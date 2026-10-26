# Testing - Quarkus Test

## Unit Testing with JUnit 5

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("Should create user successfully")
    void shouldCreateUser() {
        // Given
        UserCreateRequest request = new UserCreateRequest(
            "test@example.com",
            "Password123",
            "testuser",
            25
        );

        when(userRepository.existsByEmail(request.email())).thenReturn(false);

        // When
        UserResponse response = userService.create(request);

        // Then
        assertThat(response).isNotNull();
        assertThat(response.email()).isEqualTo(request.email());
        verify(userRepository).existsByEmail(request.email());
        verify(userRepository).persist(any(User.class));
    }

    @Test
    @DisplayName("Should throw exception when email already exists")
    void shouldThrowExceptionWhenEmailExists() {
        // Given
        UserCreateRequest request = new UserCreateRequest(
            "test@example.com", "Password123", "testuser", 25
        );
        when(userRepository.existsByEmail(request.email())).thenReturn(true);

        // When & Then
        assertThatThrownBy(() -> userService.create(request))
            .isInstanceOf(DuplicateResourceException.class)
            .hasMessageContaining("Email already registered");

        verify(userRepository, never()).persist(any(User.class));
    }
}
```

This is a plain JUnit 5 + Mockito test with no Quarkus annotation at all — fastest option, use it whenever the class under test has no CDI dependencies that are awkward to construct by hand.

## Integration Testing with `@QuarkusTest` + RestAssured

```java
@QuarkusTest
class UserIntegrationTest {

    @Inject
    UserRepository userRepository;

    @BeforeEach
    @Transactional
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    @Order(1)
    @DisplayName("Should create user via API")
    void shouldCreateUserViaApi() {
        var request = new UserCreateRequest("test@example.com", "Password123", "testuser", 25);

        given()
            .contentType(ContentType.JSON)
            .body(request)
            .when().post("/api/v1/users")
            .then()
                .statusCode(201)
                .header("Location", notNullValue())
                .body("email", is(request.email()));
    }

    @Test
    @Order(2)
    @DisplayName("Should return validation error for invalid request")
    void shouldReturnValidationError() {
        var request = new UserCreateRequest("invalid-email", "short", "u", 15);

        given()
            .contentType(ContentType.JSON)
            .body(request)
            .when().post("/api/v1/users")
            .then()
                .statusCode(400)
                .body("errors", not(empty()));
    }
}
```

`RestAssured`'s `given()/when()/then()` is the one and only HTTP test client in Quarkus — the same API tests blocking resources, Mutiny-reactive resources, and (as shown further below) security-protected ones.

## Resource Testing with Mocked Services

```java
@QuarkusTest
class UserResourceTest {

    @InjectMock
    UserService userService;

    @Test
    @DisplayName("Should get all users")
    void shouldGetAllUsers() {
        List<UserResponse> users = List.of(
            new UserResponse(1L, "user1@example.com", "user1", 25, true, null, null),
            new UserResponse(2L, "user2@example.com", "user2", 30, true, null, null)
        );
        when(userService.findAll(0, 20)).thenReturn(users);

        given()
            .when().get("/api/v1/users")
            .then()
                .statusCode(200)
                .body("$.size()", is(2))
                .body("[0].email", is("user1@example.com"));
    }

    @Test
    @DisplayName("Should create user")
    void shouldCreateUser() {
        var request = new UserCreateRequest("test@example.com", "Password123", "testuser", 25);
        var response = new UserResponse(1L, request.email(), request.username(), request.age(),
            true, LocalDateTime.now(), LocalDateTime.now());

        when(userService.create(any(UserCreateRequest.class))).thenReturn(response);

        given()
            .contentType(ContentType.JSON)
            .body(request)
            .when().post("/api/v1/users")
            .then()
                .statusCode(201)
                .header("Location", notNullValue())
                .body("email", is(request.email()))
                .body("username", is(request.username()));
    }

    @Test
    @DisplayName("Should return 403 for non-admin user")
    @TestSecurity(user = "alice", roles = "USER")
    void shouldReturn403ForNonAdmin() {
        given()
            .when().get("/api/v1/admin/users")
            .then()
                .statusCode(403);
    }
}
```

`@InjectMock` (from `quarkus-junit5-mockito`) mocks a single CDI bean without disabling the rest of the application context. `@TestSecurity` (from the `quarkus-test-security` test-scoped dependency) installs a fake `SecurityIdentity` with the given username/roles for the duration of the test, with no real token needed. It only works with `@QuarkusTest`, not `@QuarkusIntegrationTest`.

When the resource reads token claims (`JsonWebToken`, `@Claim`), add `io.quarkus:quarkus-test-security-jwt` (test scope) and put the claims next to `@TestSecurity`:

```java
@Test
@DisplayName("Should return the profile built from JWT claims")
@TestSecurity(user = "alice", roles = "USER")
@JwtSecurity(claims = {
    @Claim(key = "email", value = "alice@example.com"),
    @Claim(key = "upn", value = "alice@example.com")
})
void shouldReturnProfileFromJwtClaims() {
    given()
        .when().get("/api/v1/profile/me")
        .then()
            .statusCode(200)
            .body("email", is("alice@example.com"));
}
```

`@JwtSecurity` and `@Claim` come from `io.quarkus.test.security.jwt`. With `quarkus-oidc`, use `quarkus-test-security-oidc` and `@OidcSecurity` instead.

## Repository Testing

```java
@QuarkusTest
class UserRepositoryTest {

    @Inject
    UserRepository userRepository;

    @Test
    @DisplayName("Should find user by email")
    @TestTransaction
    void shouldFindUserByEmail() {
        var user = new User();
        user.email = "test@example.com";
        user.password = "password";
        user.username = "testuser";
        user.active = true;
        userRepository.persist(user);

        Optional<User> found = userRepository.findByEmail("test@example.com");

        assertThat(found).isPresent();
        assertThat(found.get().email).isEqualTo("test@example.com");
    }

    @Test
    @DisplayName("Should check if email exists")
    @TestTransaction
    void shouldCheckIfEmailExists() {
        var user = new User();
        user.email = "test@example.com";
        user.password = "password";
        user.username = "testuser";
        user.active = true;
        userRepository.persist(user);

        assertThat(userRepository.existsByEmail("test@example.com")).isTrue();
    }
}
```

`@TestTransaction` wraps the test method in a transaction that's rolled back at the end — a lightweight alternative to manually deleting rows in `@BeforeEach` when a test only needs its own writes to disappear afterward. There's no separate repository-slice annotation: this is a plain `@QuarkusTest`, which is already fast because the Quarkus app only boots once for the whole test run.

## Dev Services — Zero-Config Testcontainers

Add the datasource extension:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-postgresql</artifactId>
</dependency>
```

No `quarkus.datasource.jdbc.url`, no `@Testcontainers`, no `@Container`, no `@ServiceConnection` annotation anywhere. Running `./mvnw test` (or `@QuarkusTest` from an IDE) makes Quarkus notice the datasource has no connection URL configured and launch a real PostgreSQL container via Testcontainers automatically, wiring the app to it and tearing it down afterward. The `UserRepositoryTest` above already runs against a real Postgres instance with zero extra test code. The same zero-config mechanism covers Kafka, Redis, MongoDB, Keycloak, and more — see the standalone `testing-patterns` skill for the full picture (knobs to disable it, pinning an image, and `QuarkusTestResource`/`@WithTestResource` for anything Dev Services doesn't cover, like WireMock).

## Testing Reactive (Mutiny) Endpoints

```java
@QuarkusTest
class OrderReactiveResourceTest {

    @InjectMock
    OrderService orderService;

    @Test
    @DisplayName("Should get order reactively")
    void shouldGetOrderReactively() {
        var order = new OrderDto(UUID.randomUUID(), "test@example.com", BigDecimal.TEN);
        when(orderService.findById(order.id())).thenReturn(Uni.createFrom().item(order));

        given()
            .when().get("/api/v1/orders/{id}", order.id())
            .then()
                .statusCode(200)
                .body("id", is(order.id().toString()));
    }
}
```

There's no separate test setup for reactive resources — the exact same `@QuarkusTest` + RestAssured combination used for blocking resources tests a `Uni<T>`-returning one, because RestAssured only ever sees real HTTP responses, not the reactive type behind them.

## Testing Configuration

```properties
# application.properties - %test profile overrides
%test.quarkus.log.level=INFO
%test.quarkus.hibernate-orm.log.sql=true

# application-test overrides are expressed with the %test. prefix, not a
# separate application-test.yml document
```

```java
public class TestClockProducer {

    @Produces
    @Alternative
    @Priority(1)
    public Clock fixedClock() {
        return Clock.fixed(Instant.parse("2024-01-01T00:00:00Z"), ZoneId.of("UTC"));
    }
}
```

A CDI `@Alternative` producer like this, enabled for a specific test class via a `QuarkusTestProfile`'s `getEnabledAlternatives()`, overrides the real bean only for tests using that profile — see the standalone `testing-patterns` skill for `@TestProfile` in full.

## Test Fixtures

```java
public class TestDataFactory {

    public static User createUser(String email, String username) {
        var user = new User();
        user.email = email;
        user.password = BcryptUtil.bcryptHash("encodedPassword");
        user.username = username;
        user.active = true;
        return user;
    }

    public static UserCreateRequest createUserRequest() {
        return new UserCreateRequest("test@example.com", "Password123", "testuser", 25);
    }
}
```

A plain static factory class (no CDI scope needed) is enough here — test fixtures don't need to be CDI beans at all.

## Quick Reference

| Annotation | Purpose |
|------------|---------|
| `@QuarkusTest` | Boots the full CDI container in-process; the default for anything touching a bean |
| `@QuarkusIntegrationTest` | Runs the packaged jar/native image/container as a black box (see `testing-patterns` skill) |
| `@InjectMock` / `@InjectSpy` | Mock/spy a single CDI bean |
| `@TestSecurity` | Install a fake authenticated identity for a test |
| `@TestTransaction` | Roll back the test method's transaction automatically |
| Dev Services (just add the extension) | Zero-config Testcontainers for DB/Kafka/Redis/etc. |
| `@TestProfile` | Run a test class with different config/alternatives |

## Testing Best Practices

- Write tests following the AAA pattern (Arrange, Act, Assert)
- Use descriptive test names with `@DisplayName`
- Mock external dependencies; use Dev Services for a real database instead of mocking the repository in integration tests
- Test the happy path and the edge cases, both positive and negative
- Use `@TestTransaction` or explicit cleanup for test data isolation
- Keep plain unit tests (no `@QuarkusTest`) separate from `@QuarkusTest` integration tests so the fast ones can run first
- Use `@ParameterizedTest` for multiple scenarios
- Test security rules with `@TestSecurity`, not by hand-rolling tokens
- Keep tests fast and independent of each other
