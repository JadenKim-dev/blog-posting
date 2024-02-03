# 중간 테이블로 이중 관계 풀어내기

엔티티를 설계하다 보면 엔티티 간 N:M 관계를 사용해야 하는 경우가 많다.  
테이블 상에서 N:M 관계는 1:N 관계 2개로 풀어내게 되는데, ORM에서는 이를 위해 중간 테이블을 자동으로 구성해서 별도로 엔티티를 정의하지 않아도 되게끔 지원한다.

```java
@Entity
public class User {
    …
    @ManyToMany
    @JoinTable(name = "USER_ORDER")  // 중간 테이블명
    private List<Order> orders = new ArrayList<>();
}
```

하지만 중간 테이블에 추가 정보를 삽입할 수 없다는 한계로 인해 보통 이 기능은 사용하지 않는 것을 권장하고 있었다.  
실제로 토이 프로젝트의 테이블 설계를 개선하는 과정에서 중간 테이블에 대한 엔티티를 별도로 정의하는 것의 장점을 크게 느꼈었다.

## 기존의 테이블 구조

기존에 설계했던 테이블 구조는 다음과 같았다.

<img src="./images/중간테이블1.svg" width="400">

먼저 한 명의 사용자(User)가 여러 미팅(Meet)의 호스트가 될 수 있으므로, 이를 1:N 관계로 설정했다.  
또한 여러 명의 사용자(User)가 여러 개의 미팅(Meet)에 참여할 수 있는 N:M 관계였기 때문에, 이를 중간 테이블인 MEET_MEMBER를 통해 두 개의 1:N 관계로 풀어냈다.

> 중간 테이블의 이름은 보통 두 테이블의 이름을 조합해서 만드는 경우가 많다.  
> 다만 위 구조에서는 Meet과 User 사이에 이중 관계가 있기 때문에, 중간 테이블을 MEET_USER와 같이 정하면 단일한 관계로 이해될 것 같아서 MEET_MEMBER로 정했다.

이에 맞춰서 엔티티는 다음과 같이 정의할 수 있다.  
우선 Meet과 User 사이의 1:N 관계에 대해서, Meet에 host라는 이름으로 프로퍼티를 넣고 @ManyToOne을 사용하여 관계를 설정했다.  
또한 Meet과 User 사이의 N:M 관계는 Meet 엔티티에 members 프로퍼티를 넣고 @ManyToMany를 사용하여 관계를 설정했다.  
이를 통해 중간 테이블인 MEET_MEMBER를 자동으로 생성하게 된다.

```java
@Entity
public class Meet {
    @Id @GeneratedValue
    @Column(name = "meet_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "host_id")
    private User host;

    @ManyToMany
    @JoinTable(name = "MEET_MEMBER")
    private List<User> members = new ArrayList<>();
}
```

## 문제점

하지만 위 구조를 사용하면서 여러 문제들이 있음을 알게 되었다.

### 데이터 조회 시 join이 추가됨

Member와 User를 이중 관계로 구성했으므로, Meet 데이터 조회 시 연관된 데이터를 한 번에 불러오기 위해서는, 단일한 관계로 구성했을 때에 비해 join이 한 번 더 이루어져야 한다.  
우선 Member와 User의 N:M 관계를 불러오기 위해서는 중간테이블을 거쳐서 조인해야 하기 떄문에, 조인이 총 2번 이루어져야 한다.

```sql
select m.*, mm.*, u.*
from meet m
         join meet_member mm on m.id = mm.meet_id
         join user u on u.id = mm.user_id
```

여기에 추가로 meet.host_id를 기반으로 한 번 더 조인이 이루어진다.

```sql
select m.*, mm.*, u.*, h.*
from meet m
         join meet_member mm on m.id = mm.meet_id
         join user u on u.id = mm.user_id
         join user h on h.id = m.host_id
```

이로 인해 db 단에서 join을 수행하는 횟수가 1회 증가하고, 조회되는 각각의 row 에는 host_id에 매핑되는 User 정보가 중복되어 포함된다.

<img src="./images/중간테이블2.svg" width="300">

### 관계의 모호성

테이블 사이에 이중 관계가 존재하면, 두 테이블 사이의 관계가 모호해지는 문제가 있다.  
이로 인해 테이블 설계에서 외래키명과 테이블명을 순수하게 결정하는 것이 어려워지고, 비즈니스적인 이름들이 삽입될 여지가 생긴다.

먼저 N:M 관계의 중간 테이블의 경우, 단순히 두 테이블의 조합인 MEET_USER로 이름을 정할 경우에 단일한 관계로 오해하지 않을지 우려되었다.  
이로 인해 비즈니스적인 의미를 바탕으로 이름을 정하게 되었고, 미팅의 멤버라는 의미를 가진 MEET_MEMBER 라는 이름을 사용하게 되었다.

> 다만 이도 좋은 결정은 아닌 것이, MEET_MEMBER는 MEET과 MEMBER의 중간 테이블인 느낌이 든다.  
> 하지만 그렇다고 해서 PARTICIPANT와 같이 아예 다른 이름으로 지으면, 관계가 있는 테이블의 이름들과 너무 동떨어지게 된다.

또한 1:N 관계에 대한 외래키의 경우에도 user_id와 같이 테이블명을 바탕으로 외래키 이름을 정할 수 없고, host_id와 같이 비즈니스적인 의미를 가진 이름을 사용해야 한다.

<img src="./images/중간테이블3.svg" width="400">

물론 위와 같이 이름을 정할 때 비즈니스적인 의미가 개입되는 것이 불가피한 상황도 있다.  
하지만 가능하다면 외부적인 의미의 개입 없이, 두 테이블 간의 관계 설정을 하는 것만으로도 명확하게 그 관계가 드러나는 것이 더 좋은 설계라고 생각한다.  
테이블을 관리하는 측면에서도 중간 테이블명이나 외래키 이름이 순수하게 구성되는 편이 관리하기 용이하다.

## 개선 방법

위 구조를 개선하기 위해서 Meet과 User 사이에 단일한 N:M 관계만 설정하고, 해당 사용자가 host인지 여부는 중간 테이블의 칼럼(meet_user_role)으로 구분하도록 변경했다.  
Meet 테이블에 있었던 host_id는 제거했다.

<img src="./images/중간테이블4.svg" width="700">

우선 각 미팅 참여자의 권한을 구분하기 위해, MeetUserRole이라는 enum으로 각 역할을 정의한다.

```java
public enum MeetUserRole {
    HOST, MEMBER
}
```

이제 MeetUserRole 타입의 값을 중간 테이블이 저장하도록 구성해야 한다.  
결국 중간 테이블에 외래키 뿐만 아닌 추가적인 데이터의 삽입이 필요하게 된 것이고, 따라서 중간 테이블을 엔티티로 별도로 정의해야 한다.  
최종적으로 다음과 같이 엔티티를 정의했다.  
이제 이중 관계에 대한 문제가 사라졌으므로, 중간 테이블의 명을 MEET_USER로 순수하게 정할 수 있었다.

```java
@Entity
public class MeetUser {
    @Id @GeneratedValue
    @Column(name = "meet_user_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "meet_id")
    private Meet meet;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @Enumerated(EnumType.STRING)
    private MeetUserRole meetUserRole;
}
```

### 마치며

비즈니스 로직을 계속 전개하다보면 중간 테이블에 데이터를 삽입할 일이 점점 많아짐을 느꼈다.  
보통 N:M 관계는 핵심적인 비즈니스 로직에서 많이 발생하기 때문에 추가적인 데이터가 필요한 상황이 대부분일 것이다.  
정말 단순한 구조가 아니라면, 중간 테이블을 엔티티로 정의하는 것이 더 좋은 선택인 것 같다.
