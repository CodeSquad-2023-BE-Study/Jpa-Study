# 들어가기
jpa에는 다양한 연관 관계 매핑이 있습니다. 그중 가장 많이 사용하고 중요한 매핑은 
다대일 관계 매핑이고 즉 @ManyToone 매핑을 중심으로 연관관계를 살펴보고 학습해보겠습니다.

연관관계중 다대일을 중심으로 학습하는 시간이기 때문에 다른 부가적인것은 간단하게 설명하겠습니다.
## 객체와 테이블의차이
테이블은 특히 관계형 데이터베이스 테이블은 외래키로 연관관계를 맺습니다. 즉 `JOIN`
을 사용합니다. 하지만 객체에는 참조라는 것을 통해 연관관계를 습니다. 즉 테이블은 외래키 하나를 사용해 두 테이블의 연관관계가 표현 가능하지만(즉 테이블은 항상 양방향입니다.) 객체는 어느 객체가 외래키를 관리할지(연관관계 주인), 두 엔티티의 방향관계를 정해줘야합니다.

### 방향
테이블은 외래키를 통한 Join을 하기때문에 두테이블은 서로 양방향 관계를 가집니다. 엔티티와 큰 차이를 가집니다. 객체에서는 양방향, 단방향 두 엔티티의 관계를 정해줘야합니다.

### 연관관계의 주인
객체를 양방향 연관관계로 만들면 연관관계 주인을 정해야합니다. 사실 는 사실 연관관계라는 것이 없기때문에 서로 다른 단방향 연관관계 2개를 어플리케이션 로직에서 엮어서 양뱡향 처럼 보이게 하는것입니다.

또한 연관관계의 주인만 데이터베이스 연관관계와 매핑 되고 외래키를 관리 할수 있습니다.
주인이 아닌쪽 (mappedBy쪽)은 읽기만 가능합니다.

# 다대일 (@ManyToOne)
- @ManyToOne 어노테이션을 사용해서 다대일 관계를 매핑한다.
- @JoinColumn(name = 값) 외래 키를 매핑할 때 사용한다. name은 매핑할 외래 키 이름이다.
- 테이블에서의 1, N 관계에서 외래 키는 항상 다쪽(N)에 있다 따라서 양방향의 경우 항상 N 쪽이 연관관계의 주인이다

이 말은 즉 양방향일 경우에는 @OneToMany를 선언한 쪽에서 mappedBy속성을 지정해야 한다는 뜻이 됩니다. 양방향을 설명할 때 더 자세히 봅시다.

## 다대일 단반향 매핑
```java
@Entity
public class Member {

    @Id
    @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    private String userName;

    @ManyToOne
    @JoinColumn(name = "team_id")
    private Team team;
}
```

```java
@Entity
public class Team {
    @Id
    @GeneratedValue
    @Column(name = "team_id")
    private Long id;

    private String name;
}
```

member엔티티에서 member에있는 team을 참조할수 있지만 team엔티티에서는 회원을 참조하는 필드가 없니다 이것을 단방향 연관 관계라고 합니다.
이말은 즉 **@JoinColumn이 붙은 team이라는 필드로 team_id라는 회원테이블의 외래키를 관리하겠다라는 말입니다**.

## 다대일 양방향 매핑

```java
@Entity
public class Member {

    @Id
    @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    private String userName;

    @ManyToOne
    @JoinColumn(name = "team_id")
    private Team team;
    
    public void setTeam(Team team){
    	this.team = team;
        //무한루프에 빠지지 않도록 체크
        if(!team.getMembers().comtains(this){
        	team.getMembers().add(this);
            }
}
```

```java
@Entity
public class Team {
    @Id
    @GeneratedValue
    @Column(name = "team_id")
    private Long id;

    private String name;

    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<Member>();
    
    public void addMember(Member member){
        this.members.add(member);
        if(member.getTeam() != this){
            members.setTeam(this);
        }
    }
}
```

일대다와 다대일 관계에서는 당연히 N쪽이 외래키가 있습니다. 위에서 말했듯이 외래키를 관리할 때 연관 관계 주인인 다대일쪽의 Member클래스의 team 필드를  사용합니다.
 Team.members는 조회를 위한 JPAQL이나 객체 그래프를 탐색할 때 사용합니다.
 
 ``` java
 // JPQL 쿼리 예제: 특정 팀의 모든 멤버 조회
String jpql = "SELECT m FROM Member m WHERE m.team = :team";
List<Member> members = entityManager.createQuery(jpql, Member.class)
        .setParameter("team", team)
        .getResultList();


// team에있는 members로 객체탐색 가능

    public void printMembers() {
        for (Member member : members) {
            System.out.println("Member name: " + member.getName());
        }
    }
 ```
 
 ### 양방향 연관관계는 항상 서로를 참조해야 한다.
 
 한쪽만 참조하는 것이 아닌 항상 서로를 참조하고 있어야 합니다.그래서 위의 setTeam()메서드나
addMember()메시드와 같은 연관관계 편의 메서드를 작성하는 편이 좋습니다.
단 편의 메서드는 한곳에 작성하거나 둘다 작성 할수 있는데 양쪽다 작성할 경우 무한 루프에 빠지지 않도록 주의합니다 위의 코드는 무한루프를 방지하는 로직이있습니다.
양쪽에 작성할경우 한 메서드만 부르면 됩니다. 보통 객체지향 적으로 책임과 역학을 생각해 그쪽에 편의 메서드를 두는 방법을 선택합니다.
 
