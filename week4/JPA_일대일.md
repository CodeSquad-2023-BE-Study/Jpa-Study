# 들어가기
일대일 1:1 매핑에 대해서 알아보겠습니다.

# OneToOne 일대일
먼저 @OneToOne 단방향 또는 양방향일 수 있습니다. 단방향 연결은 클라이언트 측이 관계를 소유하는 관계형 데이터베이스 외래 키 의미 체계를 따릅니다. 일대일 관계에서는 반대도 일대일 관계가 됩니다. 다대일 관계에서는 다 쪽이 항상 외래 키를 가지고 있지만, 일대일 관계에서는 주 테이블이나 대상 테이블에 외래 키를 둘 수 있어서 개발 시 어느 쪽에 둘지를 선택해야 합니다.

또한 어느 테이블에 외래키에 있는지 관계 없이 단방향 양방향 모두 가능합니다.


## 주 테이블에 외래 키가 있는 경우
주 객체가 대상 객체를 참조하는 것처럼  주 테이블에 외래키를 두고 사용하는 방식입니다. 객체지향 개발자들이 선호하는 방식. (강의에서는 김영한님은 이방법을 추천)

- 장점
아무래도 밑에있는 member 주테이블에서 외래키를 가지고 있기 때문에 이 테이블만 확인해도 연관관계를 확인 하기쉽습니다.

- 단점
값이없으면 외래키에 null허용 해야한다.


![](https://velog.velcdn.com/images/leekhy02/post/287c8064-83ec-464f-b775-bd2fcac2b492/image.png)

이그림을 보면 주 테이블만 봐도 일단 외래키가 있다는 것은 대상 테이블의 데이터 존재 유무 확인이 가능합니다. 그림과 같이 password가 아직 할당 되지 않은 member는 null을 허용합니다.


- 테이블 칼럼의 값이 null인 것이 왜 단점인 이유

대상 테이블에 외래키 경우 null 허용 없이 처리를 완료할 수 있습니다. (Password가 없는 경우 대상 테이블 자체에 데이터가 들어가지 않으니까요)

테이블에 컬럼이 null을 허용하면 혹시 실수로 FK에 값을 넣지 않는 경우를 방어할 수 없는 단점이 있습니다.



### 단방향
``` java
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

    @OneToOne
    @JoinColumn(name = "member_profile_id")
    private MemberProfile memberProfile;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_password_id")
    private MemberPassword memberPassword;
   ```
   
OneToOne 매핑에도 Lazy Loading 전략이 사용 가능합니다.

하지만 주의 할 것이 있는데 밑에서 설명하겠습니다.
### 양방향

``` java
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

    @OneToOne
    @JoinColumn(name = "member_profile_id")
    private MemberProfile memberProfile;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_password_id")
    private MemberPassword memberPassword;
   ```
   
   
 ```	java
 public class MemberProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_profile_id")
    private Long id;

    @OneToOne(mappedBy = "memberProfile")
    private Member member;
}
 ```
 
당영한 것이겠지만 객체 연관관계의 변화가 있을 뿐 실제 테이블에는 변화가 없습니다.  또한 OneToOne 양방향 매핑에서 주의 할 점이 있는데 이또한 밑에서 알아봅니다.
## 대상 테이블에 외래 키가 있는 경우
전통적인 DBA 개발자들이 선호 하는 방식입니다. 대상 테이블에 외래키를 주도 사용합니다.

### 단방향
일대일 관계 중 대상 테이블에 외래키가 있는 단방향 관계는 JPA에서 지원하지 않습니다.

### 양방향
``` java
  @Entity
  public class Member {
  	@Id @GeneratedValue
  	@Column(name = "MEMBER_ID")
  	private Long id;
                
  	@OneToOne(mappedBy = "member")
  	private Locker locker;
        
  }
```

``` java
  @Entity
  public class Locker {
  	@Id @GeneratedValue
  	@Column(name = "LOCKER_ID")
  	private Long id;
                
  	@OneToOne
  	@JoinColumn(name = "MEMBER_ID")
  	private Member member;
  }
```
# OneToOne Fetch 전략

앞에서 @OneToOne 매핑에 대해 알아봤습니다.  기본값은 즉시 로딩이고 즉시 로딩의 N+1 등의 문제들을 페치조인으로 회피하기 위해 지연로딩을 권장하고있습니다. 하지만 위에서 말했듯 OneToOne 은 양방향 매핑시 문제가 있고 이것은 지연로딩이 적용되지 않는 문제가 존재입니다.

다시말해  @OneToOne 양방향 연관 관계에서 연관 관계의 주인이 아닌 쪽 엔티티를 조회할 때, Lazy로 동작할 수 없습니다.

지연(LAZY)로딩을 사용하기 위해서는 JPA 구현체에서 "프록시를 만들어 주어야 한합니다."


### 프록시란?

![](https://velog.velcdn.com/images/leekhy02/post/48366ad6-181f-44d0-adc6-519a82604b34/image.png)


1) Client가 getName()을 호출한다.
2) 호출받은 프록시가영속성 콘텍스트에 초기화를 요청한다.
3) 영속성 콘텍스트에 이미 값이 있다면 바로 4번으로 갈 테지만 그렇지 않다면 DB에 조회한다.
4) 조회된 데이터로 실제 엔티티를 생성한다.
5) 생성된 실제 엔티티를 프록시가 조회하여 반환해준다.


프록시를 통해 실제 엔티티에 접근합니다.

- 지연 로딩이란 객체 A를 조회할 때는 A만 가져오고 연관된 객체는 프록시 초기화 방법으로 가져옵니다.

결론적으로 지연(LAZY)로딩을 사용하기 위해서는 JPA 구현체에서 "프록시를 만들어 주어야 합니다." 여기 전제 조건이 하나 더있는데 JPA 구현체는 연관 관계 엔티티에 null 또는 프로시 객체가 할당되어야만 합니다.

1. @OneToOne 에서 null 은 연관 관계 해당 엔티티가 없는 것을 의미하고, 프록시 객체는 연관관계 해당 엔티티가 존재한다 볼 수 있습니다.

2. @OneToOne 매핑에서 나오는 null이 허용 되는 관계에서 OneToOne Lazy Loading이 잘 작동하지 않습니다.

- 만약 null 값이 가능한 OneToOne 에 프록시 객체를 넣는다면, 이미 그 순간 결코 null 이 아닌 프록시 객체를 리턴하는 상태가 돼 버리기 때문이다.

- 따라서 JPA 구현체는 기본적으로 OneToOne 관계에 Lazy 를 허용하지 않고, 즉시 로딩을 사용한다.

3.  연관관계의 주인이 아닌 테이블에서는 컬럼 자체가 존재하지 않습니다.즉 해당 테이블만 읽어서는 연관관계 엔티티 존재 여부를 알 수 없습니다.

4. 연관관계의 주인이 아닌 경우 값이 있는지 없는 지 알 수 없기 때문에 null 을 넣기도 애매하고 null 이 아니라는전제가 없기 때문에 프록시를 만들기도 애매합니다.
또한 2번에서 언급했듯이 null 을 프록시로 감싸버리면 null 이 아닌 프록시 객체상태로 리턴하게 됨으로 null 값 처리가 불가능해진다.

하지만 DB 관점에서 연관관계 주인인 엔티티에 매핑된 테이블에는 연결된 엔티티의 존재를 알 수 있는 컬럼이 존재합니다.

null 이 있으면 연관 엔티티가 없는것 이고, null 이 아니면 프록시 객체 설정이 가능하여 LAZY 처리가 가능합니다.

### 결론
@OneToOne 양방향 연관 관계에서 연관 관계의 주인이 아닌 쪽 엔티티를 조회할 때, Lazy로 동작할 수 없다.
