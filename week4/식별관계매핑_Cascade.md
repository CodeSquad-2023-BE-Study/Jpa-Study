### 1. 문제 상황

StoreMeta ↔ StoreDetail

기존의 구현 방식

- StoreMeta와 StoreDetail이 각각 인공키를 사용
- 각각 인공키를 사용하면, 동일하게 Insert하더라도 DB Sequence의 작동 원리에 따라 언젠가는 틀어질 수 밖에 없다.
    - ex) A Insert 1 → B Insert(실패) → A roll-back → (재시도) → A Insert 2 → B Insert 1
- StoreMeta와 StoreDetail이 동일한 협업 지점을 나타내도, API에 따라 ID가 다른 문제점이 있었음.
- API에 따라 클라이언트에서 ID로 요청을 보내는데, Meta와 Detail이 키가 다르다보니 혼선이 생겼다. → 둘 중 하나로 통일해서 헷갈리지 않도록 만들긴했지만, 뭔가 더 좋은 방법이 없을까?

→ 식별관계를 사용해 해결

### 식별 관계란

부모테이블의 기본키를 자식 테이블로 전파하여 자식테이블의 기본키로 사용하는 관계를 식별관계라고 한다.

식별관계의 문제점

- 부모 테이블의 기본키가 전파되면서 기본 키 컬럼이 점점 늘어난다. → OneToMany에 해당
- 자연 키 컬럼을 사용하는 경우, 비즈니스 요구사항의 변화에 따라 수정이 필요한 경우가 발생한다. → OneToOne으로 인공키를 가져와서 사용한다면 인공키를 사용하는 셈이다.

### 변경한 설계

```
StoreMeta {
	@Id @GeneratedValue
	private long id;

	//@OneToOne(mappedBy = "storeMeta")
	//private StoreDetail storeDetail;
	}

StoreDetail {
	
	@Id
	private long id;
	@MapsId
	@OneToOne
	@JoinColumns("id")
	private StoreMeta storeMeta;

}
```

## Cascade

Cascade는 특정 엔티티를 영속 상태로 만들 때, 관련 엔티티도 함께 영속 상태로 만들어 주는 것.

궁금한 점 → 예시코드를 보면 아래와 같은 코드들이 있다.

```
public class Parent {
	@OneToMany(mappedBy="parent", cascade=cascadeType.All)
	private List<Child> children = new Arraylist<>();
}
```

mappedBy를 사용하면 기본적으로 readOnly가 된다. 그러나, cascadeType.ALL을 허용하면, PERSIST, REMOVE, MERGE 등이 허용된다. 즉 부모를 통해서 자식 데이터에 접근이 가능해진 셈이다.

### Cascade Type?

- Persist : 연관 엔티티를 Context에 등록시킨다.
- Merge : 연관 엔티티를 Update할 수 있게 된다.
- Detach : 연관 엔티티를 Context에서 떼어낸다.
- Remove : 연관 엔티티를 제거한다.

즉, CascadeType.All을 설정하면, 데이터 Insert가 아닌 Delete, Update는 전이가 된다.

### mappedBy를 설정하는 이유?

- 코드 가독성 측면에서 양방향 매핑이 되어있음을 알리는 역할
- **외래 키가 있는 곳에서만 외래키를 관리하여 복잡도를 줄이기 위함**

### 써도 될까?

CascadeType.All을 설정하면, 외래 키가 있는 곳이 아닌 반대 엔티티에서도 데이터에 영향을 끼칠 수 있게 되어 관리 복잡도가 늘어날 수 있다. 따라서 이런점을 잘 알고 고려해서 쓰는 것이 중요하다고 할 수 있다.

### 언제 쓰면 안될까?

여기까지 Cascade에서 발생할 수 있는 부작용에 대해 알아보았다. 그렇다면 어떤 상황에서 사용하면 관리 복잡도가 증가할지 생각해보자.

→ 전이하는 대상 엔티티를 다른 엔티티가 참조하는 경우

즉, 특정 엔티티에만 종속되는 것이 아닌 여러 군데에 종속된 엔티티인 경우 Cascade Option을 사용하는 것이 큰 문제를 일으킬 수 있다.

```json
@Entity
public class Parent1{
	...
	@OneToMany(mappedBy = "parent", cascade=CascadeType.ALL)
	private List<Child> childList = new ArrayList<>();

	...
}

@Entity
public class Parent2{
	...
	@OneToMany(mappedBy = "parent", cascade=CascadeType.ALL)
	private List<Child> childList = new ArrayList<>();

	...
}
@Entity
public class Child{
	...
	@ManyToOne
	@JoinColumn(name = "parent_id")
	private Parent1 parent;

	@ManyToOne
	@JoinColumn(name = "parent_id")
	private Parent2 parent;
	...
}

...
Child child = new Child();
Parent1 parent1 = new Parent();
Parent2 parent2 = new Parent();

parent1.addChild(child);
parent2.addChild(child);

em.persist(parent);// parent만 persist 해주니 child도 같이 persist된다.

em.remove(parent1);
// 실행 시 parent1에 속한 child 엔티티가 삭제되어 parent2는 child가 없어진다.

em.flush();
em.clear();

Parent findParent = em.find(Parent.class, parent.getId());
findParent.getChildList().remove(0); // orphanRemoval 동작
```
