저번에 input-output 기반의 테스트가 이상적이라는 글을 썼다.
이 글에서는 javascript로 작성된 Node.js 프로젝트에서 input-output 기반의 테스트 코드를 작성하는 방법을 다루려고 한다.

typescript로 프로젝트가 구성되어 있다면 더 나은 해결책이 많지만, 레거시 코드를 다루다보면 javascript로 되어 있는 프로젝트에서 작업을 해야 하는 경우가 많다.
나와 같이 레거시 환경에서 단위 테스트를 시도해보려고 하는 사람들에게 이 글을 바친다...😇

## 1. 의존성을 가진 모듈 확인하기

먼저 테스트 대상이 되는 코드를 분석해서, 해당 메서드가 어떤 모듈에 의존성을 가지는지 파악해야 한다.
저번 글에서 다뤘던 getPosts 메서드를 다시 살펴보자.

<img src="https://velog.velcdn.com/images/jadenkim5179/post/932389a2-0d62-4203-8fb8-ba6f2cd883aa/image.png" width="60%" height="n%">

getPosts는 크게 3개의 작업을 한다.

1. `velogUidRepository.getByUserId` 메서드를 통해 사용자에 해당하는 velogUid 데이터를 가져온다.

2. `velogPostAxios.reqGetPosts` 메서드를 통해 외부 API와 통신하여 posts 정보를 가져온다.

3. 받아온 posts 정보를 반환한다.

여기서 1, 2 단계는 외부(db, velog api)와 통신하는 부분이며, 실제로 수행하기 위해서는 시간이 드는 작업들이다.

## 2. 외부 의존 모듈을 mock으로 감싸기

테스트 상황에서 매번 외부 API를 직접 호출하거나, db와 통신을 해야 한다면 테스트 시간이 매우 길어질 수밖에 없다.
단위 테스트 상황에서는 해당 메서드들을 stubbing해서 구현한 후, 이를 테스트 대상 메서드가 사용하도록 하는 것이 바람직하다.

다만 문제는, 일반적인 레거시 Node.js 프로젝트는 의존성 주입을 사용하고 있지 않다는 것이다.
따라서 테스트 대상 메서드가 stubbing한 구현체를 사용하게 하기 위해서는 **모듈 자체를 모킹해야 한다.**
모듈 자체를 모킹하면, 해당 모듈을 참조하는 모든 곳에서 모킹된 내용을 바탕으로 동작하게 할 수 있다.

jest.mock()을 사용하면 테스트 대상 모듈을 손쉽게 모킹할 수 있다.

```jsx
const velogUidRepository = require("#velog-post/repositories/velog-uid.repository");
const velogPostAxios = require("#velog-post/velog-post.axios");

jest.mock("#velog-post/repositories/velog-uid.repository");
jest.mock("#velog-post/velog-post.axios");
```

이렇게 하면 해당 모듈을 참조하는 모든 곳에서 모킹된 모듈을 사용하게 되어서, stub으로 구현한 메서드를 사용하도록 할 수 있다.

## 3. 의존하고 있는 메서드에 대한 stub 구현하기

이제 필요한 메서드들에 대한 stub을 구현해야 한다.
테스트에 사용할 임시 데이터들을 상수로 정의해두고, 이를 바탕으로 최소한의 로직을 담고 있는 stub 메서드를 구현할 수 있다.

먼저 `velogUidRepository.getByUserId`에 대한 stub을 구현해보자.

```js
const EXISTING_USER_ID = 1;
const VELOG_UID = 123;

const VELOG_UID_DATA_LIST = [{ userId: EXISTING_USER_ID, velogUid: VELOG_UID }];

const mockGetVelogUidByUserId = ({ userId }) =>
  Promise.resolve(VELOG_UID_DATA_LIST.find((data) => data.userId === userId));
```

임의의 velog_uid 정보의 리스트를 담고 있는 `VELOG_UID_DATA_LIST`를 정의했다.
`mockGetVelogUidByUserId`는 매개변수로 userId를 받아서, 해당 userId가 담겨있는 데이터를 `VELOG_UID_DATA_LIST`에서 찾아서 반환한다.

데이터베이스 이용 여부를 제외하면, `mockGetVelogUidByUserId`는 `velogUidRepository.getByUserId`와 하는 역할이 비슷하다.
이와 같이 기능을 단순화해서 stub 메서드를 구현하면 된다.

이번엔 `velogPostAxios.reqGetPosts`에 대한 stub을 구현해보자.

```js
const VELOG_UID = 123;
const VELOG_TOKEN = "valid_token";

const VELOG_POST_LIST = [
  {
    name: "post1",
    content: "hello world",
  },
];

const isValidReq = ({ velogUid, velogToken }) =>
  velogUid === VELOG_UID && velogToken === VELOG_TOKEN;

const mockReqGetVelogPost = ({ velogUid, velogToken }) =>
  isValidReq({ velogUid, velogToken })
    ? Promise.resolve(VELOG_POST_LIST)
    : Promise.reject(VelogAxiosError);
```

`mockReqGetVelogPost`는 매개변수로 velogUid, velogToken를 받아서, 이것이 미리 정의해둔 데이터와 일치하는지를 확인한다.
일치할 경우 유효한 요청이라고 판단해서 `VELOG_POST_LIST`를 resolve 하고, 일치하지 않다면 VelogAxiosError를 reject 한다.

## 4. stub한 메서드를 모킹한 모듈에 적용시키기

이제 위와 같이 구현해 둔 stub 메서드를 모킹한 모듈에 적용시켜야 한다.
`jest.mock()`으로 모듈 자체를 모킹해두었기 때문에, 다음과 같이 간단하게 stub한 메서드를 적용시킬 수 있다.

```js
velogUidRepository.getByUserId.mockImplementation(mockGetVelogUidByUserId);
velogPostAxios.reqGetPosts.mockImplementation(mockReqGetVelogPost);
```

3,4 단계에서 작성한 내용을 종합하면 아래와 같다.

```js
describe("velogPostService", () => {
  const EXISTING_USER_ID = 1;
  const VELOG_UID = 123;
  const VELOG_TOKEN = "valid_token";

  const isValidReq = ({ velogUid, velogToken }) =>
    velogUid === VELOG_UID && velogToken === VELOG_TOKEN;

  const VELOG_UID_DATA_LIST = Object.freeze([
    { userId: EXISTING_USER_ID, velogUid: VELOG_UID },
  ]);
  const POST_LIST = Object.freeze([{ name: "post1", content: "hello world" }]);

  describe("getPosts", () => {
    beforeEach(() => {
      const mockGetVelogUidByUserId = ({ userId }) =>
        Promise.resolve(
          VELOG_UID_DATA_LIST.find((data) => data.userId === userId)
        );
      velogUidRepository.getByUserId.mockImplementation(
        mockGetVelogUidByUserId
      );

      const mockReqGetVelogPost = ({ velogUid, velogToken }) =>
        isValidReq({ velogUid, velogToken })
          ? Promise.resolve(POST_LIST)
          : Promise.reject(VelogAxiosError);
      velogPostAxios.reqGetPosts.mockImplementation(mockReqGetVelogPost);
    });
  });
});
```

데이터를 정의하는 상수 부분은 describe 최상단에 두거나 beforeAll에 둔다.
그리고 beforeEach 안에 메서드 모킹을 정의하고, 적용하는 부분을 넣는다.

이렇게 하는 이유는 필요에 따라서 특정 테스트 케이스에서는 beforeEach에서 구현한 것과 다른 stub 메서드를 해당 mock에 적용시킬 수도 있기 때문이다.
이런 특이 케이스에 대한 테스트가 끝난 후에는 기본 stub 메서드로 매번 변경해주기 위해서 beforeEach에 mock 적용 부분을 넣었다.

## 4. 실제 테스트 케이스 작성

이제 각 테스트 케이스에 대한 테스트 코드를 작성하면 된다.

stub으로 구현한 메서드를 적용시키는 방법을 사용했기 때문에
행위 기반이 아닌 상태 기반의 직관적인 테스트 코드를 작성할 수 있다.

```jsx
describe("postService", () => {
  // 필요한 상수 정의
  describe("getPosts", () => {
    beforeEach(() => {
      // 필요한 의존성 mocking 적용
    });

    it("userId, velogToken이 유효할 경우 post 목록을 반환", async () => {
      // given
      const userId = EXISTING_USER_ID;
      const velogToken = VELOG_TOKEN;

      // when
      const posts = await postService.getPosts({
        userId,
        velogToken,
      });

      // then
      expect(posts).toEqual(POST_LIST);
    });
  });
});
```

에러가 발생하는 경우는 아래와 같이 테스트할 수 있다.

```jsx
describe("postService", () => {
  describe("getPosts", () => {
    beforeEach(() => {
      // 필요한 의존성 mocking 적용
    });

    it("userId, velogToken이 유효할 경우 post 목록을 반환", async () => {
      // 정상 호출 상황 테스트
    });

    it("유효하지 않은 userId로 호출하면 UserNotFoundError 발생", async () => {
      // given
      const userId = 999; // 임의의 user id
      const velogToken = VELOG_TOKEN;

      // when
      // then
      await expect(() =>
        postService.getPosts({
          userId,
          velogToken,
        })
      ).rejects.toEqual(new UserNotFoundError());
    });
  });
});
```
