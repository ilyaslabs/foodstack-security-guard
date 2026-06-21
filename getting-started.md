# Getting Started with security-guard

`security-guard` is a Spring Boot library for protecting service-to-service HTTP calls in a microservice environment. It
wires Spring Security with a lightweight header-based authentication model so your services can:

- create a request-scoped `AuthenticationContext`
- expose caller scopes as Spring authorities
- block endpoints annotated with `@Secured` from external gateway traffic
- return consistent `401` and `403` responses through Spring Security

The project is designed for internal microservice boundaries, not as a full identity provider or JWT verification layer.
It trusts upstream infrastructure to set the expected headers correctly.

## What the library does

When `@EnableMicroserviceSecurity` is enabled, the library registers:

| Component                       | Purpose                                                                          |
|---------------------------------|----------------------------------------------------------------------------------|
| `CustomAuthenticationWebFilter` | Reads internal security headers and populates Spring Security's context          |
| `AuthenticationContext`         | Stores `userId`, `authorities`, and whether the request came through the gateway |
| `@Secured`                      | Restricts endpoints to internal service calls only                               |
| `AuthenticationContextProvider` | Gives application code access to the current `AuthenticationContext`             |
| `SecurityControllerAdvice`      | Converts access denials into a structured `403 Forbidden` response               |

## Requirements

- Java 25
- Maven
- A Spring Boot Web MVC application

## Add the dependency

Add the library to your service:

```xml

<dependency>
    <groupId>io.github.ilyaslabs</groupId>
    <artifactId>security-guard</artifactId>
    <version><!-- use the released version --></version>
</dependency>
```

The module already brings in Spring Security support, Web MVC integration, and BSON support for `ObjectId` user
identifiers.

## Enable microservice security

Annotate your Spring Boot application or configuration class:

```java
import io.github.ilyaslabs.microservice.security.guard.annotation.EnableMicroserviceSecurity;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableMicroserviceSecurity
public class MyApplication {
}
```

This imports `HttpSecurityConfigurer`, which:

- disables CSRF, CORS, form login, and HTTP Basic
- sets stateless session management
- installs the custom authentication filter before `UsernamePasswordAuthenticationFilter`
- enables method security for `@Secured` and other Spring authorization annotations

## Request headers

The filter reads three headers from incoming requests:

| Header          | Meaning                       | Notes                                                 |
|-----------------|-------------------------------|-------------------------------------------------------|
| `X-USER-ID`     | Current user id               | Must be a valid MongoDB `ObjectId` to be stored       |
| `X-SCOPES`      | Space-delimited scopes        | Converted into Spring `SimpleGrantedAuthority` values |
| `X-API-GATEWAY` | Marks the request as external | If present, the request is treated as a gateway call  |

### Internal vs external calls

`security-guard` uses a simple rule:

- if `X-API-GATEWAY` is present, the call is **external**
- if `X-API-GATEWAY` is absent, the call is **internal**

That rule matters for `@Secured`. A secured endpoint only allows internal calls.

## Protect internal-only endpoints

Use `@Secured` on a controller class or handler method:

```java
import io.github.ilyaslabs.microservice.security.guard.annotation.Secured;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class InternalController {

    @Secured
    @GetMapping("/internal/reindex")
    public String reindex() {
        return "ok";
    }
}
```

Behavior:

- internal service call -> allowed
- gateway/external call -> rejected with `403 Forbidden`

For role- or scope-based access, use standard Spring annotations such as `@PreAuthorize("hasAuthority('admin')")`.
`X-SCOPES` is mapped directly into authorities.

## Read the authentication context in application code

Inject `AuthenticationContextProvider` anywhere you need request identity information:

```java
import io.github.ilyaslabs.microservice.security.guard.AuthenticationContextProvider;
import io.github.ilyaslabs.microservice.security.guard.model.AuthenticationContext;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class ProfileController {

    private final AuthenticationContextProvider authenticationContextProvider;

    ProfileController(AuthenticationContextProvider authenticationContextProvider) {
        this.authenticationContextProvider = authenticationContextProvider;
    }

    @GetMapping("/me")
    public AuthenticationContext me() {
        return authenticationContextProvider.current();
    }
}
```

`AuthenticationContextProvider.current()` throws if no authentication context is available. Use `currentOptional()` when
the request may be anonymous or unauthenticated.

## Typical request flow

1. Upstream infrastructure forwards request metadata using `X-USER-ID`, `X-SCOPES`, and optionally `X-API-GATEWAY`.
2. `CustomAuthenticationWebFilter` creates an `AuthenticationContext`.
3. Spring Security stores that context as the authenticated principal.
4. Your controllers and services use `@Secured`, `@PreAuthorize`, or `AuthenticationContextProvider`.

## Local development

Build and run the tests with Maven:

```bash
mvn test
```

Package the jar:

```bash
mvn package
```

The current test suite covers:

- `@Secured` blocking external gateway traffic
- `@Secured` allowing internal calls
- authority mapping from `X-SCOPES`
- `AuthenticationContext` population from headers

## Developer notes

- `security-guard` is intentionally stateless.
- The filter does not authenticate against a database or user store.
- The bundled `UserDetailsService` is disabled by design.
- `X-SCOPES` should contain space-separated values such as `read write admin`.
- If `X-USER-ID` is missing or invalid, the request can still proceed, but `AuthenticationContext.userId()` will be
  `null`.
- Only add `X-API-GATEWAY` when the request really originated from the edge, because its presence flips the request into
  the external-call path.
