# 들어가기
지난 학습했던 OneToOne 매핑에 대해 공부하다보니 양방향 관계에서 Lazy로딩이 먹히지 않는 문제에 대해서 공부했었습니다. 그로인해 Proxy에 대해 더 궁금증이 생겼고 이를 학습해보는 시간을 가졌습니다.

일단 Jpa에서 말하는 프록시란, JPA구현체인 하이버네이트(Hibernate)가 프록시 객체를 통해 지연 로딩을 구현하고 있습니다. JPA 프록시는 지연 로딩이 가능하게 해주는 고마운 기술이지만 잘못 사용한다면 여러 장애나 버그를 겪을수 있고 프록시에 대해 자세히 알아보는 시간을 가져보겠습니다.

# 소개

JPA는 왜 프록시는 왜 사용하며, 어떤 기술일까요?

간단히 프록시에 대해 설명하자면
- JPA구현체들은 프록시라는 기술을 사용하여, 연관된 객	체를 처음부터 조회 하는것이 아니라, 실제 사용하는 시점에 데이터베이스를 조회 할수 있도록 해줍니다.

- Hibernate 프록시는 실제 엔터티 POJO(Plain Old Java Object)를 대체하는 데 사용됩니다. 

>
Jpa표준 명세서는 지연 로딩의 방법을 JPA 구현체 즉 하이버네이트에게 위임 했습니다. 하이버네이트는 지연 로딩을 지원하기 위해 프록시를 사용하는 방법과 바이트 코드를 수정하는 두가지 방법을 제공합니다.

지연 로딩을 하려면 연관된 엔티티의 실제 데이터가 필요할 때 까지 조회를 미뤄야 합니다. 하이버네이트는 지연 로딩을 사용하는 연관관계 자리에 프록시 객체를 주입하여 실제 객체가 들어있는 것처럼 동작하도록 합니다.


즉 간단히 말해 JPA 구현체인 하이버네이트가 구현한 방법을 통해 JPA는 지연로딩을 할 수있게 됩니다.

# 프록시 초기화,로드
일단은 밑의 두 메서드의 차이점을 알아야합니다.
JPA는 EntityManager주어진 엔터티를 로드하는 두 가지 방법을 정의합니다.
## find()
엔터티는 첫 번째  캐시, 두 번째  캐시 또는 데이터베이스에서 불러옵니다. 이렇게 조회한 엔티티는 실제 사용하던 사용하지 않든 데이터베이스를 거칠 경우 SELECT 문이 나가게 됩니다. 
## getReference()
이 메서드를 호출 시 JPA는 **데이터베이스를 조회 하지않고 **실제 엔티티 객체도 생성하지 않습니다. 대신에 데이터베이스를 접근을 위임한 프록시 객체를 반환합니다.

``` java
Post post = entityManager.getReference(Post.class, 1L);
 
PostComment comment = new PostComment();
comment.setId(1L);
comment.setPost(post);
comment.setReview("test");
entityManager.persist(comment);

//결과
INSERT INTO post_comment (post_id, review, id)
VALUES (1, 'test', 1)
```
위와 같이Hibernate는 어떤 SELECT 문도 실행할 필요 없이 단일 INSERT 문을 실행합니다. 만약  find()를 사용했다면 SElECT문이 나갔을 겁니다.


그 다음으로
``` java
@Entity
public class PostComment {
 
    @Id
    private Long id;
 
    @ManyToOne(fetch = FetchType.LAZY)
    private Post post;
 
    private String review;
}

```

``` java
PostComment comment = entityManager.find(
    PostComment.class,1L);
 
System.out.println("========");

assertEquals(
    "test",
    comment.getPost().getTitle() // 이 시점에 프록시 초기화
);


```

``` java

SELECT pc.id AS id1_1_0_,
       pc.post_id AS post_id3_1_0_,
       pc.review AS review2_1_0_
FROM   post_comment pc
WHERE  pc.id = 1
 
========
 
SELECT p.id AS id1_0_0_,
       p.title AS title2_0_0_
FROM   post p
WHERE  p.id = 1
```

첫 번째 SELECT 문은 PostComment의 Post 가 FetchType.LAZY 이므로 프록시를 초기화하지 않고 PostComment 엔티티를 가져옵니다. 선택한 FOREIGN KEY 열을 검사하여 Post 연결을 null로 설정할지 프록시로 설정할지 여부를 파악합니다. FOREIGN KEY 열 값이 null이 아닌 경우 프록시는 연결 식별자만 채웁니다.
 
하지만 Post를 조회할 때 Post 프록시를 초기화하기 위해 SELECT 문을 실행 합니다.

최초 지연 로딩을 설정한 시점에 하는 것이아닌. 실제 객체의 메서드를 호출할 필요가 있을 때 데이터베이스를 조회해서 참조 값을 채우게 되는데요, 이를 프록시 객체를 초기화한다고 합니다.

### 순서 정리

![](https://velog.velcdn.com/images/leekhy02/post/22979837-2317-44e5-833a-2ed1da453c4b/image.png)

``` java
Post post = em.getReference(Post.class, 1L);
post.getTitle();
```
위 문장을 실행시

1.em.getReference()로 프록시 객체를 가져온 다음, getTitle() 메서드를 호출

2.Proxy 객체에 처음에 target 값이 존재하지 않는다(연결 식별자 값만 존재). JPA가 영속성 컨텍스트에 초기화 요청을 한다.

3.영속성 컨텍스트가 DB에서 조회, 실제 Entity를 생성

4. 프록시 객체가 가지고 있는 target(실제 Post)의 getTitle()을 호출해서 결국 post.getTitle()을 호출한 결과를 받을 수 있다.

5. 프록시 객체에 target이 할당 되고 나면, 더이상 프록시 객체의 초기화 동작은 없어도 된다.(프록시의 특징중 하나 입니다 프록시 객체는 처음 사용할 때 한번만 초기화합니다.)

## LazyInitializationException

위에서 알 수 있듯이 결국 프록시는 영속성 컨텍스트의 도움을 받아 실제 객체를 참조하게 됩니다. 때문에
1. 영속성 컨텍스트의 관리를 받지 못하는 상태(준영속)
2. OSIV옵션이 꺼져있는 상황
3. 트랜잭션 바깥에서 프록시를 초기화 할 경우

위의 경우에는 LazyInitializationException가 발생합니다.때문에 프록시를 초기화 할 시에는 반**드시 영속 상태**여야합니다.


## 프록시와 식별자
**하이버네이트에서 식별자를 꺼내는 getId를 호출할 때는 초기화되지 않습니다.**

이는 하이버네이트 설정 중 hibernate.jpa.compliance.proxy 때문입니다.

하이버네이트는 JPA 명세와는 다르게, 식별자를 호출할 때는 엔티티를 초기화하지 않습니다.

![](https://velog.velcdn.com/images/leekhy02/post/136da42b-d244-48ea-9982-0ac192536f7e/image.png)

다만 여기서 주의하실 점은, id는 null이라는 점입니다. 프록시 객체는 기본적으로 모든 필드 값을 null로 가지고 있습니다. 하지만 프록시 객체는 추가로 interceptor(ByteBuddyInterceptor)를 가지고 있는데, 이 인터셉터가 id값, 원래 엔티티의 타입 정보 등을 가지고 있습니다. 그리고 target 이라는 필드가 있습니다.

프록시를 초기화하고 나면 select 쿼리의 결과로 생성된 엔티티를 바로 이 target에 저장하게 됩니다. 때문에 select 쿼리를 통해 프록시가 초기화되지 않은 상태에서 프록시는 id값은 알 수 있지만 target이 없기 때문에 id를 제외한 다른 필드 값은 알 수가 없는 것입니다.


- @Access(AccessType.PROPERTY)로 설정한 경우에만 초기화되지 않는다.

- 엔티티 접근 방식을 필드@Access(AccessType.FIELD) 로 설정하면  프록시 객체를 초기화한다.


# 프록시의 특징


## 프록시는 실제 객체의 상속받아 만든다

![](https://velog.velcdn.com/images/leekhy02/post/a5d22b59-52d9-4c31-9c0f-5353a75de60d/image.png)

프록시가  실제 객체처럼 동작을 할 수 있는 이유는 프록시가 실제 객체를 상속한 타입을 가지고 있기 때문입니다.

프록시 객체는 실제 객체에 대한 참조를 보관하여, 프록시 객체의 메서드를 호출했을 때 실제 객체의 메서드를 호출합니다.

그렇기 때문에 사용자는 타입에 상관없이 사용할 수 있는 것입니다.

JPA 엔티티 생성 규칙
- 기본 생성자는 최소 protected 접근 제한자를 가져야 한다
- 엔티티 클래스는 final로 정의할 수 없다

위 두 규칙을 정해줘야 하는 이유 또한 프록시가 실제 엔티티를 상속 하기때문에입니다.

클래스가 final 일시 상속이 불가능하다.
기본 생성자가 private 라면 프록시 생성시 super 호출이 불가능하다.


## 프록시의 equals에 주의하자

``` java

 @Override
    public boolean equals(final Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        final Member member = (Member) o;
        return Objects.equals(id, member.id);
    }
```

일반적인 equals를 오버라이딩시 이런 형태를 가집니다.
equals를 호출하는 객체의 타입과 비교 대상 객체의 타입이 같은지 getClass를 통해 비교시, if문에 걸려서 false를 반환하게 됩니다.

``` java
Team team = Repository.findById(1L).get();
Member member = memberRepository.findById(team.getId).get();

assertThat(member).isEqualTo(team.getMember());
```
위 와같은 테스트가 있을때.
member와 team.member 모두 같은 프록시 객체를 가리키고 있습니다 둘은 동일성(==) 비교는 통과하는데 동등성(equals) 비교는 실패합니다. 

team.getMember()를 인자로 하여 member 객체의 equals를 호출합니다. 그런데 여기서 member는 프록시이기 때문에 equals를 직접 쓰지 못하고 실제 엔티티의 equals를 호출해야 합니다. 따라서 team.getMember()를 다시 한번 인자로 하여, 이번에는 target으로 지정된 실제 엔티티의 equals를 호출합니다. 그렇다면 다음과 같은 상황이 됩니다.

getClass()는 Member, o.getClass()는 Member의 프록시 타입이 되어 if에 걸리게 되는 것입니다.

그러므로 equals 사용시 다음과 같으 오버라이딩 해줍니다.

``` java
@Override
public boolean equals(Object o) {
    if (this == o) {
        return true;
    }
    if (!(o instanceof Team)) {
        return false;
    }
    Team team = (Team) o;
    return Objects.equals(id, team.getId());
}
```
