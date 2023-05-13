## 0. 테스트 커버리지를 높이게 된 이유

기존에 해왔던 사이드 프로젝트에서는 Service 로직과 일부 커스텀 클래스에 대한 테스트 코드 정도만 작성해 둔 상태였다.
'어차피 Controller는 Service의 메서드를 호출하는 정도의 역할만 하는데 굳이 테스트가 필요할까?' 등의 이유로 나머지 부분들에 대한 테스트는 작성하지 않고 있었다.

하지만 최근에 토스의 이응준 개발자님께서 올리신 [테스트 커버리지 100%](https://www.youtube.com/watch?v=jdlBu2vFv58) 영상을 보면서 테스트 코드 작성에 대한 생각을 바로잡게 되었다.
특히 모든 파일, 메서드, 로직에 대해서 테스트를 작성하시면서 '테스트 커버리지는 얼마든지 높일 수 있다'는 생각을 굳히게 되신 점이 인상깊었다.

그래서 이번 사이드 프로젝트에서는 테스트 커버리지 100%를 목표로 두고 프로젝트에 테스트 코드를 추가해 나가기로 결심했다.

## 1. 문제 상황

테스트 커버리지를 높이려면 모든 파일의 모든 선언/브랜치/함수/라인을 커버할 수 있어야 한다.
처음에는 이 부분이 꽤나 까다롭게 느껴졌지만, 그 번거로움을 넘어설 만큼의 이점이 있다는 것을 느꼈다.
특히 테스트 커버리지의 부족한 부분을 메꾸는 과정에서 본 로직의 잘못되거나 부족한 부분을 찾게 되는 경우도 있었다.

아래의 예시에서 validateKeyAndLock은 lockName, key를 받아서 유효한지 검증하는 메서드이다.
메서드에서는 lock의 verifyKey 메서드를 이용해서 key에 대한 검증을 수행한다.

```ts
async validateKeyAndLock(
  lockName: string,
  key: string,
): Promise<void> {
  const lock = await this.lockRepository.findOneByName(lockName);
  if (!lock?.verifyKey(key)) {
    throw new InvalidKeyException();
  }
  if (lock?.isExpired()) {
    throw new ExpiredLockException();
  }
}
```

얼핏 보면 해당 메서드에 대한 테스트 케이스는 3개면 충분하다고 생각할 수 있다.

- 적절한 lockName & key가 주어졌을 때 에러가 발생하지 않음
- lock과 매칭되지 않은 key를 제공하면 InvalidKeyException이 발생
- lock이 만료되었다면 ExpiredLockException이 발생

```ts
it("매칭되는 lockName과 key가 주어졌을 때 에러가 발생하지 않음", async () => {
  // given
  const lock = Lock.create("테스트 자물쇠");
  await lockRepository.save(lock);

  const lockName = "테스트 자물쇠";
  const key = lock.getKey();

  // when
  // then
  await expect(
    lockService.validateKeyAndLock(lockName, key)
  ).resolves.not.toThrow();
});

it("lock과 매칭되지 않은 key를 제공하면 InvalidKeyException이 발생", async () => {
  // given
  const lock = Lock.create("테스트 자물쇠");
  await lockRepository.save(lock);

  const lockName = "테스트 자물쇠";
  const key = "invalid_key";

  // when
  // then
  await expect(keyService.validateKeyAndLock(lockName, key)).rejects.toThrow(
    InvalidKeyException
  );
});

it("lock이 만료되었다면 ExpiredLockException이 발생", async () => {
  // given
  const lock = Lock.create(
    "테스트 자물쇠",
    new Date(new Date().getTime() - 10000)
  );
  await lockRepository.save(lock);

  const lockName = "테스트 자물쇠";
  const key = lock.getKey();

  // when
  // then
  await expect(keyService.validateKeyAndLock(lockName, key)).rejects.toThrow(
    ExpiredLockException
  );
});
```

하지만 위와 같이 작성하면 테스트 커버리지가 100%를 달성하지 못한다.

## 2. 커버리지가 100%가 되지 않는 이유

위의 테스트에서 커버리지가 부족한 부분은 branch 부분이다.
잘 살펴보면 key를 검증하거나 lock의 만료 상태를 검증하는 부분에서 optional chaining을 사용하고 있다.

```ts
const lock = await this.lockRepository.findOneByName(lockName);
if (!lock?.verifyKey(key)) {
  throw new InvalidKeyException();
}
```

따라서 해당 제어문에 대한 모든 케이스를 검증하기 위해서는 두가지 케이스를 모두 검사해야 한다.

- 존재하지 않는 lock name인 경우
- key가 lock에 매칭되지 않는 경우
- lock이 만료된 경우

기존에 작성한 테스트 코드의 경우 `lock이 null인 경우`에 대한 테스트를 누락했다.

## 3.1. 개선 1 - 테스트 추가해보기

`존재하지 않는 lock name인 경우`에 대한 테스트를 추가해야 한다.
하지만 이를 추가하다보면 이상한 점을 발견하게 된다.

```ts
it("존재하지 않는 lock name이면 InvalidKeyException이 발생", async () => {
  // given
  const lockName = "invalid_lock_name";
  const key = "key";

  // when
  // then
  await expect(keyService.validateKeyAndLock(lockName, key)).rejects.toThrow(
    InvalidKeyException
  );
});
```

InvalidKeyException은 lock에 매칭되지 않는 key를 제공했을 때 발생하는 게 적절하다.
위와 같이 존재하지 않는 lock name을 제공했을 때 InvalidKeyException을 발생시키면 명확성이 떨어진다.

## 3.2. 개선 2 - 본 로직 수정하기 + 테스트 수정하기

존재하지 않는 lock 이름을 사용했을 때 발생시킬 별도의 에러를 정의할 필요가 있다.
이렇게 정의한 에러를 lock 조회에 실패했을 때 발생하도록 구성하면 된다.
이렇게 별도로 if문을 나눠서 구성하게 되면 optional chaining을 제거할 수 있다.

```ts
async validateKeyAndLock(
  lockName: string,
  key: string,
): Promise<void> {
  const lock = await this.lockRepository.findOneByName(lockName);
  if (!lock) {
  	throw new NotExistsLockNameException();
  }
  if (!lock.verifyKey(key)) {
    throw new InvalidKeyException();
  }
  if (lock.isExpired()) {
    throw new ExpiredLockException();
  }
}
```

수정한 내용에 맞춰서 방금 작성한 테스트 코드도 함께 수정해보자.

```ts
it('존재하지 않는 lock name이면 NotExistsLockException이 발생', async () => {
  // given
  const lockName = 'invalid_lock_name';
  const key = 'key';

  // when
  // then
  await expect(keyService.validateKeyAndLock(lockName, key))
  	.rejects.toThrow(NotExistsLockException);
```

이렇게 작성하면 테스트 커버리지 100%를 달성할 수 있다.

## 4. 마치며

위 영상에서 이응준 개발자님께서는 테스트 작성에 필요한 것은 `테스트가 필요하다는 믿음`과 `테스트를 작성할 시간` 이라고 말씀하셨다.
테스트 코드를 작성할수록 테스트가 필요하다는 믿음을 더 굳게 가지게 되는 것 같다.

테스트 코드는 본 로직의 안정성을 검증하는 것을 넘어서, 그것을 작성하는 과정에서 본 로직을 더 깊이 이해하고 개선할 기회를 제공해준다.
앞으로도 테스트 코드를 보다 적극적으로 작성하면서 테스트의 이점을 더 많이 느껴보고 싶다.
