## 커스텀 레포지토리

Sprig Data JPA에서는 여러 이유로 사용자가 직접 레포지토리 메서드를 정의할 수 있는 기능을 제공해줍니다.
어떨 때 사용할 수 있는지 예를 들어 보겠습니다.

### 문제 상황

JPA에서 MySQL을 사용하고 `saveAll` 메서드를 호출하려 할 때 다음과 같은 쿼리가 날라갔습니다.

```sql
Hibernate:
    insert
    into
        item_image
        (image_url, item_id)
    values
        (?, ?)
Hibernate:
    insert
    into
        item_image
        (image_url, item_id)
    values
        (?, ?)
```

bulk insert 쿼리가 날라갈 것으로 예상했지만 INSERT 쿼리가 여러 번 날라가고 있었습니다.

이는 MySQL과 같은 데이터베이스에 기본키 생성을 위임하고 있었기 때문입니다.

### 내부를 들여다보자

`saveAll` 메서드를 들여다 보겠습니다.

```java
    @Transactional
	@Override
	public <S extends T> List<S> saveAll(Iterable<S> entities) {

		Assert.notNull(entities, "Entities must not be null");

		List<S> result = new ArrayList<>();

		for (S entity : entities) {
			result.add(save(entity));  // 문제 지점
		}

		return result;
	}

    @Transactional
	@Override
	public <S extends T> S save(S entity) {

		Assert.notNull(entity, "Entity must not be null");

		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}
```

```java
    public boolean isNew(T entity) {
		ID id = getId(entity);
		Class<ID> idType = getIdType();

		if (!idType.isPrimitive()) {
			return id == null;
		}

		if (id instanceof Number) {
			return ((Number) id).longValue() == 0L;
		}

		throw new IllegalArgumentException(String.format("Unsupported primitive id type %s!", idType));
	}
```

JPA의 영속성 컨텍스트에 엔티티가 존재하기 위해서는 식별자가 필요합니다.
그런데 IDENTITY 방식에서는 PK값 생성을 DB에 위임하기 때문에 `persist` 호출 시 엔티티를 영속성 컨텍스테 등록하기 위해 INSERT 쿼리가 실행됩니다.

그렇다면 기본 키 생성전략이 IDENTITY인 경우에는 bulk insert를 어떻게 수행할 수 있을까요?
생각한 방법은 jdbcTemplate을 사용하는 방법이였습니다.

### 사용자 정의 레포지토리

이를 위해서는 커스텀 레포지토리를 생성할 필요가 있었습니다. 이를 위해 JPA에서 제공하는 커스텀 레포지토리 기능을 이용했습니다.

```java
public interface ItemImageRepositoryCustom {

    void saveAllItemImages(List<ItemImage> itemImages);
}
```

위와 같이 bulk insert 연산을 수행할 메서드를 정의하고 있는 커스텀 레포지토리 인터페이스를 생성합니다.
이후 실제 구현을 담고 있는 클래스를 하나 생성합니다. 이때 주의해야할 점은 이름을 짓는 규칙이 존재한다는 것입니다.
`레포지토리 인터페이스 이름 + Impl`로 네이밍을 해야 합니다.
이렇게 하면 Spring Data JPA가 사용자 정의 레포지토리로 인식하게 됩니다.

```java
@RequiredArgsConstructor
public class ItemImageRepositoryImpl implements ItemImageRepositoryCustom {

    private final NamedParameterJdbcTemplate jdbcTemplate;

    @Override
    public void saveAllItemImages(List<ItemImage> itemImages) {
        // IDENTITY 방식의 한계로 bulk insert query 직접 구현
        String sql = "INSERT INTO item_image "
                + "(image_url, item_id) VALUES (:imageUrl, :itemId)";
        MapSqlParameterSource[] params = itemImages.stream()
                .map(itemImage -> new MapSqlParameterSource()
                        .addValue("imageUrl", itemImage.getImageUrl())
                        .addValue("itemId", itemImage.getItem().getId()))
                .collect(Collectors.toList())
                .toArray(MapSqlParameterSource[]::new);
        jdbcTemplate.batchUpdate(sql, params);
    }
}
```

마지막으로 레포지토리 인터페이스에서 사용자 정의 인터페이스를 상속받으면 됩니다.

```java
public interface ItemImageRepository extends JpaRepository<ItemImage, Long>, ItemImageRepositoryCustom {
}
```

## JPA에서의 페이징

우리는 DB를 사용하면서 데이터를 모두 불러오는 것이 아닌 페이지 단위로 불러오기 싶은 경우가 종종 있습니다.
이때 사용할 수 있는 것이 JPA 에서 제공하는 페이징 기능인데 어떻게 사용하는지 알아보겠습니다.

## JPA에서의 Pagination

JPA에서는 아래와 같이 별도의 페이징 쿼리가 필요없습니다.

```java
    @Test
    void paging() {
        for (int i = 0; i < 100; i++) {
            em.persist(new Member("member - " + i));
        }

        em.flush();
        em.clear();

        em.createQuery("SELECT m FROM Member m", Member.class)
                .setFirstResult(3)
                .setMaxResults(10)
                .getResultList();
    }
```

```sql
Hibernate:
    select
        member0_.id as id1_2_,
        member0_.name as name2_2_,
        member0_.team_id as team_id3_2_
    from
        member member0_ limit ?,?
```

## Spring Data JPA에서의 Pagination

Spring Data JPA에서의 페이징은 더 간단합니다. 코드로 보겠습니다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    Page<Member> findAll(Pageable pageable);
}
```

```java
    @Test
    void springDataJpaPaging() {
        for (int i = 0; i < 100; i++) {
            em.persist(new Member("member - " + i));
        }

        em.flush();
        em.clear();

        Page<Member> result = memberRepository.findAll(PageRequest.of(10, 5));
        assertThat(result.getTotalPages()).isEqualTo(20);   // 5개씩 20페이지
        Pageable pageable = result.getPageable();
        assertThat(pageable.getPageNumber()).isEqualTo(10); // 10페이지
        assertThat(pageable.getOffset()).isEqualTo(50);     // 5개씩 10페이지니까 50번쨰
    }
```

```sql
Hibernate:
    select
        member0_.id as id1_2_,
        member0_.name as name2_2_,
        member0_.team_id as team_id3_2_
    from
        member member0_ limit ?,?
Hibernate:
    select
        count(member0_.id) as col_0_0_
    from
        member member0_
```

쿼리를 보면 `COUNT` 쿼리도 같이 나가는 것을 확인할 수 있는데 Spring Data JPA에서 `Page` 타입을 사용할때 나타나는 특징입니다.

이렇게 `Pageable` 클래스를 사용해서 간단하게 페이징을 수행할 수 있습니다. 뿐만 아니라 페이징 정렬 조건까지 지정할 수 있다는 것이 특징입니다.

```java
    @Test
    void springDataJpaPagingWithSorting() {
        for (int i = 0; i < 100; i++) {
            em.persist(new Member("member - " + i));
        }

        em.flush();
        em.clear();

        Page<Member> result = memberRepository.findAll(
                PageRequest.of(
                        10,
                        5,
                        Sort.by(Sort.Direction.DESC, "name"))
        );
        Member member = result.get().findFirst().get();
        // 0번째 부터 10번째 페이지
        // 49 ~ 53
        assertThat(member.getName()).isEqualTo("member - 53");
    }
```

불필요한 COUNT 쿼리를 날리기 싫다면 `Slice`를 이용할 수 있습니다.
그런데 `Page`, `Slice` 타입은 OFFSET 방식의 페이징을 수행하는 것이 특징입니다.
