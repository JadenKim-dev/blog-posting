# Promise를 사용할 때 주의할 점

javascript로 개발을 하기 위해서 Promise 사용은 필수이다.  
기본적인 개념들만 제대로 이해하면 어려움 없이 사용할 수 있지만, 개발 과정에서 한 번씩 헷갈렸던 지점들이 있어서 정리해봤다.

## reject는 콜백의 호출을 종료시키지 않는다.

javascript로 애플리케이션을 구성하다 보면 Promise를 직접 생성해서 사용할 일이 자주 있었다.  
대표적으로 setTimeout이나 setInterval 처럼 콜백을 기반으로 동작하는 함수들을 async - await으로 사용하기 위해 많이 사용했다.  
특히 기존의 콜백 기반의 레거시 메서드를 async - await 기반으로 리팩토링 하는 과정에서, 한 번에 모든 부분을 프로미스 기반으로 변경하는 것은 불가능하다 보니 콜백을 프로미스로 감싸서 await 처리하는 경우가 많았다.

Promise 내에서는 reject를 호출해서 프로미스의 상태를 Rejected로 변경할 수 있다.  
reject에는 문자열, 객체, 배열 등 어떤 값이든 넘길 수 있고, 이 값은 프로미스의 catch 콜백으로 전달된다.  
이 때 프로미스에 await을 적용하면 해당 로직을 try - catch로 묶어내서, catch에서 reject에 전달된 값을 받을 수도 있다.

```javascript
try {
  await new Promise((resolve, reject) => {
    reject("promise reject");
  });
} catch (err) {
  console.log(err); // err === "promise reject"
}
```

하지만 catch로 넘어온 값이 모든 타입이 될 수 있게 두는 것은 모호성을 너무 열어두는 행위이다.  
개발 과정에서는 모호성을 최대한 줄여야 이해하기 쉽고 예측할 수 있는 코드를 작성할 수 있다.  
이러한 이유 때문에 reject를 호출할 때에는 에러 객체를 넘기는 식으로 처리했었다.

```javascript
try {
  await new Promise((resolve, reject) => {
    reject(new Error("promise reject"));
  });
} catch (err) {
  console.error(err); // Error: promise reject
}
```

> 만약 객체 등을 이용해서 예외의 원인을 나타내고 싶다면, Error 생성자의 cause에 넘기는 것이 좋다.  
> ex) `new Error('error message', { cause: { value1: 'a' } })`

여기서 문제는, reject의 호출은 프로미스 콜백의 호출을 종료시키지는 않는다는 것이다.  
에러를 throw 하면 해당 메서드의 호출을 즉시 종료시키지만, reject는 종료시키지 않고 이후의 로직을 그대로 실행시킨다.

```javascript
try {
  await new Promise((resolve, reject) => {
    if (isError) {
      reject(new Error("promise reject"));
    }
    console.log("after reject"); // -> 앞에서 reject 해도 그대로 진행 됨
  });
} catch (err) {
  console.error(err);
}
```

```
Error: promise reject
  at /to/file/path/test.js:6:14
  ...
after reject
```

따라서 reject를 throw처럼 사용하기 위해서는, reject 이후에 즉시 return을 해주어야 한다.

```javascript
try {
  await new Promise((resolve, reject) => {
    if (isError) {
      reject(new Error("promise reject"));
      return;
    }
    console.log("after reject"); // 실행되지 않음
  });
} catch (err) {
  console.error(err);
}
```

다만 좀 더 알아보니 프로미스에서는 그 안에서 에러를 던질 경우, 알아서 적절히 프로미스의 상태를 Rejected로 처리한다고 한다.  
따라서 그냥 throw로 에러를 던지는게 가장 간결하다.

```javascript
try {
  await new Promise((resolve, reject) => {
    if (isError) {
      throw new Error("promise reject");
    }
    console.log("after reject"); // 실행되지 않음
  });
} catch (err) {
  console.error(err);
}
```

## async 메서드 내의 모든 비동기 작업에는 await을 거는게 좋다.

async - await을 사용하면 비동기 작업들을 매우 간결하게 동기적으로 처리할 수 있다.

```javascript
const waitFor = (ms) =>
  new Promise((resolve) => {
    setTimeout(() => {
      resolve();
    }, ms);
  });

const main = async () => {
  logic1();
  await waitFor(1000);
  logic2();
};

main();
```

async 메서드를 사용하면서 착각했던 점은, '해당 메서드 내에서 특정 비동기 작업 이후에 수행할 동기적 작업이 없다면 그냥 await을 생략해도 되지 않을까?' 라는 것이었다.  
특히 메서드의 마지막에 수행하는 비동기 작업의 경우 그런 착각이 더 크게 들었다.

```javascript
const main = async () => {
  logic1();
  await waitFor(1000);
  logic2();
  waitFor(1000);
};

await main();
logic3();
```

하지만 기대와 달리, 위와 같이 작성하면 마지막 비동기 작업은 기다리지 않고 바로 다음 로직(logic3)이 실행된다.  
async - await을 사용하면 내부적으로는 generator와 Promise를 사용하여, 프로미스 체이닝으로 각 비동기 작업들을 연결한다.  
만약 위에서 작성한 코드의 Promise로 변환된 결과를 극도로 단순화 하면, 다음과 같은 프로미스 체이닝이 완성될 것이다.

```javascript
logic1();
waitFor(1000).then(() => {
  logic2();
  waitFor(1000);
  logic3();
});
```

main() 메서드 내에서 수행한 마지막 waitFor(1000)에는 프로미스 체이닝이 걸리지 않고 그냥 실행되고, logic3가 동기적으로 기다리지 않고 바로 실행된다.  
따라서 async 메서드 내의 모든 비동기 작업에는 await을 거는 게 안전하다.

```javascript
const main = async () => {
  logic1();
  await waitFor(1000);
  logic2();
  await waitFor(1000);
};

await main();
logic3();
```

> 다만 만약 마지막에 수행하는 비동기 작업의 결과가 반환되어야 한다면, await을 걸지 않고 반환해도 괜찮다.  
> `const main = async () => { return waitFor(1000); };`
> 이렇게 하면 프로미스가 반환되기 때문에, 반환 받는 쪽에서 await을 걸어서 결과를 받을 수 있다.

실무에서 문제가 되었던 것은 Promise.all과 async 메서드를 함께 사용할 때였다.  
비동기적인 여러 작업들을 동시에 수행하고, 모든 작업이 끝난 후에 결과를 모아서 처리해야 하는 경우에는 Promise.all을 사용한다.  
보통 특정 배열의 값들에 대해 각각 비동기 작업을 수행하는 경우가 많았다.  
이 때에는 배열의 각 요소를 map을 통해 각각 Promise로 변환하여, Promise.all에 넘겨주는 방식으로 처리했다.

```javascript
const arr = [1, 2, 3, 4, 5];
const result = await Promise.all(arr.map((i) => Promise.resolve(i + 5)));
console.log("done", result); // done [ 6, 7, 8, 9, 10 ]
```

각 요소에 대해서 복잡한 비동기 작업을 수행해야 하는 경우에는, 다음과 같이 map에 async 메서드를 넘겨주는 식으로 처리했다.

```javascript
const task1 = async (i) => {
  await waitFor(1000);
  console.log("task1 done", i);
};
const task2 = async (i) => {
  await waitFor(1000);
  console.log("task2 done", i);
};
const arr = [1, 2, 3];
const result = await Promise.all(
  arr.map(async (i) => {
    await task1(i);
    await task2(i);
  })
);
console.log("done", result);
```

```
task1 done 1
task1 done 2
task1 done 3
task2 done 1
task2 done 2
task2 done 3
done [ 6, 7, 8 ]
```

이 때 아까 했던 착각처럼, async 메서드에서 마지막 비동기 작업(task3)의 경우 뒤에서 수행해야 할 동기적 작업이 없으므로, await을 생략해도 되겠다 싶어서 생략했었다.

```javascript
const arr = [1, 2, 3];
await Promise.all(
  arr.map(async (i) => {
    await task1(i);
    task2(i);
  })
);
console.log("complete");
```

하지만 이렇게 할 경우 task2가 실행을 마치기 전에, await Promise.all 뒤의 complete 로깅이 먼저 실행되어 버렸다.

```
task1 done 1
task1 done 2
task1 done 3
complete
task2 done 1
task2 done 2
task2 done 3
```

이 경우에도 모든 비동기 작업에 await을 거는 식으로 해결했다.  
메서드의 맨 마지막 줄에 await을 거는게 이상하게 느껴질 수도 있겠지만, 비동기 작업을 정상적으로 await 처리하기 위해서는 필요한 작업이라고 느꼈다.
