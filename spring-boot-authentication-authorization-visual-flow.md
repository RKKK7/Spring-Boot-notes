# Spring Boot Authentication & Authorization — Visualizing What Actually Happens

> These notes are intentionally **flow-first, visual, and explanation-driven**.
>
> The goal is not to memorize `AuthenticationManager`, `AuthenticationProvider`, `SecurityContextHolder`, JWT, or other Spring Security terms in isolation. The goal is to build a mental movie of what actually happens when a user registers, logs in, receives authentication state, sends a protected request, and reaches the business logic.
>
> **These notes also follow the failure paths:** wrong passwords, missing users, expired JWTs, invalid signatures, failed authorization, and the HTTP responses produced by those failures. Real authentication is not only about the happy path.

Spring Security is often taught as a list of classes, annotations, and configuration snippets. That approach can make the framework feel like a collection of disconnected pieces. These notes take the opposite approach: **start with the problem, follow the request, introduce each component exactly when it becomes necessary, and visualize the state moving through the system.**

Throughout the notes, we will build the complete story from beginning to end:

```text
REGISTRATION
     |
     v
Password hashing + user storage
     |
     v
LOGIN
     |
     v
AuthenticationManager
     |
     v
AuthenticationProvider
     |
     v
UserDetailsService + PasswordEncoder
     |
     v
Authentication succeeds
     |
     v
JWT generation
     |
     v
CLIENT RECEIVES ACCESS TOKEN
     |
     v
PROTECTED REQUEST
     |
     v
JWT VALIDATION
     |
     +---- invalid / expired / bad signature --> REJECT
     |
     v
AUTHENTICATION + SECURITY CONTEXT
     |
     v
AUTHORIZATION
     |
     +---- denied --> 403
     |
     v
BUSINESS LOGIC
     |
     v
REFRESH TOKEN / LOGOUT / TOKEN LIFECYCLE
     |
     v
JWT validation
     |
     v
Identity restored
     |
     v
SecurityContext
     |
     v
Authorization
     |
     v
Controller → Service → Repository
```

We will also first understand the traditional **session + cookie** model, because it makes the motivation for stateless JWT authentication much clearer. Along the way, we will connect Servlet filters, Spring Security's filter chain, authentication, authorization, roles and authorities, BCrypt, CSRF, JWT structure, signing and validation, and the `SecurityContext`.

### What makes these notes different

- **Flow before terminology** — every important class is introduced inside the flow where it actually participates.
- **Visualization before memorization** — diagrams show what is happening before the implementation details are discussed.
- **Complete lifecycle** — registration, password storage, login, authentication, JWT creation, protected requests, JWT validation, identity restoration, and authorization are treated as one connected system.
- **Under-the-hood thinking** — the focus is on what the framework is doing behind the annotations and abstractions we normally use.
- **Concepts connected to one another** — instead of learning `AuthenticationManager`, `AuthenticationProvider`, `UserDetailsService`, `Authentication`, and `SecurityContext` as separate definitions, you will see how one leads to the next.
- **Interview-ready mental models** — the final sections compress the detailed flows into diagrams and questions you can use for revision.

The objective is simple:

> **Do not just know how to configure Spring Security. Understand what actually happens from the moment credentials enter the application to the moment an authenticated request is allowed to execute business logic.**

---

## Table of Contents

1. [Security Foundations — From HTTP Request to Identity](#security-foundations--from-http-request-to-identity)
2. [Before JWT — Registration, Sessions, Cookies, Scaling and CSRF](#before-jwt--registration-sessions-cookies-scaling-and-csrf)
3. [Spring Security Under the Hood — Filters, Authentication and Authorization](#spring-security-under-the-hood--filters-authentication-and-authorization)
4. [JWT Authentication — Generation, Validation and Protected Requests](#jwt-authentication--generation-validation-and-protected-requests)
5. [Putting Everything Together — Common Doubts and Session vs JWT](#putting-everything-together--common-doubts-and-session-vs-jwt)
6. [Final Mental Model and Interview Revision](#final-mental-model-and-interview-revision)

---

# Security Foundations — From HTTP Request to Identity

## Where Security Fits

The starting question is:

> **How does an HTTP request physically move through a Spring Boot application?**

We built this model:

```text
Browser / Client
       |
       v
    Network
       |
       v
     Tomcat
       |
       v
Servlet infrastructure
       |
       v
Servlet Filter Chain
       |
       v
DispatcherServlet
       |
       v
HandlerMapping
       |
       v
HandlerAdapter
       |
       v
Controller
       |
       v
Service
       |
       v
Repository
       |
       v
Database
```

Now we introduce security.

The question becomes:

> **Who is making this request, and is that user allowed to perform this operation?**

So security sits inside the request pipeline:

```text
Browser / Client
       |
       | HTTP request
       v
     Tomcat
       |
       v
Servlet Filter Chain
       |
       v
+----------------------+
|   SPRING SECURITY    |
|                      |
| Authentication       |
| Identity             |
| Authorities          |
| Authorization        |
+----------------------+
       |
       v
DispatcherServlet
       |
       v
Controller
```

### The key separation

```text
INFRASTRUCTURE
    |
    +--> How does the request move?

SECURITY
    |
    +--> Who is the requester?
    |
    +--> What is the requester allowed to do?
```

Security does **not** replace Tomcat, Servlet infrastructure, or Spring MVC. It plugs into the request pipeline.

---

## The Problem Security Solves

Consider:

```http
GET /api/notes
```

The HTTP request tells the server what operation was requested. It does not automatically tell the application which user is making that request.

So the application needs to answer two questions:

```text
                 SECURITY
                    |
           +--------+--------+
           |                 |
           v                 v
   AUTHENTICATION      AUTHORIZATION
           |                 |
           v                 v
      "Who are you?"   "What can you do?"
```

Everything we study in Spring Security eventually supports one or both of these questions.

---

## Authentication vs Authorization

## Authentication

Authentication establishes identity.

```text
Credentials / Token
        |
        v
     Server
        |
        v
  "Who is this?"
        |
        v
       Ravi
```

## Authorization

Authorization checks permission.

```text
       Ravi
         |
         | wants to DELETE /admin/users/10
         v
   Authorization
         |
    +----+----+
    |         |
  allow      deny
```

Example:

```text
Authentication
    |
    v
User = ravi
    |
    v
Authorization
    |
    v
Does ravi have ADMIN authority?
```

### Memory trick

```text
Authentication = WHO?
Authorization   = WHAT CAN THEY DO?
```

---

## The Bridge: Servlet Filters → Spring Security

A Servlet filter is part of the request pipeline:

```text
Request
   |
   v
Filter 1
   |
   v
Filter 2
   |
   v
Filter 3
   |
   v
DispatcherServlet
```

Spring Security uses the same filtering mechanism.

```text
              REQUEST PIPELINE
                    |
                    v
            Servlet Filter Chain
                    |
          +---------+---------+
          |                   |
          v                   v
  General Filters       Security Filters
                              |
                              v
                       Spring Security
```

For example, a `JwtAuthenticationFilter` is a security-purpose filter participating in the Servlet filter pipeline.

This is the bridge between the HTTP infrastructure and security:

```text
INFRASTRUCTURE
      |
      v
Servlet Filter Chain
      |
      v
SPRING SECURITY
```

This explains why filters are central to both HTTP processing and Spring Security, even though they solve different problems.

---

### Visual checkpoint — the security problem

```text
                         HTTP REQUEST
                              |
                              v
                    Servlet Filter Chain
                              |
                              v
                    +------------------+
                    | Spring Security  |
                    +------------------+
                         /         \
                        /           \
                       v             v
              AUTHENTICATION    AUTHORIZATION
                  |                  |
                  v                  v
              WHO IS IT?       WHAT CAN THEY DO?
```

Everything that follows is an increasingly detailed answer to these two questions.

# Before JWT — Registration, Sessions, Cookies, Scaling and CSRF

## Before JWT: Session-Based Authentication

Before learning JWT, first understand the traditional question:

> **After a user logs in successfully, how does the server remember that the user is authenticated on the next request?**

A traditional answer is a server-side session.

High-level flow:

```text
LOGIN
  |
  v
username + password
  |
  v
Server verifies credentials
  |
  v
Authentication succeeds
  |
  v
Server creates session
  |
  v
Session ID
  |
  v
Browser receives session ID
```

This is the foundation for understanding why JWT changes the architecture.

---

## Registration

Before authentication can happen, the application needs a user account.

For example:

```http
POST /api/auth/register
```

```json
{
  "username": "ravi",
  "password": "secret"
}
```

The request travels through the application:

```text
Client
  |
  v
Tomcat
  |
  v
Servlet Filter Chain
  |
  v
DispatcherServlet
  |
  v
AuthController (or equivalent authentication endpoint)
  |
  v
Authentication service
  |
  +--> Validate input
  |
  +--> Check username
  |
  +--> Hash password
  |
  v
User repository
  |
  v
Database
```

The security-critical part is password storage.

We do **not** want:

```text
ravi | secret
```

in the database.

We want a password hash.

---

## Password Storage and BCrypt

Registration transforms a plain password into a stored password hash:

```text
Plain password
      |
      v
BCryptPasswordEncoder
      |
      v
Password hash
      |
      v
Database
```

Later, during login:

```text
Submitted password
       |
       v
PasswordEncoder
       |
       v
Compare against stored hash
       |
       v
Match?
```

BCrypt is not decrypted.

There is no normal operation like:

```text
BCrypt hash -> original password
```

Instead, the submitted password is verified against the stored hash.

---

## Login with a Session

Suppose Ravi sends:

```http
POST /api/auth/login
```

```json
{
  "username": "ravi",
  "password": "myPassword123"
}
```

After successful verification:

```text
username + password
        |
        v
   Verify credentials
        |
        v
     SUCCESS
        |
        v
  Create server session
        |
        v
  Session ID = abc123
        |
        v
 Set-Cookie: JSESSIONID=abc123
        |
        v
      Browser
```

The browser now has a session identifier.

---

## What a Session Actually Is

A session is server-side state associated with a session identifier.

Imagine:

```text
SERVER SESSION STORE
-----------------------------
abc123  ->  ravi
xyz789  ->  admin
```

The browser may only carry:

```text
JSESSIONID=abc123
```

The server uses that ID to find:

```text
abc123
  |
  v
ravi
```

So:

```text
Browser
  |
  | session ID
  v
Server
  |
  v
Session Store
  |
  v
Authenticated User
```

### Important distinction

```text
Cookie
  = browser mechanism for storing/sending data

Session
  = server-side state associated with a session ID
```

A cookie and a session are related, but they are not the same thing.

---

## Cookies

The server can ask the browser to store a cookie:

```http
Set-Cookie: JSESSIONID=abc123
```

For a later applicable request, the browser can send:

```http
Cookie: JSESSIONID=abc123
```

Visualize:

```text
LOGIN RESPONSE
      |
      v
Set-Cookie
      |
      v
Browser stores cookie
      |
      v
NEXT REQUEST
      |
      v
Browser sends cookie
```

This automatic browser behavior is extremely useful for session authentication.

It also creates an important security consideration: **CSRF**.

---

## Session + Cookie: Complete Flow

First request:

```text
                    LOGIN
                      |
                      v
             username + password
                      |
                      v
                    Server
                      |
                      v
             Verify credentials
                      |
                      v
                   SUCCESS
                      |
                      v
                Create session
                      |
                      v
             Session ID = abc123
                      |
                      v
          Set-Cookie: JSESSIONID=abc123
                      |
                      v
                   Browser
```

Second request:

```text
                  GET /api/notes
                        |
                        v
             Cookie: JSESSIONID=abc123
                        |
                        v
                      Server
                        |
                        v
               Find session abc123
                        |
                        v
                 User = ravi
                        |
                        v
                 Authorization
                        |
                        v
                   Controller
```

The server does not need Ravi to send his password again.

---

## The Next Request

Suppose:

```text
abc123 -> ravi
```

exists in the session store.

Ravi sends:

```http
GET /api/notes
Cookie: JSESSIONID=abc123
```

The request becomes:

```text
Browser
   |
   v
Tomcat
   |
   v
Servlet Filter Chain
   |
   v
Session authentication
   |
   v
Find session abc123
   |
   v
Authenticated user = ravi
   |
   v
Authorization
   |
   v
DispatcherServlet
   |
   v
Controller
```

This is the core session mental model:

> **The client sends an identifier; the server uses it to recover server-side authentication state.**

---

## Session State and Scaling

A traditional session model requires the server side to remember session state.

For many users:

```text
Session Store
--------------------------------
session-001 -> user-1
session-002 -> user-2
session-003 -> user-3
...
session-10000 -> user-10000
```

Now consider two application instances:

```text
                 Load Balancer
                       |
                 +-----+-----+
                 |           |
                 v           v
             Server A    Server B
```

Ravi logs in through Server A:

```text
Ravi
 |
 v
Server A
 |
 v
Session abc123
 |
 v
Server A's session state
```

Then the next request reaches Server B:

```text
Ravi
 |
 v
Load Balancer
 |
 v
Server B
 |
 v
Where is abc123?
```

Architectures can solve this with shared session stores, Redis-backed sessions, database-backed sessions, sticky sessions, and other approaches.

The point is not that sessions cannot scale. They can.

The point is:

> **Session authentication introduces server-side state that has to be managed as the system scales.**

That gives us the motivation to understand stateless authentication.

---

## CSRF

CSRF means **Cross-Site Request Forgery**.

The important idea is:

> A malicious website attempts to cause a victim's authenticated browser to send an unwanted request to another website.

The classic concern appears when authentication credentials are automatically attached by the browser, such as cookies.

---

## Why Cookies Create a CSRF Concern

Suppose Ravi is logged into:

```text
bank.example
```

His browser has:

```text
Cookie: session=abc123
```

Ravi then visits a malicious site.

Conceptually:

```text
                 Ravi's Browser
                       |
             +---------+---------+
             |                   |
             v                   v
       bank.example        evil.example
             ^                   |
             |                   |
             +-------------------+
                 unwanted request
```

The important part is that the browser may automatically attach the applicable authentication cookie to the request to the target site.

The attacker does not necessarily need to know the cookie value.

The danger is:

```text
Browser automatically attaches credential
                |
                v
Server sees valid session
                |
                v
Server may think the request is from Ravi
```

---

## CSRF Attack Visualization

Imagine a state-changing operation:

```http
POST /api/change-email
Cookie: JSESSIONID=abc123
```

Normal request:

```text
Ravi's browser
      |
      v
Your application
      |
      v
Session abc123
      |
      v
Ravi
      |
      v
Change email
```

CSRF attempt:

```text
Attacker
   |
   v
Malicious page
   |
   v
Ravi's Browser
   |
   | applicable auth cookie may be attached
   v
Your application
   |
   v
Session abc123
   |
   v
Server sees Ravi's authenticated session
```

That is why CSRF protection exists.

The precise problem is **not** simply "cookies are insecure." The problem is automatic credential attachment combined with an attacker's ability to cause a browser to make a request.

---

## CSRF Protection

One common defense is a CSRF token.

Conceptually:

```text
Server
  |
  | CSRF token
  v
Browser
  |
  | state-changing request
  | + CSRF token
  v
Server
  |
  v
Verify token
```

Now the server can require both:

```text
Authentication credential
        +
CSRF token
        |
        v
Valid request
```

Spring Security provides CSRF protection mechanisms for architectures where they are appropriate.

### Important architectural point

A browser application using cookie-based authentication has a different CSRF threat model from a stateless API where a client explicitly sends a bearer token in an `Authorization` header.

That does **not** mean JWT makes an application automatically secure. XSS, token theft, weak secrets, missing HTTPS, broken authorization, insecure storage, and other threats still matter.

---

## Other Security Concerns

Once we understand authentication, we can see why application security is broader than login.

Important areas include:

```text
Authentication
Authorization
Password storage
Sessions
Cookies
CSRF
XSS
CORS
HTTPS
Token integrity
Token expiration
Secret management
Input validation
Database security
Dependency security
```

These notes focus on the authentication/authorization flow, but the broader lesson is:

> **A correct login mechanism is only one part of application security.**

---

### Visual checkpoint — session authentication

```text
REGISTER
   |
   v
Hash password
   |
   v
Store user

LOGIN
   |
   v
Verify password
   |
   v
Create server-side session
   |
   v
Send session ID in cookie
   |
   v
NEXT REQUEST
   |
   v
Cookie → session lookup → authenticated identity
   |
   v
Authorization → Controller
```

The key idea is **server-side authentication state**. JWT will change where that state is represented.

# Spring Security Under the Hood — Filters, Authentication and Authorization

## Why Spring Security

Without a security framework, developers could end up doing this in every controller:

```java
if (!authenticated) {
    // reject
}

if (!hasRole("ADMIN")) {
    // reject
}

// business logic
```

Then the same checks get repeated across controllers.

A better architecture is:

```text
HTTP Request
     |
     v
Security layer
     |
     +--> Authentication
     +--> Authorization
     +--> Security context
     |
     v
Application logic
```

Spring Security provides the infrastructure and abstractions for this security layer.

---

## Spring Security in the Request Pipeline

The full pipeline now becomes:

```text
Browser
   |
   v
Tomcat
   |
   v
Servlet Filter Chain
   |
   v
+----------------------+
|   SPRING SECURITY    |
|                      |
| Authentication       |
| Authorization        |
| SecurityContext      |
+----------------------+
   |
   v
DispatcherServlet
   |
   v
Controller
```

This is the main bridge between the Servlet infrastructure and Spring Security.

Security commonly gets the opportunity to process the request **before protected controller logic runs**.

---

## DelegatingFilterProxy and FilterChainProxy

Spring Security has infrastructure connecting the Servlet filter world to Spring-managed security components.

Conceptually:

```text
Tomcat
  |
  v
Servlet Filter mechanism
  |
  v
DelegatingFilterProxy
  |
  v
FilterChainProxy
  |
  v
SecurityFilterChain
  |
  v
Security filters
```

### `DelegatingFilterProxy`

Think:

> **A bridge from the Servlet container's filtering mechanism into Spring-managed infrastructure.**

### `FilterChainProxy`

Think:

> **The Spring Security component that routes a request through the appropriate security filters.**

Do not memorize these as isolated definitions. Visualize the bridge:

```text
Servlet world
     |
     v
DelegatingFilterProxy
     |
     v
Spring Security world
     |
     v
FilterChainProxy
     |
     v
SecurityFilterChain
```

---

## SecurityFilterChain

A `SecurityFilterChain` represents the security filters and rules applied to requests.

Conceptually:

```text
HTTP Request
     |
     v
SecurityFilterChain
     |
     +--> Security filter
     |
     +--> Security filter
     |
     +--> JwtAuthenticationFilter
     |
     +--> Security filter
     |
     v
Next stage
```

A simplified configuration can look like:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http)
        throws Exception {

    return http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/auth/**")
                    .permitAll()
                    .anyRequest()
                    .authenticated()
            )
            .build();
}
```

Read the configuration as rules:

```text
/api/auth/**
     |
     v
public / permitted

Everything else
     |
     v
authentication required
```

The syntax can vary by Spring Security version. The important part is the request-processing model.

---

## Authentication Flow

Now we can introduce the Spring Security authentication architecture **because we have already seen the problem it is solving**.

Suppose the user sends username/password.

```text
Username + Password
        |
        v
AuthenticationManager
        |
        v
AuthenticationProvider
        |
        v
UserDetailsService
        |
        v
Database
```

But we need to understand every box in context.

---

## AuthenticationManager

The `AuthenticationManager` is the main entry point/coordinator for an authentication attempt.

When code does:

```java
authenticationManager.authenticate(authentication)
```

the mental translation is:

> **"Spring Security, authenticate this authentication attempt."**

Flow:

```text
Authentication request
        |
        v
AuthenticationManager
        |
        v
Delegates to suitable provider
```

It is not the database and it is not the password encoder.

It coordinates authentication.

---

## AuthenticationProvider

An `AuthenticationProvider` performs a particular authentication mechanism.

The relationship is:

```text
AuthenticationManager
        |
        v
AuthenticationProvider
        |
        v
Authentication strategy
```

For username/password authentication in a typical application:

```text
AuthenticationManager
        |
        v
DaoAuthenticationProvider
```

Think:

```text
Manager = coordinates
Provider = performs a supported authentication strategy
```

---

## DaoAuthenticationProvider

Now the username/password flow becomes concrete:

```text
Username + Password
        |
        v
AuthenticationManager
        |
        v
DaoAuthenticationProvider
        |
        +-------------------+
        |                   |
        v                   v
UserDetailsService    PasswordEncoder
        |                   |
        v                   |
Database                    |
        |                   |
        +---------+---------+
                  |
                  v
            Password match?
              /       \
            YES        NO
             |          |
             v          v
       Authentication  Failure
          succeeds
```

### Read the flow left-to-right

1. Credentials enter the authentication system.
2. `AuthenticationManager` delegates to `DaoAuthenticationProvider`.
3. The provider asks `UserDetailsService` for the user.
4. The user is loaded from the database.
5. The provider uses `PasswordEncoder` to compare the submitted password with the stored hash.
6. Success produces an authenticated result; failure rejects the attempt.

This is the important flow. The class definitions only make sense after this picture is clear.

---

## UserDetailsService

The `UserDetailsService` exists in the flow because the provider needs user security information.

```text
DaoAuthenticationProvider
        |
        | "Give me security information for this username."
        v
UserDetailsService
        |
        v
User repository
        |
        v
Database
```

Its central method is:

```java
UserDetails loadUserByUsername(String username)
```

Its primary job is to **load** the user's security information.

It is not itself the password comparison component.

---

## UserDetails

Spring Security uses `UserDetails` as a standard representation of security-related user information.

Visualize the conversion:

```text
Your database
      |
      v
User entity
      |
      v
UserDetailsService
      |
      v
UserDetails
      |
      v
Spring Security
```

`UserDetails` can provide:

```text
username
password hash
authorities
account status
```

Your application `User` entity can contain many more domain fields. `UserDetails` is the security-facing representation.

---

## PasswordEncoder in the Login Flow

Now connect BCrypt to the actual provider flow.

```text
Client
  |
  | password = "secret"
  v
DaoAuthenticationProvider
  |
  +---------------------------+
  |                           |
  v                           v
UserDetailsService       PasswordEncoder
  |                           |
  v                           |
Stored BCrypt hash            |
  |                           |
  +-------------+-------------+
                |
                v
          matches(password, hash)
                |
             +--+--+
             |     |
            YES    NO
             |     |
             v     v
          success failure
```

Again:

> The stored BCrypt value is not decrypted. The submitted password is checked against it.

---

## UsernamePasswordAuthenticationToken

During login, your code creates:

```java
new UsernamePasswordAuthenticationToken(
        request.getUsername(),
        request.getPassword()
)
```

Visualize:

```text
username + password
        |
        v
UsernamePasswordAuthenticationToken
        |
        v
AuthenticationManager
```

At this point the token represents an **authentication attempt**.

It means:

> "Here are the credentials I want Spring Security to authenticate."

It does not mean that the credentials have already been proven valid.

Later, an authenticated token can represent:

```text
UserDetails
Authorities
Authenticated = true
```

That distinction becomes very important when we reach JWT.

---

## The Different “Tokens” You Will Encounter

The word **token** is overloaded in authentication discussions. These are not interchangeable objects.

```text
                 AUTHENTICATION-RELATED VALUES
                              |
       +----------------------+-----------------------+
       |                      |                       |
       v                      v                       v
Session ID       UsernamePasswordAuthenticationToken       JWT
   |                         |                              |
server-side state      Spring Security object          signed credential
   |                         |                              |
   v                         v                              v
JSESSIONID            login credentials / identity       access token

Other common values:

CSRF token    -> anti-CSRF value used to prove a browser request is expected
Refresh token -> credential used to obtain a new access token
```

### Session ID

A session ID such as `JSESSIONID=abc123` is usually an identifier. The server uses it to locate server-side session state.

### `UsernamePasswordAuthenticationToken`

Despite its name, this is primarily a **Spring Security `Authentication` implementation**, not a JWT and not a network token by itself. It can represent submitted username/password credentials during login and can also represent an authenticated principal after verification.

### Access token / JWT

A JWT access token is a signed credential sent with protected requests, commonly through:

```http
Authorization: Bearer <access-token>
```

### Refresh token

A refresh token is a longer-lived credential used at a token endpoint to obtain a new access token. It is not normally sent to every API endpoint.

### CSRF token

A CSRF token has a different purpose: it helps the server determine whether a state-changing browser request is expected rather than forged by another site.

> **Not every object called a token is a JWT, and not every token is an authentication credential.**

---

## Authentication

`Authentication` is the Spring Security abstraction representing security identity and authentication state.

After successful authentication, conceptually:

```text
Authentication
-------------------------
Principal: ravi
Authorities: ROLE_USER
Authenticated: true
```

This gives security code answers to:

```text
Who?
  -> ravi

What authorities?
  -> ROLE_USER

Authenticated?
  -> true
```

---

## SecurityContext

Now we have an `Authentication` object.

Where does the current security identity live during processing?

Inside the `SecurityContext`.

```text
SecurityContext
      |
      v
Authentication
      |
      +--> Principal
      +--> Authorities
      +--> Authenticated
```

The important mental model is:

```text
Authentication
      |
      v
SecurityContext
```

---

## SecurityContextHolder

How does code access the current security context?

Through `SecurityContextHolder`.

```text
SecurityContextHolder
        |
        v
SecurityContext
        |
        v
Authentication
        |
        v
Current authenticated user
```

For example:

```java
Authentication authentication =
        SecurityContextHolder
                .getContext()
                .getAuthentication();
```

Then application/security code can inspect the established identity and authorities.

### The complete chain

```text
Credentials
    |
    v
AuthenticationManager
    |
    v
AuthenticationProvider
    |
    v
Authentication
    |
    v
SecurityContext
    |
    v
SecurityContextHolder
```

---

## Authorization

Authentication established:

```text
User = Ravi
```

Now Ravi requests:

```http
GET /api/notes
```

Authorization asks:

```text
Is Ravi allowed to access this endpoint/resource?
```

Flow:

```text
Authentication
      |
      v
User = ravi
      |
      v
Authorities
      |
      v
Authorization rule
      |
   +--+--+
   |     |
 allow  deny
```

This is why authentication and authorization are sequential but distinct.

---

## Roles and Authorities

Suppose Ravi has:

```text
ROLE_USER
```

An admin has:

```text
ROLE_ADMIN
```

Authorization can use these authorities to decide access.

```text
Authenticated user
        |
        v
Authorities
        |
        v
Security rule
        |
    +---+---+
    |       |
  allow    deny
```

A role is commonly represented as an authority using the `ROLE_` prefix:

```text
Role: USER
Authority: ROLE_USER
```

---

## Putting Login and Authorization Together

Now the complete non-JWT security story is visible:

```text
                         LOGIN
                           |
                           v
                  username + password
                           |
                           v
                 AuthenticationManager
                           |
                           v
                DaoAuthenticationProvider
                           |
                    +------+------+
                    |             |
                    v             v
            UserDetailsService PasswordEncoder
                    |             |
                    v             |
                Database          |
                    |             |
                    +------+------+
                           |
                           v
                    Authentication
                           |
                           v
                    SecurityContext
                           |
                           v
                      User = Ravi
                           |
                           v
                    NEXT REQUEST
                           |
                           v
                    Authorization
                           |
                    +------+------+
                    |             |
                  allow          deny
                    |             |
                    v             v
                Controller        403
```

At this point, we understand security without needing JWT.

Now we can ask the next architectural question.

---

### Visual checkpoint — Spring Security authentication pipeline

```text
HTTP REQUEST
     |
     v
SecurityFilterChain
     |
     v
Authentication request
     |
     v
AuthenticationManager
     |
     v
AuthenticationProvider
     |
     v
DaoAuthenticationProvider
     |
     +--------------------+-------------------+
     |                                        |
     v                                        v
UserDetailsService                     PasswordEncoder
     |                                        |
     v                                        |
UserDetails ----------------------------> password match
     |
     v
Authenticated Authentication
     |
     v
SecurityContext
     |
     v
Authorization
```

The diagram is the reason the individual classes matter: each component owns a different step of the same authentication operation.

---

## Failure Paths — What Happens When Authentication Fails?

Authentication diagrams are often taught as if every credential is correct. Real systems spend just as much time handling failure as success. The important interview skill is to know **where the failure occurs, which exception represents it, and which layer turns it into the HTTP response seen by the client**.

### Wrong password

```text
CLIENT
  |
  | username + WRONG password
  v
AuthenticationManager
  |
  v
DaoAuthenticationProvider
  |
  +--> UserDetailsService
  |        |
  |        v
  |     UserDetails
  |
  +--> PasswordEncoder.matches(...)
           |
           v
       PASSWORD MISMATCH
           |
           v
   BadCredentialsException
           |
           v
 Authentication failure
           |
           v
 AuthenticationEntryPoint
           |
           v
      HTTP 401
```

The sequence is:

1. `UserDetailsService` loads the user's stored credentials.
2. `DaoAuthenticationProvider` asks `PasswordEncoder` to compare the submitted password with the stored hash.
3. The comparison fails.
4. Spring Security represents the failure as an `AuthenticationException`, commonly `BadCredentialsException` for a wrong password.
5. For a REST API configured to return HTTP status codes, the configured `AuthenticationEntryPoint` typically produces **401 Unauthorized**.

### User does not exist

```text
AuthenticationManager
       |
       v
DaoAuthenticationProvider
       |
       v
UserDetailsService
       |
       v
Repository / User Store
       |
       v
USER NOT FOUND
       |
       v
UsernameNotFoundException
       |
       v
Authentication fails
       |
       v
AuthenticationEntryPoint
       |
       v
HTTP 401
```

`UsernameNotFoundException` is what `UserDetailsService` throws when it cannot find the requested user. There is an important Spring Security detail: `DaoAuthenticationProvider` commonly hides the distinction between "username not found" and "wrong password" and exposes a `BadCredentialsException` instead. This helps avoid username-enumeration leaks.

So the practical mental model is:

```text
User missing
    |
    v
UsernameNotFoundException
    |
    v
DaoAuthenticationProvider may hide it
    |
    v
AuthenticationException / BadCredentialsException
    |
    v
AuthenticationEntryPoint
    |
    v
HTTP 401
```

### Authentication succeeds, authorization fails

```text
Credentials / JWT
       |
       v
Authentication succeeds
       |
       v
SecurityContext
       |
       v
Authorization check
       |
       v
Required authority: ROLE_ADMIN
       |
       X
User has: ROLE_USER
       |
       v
HTTP 403 Forbidden
```

The identity is valid. The permission is not. This is why 401 and 403 should not be mentally merged.

### Failure summary

| Failure | Where it occurs | Security meaning | Typical REST result |
|---|---|---|---|
| Wrong password | `PasswordEncoder` / `DaoAuthenticationProvider` | Authentication failed | **401** |
| User not found | `UserDetailsService` | Authentication failed | **401** |
| Missing/invalid JWT | JWT authentication filter | Authentication not established | **401** |
| Expired JWT | JWT parsing/validation | Authentication not established | **401** |
| Bad JWT signature | JWT parsing/validation | Authentication not established | **401** |
| Valid user, insufficient authority | Authorization | Authentication succeeded, permission denied | **403** |

The exact response body and handler are determined by the application's security configuration, but this is the standard REST API mental model.

# JWT Authentication — Generation, Validation and Protected Requests

## Now Why JWT?

The session model is not "bad." It is a valid architecture.

The question is:

> **What if we want a stateless authentication model where the client sends a verifiable authentication token on each request instead of relying on a traditional server-side login session?**

Session model:

```text
Client
  |
  | session ID
  v
Server
  |
  v
Session Store
  |
  v
User
```

JWT model:

```text
Client
  |
  | JWT
  v
Server
  |
  v
Validate token
  |
  v
Establish identity
```

JWT is one way of implementing this stateless model.

---

## How JWT Changes the Flow

Session:

```text
LOGIN
  |
  v
Authenticate
  |
  v
Create server session
  |
  v
Send session ID
```

Later:

```text
Session ID
   |
   v
Find server-side session
   |
   v
Identity
```

JWT:

```text
LOGIN
  |
  v
Authenticate
  |
  v
Generate JWT
  |
  v
Send JWT to client
```

Later:

```text
JWT
 |
 v
Validate token
 |
 v
Extract identity
 |
 v
Build Authentication
 |
 v
SecurityContext
```

So the major architectural difference is where the authentication representation travels.

---

## JWT Structure

A JWT is commonly represented as:

```text
HEADER.PAYLOAD.SIGNATURE
```

Visualize:

```text
+------------------+
| HEADER           |
| alg = HS256      |
| typ = JWT        |
+------------------+
          .
+------------------+
| PAYLOAD          |
| sub = ravi       |
| iat = ...        |
| exp = ...        |
+------------------+
          .
+------------------+
| SIGNATURE        |
| integrity check  |
+------------------+
```

Now each section has a specific job.

---

## Header

Example:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

The header contains metadata such as the signing algorithm.

In a typical application:

```text
alg = HS256
```

---

## Payload and Claims

Example:

```json
{
  "sub": "ravi",
  "iat": 1780000000,
  "exp": 1780003600
}
```

Important claims:

```text
sub = subject
      |
      +--> a typical application uses username

iat = issued at

exp = expiration time
```

Your token generation does:

```java
.setSubject(username)
```

Therefore:

```text
username = ravi
      |
      v
JWT sub = ravi
```

### Important

A normal signed JWT is not automatically encrypted.

```text
JWT payload
     |
     v
Can generally be decoded
```

The signature protects integrity/authenticity. It does not provide confidentiality.

Do not put secrets into an ordinary signed JWT payload.

---

## Signature

The signature allows the server to detect modification of the signed content.

Conceptually:

```text
Header
   +
Payload
   +
Signing key
   |
   v
Signing algorithm
   |
   v
Signature
```

If an attacker changes:

```text
sub = ravi
```

to:

```text
sub = admin
```

without producing a valid corresponding signature, validation fails.

So:

```text
Modified token
      |
      v
Signature verification
      |
      v
Mismatch
      |
      v
Invalid token
```

---

## HS256 and the Signing Key

A typical application uses:

```java
SignatureAlgorithm.HS256
```

HS256 is an HMAC-based SHA-256 signing algorithm.

Conceptually:

```text
Header + Payload + Secret Key
              |
              v
            HS256
              |
              v
          Signature
```

The server must protect the secret signing key.

A typical application loads it through configuration:

```java
@Value("${app.jwt.secret}")
private String jwtSecret;
```

and creates the signing key from it.

In production, secrets should be managed securely rather than hardcoded into source code.

### HS256 vs RS256

JWTs do not require a shared secret. Another common choice is **RS256**, an asymmetric signing algorithm.

```text
HS256

        SHARED SECRET
        /          \
       v            v
   Sign token    Verify token

Same secret must be protected by every verifier.

RS256

   PRIVATE KEY                 PUBLIC KEY
       |                            |
       v                            v
   Sign token                  Verify token

Private key stays with the trusted signer.
Public key can be distributed to services that only need verification.
```

| | HS256 | RS256 |
|---|---|---|
| Cryptography | Symmetric | Asymmetric |
| Signing key | Shared secret | Private key |
| Verification | Same secret | Public key |
| Useful when | Simpler deployments | Many services need verification without receiving the private signing key |

The important architectural distinction is that RS256 lets verification happen with a public key while the private signing key remains with the trusted signer.

---

## JWT Generation

A typical application has logic like:

```java
public String generateToken(String username) {
    return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(
                    new Date(
                        System.currentTimeMillis()
                        + jwtExpiration
                    )
            )
            .signWith(
                    getSigningKey(),
                    SignatureAlgorithm.HS256
            )
            .compact();
}
```

Visualize it as:

```text
username
   |
   v
setSubject()
   |
   v
sub claim
```

then:

```text
current time
   |
   v
setIssuedAt()
   |
   v
iat claim
```

then:

```text
current time + expiration
   |
   v
setExpiration()
   |
   v
exp claim
```

then:

```text
Header + Payload
      |
      + secret key
      |
      v
     HS256
      |
      v
  Signature
```

finally:

```text
compact()
   |
   v
HEADER.PAYLOAD.SIGNATURE
```

---

## JWT Login Flow

Now the session creation step is replaced by JWT generation:

```text
                         LOGIN
                           |
                           v
                  username + password
                           |
                           v
                         Tomcat
                           |
                           v
                 Servlet Filter Chain
                           |
                           v
                  Spring Security
                           |
                           | login allowed
                           v
                  DispatcherServlet
                           |
                           v
                    AuthController (or equivalent authentication endpoint)
                           |
                           v
                      Authentication service
                           |
                           v
             UsernamePasswordAuthenticationToken
                           |
                           v
                  AuthenticationManager
                           |
                           v
                DaoAuthenticationProvider
                           |
                    +------+------+
                    |             |
                    v             v
            UserDetailsService PasswordEncoder
                    |             |
                    v             |
                Database          |
                    |             |
                    +------+------+
                           |
                           v
                Authentication SUCCESS
                           |
                           v
                         JwtUtil
                           |
                           v
                      Generate JWT
                           |
                           v
                         Client
```

Notice the sequence:

```text
Authenticate first
       |
       v
Generate JWT second
```

A JWT should not be issued merely because a login request arrived. The credentials must first be successfully authenticated.

---

## Where Should the Client Store the JWT?

Saying “the client receives the JWT” is only half the story. The next question is **where the client keeps it**. Storage changes the threat model.

### `localStorage`

```text
Login response
     |
     v
JavaScript
     |
     v
localStorage
     |
     v
JavaScript reads token
     |
     v
Authorization: Bearer <JWT>
```

It is convenient for browser JavaScript, but JavaScript can read the token. If an attacker achieves XSS, malicious code may be able to steal it.

### `HttpOnly` cookie

```text
Server
  |
  | Set-Cookie: access=...; HttpOnly; Secure
  v
Browser cookie jar
  |
  | JavaScript cannot directly read HttpOnly cookie
  v
Browser sends cookie on applicable requests
```

`HttpOnly` prevents JavaScript from directly reading the cookie, while `Secure` restricts transmission to HTTPS. `SameSite` can reduce cross-site request exposure.

The trade-off is important: cookies are automatically attached by the browser, so **CSRF becomes part of the security design**. `HttpOnly` also does not make XSS harmless; malicious JavaScript can still cause authenticated requests even if it cannot read the cookie.

### In-memory storage

```text
Login
  |
  v
Application memory
  |
  v
Access token
```

This reduces persistence, but a full page reload can lose the token unless another mechanism obtains a new one.

### Compare the mental models

```text
                  JWT STORAGE
                       |
          +------------+------------+
          |            |            |
          v            v            v
     localStorage   HttpOnly       Memory
                    cookie
          |            |            |
          v            v            v
       XSS risk     CSRF model    lost on reload
                    matters
```

There is no single storage choice that makes an application secure by itself. The choice depends on the architecture and threat model.

---

## The Protected Request

Suppose login returned a JWT.

The client later sends:

```http
GET /api/notes
Authorization: Bearer eyJhbGciOiJIUzI1Ni...
```

Now the security flow is:

```text
Client
  |
  | Authorization: Bearer JWT
  v
Tomcat
  |
  v
Servlet Filter Chain
  |
  v
Spring Security
  |
  v
JwtAuthenticationFilter
  |
  v
Validate JWT
  |
  v
Identify user
  |
  v
Create Authentication
  |
  v
SecurityContext
  |
  v
Authorization
  |
  v
DispatcherServlet
  |
  v
Controller
```

This is the most important JWT request diagram.

---

## JwtAuthenticationFilter

Your custom filter has logic like:

```java
final String authHeader =
        request.getHeader("Authorization");

if (authHeader == null ||
        !authHeader.startsWith("Bearer ")) {

    filterChain.doFilter(request, response);
    return;
}

final String token = authHeader.substring(7);
```

Visualize:

```text
HTTP Request
     |
     v
Read Authorization header
     |
     v
Does it exist?
     |
   +---+---+
   |       |
  no      yes
   |       |
   v       v
continue  "Bearer <JWT>"
             |
             v
        Extract JWT
```

If there is no JWT, this filter does not magically authenticate the user.

It can continue the chain, allowing the rest of the security configuration to decide whether the endpoint is public or requires authentication.

---

## JWT Validation and `parseClaimsJws()`

Your `JwtUtil` has logic like:

```java
private Claims getClaims(String token) {
    return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
}
```

Visualize:

```text
JWT
 |
 v
Jwts.parserBuilder()
 |
 v
Configure trusted signing key
 |
 v
build()
 |
 v
JWT parser
 |
 v
parseClaimsJws(token)
 |
 +--> parse signed token
 +--> verify signature
 +--> validate relevant token conditions
 |
 v
Claims
```

This is **not simply decoding JSON**.

Your validation method:

```java
public boolean isTokenValid(String token) {
    try {
        getClaims(token);
        return true;
    } catch (JwtException | IllegalArgumentException e) {
        return false;
    }
}
```

maps to:

```text
parse / validate
      |
  +---+---+
  |       |
valid   exception
  |       |
  v       v
true    false
```

Expiration is one of the relevant claims checked by JWT processing. An expired token therefore cannot be treated as a valid authenticated credential.

### JWT failure paths — expired token and bad signature

These failures happen **inside JWT validation in the authentication filter**, before the request reaches the controller.

#### Expired JWT

```text
HTTP request
     |
     v
JwtAuthenticationFilter
     |
     v
Extract Bearer token
     |
     v
parseClaimsJws(token)
     |
     v
exp claim is in the past
     |
     v
ExpiredJwtException / JwtException
     |
     v
JWT authentication fails
     |
     v
HTTP 401
     X
DispatcherServlet / Controller not reached
```

#### Bad signature

```text
HTTP request
     |
     v
JwtAuthenticationFilter
     |
     v
Extract Bearer token
     |
     v
parseClaimsJws(token)
     |
     v
Verify signature
     |
     v
SIGNATURE MISMATCH
     |
     v
SignatureException / JwtException
     |
     v
JWT authentication fails
     |
     v
HTTP 401
     X
Controller not reached
```

The exact exception class depends on the JWT library/version, but the security meaning is the same: **the presented credential cannot establish a trusted identity**.

A common REST implementation catches the validation exception in the JWT filter and returns 401, or delegates to an appropriate authentication failure mechanism. The crucial point is that the token is rejected during the filter stage, before business logic executes.

### Why a bad signature is different from decoding

```text
Decode payload   -> data can be read
Verify signature -> token integrity and trusted signer are checked
```

Therefore:

```text
Readable JWT
   !=
Trusted JWT
```

---

---

## Extracting the Subject

A typical application has:

```java
public String extractUsername(String token) {
    return getClaims(token).getSubject();
}
```

Flow:

```text
JWT
 |
 v
getClaims()
 |
 v
Claims
 |
 v
getSubject()
 |
 v
"ravi"
```

Why does this work?

Because generation did:

```java
.setSubject(username)
```

So:

```text
LOGIN
username = ravi
     |
     v
setSubject("ravi")
     |
     v
JWT sub = ravi
```

Later:

```text
JWT
 |
 v
getSubject()
 |
 v
ravi
```

---

## JWT → Authentication

Extracting the username is not the end of authentication.

We need to establish an authenticated identity that Spring Security can use.

```text
JWT
 |
 v
Validate
 |
 v
subject = ravi
 |
 v
UserDetailsService
 |
 v
UserDetails(ravi)
 |
 v
Authorities
 |
 v
Authentication
```

Conceptually:

```java
UserDetails userDetails =
        userDetailsService
            .loadUserByUsername(username);

UsernamePasswordAuthenticationToken authentication =
        new UsernamePasswordAuthenticationToken(
                userDetails,
                null,
                userDetails.getAuthorities()
        );
```

Notice what is different from login.

Login:

```text
username + password
```

Protected JWT request:

```text
validated JWT
    |
    v
username
    |
    v
UserDetails + authorities
```

---

## JWT + SecurityContext

Now place the authenticated identity into the security context:

```text
JWT
 |
 v
Validate
 |
 v
Extract username
 |
 v
Load UserDetails
 |
 v
Create Authentication
 |
 v
SecurityContextHolder
 |
 v
SecurityContext
 |
 v
Authentication
```

Conceptually:

```java
SecurityContextHolder
        .getContext()
        .setAuthentication(authentication);
```

Now downstream code can ask:

```text
Who is authenticated for this request?
```

and Spring Security can answer:

```text
Ravi
ROLE_USER
```

---

## Authorization After JWT

Authentication is now complete.

```text
Authentication
   |
   +--> User = Ravi
   +--> ROLE_USER
   |
   v
Authorization
   |
   v
Does this identity satisfy the security rule?
```

For example:

```text
GET /api/notes
       |
       v
authenticated()?
       |
       v
YES
       |
       v
Controller
```

But:

```text
DELETE /api/users/10
       |
       v
hasRole("ADMIN")?
       |
       v
Ravi = ROLE_USER
       |
       v
NO
       |
       v
403 Forbidden
```

This shows why authentication must happen before authorization.

---

## Using the Current User to Retrieve Resources

In a typical secure application, suppose Ravi requests:

```http
GET /api/notes
Authorization: Bearer <Ravi JWT>
```

The security flow establishes:

```text
Authenticated identity
        |
        v
Ravi
```

Then application logic can retrieve Ravi's notes:

```text
JWT
 |
 v
Authentication
 |
 v
SecurityContext
 |
 v
Current user = ravi
 |
 v
Business service
 |
 v
NoteRepository
 |
 v
Database
 |
 v
Ravi's notes
```

Conceptually:

```sql
SELECT *
FROM notes
WHERE owner = 'ravi';
```

The important security principle is:

> **Use the server-established authenticated identity when determining which protected resources the requester may access.**

Do not blindly trust a client-provided user ID when the authenticated identity is already available.

---

## Complete JWT Request Flow

This is the main diagram to memorize visually:

```text
                         CLIENT
                           |
                           |
             Authorization: Bearer JWT
                           |
                           v
                         TOMCAT
                           |
                           v
                SERVLET FILTER CHAIN
                           |
                           v
                  SPRING SECURITY
                           |
                           v
               JwtAuthenticationFilter
                           |
                           v
                 Read Authorization
                           |
                           v
                    Extract JWT
                           |
                           v
                  Validate JWT
                           |
                      +----+----+
                      |         |
                   invalid     valid
                      |         |
                      v         v
                 reject /    Extract
                 continue    username
                                |
                                v
                       UserDetailsService
                                |
                                v
                           UserDetails
                                |
                                v
                   Create Authentication
                                |
                                v
                    SecurityContextHolder
                                |
                                v
                         SecurityContext
                                |
                                v
                         Authorization
                                |
                       +--------+--------+
                       |                 |
                    allowed            denied
                       |                 |
                       v                 v
               DispatcherServlet       403
                       |
                       v
                   Controller
                       |
                       v
                    Service
                       |
                       v
                  Repository
                       |
                       v
                    Database
```

Read this diagram top-to-bottom as one continuous story.

---


### Visual checkpoint — the complete JWT lifecycle

```text
┌──────────────────── REGISTRATION ────────────────────┐
│ credentials → validate → BCrypt hash → database     │
└─────────────────────────┬────────────────────────────┘
                          v
┌────────────────────── LOGIN ─────────────────────────┐
│ username + password                                  │
│        ↓                                               │
│ AuthenticationManager                                 │
│        ↓                                               │
│ AuthenticationProvider                                │
│        ↓                                               │
│ UserDetailsService + PasswordEncoder                  │
│        ↓                                               │
│ authenticated                                         │
│        ↓                                               │
│ JWT generated                                         │
└─────────────────────────┬────────────────────────────┘
                          v
                     CLIENT STORES
                         JWT
                          |
                          v
┌────────────────── PROTECTED REQUEST ──────────────────┐
│ Authorization: Bearer <JWT>                          │
│        ↓                                               │
│ JWT authentication filter                             │
│        ↓                                               │
│ parse + verify signature + validate claims             │
│        ↓                                               │
│ extract subject / identity                             │
│        ↓                                               │
│ load user details when required                        │
│        ↓                                               │
│ create authenticated Authentication                    │
│        ↓                                               │
│ SecurityContextHolder                                 │
│        ↓                                               │
│ authorization check                                   │
│        ↓                                               │
│ controller → service → repository → database          │
└───────────────────────────────────────────────────────┘
```

This is the full story the rest of the JWT chapter builds step by step.

## Access Tokens and Refresh Tokens

A common real-world JWT architecture separates credentials into a **short-lived access token** and a **longer-lived refresh token**.

The access token is presented frequently, so limiting its lifetime reduces the window in which a stolen access token remains useful. The refresh token is used less frequently and can be protected, rotated, and revoked more carefully.

### The pair

```text
LOGIN
  |
  v
Authenticate username + password
  |
  v
+-------------------------------+
| Issue two credentials         |
|                               |
| Access Token  -> short-lived  |
| Refresh Token -> longer-lived |
+-------------------------------+
       |                 |
       v                 v
 API requests       /refresh endpoint
```

### Normal API request

```text
Client
  |
  | Authorization: Bearer <access token>
  v
API
  |
  v
Validate access token
  |
  +--> valid -> continue
  |
  +--> expired -> 401
```

### What happens when the access token expires?

The user does **not necessarily need to log in again**. The client can use a valid refresh token to request a new access token.

```text
Access token expired
        |
        v
API returns 401
        |
        v
Client calls /auth/refresh
        |
        | refresh token
        v
Refresh-token validation
        |
   +----+----+
   |         |
 valid     invalid/revoked
   |         |
   v         v
Issue new    require login
access token
```

Refresh tokens should be treated as sensitive credentials. Real systems may rotate them, store their identifiers server-side, associate them with a device/session, detect reuse, and revoke them.

> Refresh tokens are not a requirement of JWT itself. They are a common **token lifecycle architecture** that makes short-lived access tokens practical.

---

## Logout and JWT Invalidation

Logout exposes one of the genuine trade-offs of stateless JWT authentication.

With a traditional session:

```text
Logout
  |
  v
Invalidate server session
  |
  v
Session ID no longer maps to authenticated state
```

With a self-contained access JWT:

```text
Logout
  |
  v
Client deletes token
  |
  X
A copied JWT may still be cryptographically valid
until it expires or the server checks additional state.
```

### Common invalidation strategies

#### Short-lived access token + refresh-token revocation

```text
Access token -> expires quickly
Refresh token -> can be revoked/rotated
```

#### Token blacklist / denylist

```text
JWT ID / token identifier
        |
        v
Revocation store
        |
        v
If revoked -> reject token
```

This adds server-side state and reduces the simplicity of completely stateless validation.

#### Database-backed token/session record

```text
Logout
  |
  v
Mark refresh credential revoked
  |
  v
Future refresh attempt -> reject
```

The key interview answer is:

> **JWT signature validity is not the same thing as revocation.** A JWT can be cryptographically valid while the application has decided that its associated session/refresh credential is no longer allowed.

---

# Putting Everything Together — Common Doubts and Session vs JWT

## Why UsernamePasswordAuthenticationToken Appears Twice

This is a common source of confusion.

## During login

```java
new UsernamePasswordAuthenticationToken(
    username,
    password
)
```

Meaning:

```text
"Here are the credentials.
Please authenticate me."
```

## During JWT authentication

Conceptually:

```java
new UsernamePasswordAuthenticationToken(
    userDetails,
    null,
    userDetails.getAuthorities()
)
```

Meaning:

```text
"This identity has been established
from the validated JWT."
```

So visualize:

```text
LOGIN
username + password
       |
       v
authentication attempt

JWT REQUEST
validated JWT
       |
       v
UserDetails + authorities
       |
       v
authenticated identity
```

Same class, different stage and meaning.

---

## Why UserDetailsService Is Used Again

A natural question is:

> **If the JWT already contains `sub = ravi`, why load Ravi again?**

Because the application may need current security information:

```text
ravi
 |
 +--> authorities
 +--> role
 +--> account status
 +--> other security information
```

So a typical application can do:

```text
JWT
 |
 v
subject = ravi
 |
 v
UserDetailsService
 |
 v
Current user security information
 |
 v
Authentication
```

There are other JWT architectures and trade-offs, but this is the model used by a typical application.

---

## Session vs JWT

## Session model

```text
LOGIN
 |
 v
Username + Password
 |
 v
Authenticate
 |
 v
Create server session
 |
 v
Session ID
 |
 v
Cookie
 |
 v
Browser
```

Later:

```text
Request
 |
 v
Cookie / Session ID
 |
 v
Server
 |
 v
Session Store
 |
 v
Authenticated User
```

### Authentication state

```text
SERVER
  |
  v
Session Store
```

---

## JWT model

```text
LOGIN
 |
 v
Username + Password
 |
 v
Authenticate
 |
 v
Generate JWT
 |
 v
Client
```

Later:

```text
Request
 |
 v
Authorization: Bearer JWT
 |
 v
Server
 |
 v
Validate JWT
 |
 v
Extract identity
 |
 v
Authentication
 |
 v
SecurityContext
```

### Authentication representation

```text
CLIENT
  |
  v
JWT
```

### Important conclusion

Do not say:

> "Sessions are bad and JWT is better."

The correct engineering question is:

> **Which authentication architecture fits the application's requirements, client model, scaling model, threat model, and operational constraints?**

---

## Common Doubts

### Is Spring Security completely separate from the Servlet filter chain?

No.

It integrates with the Servlet filtering mechanism:

```text
Tomcat
 ↓
Servlet Filter Chain
 ↓
Spring Security filters
 ↓
DispatcherServlet
```

---

### Is `JwtAuthenticationFilter` part of the basic HTTP infrastructure?

It participates in the Servlet filter pipeline, but its purpose is specifically security:

```text
Read JWT
Validate JWT
Identify user
Create Authentication
Set SecurityContext
```

---

### Does the controller authenticate the user?

Not normally in a Spring Security architecture.

Authentication is commonly established earlier in the security filter/authentication pipeline.

```text
Request
 ↓
Security filters
 ↓
Authentication
 ↓
Authorization
 ↓
Controller
```

---

### Does `AuthenticationManager` directly query the database?

Not normally.

The flow is:

```text
AuthenticationManager
 ↓
AuthenticationProvider
 ↓
DaoAuthenticationProvider
 ↓
UserDetailsService
 ↓
Repository
 ↓
Database
```

---

### Does `DaoAuthenticationProvider` store users?

No.

It uses `UserDetailsService` to load user information and a `PasswordEncoder` to verify credentials.

---

### Does `UserDetailsService` check the password?

Its primary responsibility is loading the user.

The provider coordinates password verification using the configured `PasswordEncoder`.

```text
DaoAuthenticationProvider
      |
      +--> UserDetailsService
      |
      +--> PasswordEncoder
```

---

### Is `UserDetails` the same as my database `User` entity?

Not necessarily.

Think:

```text
Database User
      |
      v
UserDetailsService
      |
      v
Spring Security UserDetails
```

---

### Is `Authentication` the same as authorization?

No.

```text
Authentication
  -> Who?

Authorization
  -> What can they do?
```

---

### Is the JWT stored in `SecurityContext`?

The `SecurityContext` primarily contains the `Authentication` object.

The flow is:

```text
JWT
 ↓
Validate
 ↓
Authentication
 ↓
SecurityContext
```

---

### Is `SecurityContextHolder` a database?

No.

It provides access to the current `SecurityContext`.

```text
SecurityContextHolder
 ↓
SecurityContext
 ↓
Authentication
```

---

### Is a JWT a session?

No.

A traditional session is server-side state associated with a session ID.

A JWT is a signed token containing claims that the server validates.

```text
SESSION
Session ID -> Server session

JWT
JWT -> Validate -> Authentication
```

---

### Is a cookie the same as a session?

No.

```text
Cookie = browser storage/transport mechanism
Session = server-side state
```

A cookie can carry a session ID.

---

### Why does CSRF matter with cookies?

Because browsers can automatically attach applicable cookies to requests.

An attacker may try to cause a victim's browser to make a request carrying the victim's authentication cookie.

---

### Does disabling CSRF automatically make an application insecure?

No.

Security configuration depends on the architecture and threat model.

A cookie-based browser session and an API using an explicit bearer token have different CSRF considerations.

---

### Does JWT solve all security problems?

No.

You still need to consider:

```text
HTTPS
XSS
Token theft
Secret management
Expiration
Authorization
Input validation
Dependency vulnerabilities
Database security
CSRF where applicable
```

---

### Is JWT encrypted?

Not by default.

A normal signed JWT is encoded and signed, not automatically encrypted.

```text
Encoded != Encrypted
Signed != Encrypted
```

---

### Can someone decode the JWT without the secret?

Generally yes.

But:

```text
Decode
  = read contents

Validate
  = verify integrity/authenticity and applicable token conditions
```

These are different operations.

---

### Why can't I just read `sub` and trust it?

Because the token must first be validated.

Correct:

```text
JWT
 ↓
Validate
 ↓
Extract subject
```

Incorrect:

```text
JWT
 ↓
Read subject
 ↓
Trust immediately
```

---

### Why use SecurityContext if JWT already contains the username?

Because the JWT is the incoming credential.

Spring Security converts that credential into an `Authentication` representation that the rest of the security/application pipeline can use.

```text
JWT
 ↓
Authentication
 ↓
SecurityContext
 ↓
Application
```

---

### Does JWT mean there is no authentication after login?

No.

Every protected request still needs authentication.

Session:

```text
Request -> Session ID -> Existing server session -> Identity
```

JWT:

```text
Request -> JWT -> Validate -> Identity
```

JWT changes the mechanism; it does not eliminate authentication.

---

## Complete Authentication Lifecycle — Happy Path + Failure Paths

At this point, the entire story can be visualized as one system rather than a collection of isolated concepts.

```text
                              CLIENT
                                |
                +---------------+---------------+
                |                               |
                v                               v
          REGISTRATION                         LOGIN
                |                               |
                v                               v
        Validate input                username + password
                |                               |
                v                               v
        BCrypt password hash            AuthenticationManager
                |                               |
                v                               v
             DATABASE                  DaoAuthenticationProvider
                                                |
                                   +------------+------------+
                                   |                         |
                                   v                         v
                            UserDetailsService        PasswordEncoder
                                   |                         |
                                   +------------+------------+
                                                |
                              +-----------------+-----------------+
                              |                                   |
                           FAILURE                              SUCCESS
                              |                                   |
                 +------------+-----------+                       v
                 |                        |                 Issue tokens
          wrong password             user missing                |
                 |                        |               +--------+--------+
                 v                        v               |                 |
        BadCredentialsException   UsernameNotFoundException   Access token   Refresh token
                 |                        |               |                 |
                 +------------+-----------+               |                 |
                              |                           |                 |
                              v                           v                 v
                         HTTP 401                    PROTECTED REQUEST   /refresh
                                                          |                 |
                                                          v                 v
                                                JwtAuthenticationFilter  Validate refresh
                                                          |                 |
                                               +----------+----------+      |
                                               |                     |      |
                                           valid JWT            invalid JWT  |
                                               |                     |      |
                                               v                     v      |
                                      Authentication        HTTP 401 <-------+
                                               |
                                               v
                                      SecurityContext
                                               |
                                               v
                                        Authorization
                                               |
                                      +--------+--------+
                                      |                 |
                                   allowed            denied
                                      |                 |
                                      v                 v
                                  Controller          HTTP 403
                                      |
                                      v
                                   Service
                                      |
                                      v
                                  Repository
                                      |
                                      v
                                  Database

LOGOUT:
  client deletes local credential
        + revoke refresh credential / blacklist if immediate invalidation is required
```

### What this diagram teaches

- Registration creates the credential material that will later be verified.
- Login establishes authentication and issues token state.
- Access tokens are used on ordinary protected requests.
- JWT validation happens in the security/filter stage, before the controller.
- Invalid credentials generally produce a 401 authentication failure.
- A valid identity with insufficient authority produces 403.
- Refresh tokens solve the usability problem created by short-lived access tokens.
- Logout is easy for server-side sessions because the server can destroy state; JWT invalidation requires an explicit lifecycle strategy if immediate revocation is required.

---

# Final Mental Model and Interview Revision

## The Mental Model

Do **not** memorize this chapter as a list:

```text
AuthenticationManager
AuthenticationProvider
DaoAuthenticationProvider
UserDetailsService
UserDetails
Authentication
SecurityContext
SecurityContextHolder
JwtUtil
JwtAuthenticationFilter
...
```

Instead, visualize one request.

### Step 1 — Request enters

```text
Client
  ↓
Tomcat
  ↓
Servlet Filter Chain
```

### Step 2 — Security gets involved

```text
Servlet Filter Chain
  ↓
Spring Security
```

### Step 3 — Authentication

Session architecture:

```text
Session ID
  ↓
Server session
  ↓
Identity
```

JWT architecture:

```text
JWT
  ↓
Validate
  ↓
Identity
  ↓
UserDetails
  ↓
Authentication
```

### Step 4 — Current identity is available

```text
Authentication
  ↓
SecurityContext
  ↓
SecurityContextHolder
```

### Step 5 — Authorization

```text
Identity + Authorities
          |
          v
Authorization rule
          |
     +----+----+
     |         |
   allow      deny
```

### Step 6 — Application logic

If allowed:

```text
DispatcherServlet
      ↓
Controller
      ↓
Service
      ↓
Repository
      ↓
Database
```

---

## Interview Revision

### What happens when the password is wrong?

`PasswordEncoder` comparison fails, the provider raises an authentication failure (commonly `BadCredentialsException`), and a configured `AuthenticationEntryPoint` typically turns it into **401 Unauthorized** for a REST API.

### What happens when the user does not exist?

`UserDetailsService` throws `UsernameNotFoundException`. `DaoAuthenticationProvider` commonly hides the distinction and exposes a generic authentication failure such as `BadCredentialsException`, which is then handled as a 401 in a REST API.

### What happens when a JWT is expired?

JWT parsing/validation fails in the JWT authentication filter. The token cannot establish authentication, so the protected request is rejected, typically with **401 Unauthorized**.

### What happens when a JWT has a bad signature?

Signature verification fails during JWT parsing. The filter must not place an authenticated identity in the `SecurityContext`; the request is rejected as unauthenticated, typically with **401**.

### What is the difference between 401 and 403?

```text
401 -> authentication is missing / invalid / failed
403 -> authentication exists, but authorization denies the operation
```

### What is the difference between an access token and a refresh token?

An access token is short-lived and sent to protected APIs. A refresh token is longer-lived and is used to obtain a new access token without requiring the user to submit credentials again.

### Where should a browser store a JWT?

The answer depends on the architecture and threat model. `localStorage` exposes the token to JavaScript and therefore increases the impact of XSS; `HttpOnly` cookies prevent JavaScript from directly reading the cookie but require careful CSRF defenses; in-memory storage reduces persistence but can lose the token on reload.

### How do you log out with JWT?

Deleting the client-side token stops that client from presenting it, but a copied self-contained JWT remains valid until expiration unless the architecture adds revocation state. Common strategies include short-lived access tokens with refresh-token revocation, token denylisting, or server-side token/session records.

### What is the difference between HS256 and RS256?

HS256 uses one shared secret for signing and verification. RS256 uses a private key for signing and a public key for verification. RS256 can be useful when many services need to verify tokens without receiving the private signing key.

### What is authentication?

Establishing the identity of a requester.

### What is authorization?

Determining whether an authenticated identity has permission to perform an operation.

### Where does Spring Security integrate with Spring Boot?

Through the Servlet filter mechanism.

```text
Tomcat
 ↓
Servlet Filter Chain
 ↓
Spring Security
```

### What is a `SecurityFilterChain`?

The configured Spring Security filters and request security rules applied to HTTP requests.

### What is `AuthenticationManager`?

The main authentication entry point/coordinator that delegates authentication to suitable providers.

### What is `AuthenticationProvider`?

A component that performs a particular authentication mechanism.

### What is `DaoAuthenticationProvider`?

A username/password authentication provider that uses `UserDetailsService` and `PasswordEncoder`.

### What does `UserDetailsService` do?

Loads user security information by username.

### What is `UserDetails`?

Spring Security's representation of security-related user information.

### What does `PasswordEncoder` do?

Provides password hashing and verification operations.

### What is BCrypt?

A password-hashing algorithm commonly used for secure password storage and verification.

### What is a session?

Server-side state associated with a session identifier.

### What is a cookie?

A browser mechanism for storing and sending data with applicable requests.

### What is CSRF?

An attack where an attacker attempts to cause a victim's authenticated browser to make an unwanted request.

### Why are cookies relevant to CSRF?

Because browsers can automatically attach applicable cookies to requests.

### What is JWT?

A token commonly represented as:

```text
Header.Payload.Signature
```

### Is JWT encrypted?

Not by default.

### What does the JWT signature provide?

It allows the server to verify the integrity/authenticity of the signed token using the appropriate key.

### What is the JWT subject in a typical application?

The `sub` claim containing the username.

### What is HS256?

An HMAC-based SHA-256 signing algorithm.

### What does `parseClaimsJws()` do?

It processes the signed JWT/JWS using the configured signing key, validates it, and exposes its claims.

### Why put the authenticated user into SecurityContext?

So downstream security and application code can access the established identity and authorities during request processing.

### What is the JWT request flow?

```text
JWT
 ↓
Validate
 ↓
Extract identity
 ↓
UserDetails
 ↓
Authentication
 ↓
SecurityContext
 ↓
Authorization
 ↓
Controller
```

---

## One-Page Quick Revision

## The problem

```text
Request arrives
     |
     v
Who is the caller?
     |
     v
Authentication
     |
     v
What can they do?
     |
     v
Authorization
```

## Traditional session model

```text
LOGIN
 |
 v
Username + Password
 |
 v
Authenticate
 |
 v
Create Session
 |
 v
Session ID
 |
 v
Cookie
 |
 v
Browser
```

Later:

```text
Request
 |
 v
Cookie / Session ID
 |
 v
Server
 |
 v
Session Store
 |
 v
Authenticated User
```

## Why CSRF matters

```text
Browser
 |
 | automatically attaches applicable cookie
 v
Server
 |
 v
Server sees authenticated session
 |
 v
Attacker may try to forge request
```

CSRF protection adds an additional validation mechanism where appropriate.

## Spring Security bridge

```text
Tomcat
 ↓
Servlet Filter Chain
 ↓
DelegatingFilterProxy
 ↓
FilterChainProxy
 ↓
SecurityFilterChain
 ↓
Security Filters
```

## Login authentication

```text
username + password
        |
        v
AuthenticationManager
        |
        v
DaoAuthenticationProvider
        |
        +--> UserDetailsService
        |       |
        |       v
        |   Database
        |
        +--> PasswordEncoder
                |
                v
          Password verification
        |
        v
Authentication
```

## Security context

```text
Authentication
      |
      v
SecurityContext
      |
      v
SecurityContextHolder
```

## Authorization

```text
Authentication
      |
      +--> User
      +--> Authorities
      |
      v
Authorization rule
      |
   +--+--+
   |     |
 allow  deny
```

## JWT generation

```text
Successful login
      |
      v
JwtUtil
      |
      +--> Header
      +--> Payload / Claims
      +--> Signature
      |
      v
JWT
```

## JWT request

```text
Authorization: Bearer JWT
              |
              v
     JwtAuthenticationFilter
              |
              v
          Validate JWT
              |
              v
       Extract subject
              |
              v
    UserDetailsService
              |
              v
          UserDetails
              |
              v
       Authentication
              |
              v
       SecurityContext
              |
              v
        Authorization
              |
              v
          Controller
```

## A typical application login

```text
AuthController (or equivalent authentication endpoint)
      ↓
Authentication service
      ↓
AuthenticationManager
      ↓
DaoAuthenticationProvider
      ↓
UserDetailsService implementation
      ↓
User repository
      ↓
Database
      +
PasswordEncoder / BCrypt
      ↓
Authentication success
      ↓
JwtUtil
      ↓
JWT
      ↓
Client
```

## A typical application protected request

```text
Client
  ↓
Bearer JWT
  ↓
JwtAuthenticationFilter
  ↓
JwtUtil
  ↓
Validate + extract username
  ↓
UserDetailsService implementation
  ↓
User repository
  ↓
UserDetails
  ↓
Authentication
  ↓
SecurityContextHolder
  ↓
Authorization
  ↓
Protected controller
  ↓
Service
  ↓
Repository
  ↓
Database
```

---


# End-to-End Authentication Story — From Registration to Authorized Request

The earlier sections introduced each concept at the moment it became necessary. This section now puts the entire lifecycle back together so you can see how all the pieces cooperate.

## 1. Registration — creating the identity

```text
CLIENT
  |
  | POST /register
  | username + password
  v
CONTROLLER
  |
  v
AUTHENTICATION / USER SERVICE
  |
  +--> validate input
  |
  +--> check whether user already exists
  |
  +--> BCryptPasswordEncoder.encode(password)
  |
  v
USER REPOSITORY
  |
  v
DATABASE
  |
  v
USER + PASSWORD HASH
```

### What is happening?

The user sends credentials for account creation. The application validates the input and hashes the password before storing it. The database stores the hash, not the original password.

The important distinction is:

```text
Registration
    |
    v
Create identity + securely store credentials
```

Registration by itself does not mean the user has already authenticated for every protected operation. Authentication is established when credentials are successfully verified during login.

---

## 2. Login — proving the identity

```text
CLIENT
  |
  | POST /login
  | username + password
  v
AUTH CONTROLLER
  |
  v
AUTHENTICATION SERVICE
  |
  | create UsernamePasswordAuthenticationToken
  v
AUTHENTICATION MANAGER
  |
  v
AUTHENTICATION PROVIDER
  |
  v
DAO AUTHENTICATION PROVIDER
  |\
  | \
  v  v
USER DETAILS SERVICE       PASSWORD ENCODER
  |                              |
  v                              |
LOAD USER DETAILS ---------------+
             |
             v
       COMPARE PASSWORD
             |
        +----+----+
        |         |
      FAIL      MATCH
        |         |
        v         v
      REJECT   AUTHENTICATED
                    |
                    v
               JWT GENERATION
                    |
                    v
                  CLIENT
```

### What is happening step by step?

**Step 1 — The client submits credentials.**

The username and password arrive as normal HTTP request data. At this point, the application has credentials, but it has not yet established a trusted authenticated identity.

**Step 2 — An authentication request object is created.**

Spring Security commonly represents username/password credentials with a `UsernamePasswordAuthenticationToken`.

At this stage it represents a request to authenticate:

```text
username + password
       |
       v
Authentication request
       |
       v
AuthenticationManager
```

**Step 3 — `AuthenticationManager` coordinates authentication.**

The `AuthenticationManager` is the main entry point for authentication. It delegates the actual authentication work to an appropriate `AuthenticationProvider`.

```text
AuthenticationManager
        |
        v
AuthenticationProvider
```

**Step 4 — `DaoAuthenticationProvider` obtains user information.**

For a common username/password setup, `DaoAuthenticationProvider` uses a `UserDetailsService` to load the user's security information.

```text
UserDetailsService
       |
       v
loadUserByUsername(username)
       |
       v
UserDetails
```

**Step 5 — The password is verified.**

The `PasswordEncoder` compares the submitted password with the stored password hash.

```text
Submitted password
       |
       v
PasswordEncoder.matches(...)
       |
       +------> stored password hash
```

A successful match allows authentication to continue. A failed match results in an authentication failure.

**Step 6 — Authentication succeeds.**

Spring Security now has a trusted `Authentication` representation of the user.

```text
Credentials verified
       |
       v
Authenticated Authentication
```

**Step 7 — The application generates a JWT.**

Only after successful authentication does the application create the signed token that the client will present on later protected requests.

```text
Authenticated identity
       |
       v
JWT generation
       |
       v
Signed JWT
       |
       v
Client
```

---

## 3. JWT generation — turning authentication into a token

A JWT is commonly represented as:

```text
HEADER.PAYLOAD.SIGNATURE
```

The generation flow is:

```text
Authenticated user
       |
       v
Create claims
       |
       +--> subject / username
       +--> issued-at
       +--> expiration
       +--> other appropriate claims
       |
       v
Serialize header + payload
       |
       v
Sign with trusted key
       |
       v
JWT
```

The client receives the token and sends it with later requests, commonly as:

```http
Authorization: Bearer <JWT>
```

The JWT is therefore not the password and not the user's database record. It is a signed representation that can be presented later as evidence of an authenticated identity.

---

## 4. Protected request — the JWT comes back

Suppose the client now calls:

```http
GET /api/notes
Authorization: Bearer <JWT>
```

The important thing is that this request is a **new HTTP request**. The server cannot simply assume that because the user logged in earlier, every later request is automatically trusted.

The request must establish authentication again for this request-processing cycle.

```text
CLIENT
  |
  | Authorization: Bearer <JWT>
  v
TOMCAT
  |
  v
SERVLET FILTER CHAIN
  |
  v
JWT AUTHENTICATION FILTER
```

---

## 5. JWT validation — do not simply decode and trust

The JWT authentication filter performs the token-processing work.

```text
Authorization Header
        |
        v
Extract Bearer token
        |
        v
Parse signed JWT
        |
        v
Verify signature
        |
        v
Validate relevant claims
        |
        v
Extract subject / identity
        |
        v
Load user details when required
        |
        v
Create authenticated Authentication
```

### Why is signature verification important?

A JWT contains data that can be decoded by a client. Decoding is not the same as trusting the data.

The server needs to determine whether the token was signed with the expected key and whether the token satisfies the application's validation rules.

```text
JWT received
   |
   v
Can the signature be verified?
   |
   +------ NO ------> reject / unauthenticated
   |
  YES
   |
   v
Are relevant claims valid?
   |
   +------ NO ------> reject / unauthenticated
   |
  YES
   |
   v
Identity can be trusted
```

For libraries such as JJWT, a call such as `parseClaimsJws()` is part of this signed-token parsing and validation process. It is not merely a JSON decoder.

---

## 6. JWT → `Authentication` → `SecurityContext`

After the token has been accepted, the application needs to turn the token's identity into the same kind of security representation used by the rest of Spring Security.

```text
Validated JWT
     |
     v
Extract subject / identity
     |
     v
Load user security information
     |
     v
Create authenticated Authentication
     |
     v
SecurityContext
     |
     v
SecurityContextHolder
```

This is the critical bridge:

```text
JWT
 |
 | proves / represents authenticated identity
 v
Authentication
 |
 | carries principal + authorities
 v
SecurityContext
 |
 | makes identity available to the request
 v
Authorization + application code
```

The JWT itself is **not** the `SecurityContext`.

The JWT is the credential/token presented by the client. The `Authentication` is Spring Security's representation of the established identity. The `SecurityContext` is where that security state is made available during request processing.

---

## 7. Authorization — authentication is not permission

Once the request has an authenticated identity, authorization can be evaluated.

```text
SecurityContext
      |
      v
Authenticated user
      |
      +--> authorities / roles
      |
      v
Authorization rule
      |
   +--+--+
   |     |
 ALLOW  DENY
   |     |
   v     v
Controller   403 / Access Denied
```

This gives us the exact separation:

```text
Authentication
    |
    v
WHO IS THE CALLER?

Authorization
    |
    v
IS THIS CALLER ALLOWED TO DO THIS?
```

A valid JWT does not automatically mean every operation is permitted. The authenticated identity still has to satisfy the authorization rule for the requested operation.

---

## 8. Authorized request — business logic finally executes

If authorization succeeds:

```text
SecurityContext
      |
      v
Authorization
      |
      v
ALLOW
      |
      v
DispatcherServlet
      |
      v
Controller
      |
      v
Service
      |
      v
Repository
      |
      v
Database
```

At this point, security has done its job for the request. The application can execute the protected business operation with an authenticated identity available through Spring Security's context.

---

## 9. The entire lifecycle in one visualization

```text
┌──────────────────────────── REGISTRATION ────────────────────────────┐
│                                                                      │
│  username + password → validate → BCrypt hash → database            │
│                                                                      │
└─────────────────────────────────┬────────────────────────────────────┘
                                  |
                                  v
┌────────────────────────────────── LOGIN ──────────────────────────────┐
│                                                                      │
│  username + password                                                 │
│          |                                                           │
│          v                                                           │
│  UsernamePasswordAuthenticationToken                                 │
│          |                                                           │
│          v                                                           │
│  AuthenticationManager                                               │
│          |                                                           │
│          v                                                           │
│  AuthenticationProvider                                               │
│          |                                                           │
│          v                                                           │
│  DaoAuthenticationProvider                                           │
│       /                 \\                                             │
│      v                   v                                            │
│  UserDetailsService   PasswordEncoder                                 │
│      |                   |                                            │
│      +--------> UserDetails <--------+                                │
│                         |                                            │
│                         v                                            │
│                  password matches?                                   │
│                         |                                            │
│                         v                                            │
│                    AUTHENTICATED                                    │
│                         |                                            │
│                         v                                            │
│                    JWT GENERATION                                   │
│                         |                                            │
└─────────────────────────┬────────────────────────────────────────────┘
                          |
                          v
                       CLIENT
                          |
                          | Authorization: Bearer <JWT>
                          v
┌──────────────────────── PROTECTED REQUEST ───────────────────────────┐
│                                                                      │
│  Servlet Filter Chain                                                │
│          |                                                           │
│          v                                                           │
│  JWT Authentication Filter                                           │
│          |                                                           │
│          +--> parse token                                            │
│          +--> verify signature                                       │
│          +--> validate claims                                        │
│          +--> extract identity                                       │
│          +--> load user details when required                        │
│          +--> create authenticated Authentication                    │
│                                                                      │
│                         |                                            │
│                         v                                            │
│                  SecurityContext                                     │
│                         |                                            │
│                         v                                            │
│                    AUTHORIZATION                                    │
│                       /     \\                                        │
│                    ALLOW     DENY                                    │
│                      |         |                                     │
│                      v         v                                     │
│                 Controller    403                                    │
│                      |                                               │
│                      v                                               │
│                   Service                                             │
│                      |                                               │
│                      v                                               │
│                  Repository                                           │
│                      |                                               │
│                      v                                               │
│                   Database                                            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### The four questions to ask when reading any Spring Security flow

```text
1. Where does the request enter?
        ↓
2. How is the caller authenticated?
        ↓
3. Where is the authenticated identity stored for this request?
        ↓
4. How is permission checked before business logic runs?
```

If you can answer those four questions, the individual Spring Security classes stop looking like unrelated framework magic and start fitting into one coherent request-processing system.

# Final Mental Model

If you remember only one diagram, remember this:

```text
                       HTTP REQUEST
                            |
                            v
                          TOMCAT
                            |
                            v
                 SERVLET FILTER CHAIN
                            |
                            v
                     SPRING SECURITY
                            |
                            v
                   WHO IS THE CALLER?
                            |
                            v
                     AUTHENTICATION
                            |
                            v
                    SecurityContext
                            |
                            v
                  WHAT CAN THEY DO?
                            |
                            v
                     AUTHORIZATION
                            |
                     +------+------+
                     |             |
                   ALLOW          DENY
                     |             |
                     v             v
              DispatcherServlet   403
                     |
                     v
                 Controller
                     |
                     v
                  Service
                     |
                     v
                Repository
                     |
                     v
                  Database
```

And authentication itself can use different architectures:

```text
                 AUTHENTICATION
                       |
          +------------+------------+
          |                         |
          v                         v
       SESSION                    JWT
          |                         |
          v                         v
   Session ID                  Signed Token
          |                         |
          v                         v
   Server-side state          Validate token
                                    |
                                    v
                              Extract identity
                                    |
                                    v
                               UserDetails
                                    |
                                    v
                               Authentication
```

The deepest mental model is:

> **Infrastructure explains how the request moves through the application.**
>
> **Security explains how the application establishes and enforces the identity and permissions of the requester while that request is moving through the pipeline.**
>
> **Sessions and JWT are different ways of establishing authentication state for subsequent requests.**
>
> **Spring Security provides the machinery that turns credentials into an authenticated identity, makes that identity available through the security context, and applies authorization rules before protected application logic executes.**

---

## Final Flow to Visualize

```text
CLIENT
  |
  | HTTP
  v
TOMCAT
  |
  v
SERVLET FILTER CHAIN
  |
  v
SPRING SECURITY
  |
  +-------------------------------+
  |                               |
  v                               v
AUTHENTICATION                 AUTHORIZATION
  |                               |
  v                               |
Who is this?                      |
  |                               |
  v                               |
Session OR JWT                    |
  |                               |
  v                               |
Authentication                    |
  |                               |
  v                               |
SecurityContext                   |
  |                               |
  +------------->-----------------+
                  |
                  v
             ALLOWED ?
              /     \
            YES      NO
             |        |
             v        v
       CONTROLLER     403
             |
             v
          SERVICE
             |
             v
        REPOSITORY
             |
             v
          DATABASE
```

This is the flow to keep in your head when learning every new Spring Security concept.
