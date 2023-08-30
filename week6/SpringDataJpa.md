**Spring Data JPA는 JPA를 사용하기 편하도록 만들어 놓은 모듈입니다.**

이미 추상화 되어있는 JPA를 한번더  추상화 시켜  Repository 인터페이스를 통해 사용합니다.

Hibernate와 같은 JPA구현체를 사용해서 JPA를 사용하게 됩니다.

![image](https://github.com/CodeSquad-2023-BE-Study/Jpa-Study/assets/53938366/16c9a057-38d0-4c99-9811-ad0587b47570)


# 쿼리 메소드

인터페이스에 메소드만 선언하면 해당 메소드의 이름으로 적절한 JPQL 쿼리를 생성해서 실행

## **파라미터 바인딩**

```java

"select m from Member m where m.username = ?1" //위치기반
"select m from Member m where m.username = :name" //이름기반
```

***코드의 가독성과 유지보수를 위해 이름 기반 파라미터 바인딩을 권장합니다.***

## 벌크성 수정 쿼리

JPA 엔티티를 하나 가져와 변경감지 기능으로 수정 하는 것이아닌  모든 데이터를 일괄적인 업데이를 날려야 할경우 어떻게 해야할까요?( ex) 모든 직원의 연봉 10퍼센트 인상)
이 경우 벌크성 수정 쿼리를 사용합니다.

벌크성 수정, 삭제 쿼리는 @Modifying 어노테이션을 사용해야 합니다. (사용하지 않으면 QueryExcutionRequestException 예외가 발생)

기존 jpql사용

```java
// 연봉이 10만원 미만인 직원들을 10% 인상하는 JPQL 쿼리
public int basicJpql(){
        int increasedRowCount = em.createQuery("UPDATE Employee e SET e.salary = e.salary * 1.1 WHERE e.salary < 100000")
                .executeUpdate();
}
```

벌크성 수정 쿼리 사용

```java
	
		@Modifying // JPA  executeUpdate() 와 같은 기능
    @Query("UPDATE Employee e SET e.salary = e.salary * 1.1 WHERE e.salary < 100000")
    int increaseSalaryForLowPaidEmployees();
```

### 주의점

벌크성 쿼리는 영속성 컨텍스트를 무시하기 때문에 영속성 컨텍스트에 있는 엔티티 값들과 db값이 달라질수 있습니다.

1. 영속성 컨텍스트에 엔티티가 없는 상태에서 벌크 연산을 먼저 실행합니다.
2. 부득이한 영속성 컨텍스트에 엔티티가 이씅면. 벌크 연산 직후 컨텍스트를 초기화 합니다.

### **반환 타입**

결과가 한 건 이상이면  컬렉션 사용 , *단건*이면 반환 타입 지정합니다.

## ****QueryHint****

****JPA 쿼리 힌트(SQL 힌트가 아니라 JPA 구현체에게 제공하는 힌트)를 말합니다.****

```java
@QueryHints(value = @QueryHint(name = "org.hibernate.readOnly", value = "true"))
Product 메서드이름 (String name);
```

쿼리 메서드 위에 적어주며 쿼리가 읽기전용으로 조회한다고 힌트를 줍니다.

인터페이스에 메서드를 선언하여 메서드 이름에따라 적절한 JPQL이 나간다.

# Lock

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
```

## ****낙관적 잠금(Optimisstic Lock)****

실무에서 실질적으로 데이터 갱신시 경합이 발생하지 않을 것이라고 낙관적으로 판단하여 잠금을 거는 방법입니다.

ex)  회원정보의 업데이트는, 회원의 요청에 의해 발생하기에 동시접근이 발생 할 확률이 낮습니다.
따라서 동시 수정이 이뤄지는 경우를 감지해서 예외를 발생해도 발생 가능성이 낮다고 보는 것으로 잠금(Lock)보다는 충돌감지(Conflict detection)에 가깝습니다.

## ****비관적 잠금(Perssimistic Lock)****

동일한 데이터를 동시에 수정 할 가능성이 높을 때 쓰는 방법입니다.

ex )상품의 재고의 경우 여러명이 같은 상품을 동시에 주문할 수 있기 때문에 데이터 수정에 의한 경합이 발생할 확률이 높다고 보는 경우 비관적 잠금(Perssimistic Lock)을 통해 예외를 발생시키지 않고 정합성을 보장하는 경우에 사용합니다. 하지만 성능적인 측면은 어느정도 포기합니다.

데이터 베이스의 베타잠금(Exclusive Lock)을 사용

## ****암시적 잠금(Implicit Lock)****

코드상에 명시적으로 지정하지 않아도 잠금이 발생하는 것을 의미합니다.

ex) JPA에서는 엔티티에 @Version이 붙은 필드가 존재하거나 @OptimisticLocking 어노테이션이 설정되어 있을 경우 자동적으로 충돌감지를 위한 잠금이 실행됩니다.

## ****명시적 잠금(Explicit Lock)****

의도적으로 잠금을 실행하는 것이 명시적 잠금입니다

ex) JPA에서 엔티티를 조회할 때 LockMode를 지정하거나 select for update 쿼리를 통해 직접 잠금을 지정할 수 있습니다.
