### JPA에서 쿼리 최적화 하기

JPA를 제대로 이해하지 못하고 사용하면, 예상하지 못한 쿼리가 난발되어 성능이 저하될 수 있다.  
이번에 김영한님의 `JPA 프로그래밍 기초편`과 `실전 스프링 데이터 JPA` 편을 수강한 뒤, 토이 프로젝트에서 발생하고 있는 쿼리를 확인해 봤다.  
수업에서 들었던 나쁜 예시들이 내 프로젝트에도 존재하는 것을 확인했고, 이를 개선해보았다.

### 1. Lazy Loading 적용

JWT_TOKEN과 USER 테이블은 다음과 같이 1:1 관계를 맺고 있었다.  
JWT_TOKEN이 대상 테이블의 역할을 하여 USER에 대한 외래키를 관리하고 있었다.

<img src="./images/쿼리최적화1.svg" width="500">

이에 맞춰서 JwtToken 엔티티는 다음과 같이 정의했다.  
예제의 단순화를 위해 기타 필드는 제외했다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class JwtToken extends BaseEntity {
    @Id
    @Column(name = "jwt_token_id")
    @GeneratedValue
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id")
    @NotNull
    private User user;

    @Builder
    public JwtToken(User user) {
        this.user = user;
    }
}
```

이제 다음과 같이 스프링 데이터 JPA 레포지토리의 메서드를 이용하여 간단하게 전체 목록을 조회할 수 있다.  
서비스 단에서는 jwtToken에 연결된 user 정보가 필요하지 않았으므로, jwtToken의 정보만 불러오길 원했다.

```java
public interface JwtTokenRepository extends JpaRepository<JwtToken, Long> {
}
```

```java
@RequiredArgsConstructor
@Service
public class AuthService {
    private final JwtTokenRepository jwtTokenRepository;

    public List<JwtToken> list() {
        return jwtTokenRepository.findAll();
    }
}
```

이 때 JwtToken 목록을 조회하는 시점에 N+1 문제가 발생했다.

```sql
select j.* from jwt_token j
select u.* from user u where u.id=1;
select u.* from user u where u.id=2;
select u.* from user u where u.id=3;
```

서비스 로직에서는 jwtToken을 통해 user 정보를 사용하지 않았는데도, user 정보를 조회하는 쿼리가 발생했다.  
이러한 문제가 발생한 이유는 JwtToken과 User 테이블 사이에 즉시 로딩이 설정되어 있기 때문이다.  
엔티티에서 user 프로퍼티에 설정한 부분을 다시 보면, @OneToOne의 fetch 타입에 값을 입력하지 않았다.  
@ManyToOne과 @OneToOne은 fetch에 기본값으로 FetchType.EAGER가 설정되기 때문에, 값을 입력하지 않으면 즉시 로딩이 된다.

```java
@OneToOne
@JoinColumn(name = "user_id")
@NotNull
private User user;
```

이를 다음과 같이 변경하면 정상적으로 JwtToken 목록에 대한 쿼리만 발생한다.

```java
@OneToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id")
@NotNull
private User user;
```

```sql
select j.* from jwt_token j
```

### 2. 쿼리 Dsl에 fetch join 적용

Meet과 User 테이블은 N:M 관계를 가지고 있었다.  
이를 중간 테이블인 MeetUser를 통해 1:N, N:1로 풀어냈다.

<img src="./images/쿼리최적화2.svg" width="500">

원하는 동작은 Meet 목록 정보를 불러올 때 MeetUser, User 정보를 함께 Join해서 가져오는 것이었다.  
처음에 작성한 코드를 간단하게 확인하면 다음과 같았다.

```java
List<Meet> meetList = jpaQueryFactory.selectFrom(meet)
    .leftJoin(meet.meetUsers, meetUser)
    .leftJoin(meetUser.user, User);

meetList.forEach(meet -> {
    meet.getMeetUsers().forEach(meetUser -> {
        meetUser.getUser().getEmail();
    });
});
```

그런데 이 상태에서 확인해보니, 다음과 같이 총 3번의 쿼리로 분리되어 나가고 있었다.  
각 엔티티 사이는 지연 로딩으로 설정되어 있었기 때문에, 연관 객체에 접근할 때마다 쿼리가 새롭게 나가고 있었다.

```sql
-- Querydsl에서 빌드한 쿼리
select m.*
  from meet m
  left join meet_user mu on m.id=mu.meet_id
  left join user u on u.id=mu.user_id

-- meetUser.getUser() 시점에 발생
select mu.*
  from meet_user mu where mu.meet_id=?

-- meetUser.getUser().getEmail() 시점에 발생
select u.*
  from user u where u.id=?
```

쿼리를 확인해보면 querydsl에서 작성한 코드에 의해서 Meet, MeetUser, User 테이블을 left join하는 jpql이 빌드되었다.  
문제는 해당 쿼리의 select 절에 조인 대상인 MeetUser, User 테이블의 칼럼들이 포함되지 않았다는 것이다.  
이로 인해 meet.meetUsers, meetUser.user는 아직 로딩 되지 않은 상태가 되고, 지연 로딩에 따라 해당 객체의 데이터에 접근 시 조회 쿼리가 추가로 발생하게 된다.

이를 해결하기 위해서는 fetch join을 사용해야 한다.  
Querydsl에서는 편리하게 fetch join을 할 수 있는 인터페이스를 제공한다.

```java
List<Meet> meetList = jpaQueryFactory.selectFrom(meet)
    .leftJoin(meet.meetUsers, meetUser)
    .fetchJoin()
    .leftJoin(meetUser.user, User)
    .fetchJoin();
```

이제 jqpl 빌드 시 다음과 같이 select 절에 연관 객체들의 필드가 정상적으로 포함된다.  
연관 엔티티들이 모두 로딩되었기 때문에, 연관 객체의 데이터에 접근해도 쿼리가 추가로 발생하지 않는다.

```sql
select m.*, mu.*, u.*
  from meet m
  left join meet_user mu on m.id=mu.meet_id
  left join user u on u.id=mu.user_id
```

### 3. 스프링 데이터 JPA에 @EntityGraph 적용

마찬가지로 위의 테이블 구조에서, 스프링 데이터 JPA 레포지토리에도 Meet에 대한 조회 메서드를 정의하고 사용했다.

> 실제 코드에서 Querydsl의 메서드에는 where 문에 복잡한 조건들을 포함하도록 구현했다.  
> 단순 목록 조회는 스프링 데이터 JPA 메서드를 사용하고, 복잡한 조건을 넣을 떄는 Querydsl 메서드를 사용하는 식으로 활용했다.

처음에는 Quertdsl에서와 마찬가지로 fetch join이 적용되어 있지 않았다.

```java
public interface MeetRepository extends JpaRepository<Meet, Long> {
    Optional<Meet> findDistinctById(Long id, Long hostId);
}
```

해당 메서드를 사용할 때에도 Meet.meetUsers, MeetUser.user의 데이터에 접근해야 했고, 이로 인해 각각 지연로딩 되어 추가적인 쿼리가 발생했다.

```sql
select m.* from meet m

select mu.* from meet_user mu where mu.meet_id=?

select u.* from user u where u.id=?
```

스프링 데이터 JPA 레포지토리의 메서드에 페치 조인을 적용할 때에는 @EntityGraph 어노테이션을 사용한다.  
@EntityGraph에 함께 조인해 올 연관 객체의 프로퍼티명을 리스트로 적어주면, 해당 엔티티들에 대해서 페치 조인이 적용된다.

```java
public interface MeetRepository extends JpaRepository<Meet, Long> {
    @EntityGraph(attributePaths = {"meetUsers", "meetUsers.user"})
    Optional<Meet> findDistinctById(Long id, Long hostId);
}
```

```sql
select m.*, mu.*, u.*
  from meet m
  join meet_user mu on m.id=mu.meet_id
  join user u on u.id=mu.user_id
```
