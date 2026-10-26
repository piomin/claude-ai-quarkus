# Web Layer - JAX-RS Resources & REST APIs

## REST Resource Pattern

```java
@Path("/api/users")
public class UserResource {
    private final UserService userService;

    public UserResource(UserService userService) {
        this.userService = userService;
    }

    @GET
    public List<UserResponse> getUsers(
            @QueryParam("page") @DefaultValue("0") int page,
            @QueryParam("size") @DefaultValue("20") int size) {
        return userService.findAll(page, size);
    }

    @GET
    @Path("/{id}")
    public UserResponse getUser(@PathParam("id") Long id) {
        return userService.findById(id);
    }

    @POST
    public Response createUser(@Valid UserCreateRequest request) {
        UserResponse user = userService.create(request);
        URI location = UriBuilder.fromResource(UserResource.class)
            .path(String.valueOf(user.id()))
            .build();
        return Response.created(location).entity(user).build();
    }

    @PUT
    @Path("/{id}")
    public UserResponse updateUser(
            @PathParam("id") Long id,
            @Valid UserUpdateRequest request) {
        return userService.update(id, request);
    }

    @DELETE
    @Path("/{id}")
    @Status(204)
    public void deleteUser(@PathParam("id") Long id) {
        userService.delete(id);
    }
}
```

There's no `@RequestBody` annotation — JAX-RS infers the entity body from whichever method parameter has no other binding annotation (`@PathParam`, `@QueryParam`, etc.). `@Path` on the class plus `@GET`/`@POST`/... on the methods is the whole mapping — no separate class-level/method-level annotation split needed.

## Request DTOs with Validation

```java
public record UserCreateRequest(
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be 8-100 characters")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).*$",
             message = "Password must contain uppercase, lowercase, and digit")
    String password,

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username must be alphanumeric")
    String username,

    @Min(value = 18, message = "Must be at least 18")
    @Max(value = 120, message = "Must be at most 120")
    Integer age
) {}

public record UserUpdateRequest(
    @Email(message = "Email must be valid")
    String email,

    @Size(min = 3, max = 50)
    String username
) {}
```

Bean Validation (`jakarta.validation.*`) is a Jakarta EE standard; Quarkus needs the `quarkus-hibernate-validator` extension on the classpath to process these annotations.

## Response DTOs

```java
public record UserResponse(
    Long id,
    String email,
    String username,
    Integer age,
    Boolean active,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {
    public static UserResponse from(User user) {
        return new UserResponse(
            user.id,
            user.email,
            user.username,
            user.age,
            user.active,
            user.createdAt,
            user.updatedAt
        );
    }
}
```

## Global Exception Handling

There is no single global exception-handling class in JAX-RS. Instead, one or more `@Provider`-annotated classes implement `ExceptionMapper<T>`, each bound to a specific exception type:

```java
@Provider
public class ResourceNotFoundExceptionMapper implements ExceptionMapper<ResourceNotFoundException> {

    private static final Logger log = LoggerFactory.getLogger(ResourceNotFoundExceptionMapper.class);

    @Override
    public Response toResponse(ResourceNotFoundException ex) {
        log.error("Resource not found: {}", ex.getMessage());
        var error = new ErrorResponse(404, ex.getMessage(), LocalDateTime.now());
        return Response.status(404).entity(error).build();
    }
}

@Provider
public class ValidationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException ex) {
        Map<String, String> errors = ex.getConstraintViolations().stream()
            .collect(Collectors.toMap(
                v -> v.getPropertyPath().toString(),
                v -> v.getMessage() != null ? v.getMessage() : "Invalid value"));

        var response = new ValidationErrorResponse(400, "Validation failed", errors, LocalDateTime.now());
        return Response.status(400).entity(response).build();
    }
}

@Provider
public class DataIntegrityExceptionMapper implements ExceptionMapper<PersistenceException> {

    private static final Logger log = LoggerFactory.getLogger(DataIntegrityExceptionMapper.class);

    @Override
    public Response toResponse(PersistenceException ex) {
        log.error("Data integrity violation", ex);
        var error = new ErrorResponse(409, "Data integrity violation - resource may already exist", LocalDateTime.now());
        return Response.status(409).entity(error).build();
    }
}

@Provider
public class GlobalExceptionMapper implements ExceptionMapper<Exception> {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionMapper.class);

    @Override
    public Response toResponse(Exception ex) {
        log.error("Unexpected error", ex);
        var error = new ErrorResponse(500, "An unexpected error occurred", LocalDateTime.now());
        return Response.status(500).entity(error).build();
    }
}

record ErrorResponse(int status, String message, LocalDateTime timestamp) {}

record ValidationErrorResponse(int status, String message, Map<String, String> errors, LocalDateTime timestamp) {}
```

Quarkus/RESTEasy resolves the **most specific** mapper for the thrown exception's type, falling back to a broader one (e.g. `Exception`) when no exact or subtype match exists — so registering several narrow mappers alongside one catch-all `Exception` mapper is the idiomatic way to centralize exception handling.

## Custom Validation

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@ApplicationScoped
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository userRepository;

    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) return true;
        return userRepository.find("email", email).firstResultOptional().isEmpty();
    }
}
```

Custom Bean Validation constraints are, again, a Jakarta standard — only the bean's scope annotation changes (`@ApplicationScoped` instead of `@Component`), and CDI constructor injection works inside a `ConstraintValidator` exactly like in any other bean.

## REST Client for External APIs

```java
@RegisterRestClient(configKey = "external-api")
public interface ExternalApiClient {

    @GET
    @Path("/data/{id}")
    ExternalDataResponse fetchData(@PathParam("id") String id);
}
```

```properties
# application.properties
quarkus.rest-client.external-api.url=https://api.example.com
quarkus.rest-client.external-api.scope=jakarta.inject.Singleton
quarkus.rest-client.logging.scope=request-response
quarkus.rest-client.logging.body-limit=1024
```

```java
@ApplicationScoped
public class ExternalApiService {

    @RestClient
    ExternalApiClient client;

    public ExternalDataResponse fetchData(String id) {
        try {
            return client.fetchData(id);
        } catch (WebApplicationException e) {
            int status = e.getResponse().getStatus();
            if (status == 404) {
                throw new ResourceNotFoundException("External resource not found");
            }
            if (status >= 500) {
                throw new ServiceUnavailableException("External service unavailable");
            }
            throw e;
        }
    }
}
```

`@RegisterRestClient` + `@RestClient` is Quarkus's single, consistent mechanism for declarative HTTP clients — no separate `@Configuration` class to write; the base URL and logging come straight from `application.properties`.

## CORS Configuration

```properties
# application.properties - no Java @Configuration class needed
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:3000,https://example.com
quarkus.http.cors.methods=GET,POST,PUT,DELETE,OPTIONS
quarkus.http.cors.headers=*
quarkus.http.cors.access-control-allow-credentials=true
quarkus.http.cors.access-control-max-age=24H
```

## Problem Details (RFC 7807)-style Responses

Quarkus has no single switch that turns every exception into an RFC 7807 body automatically. Build the shape explicitly in the exception mappers above:

```java
public record ProblemDetail(
    String type,
    String title,
    int status,
    String detail,
    String instance
) {
    public static ProblemDetail of(int status, String title, String detail, String instance) {
        return new ProblemDetail("about:blank", title, status, detail, instance);
    }
}
```

```java
@Provider
public class UserNotFoundExceptionMapper implements ExceptionMapper<UserNotFoundException> {

    @Context
    UriInfo uriInfo;

    @Override
    public Response toResponse(UserNotFoundException ex) {
        var problem = ProblemDetail.of(404, "User Not Found", ex.getMessage(),
            uriInfo.getPath());
        return Response.status(404)
            .type("application/problem+json")
            .entity(problem)
            .build();
    }
}
```

## OpenAPI & Swagger UI

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-openapi</artifactId>
</dependency>
```

The spec is generated at build time from the JAX-RS resources and served at `/q/openapi`; Swagger UI is at `/q/swagger-ui` in dev and test mode only. Paths, parameters, and record schemas are inferred, so annotate only what can't be (summaries, tags, error responses):

```java
@Path("/api/users")
@Tag(name = "Users", description = "User management endpoints")
public class UserResource {
    private final UserService userService;

    public UserResource(UserService userService) {
        this.userService = userService;
    }

    @GET
    @Path("/{id}")
    @Operation(summary = "Get user by ID")
    @APIResponse(responseCode = "200", description = "User found",
        content = @Content(mediaType = "application/json",
            schema = @Schema(implementation = UserResponse.class)))
    @APIResponse(responseCode = "404", description = "User not found",
        content = @Content(mediaType = "application/problem+json"))
    public UserResponse getUser(@PathParam("id") Long id) {
        return userService.findById(id);
    }
}
```

```properties
quarkus.smallrye-openapi.info-title=User API
quarkus.smallrye-openapi.info-version=1.0.0
# Write the spec to the build output, e.g. to diff it in CI as an API-contract check
quarkus.smallrye-openapi.store-schema-directory=target/openapi
# Only if Swagger UI must be served in prod (and then behind auth)
# quarkus.swagger-ui.always-include=true
```

The annotations are the MicroProfile OpenAPI ones (`org.eclipse.microprofile.openapi.annotations.*`).

## Quick Reference

| Annotation | Purpose |
|------------|---------|
| `@Path` | Maps a class/method to a URI |
| `@GET`/`@POST`/`@PUT`/`@DELETE`/`@PATCH` | HTTP method-specific mappings |
| `@PathParam` | Extracts values from the URI path |
| `@QueryParam` | Extracts query parameters |
| *(unannotated parameter)* | Binds the request body — no `@RequestBody` needed |
| `@Valid` | Triggers Bean Validation on a parameter |
| `@Provider` + `ExceptionMapper<T>` | Exception-type-specific response mapping (no global advice class) |
| `@Status` | Sets the HTTP status code for a resource method |
| `@Tag` / `@Operation` / `@APIResponse` | Enrich the generated OpenAPI spec (`quarkus-smallrye-openapi`) |
