# OSIV

- JPA에는 [Open EntityManager In View](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data.sql.jpa-and-spring-data.open-entity-manager-in-view) 패턴이 라고 하지만 Hibernate에서는 Open Session In View 라고합니다.

<aside>
💡 If you are running a web application, Spring Boot by default registers `[OpenEntityManagerInViewInterceptor](https://docs.spring.io/spring-framework/docs/6.0.11/javadoc-api/org/springframework/orm/jpa/support/OpenEntityManagerInViewInterceptor.html)` to apply the “Open EntityManager in View” pattern, to allow for lazy loading in web views. If you do not want this behavior, you should set `spring.jpa.open-in-view` to `false` in your `application.properties`.

---

웹 애플리케이션을 실행하면 Spring Boot는 기본적으로 `OpenEntityManagerInViewInterceptor`를 등록하여 웹 보기에서 지연 로딩을 허용하기 위해 **"Open EntityManager in View" 패턴**을 적용합니다. 이 동작을 원하지 않는 경우 application.properties에서 spring.jpa.open-in-view를 `false`로 설정해야 합니다.

</aside>

- 직역하자면 view영역에서도 session을 열어둔다는 뜻인데, 여기서 session은 영속성 컨텍스트를 의미합니다. [참고링크](https://www.tutorialspoint.com/hibernate/hibernate_sessions.htm) 이게 가능할 경우 엔티티는 영속인 상태를 view까지 유지하며 view에서 지연 로딩을 하는 데 자유로워 집니다.

## ****Session per operation****

- 단일 스레드에서 각 데이터베이스 호출에 대해 Session을 열고 닫는 패턴을 말합니다.
- auto-commit 모드를 켠 것과 같은 효과를 가집니다.
    - Hibernate는 애플리케이션 서버가 자동 커밋 모드를 비활성화하는 것을 기본값으로 예상합니다.
- 데이터베이스 트랜젝션 측면에서도 안티 패턴입니다.
- 여러 작은 트랜잭션은 명확한 하나의 작업 단위보다 성능이 떨어지고 유지 관리 및 확장이 더 어렵습니다.

![image](https://github.com/CodeSquad-2023-BE-Study/Jpa-Study/assets/103120173/386003a7-73ce-4e77-9653-b3904b03d6e6)

## ****Session per Request****

- 책에서는 과거 OSIV라고 표현되었습니다.
- 요청처리를 하기 시작하면 애플리케이션은 session을 열고 트랜젝션을 시작합니다. 모든 데이터 관련 작업을 수행하고 나서야 트랜잭션을 종료하고 session을 닫습니다.
- 트랜잭션과 세션간 1:1 관계라는 면이 특징적입니다.
- 구현
    - `javax.servlet.Filter` implementation
    - AOP interceptor with a pointcut on the service methods
    - A proxy/interception container
![image](https://github.com/CodeSquad-2023-BE-Study/Jpa-Study/assets/103120173/09c209db-1f39-43d9-aabe-da7c44b0eac0)

- 여기서 ‘요청’ 은 비단 웹 어플리케이션의 web request라기보다는 하나의 어플리케이션의 요청 단위로 해석할 수 있습니다.

## Session per application

- 어플리케이션 1개당 session을 1개 유지하는 것으로 매우 안티패턴입니다.
- 멀티 스레드 환경에서 session이 공유될 수 있습니다.
- 오랫동안 session을 열어두면 session이 entity를 캐시하면서 사용중인 메모리가 너무 커지는 문제점이 있습니다.

# Session per Resquest 패턴의 OSIV

- Session per Resquest 패턴의 원형이 가진 문제점이 있습니다.
- 프레젠테이션 계층에서 엔티티를 변형시킬 수 있다는 점입니다.
    - 뷰를 렌더링한 후 요청에 대한 트랜잭션을 커밋할 때 영속성 컨텍스트도 플러시됩니다.
    - 의도치않게 프레젠테이션 계층에서 DB를 수정할 수 있습니다.
- 그래서 이 패턴에서 해결 방법은
    - **엔티티를 읽기 전용의 인터페이스로 제공**
        - 값을 변형하는 인터페이스를 허용하지 않습니다.
    - **엔티티를 래핑**
        - 마찬가지로 값을 변형하는 인터페이스가 없는 Wrapper 객체로 엔티티를 감쌉니다.
    - **DTO만 반환**
        - 가장 전통적인 방법으로 엔티티에 값을 채워 프레젠테이션 계층에서 사용합니다.

## Spring 프레임워크에서 OSIV

- spring boot 2.0부터는 명시적으로 표시하지 않으면 다음과같은 경고 문구를 줍니다.
    - open in view 가 기본으로 true 값을 가지므로 데이터베이스 쿼리가 뷰 렌더링 중에도 작동할 수 있다는 뜻입니다.

```java
2023-09-04T16:12:07.211+09:00  WARN 19525 --- [           main]
JpaBaseConfiguration$JpaWebConfiguration :
spring.jpa.open-in-view is enabled by default.
Therefore, database queries may be performed during view rendering.
Explicitly configure spring.jpa.open-in-view to disable this warning
```

```java
spring.jpa.open-in-view=false // 이렇게 비활성화할 수 있습니다.
```

- 스프링 프레임워크가 제공하는 OSIV는 `비즈니스 계층`에서 트랜잭션을 사용합니다.

### OSIV 옵션이 `false`일 경우
![image](https://github.com/CodeSquad-2023-BE-Study/Jpa-Study/assets/103120173/c3c597bf-c3e0-4b53-a181-22cd5f3f9f7a)

- 트랜잭션 종료시 영속성 컨텍스트도 `flush()` → `close()` 가 되었고 `connection`도 반환한 상태입니다.
- 이경우 view에서 lazy loading 을 시도할 경우 에러가 발생합니다.

```java
LazyInitializationException: could not initialize proxy [com.practice.playground.jpa] - no Session
```

- 예시

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String username;

    @ElementCollection // 기본적으로 지연로딩을 합니다.
    private Set<String> permissions;

    // 기타등등등...
}
```

```java
@Service
public class SimpleUserService implements UserService {

    private final UserRepository userRepository;

    public SimpleUserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    @Transactional(readOnly = true)
    public Optional<User> findOne(String username) { // 이게 실행되면?
        return userRepository.findByUsername(username);
    }
}
```

- 트랜잭션 AOP에서 트랜잭션을 시작합니다.
- `permissions` 는 초기화 되지 않은 상태로 메서드 호출이 끝납니다.
- 다시 트랜잭션 AOP를 통해 트랜잭션이 커밋됨과 동시에 영속성 컨텍스트도 종료됩니다.
- 이제 프레젠테이션 계층에서는 `permissions`을 호출할 수 없습니다.

### OSIV 옵션이 `true`일 경우

![image](https://github.com/CodeSquad-2023-BE-Study/Jpa-Study/assets/103120173/7c1bfb9d-ae65-45e0-81be-e15b4a8b480c)

- 트랜잭션 범위는 끝나서 영속성 컨텍스트 `flush()` 가 되었지만 `close()` 가 되지 않은 채로 계속 유지되고 있습니다.
- DB connection도 유지되었으므로 프레젠테이션 계층에서 지연 로딩이 가능합니다.
- 요청 범위가 끝나고 Servlet Filter나 Spring interceptor로 돌아오면 이 때 영속성 컨텍스트도 `close()` 됩니다.
- **주의할 점은 트랜잭션 범위를 벗어나고 다시 트랜잭션 범위로 진입하게 되면, 프레젠테이션 계층에서 변화한 엔티티가 `flush()` 될 수 있다는 것입니다.**

## `OSIV` 활성화시 주의할 점!

### IO를 너무 섞어서 사용하지 않아야 합니다.

- 만약 하나의 트랜잭션 범위에서 데이터베이스 IO 작업을 하거나 다른 IO를 섞어서 사용할 경우 어떤 서비스가 조금 더 느릴 때 데이터베이스 연결이 부족해질 수도 있습니다.
- 이 경우 `@Transactional` 주석 범위를 바꾸거나 제거하여 해결할 수도 있습니다.

### DB connection pool 소진

- 만약 OSIV가 활성화되었을 때 요청범위에 세션을 항상 유지하게 되면서 DB 커넥션 풀이 부족할 수 있음을 주의해야합니다.
- 초반에는 별 문제 없지만 처음 DB connection 이후 요청이 끝날 때까지 세션이 유지됩니다.
    1. 요청 시작 시 해당 필터는 새 *`Session`*을 **생성합니다 .
    2. *findByUsername* 메소드를 호출하면  해당 세션은 풀에서 DB connection을 빌립니다.
    3. 세션은 요청이 끝날 때까지 이 DB connection을 유지합니다.
 
- [OKKY - server sent event (sse) 사용시 connection pool 부족 문제](https://okky.kr/questions/1399411)

### 불필요한 쿼리를 더 요청할 수 있습니다.

- 세션이 요청이 끝날 때까지 열려있으므로 의도하지 않게 쿼리를 여러개 더 실행시킬 수 있습니다.
- n+1 문제를 일으킬 수도 있지만 명시적인 쿼리 호출이 아니기 때문에 이런 추가 쿼리를 알아차리기 힘들 수도 있습니다.

# 질문사항

## 그래서 어떻게 하는게 좋다는 말이지? 🤔

- OSIV가 false 옵션인 상태에서 복잡성을 관리하는 것이 좋은 방법인 것 같습니다.
    - 간단한 서비스 외에는 OSIV옵션을 명시적으로 비활성화 한 채로 구현하는 편이 메모리나 DB 커넥션 자원 관리에 유리할 것입니다.
- Lazy Loading을 사용하기 위해 OSIV를 설정하지 않습니다.
    - 전통적인 방식으로 DTO를 사용합니다. *(개인적으로 선호합니다.)*
    - `@EntityGraph` 로 쿼리 메서드에 주석을 달아 일부 엔티티를 임시로 일찍 로딩할 수 있습니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    **@EntityGraph(attributePaths = "permissions") // left join입니다. 주의할 것**
    Optional<User> findByUsername(String username);

		Optional<User> findSummaryByUsername(String username); // permissions가 필요 없는 경우 따로 구현해야한다는 단점이 있습니다.
}
```

- 만약 관리자를 위한 API처럼 트래픽이 많지 않은 곳은 OSIV를 켜도 무방하다고 합니다.

## 파사드에서 `@Transaction` 을 시작하는 것이 좋을까?

- 작업 단위를 하나의 트랜잭션으로 만드는 것이 좋을까?
- 파사드에 트랜잭션을 걸면 트랜잭션 단위가 너무 길어지지 않을까?

## `DB connection pool`의 크기는 어느정도가 좋을까?

```java
Connection Pool Size = (core_count * 2) + effective_spindle_count

- core_count: CPU 코어 개수.
- effective_spindle_count: 하드 디스크의 개수. spindle은 DB 서버가 관리할 수 있는 동시 I/O 요청의 개수를 의미한다.
```

- [HikariCP와 적절한 풀 사이즈 고민하기 (1) - 이론편](https://dallog.github.io/hikari-cp-1-theory/)
- [Connection Pool과 적절한 Size 설정하기 (with HikariCP)](https://colour-my-memories-blue.tistory.com/15)

---

# Refs.
- [Hibernate - Sessions](https://www.tutorialspoint.com/hibernate/hibernate_sessions.htm)
- [CQS(Command Query Separation) Pattern 이란?](https://medium.com/@su_bak/cqs-command-query-separation-pattern-이란-f701eabf8754)
- [A Guide to Spring's Open Session In View](https://www.baeldung.com/spring-open-session-in-view)
- [[Spring] OSIV로 알아보는 Spring Transaction 헤짚기](https://blog.neonkid.xyz/297)
- [왜 LazyInitializationException이 발생하지? - OSIV편](https://veluxer62.github.io/explanation/osiv/)
- [OpenEntityManagerInViewInterceptor (Spring Framework 6.0.11 API)](https://docs.spring.io/spring-framework/docs/6.0.11/javadoc-api/org/springframework/orm/jpa/support/OpenEntityManagerInViewInterceptor.html)
- [Hibernate ORM 6.2.8.Final User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#architecture-current-session)
 
