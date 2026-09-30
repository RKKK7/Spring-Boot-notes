# Spring Boot Infrastructure & HTTP Request–Response Lifecycle

> A practical learning guide to understanding what happens to an HTTP request before, during, and after it reaches a Spring Boot controller.

## Table of Contents

1. Why This Topic Matters
2. The Big Picture
3. What Actually Receives an HTTP Request?
4. Servlet Container
5. Embedded Tomcat
6. What Is a Servlet?
7. Servlet vs Controller
8. How HTTP Becomes `HttpServletRequest`
9. Servlet Filter Chain
10. `DispatcherServlet`
11. `HandlerMapping` and `HandlerAdapter`
12. How a Controller Method Is Invoked
13. JSON to Java Object
14. Complete Request Lifecycle
15. Response Lifecycle
16. Concrete Example
17. Common Student Doubts
18. Important Classes and Interfaces
19. Mental Model
20. Interview Questions
21. Summary

---

# 1. Why This Topic Matters

When learning Spring Boot, it is easy to write:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

and think:

> A request comes in and Spring calls this method.

That is true at a high level, but a real HTTP request passes through several layers first.

```text
Client
  ↓
Network
  ↓
Tomcat
  ↓
Servlet infrastructure
  ↓
Filters
  ↓
DispatcherServlet
  ↓
HandlerMapping
  ↓
HandlerAdapter
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Database
```

Understanding this pipeline makes Spring MVC, Spring Security, filters, `HttpServletRequest`, `@RequestBody`, Jackson, exception handling, and REST APIs much easier to understand.

---

# 2. The Big Picture

Suppose a client sends:

```http
GET /api/users/42 HTTP/1.1
Host: example.com
Authorization: Bearer <token>
```

A useful conceptual model is:

```text
┌─────────────────────┐
│ Client / Browser    │
└──────────┬──────────┘
           │ HTTP request
           ▼
┌─────────────────────┐
│ Network / TCP       │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Embedded Tomcat      │
│ HTTP Connector       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Servlet Filter Chain │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ DispatcherServlet    │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ HandlerMapping       │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ HandlerAdapter       │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Controller           │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Service              │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Repository           │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Database             │
└─────────────────────┘
```

There are two useful categories here.

### Infrastructure

Infrastructure answers:

> **How does an HTTP request enter and move through the application?**

Examples:

- Tomcat
- Servlet container
- Servlet
- Filters
- `HttpServletRequest`
- `HttpServletResponse`
- `DispatcherServlet`

### Application logic

Application logic answers:

> **What should the application do with this request?**

Examples:

- Controller
- Service
- Repository
- Database operations

These layers are connected, but they are not the same thing.

---

# 3. What Actually Receives an HTTP Request?

### Common student doubt

> Does Spring Boot itself directly receive the HTTP request?

Not exactly.

A useful mental model is:

```text
Operating System
      ↓
TCP connection
      ↓
Tomcat
      ↓
Servlet infrastructure
      ↓
Spring MVC
      ↓
Controller
```

Spring Boot configures and starts the web server for you.

For a typical Spring Boot MVC application, that server is embedded Tomcat.

So the first major component handling the incoming HTTP connection is the web server/container, not your controller.

---

# 4. What Is a Servlet Container?

A **Servlet container** is software that manages Java servlets and their request/response lifecycle.

Common examples include:

- Tomcat
- Jetty
- Undertow

It is responsible for things such as:

- accepting HTTP connections
- parsing HTTP requests
- creating servlet request/response abstractions
- managing servlet lifecycle
- running filters
- invoking servlets
- managing request-processing threads
- sending responses back to clients

Conceptually:

```text
HTTP request
     ↓
Servlet container
     ↓
Parse HTTP
     ↓
Create HttpServletRequest
Create HttpServletResponse
     ↓
Run filters
     ↓
Invoke servlet
```

The container is therefore the bridge between the external HTTP world and the Java Servlet API.

---

# 5. Embedded Tomcat in Spring Boot

### Common student doubt

> Where is Tomcat? I did not install Tomcat separately.

Spring Boot commonly uses **embedded Tomcat** for web applications.

Conceptually:

```text
Your Java process
┌─────────────────────────────────────┐
│ Spring Boot                         │
│   ├── Spring MVC                    │
│   ├── Spring Security               │
│   └── Embedded Tomcat               │
│          └── HTTP server            │
└─────────────────────────────────────┘
```

When you run:

```bash
java -jar application.jar
```

the application starts and its embedded Tomcat server starts with it.

That is why a Spring Boot application can listen on something such as:

```text
http://localhost:8080
```

without requiring a separately installed Tomcat server.

---

# 6. What Is a Servlet?

A **Servlet** is a Java component managed by a servlet container that participates in HTTP request/response processing.

At a conceptual level:

```text
HTTP request
     ↓
Servlet container
     ↓
Servlet
     ↓
HTTP response
```

The Servlet API provides abstractions such as:

```java
HttpServletRequest
HttpServletResponse
```

Spring MVC uses an important servlet called:

```text
DispatcherServlet
```

This distinction is fundamental.

---

# 7. Servlet vs Controller

### Common misunderstanding

> Every class that handles an HTTP request is a servlet, right?

**No.**

Consider:

```java
@RestController
public class UserController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.getUsers();
    }
}
```

`UserController` is a Spring MVC controller. It is not a servlet.

Spring MVC has a central servlet:

```text
DispatcherServlet
```

The architecture is:

```text
HTTP request
      ↓
Tomcat
      ↓
Servlet Filter Chain
      ↓
DispatcherServlet     ← actual servlet
      ↓
Spring MVC infrastructure
      ↓
UserController        ← Spring controller
```

So do not imagine every controller as a tiny servlet.

A better mental model is:

```text
Many Controllers
       ↑
       │
DispatcherServlet
       ↑
Servlet Container
```

| Component | What it is |
|---|---|
| Tomcat | Servlet container / web server |
| `DispatcherServlet` | Spring MVC servlet |
| Filter | Servlet infrastructure component |
| Controller | Spring MVC component |
| Service | Business/application layer |
| Repository | Data-access layer |

---

# 8. How HTTP Becomes `HttpServletRequest`

Consider this request:

```http
POST /api/users HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
    "name": "Ravi",
    "email": "ravi@example.com"
}
```

At the network level, the HTTP message travels as bytes.

Tomcat receives those bytes.

Its HTTP connector parses the HTTP message and exposes request information through Servlet API objects such as:

```java
HttpServletRequest
```

and:

```java
HttpServletResponse
```

So:

```text
Raw HTTP bytes
      ↓
Tomcat HTTP connector
      ↓
HTTP parsing
      ↓
HttpServletRequest
HttpServletResponse
      ↓
Servlet/filter infrastructure
      ↓
Spring MVC
```

You normally do not manually construct these objects. The container creates and manages them for the request.

---

# 9. The Servlet Filter Chain

Before the request reaches `DispatcherServlet`, servlet filters can process it.

```text
Request
  ↓
Filter 1
  ↓
Filter 2
  ↓
Filter 3
  ↓
DispatcherServlet
  ↓
Controller
```

Filters are useful for cross-cutting request processing:

- authentication
- logging
- CORS
- request/response processing
- tracing
- security checks

A filter can also stop a request:

```text
Request
  ↓
Authentication Filter
  ↓
Token valid?
  ├── No  → reject
  └── Yes → continue
              ↓
        DispatcherServlet
```

This is also where Spring Security fits into the broader request architecture.

---

# 10. `DispatcherServlet`: Spring MVC's Front Controller

`DispatcherServlet` is Spring MVC's central servlet.

It does not contain your business logic.

Its job is to coordinate request processing.

A useful mental model:

> **DispatcherServlet is the traffic controller of Spring MVC.**

For example:

```text
                    DispatcherServlet
                           │
          ┌────────────────┼────────────────┐
          ↓                ↓                ↓
     UserController   OrderController   NoteController
```

It asks Spring MVC infrastructure:

> Which handler should process this request?

That leads to `HandlerMapping`.

---

# 11. `HandlerMapping` and `HandlerAdapter`

## 11.1 `HandlerMapping`

Suppose the request is:

```http
GET /api/users/42
```

and we have:

```java
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable Long id) {
    ...
}
```

`HandlerMapping` helps determine the appropriate handler.

Conceptually:

```text
GET /api/users/42
        ↓
HandlerMapping
        ↓
UserController.getUser(...)
```

### Shortcut

> **HandlerMapping = Find who handles the request.**

---

## 11.2 `HandlerAdapter`

Finding the handler and invoking it are separate responsibilities.

Conceptually:

```text
DispatcherServlet
       ↓
HandlerMapping
       ↓
Find handler
       ↓
HandlerAdapter
       ↓
Invoke handler
       ↓
Controller method
```

### Shortcut

> **HandlerAdapter = Invoke the selected handler.**

So:

```text
Mapping = Find
Adapter = Invoke
```

This separation allows Spring MVC to support different kinds of handlers without making `DispatcherServlet` directly know how to invoke every possible handler type.

---

# 12. How a Controller Method Is Actually Invoked

Consider:

```java
@PostMapping("/users")
public User createUser(
        @RequestBody CreateUserRequest request) {

    return userService.createUser(request);
}
```

The client sends:

```json
{
    "name": "Ravi",
    "email": "ravi@example.com"
}
```

Spring must do more than simply "call the method."

Conceptually:

```text
HTTP request
     ↓
DispatcherServlet
     ↓
Find controller method
     ↓
Resolve method arguments
     ↓
Read request body
     ↓
JSON → CreateUserRequest
     ↓
Invoke controller method
```

The controller receives a Java object rather than raw JSON.

---

# 13. How JSON Becomes a Java Object

### Common student doubt

> Who converts JSON into my DTO?

Usually Spring MVC uses an `HttpMessageConverter`.

For JSON, Jackson is commonly used.

```text
HTTP body
   ↓
JSON
   ↓
HttpMessageConverter
   ↓
Jackson
   ↓
Java object
```

For example:

```json
{
    "username": "ravi",
    "password": "secret"
}
```

can become:

```java
LoginRequest request
```

when the controller contains:

```java
@RequestBody LoginRequest request
```

A simplified sequence:

```text
Controller method needs LoginRequest
            ↓
Spring sees @RequestBody
            ↓
Select suitable HttpMessageConverter
            ↓
JSON converter uses Jackson
            ↓
Jackson deserializes JSON
            ↓
LoginRequest created
            ↓
Controller method invoked
```

The reverse process is serialization:

```text
Java object
    ↓
Jackson
    ↓
JSON
    ↓
HTTP response body
```

So:

- **Deserialization:** JSON → Java object
- **Serialization:** Java object → JSON

---

# 14. Complete Request Lifecycle

Suppose the client sends:

```http
POST /api/auth/login
Content-Type: application/json

{
    "username": "ravi",
    "password": "secret"
}
```

The conceptual request lifecycle is:

```text
1. Client
   ↓
2. Network / TCP
   ↓
3. Tomcat
   ↓
4. Tomcat HTTP Connector
   ↓
5. HttpServletRequest / HttpServletResponse
   ↓
6. Servlet Filter Chain
   ↓
7. DispatcherServlet
   ↓
8. HandlerMapping
   ↓
9. HandlerAdapter
   ↓
10. Message Conversion
    JSON → Java object
   ↓
11. Controller
   ↓
12. Service
   ↓
13. Repository
   ↓
14. Database
```

A more visual architecture:

```text
┌──────────────────────────┐
│ Client                   │
│ Sends HTTP request       │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Network / TCP            │
│ Request travels as bytes │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Tomcat                   │
│ Accepts HTTP connection  │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Servlet infrastructure   │
│ Request / response       │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Filter chain             │
│ Security / logging / etc │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ DispatcherServlet        │
│ Spring MVC entry point   │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ HandlerMapping            │
│ Find handler             │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ HandlerAdapter            │
│ Invoke handler           │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Controller               │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Service                  │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Repository               │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Database                 │
└──────────────────────────┘
```

---

# 15. Response Lifecycle

The response travels back through the application's response-processing infrastructure.

```text
Database
   ↓
Repository
   ↓
Service
   ↓
Controller
   ↓
Java object
   ↓
HttpMessageConverter
   ↓
Jackson
   ↓
JSON
   ↓
HttpServletResponse
   ↓
Tomcat
   ↓
Network
   ↓
Client
```

For example:

```java
User user = new User(42L, "Ravi");
```

can become:

```json
{
    "id": 42,
    "name": "Ravi"
}
```

and the HTTP response is conceptually:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": 42,
    "name": "Ravi"
}
```

Important: this is a mental model. Internally, Spring MVC has additional return-value handlers, converters, exception handling, and other infrastructure, so the response path is not literally a perfect reverse of every internal method call.

---

# 16. Concrete Example

Consider:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

The client sends:

```http
GET /api/users/42
```

## Step 1 — Client

The client creates an HTTP request.

## Step 2 — Network

The request travels through the network using TCP/IP.

Your controller is not involved yet.

## Step 3 — Tomcat

The request reaches embedded Tomcat.

Tomcat accepts the connection and processes the HTTP request.

## Step 4 — Servlet request

Tomcat exposes request information through:

```java
HttpServletRequest
```

and provides:

```java
HttpServletResponse
```

for the response.

## Step 5 — Filters

Configured filters can inspect or process the request.

For example:

```text
Security filter
      ↓
Logging filter
      ↓
Other filters
```

## Step 6 — DispatcherServlet

Spring MVC's `DispatcherServlet` receives the request.

It needs to determine:

> Which controller method handles `/api/users/42`?

## Step 7 — HandlerMapping

Spring finds:

```java
UserController.getUser(...)
```

because the class-level and method-level mappings match the request.

## Step 8 — HandlerAdapter

Spring uses a suitable `HandlerAdapter` to invoke the selected handler.

## Step 9 — Path variable resolution

Spring sees:

```java
@PathVariable Long id
```

and extracts:

```text
42
```

from:

```text
/api/users/42
```

It converts the value to:

```java
Long
```

## Step 10 — Controller

The controller executes:

```java
userService.getUser(42L);
```

## Step 11 — Service

The service performs business logic.

For example:

```java
return userRepository.findById(id)
        .orElseThrow(...);
```

## Step 12 — Repository

The repository interacts with the database.

Conceptually:

```sql
SELECT * FROM users WHERE id = 42;
```

## Step 13 — Response object

The database result becomes a Java object:

```java
User user
```

## Step 14 — Serialization

Spring uses an `HttpMessageConverter`, commonly backed by Jackson, to serialize the object.

```text
User object
    ↓
Jackson
    ↓
JSON
```

## Step 15 — HTTP response

The response is ultimately sent:

```text
User object
    ↓
JSON
    ↓
HTTP response
    ↓
Tomcat
    ↓
Network
    ↓
Client
```

---

# 17. Common Student Doubts and Misunderstandings

## Doubt 1 — "Is Spring Boot the web server?"

Not exactly.

Spring Boot is the framework/tooling that simplifies application startup and configuration. A Spring Boot web application commonly contains an embedded web server such as Tomcat.

Think:

```text
Spring Boot
    │
    ├── Configures application
    ├── Auto-configures components
    ├── Starts application
    │
    └── Starts embedded Tomcat
```

---

## Doubt 2 — "Is Tomcat only a web server?"

Tomcat is commonly described as both a web server and a Servlet container.

For Spring MVC, its Servlet-container role is especially important.

It can:

- accept HTTP requests
- manage servlets
- manage filters
- provide request/response objects
- manage request-processing threads
- send HTTP responses

---

## Doubt 3 — "Is every request handled by a servlet?"

At the Servlet API level, requests are dispatched through servlet infrastructure.

In a normal Spring MVC application, the important servlet is:

```text
DispatcherServlet
```

Your controllers are not individual servlets.

---

## Doubt 4 — "Does DispatcherServlet contain all my controller code?"

No.

It coordinates request processing.

Your application logic remains in:

```text
Controller
Service
Repository
```

Think of it as:

```text
Request
   ↓
DispatcherServlet
   ├── Which handler?
   ├── What arguments?
   ├── How should it be invoked?
   └── How should the result become a response?
```

---

## Doubt 5 — "Who creates HttpServletRequest?"

The servlet container creates and manages the request object for the current HTTP request.

Your application receives a reference to it through the Servlet API.

You normally do not create it yourself.

---

## Doubt 6 — "Why do I need HttpServletRequest if I can use @RequestBody, @PathVariable and @RequestParam?"

Because Spring MVC provides convenient abstractions on top of the lower-level Servlet request.

For example:

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    ...
}
```

is much easier than manually reading the URI from `HttpServletRequest`.

Spring extracts and converts the required data for you.

---

## Doubt 7 — "Who converts JSON into my DTO?"

Usually:

```text
HttpMessageConverter
       ↓
Jackson
       ↓
DTO
```

For example:

```java
@RequestBody LoginRequest request
```

tells Spring that the request body should be converted into a `LoginRequest`.

---

## Doubt 8 — "Does Jackson directly call my controller?"

No.

Jackson handles data conversion. It does not decide which controller handles the request.

A simplified sequence is:

```text
DispatcherServlet
      ↓
HandlerAdapter
      ↓
Argument resolution
      ↓
HttpMessageConverter
      ↓
Jackson
      ↓
Java argument
      ↓
Controller method
```

---

## Doubt 9 — "What is the difference between HandlerMapping and HandlerAdapter?"

Remember:

```text
HandlerMapping = Find
HandlerAdapter = Invoke
```

`HandlerMapping` determines the handler.

`HandlerAdapter` knows how to invoke the selected handler.

---

## Doubt 10 — "Why does Spring need DispatcherServlet if Tomcat already handles servlets?"

Because they solve different problems.

Tomcat provides the Servlet infrastructure:

```text
HTTP
  ↓
Servlet container
  ↓
Servlet
  ↓
HTTP response
```

Spring MVC provides the MVC infrastructure:

```text
Request
  ↓
Find controller
  ↓
Resolve arguments
  ↓
Invoke controller
  ↓
Convert result
  ↓
Response
```

---

## Doubt 11 — "Are filters part of Spring MVC?"

Servlet filters are part of the Servlet infrastructure.

Spring can register and use filters, and Spring Security heavily uses the Servlet filter chain.

A useful relationship is:

```text
Servlet infrastructure
        │
        ├── Filters
        │
        └── DispatcherServlet
                 │
                 └── Spring MVC
```

This distinction becomes important when learning Spring Security.

---

## Doubt 12 — "Does the request go directly from Tomcat to my controller?"

No.

A simplified request path is:

```text
Tomcat
  ↓
Filters
  ↓
DispatcherServlet
  ↓
HandlerMapping
  ↓
HandlerAdapter
  ↓
Controller
```

There are additional internal components, but this is an excellent learning model.

---

# 18. Important Classes and Interfaces

| Component | Main responsibility |
|---|---|
| Tomcat | Runs the web/Servlet infrastructure |
| Servlet container | Manages servlets, filters, requests and responses |
| `HttpServletRequest` | Represents the incoming servlet request |
| `HttpServletResponse` | Represents the outgoing servlet response |
| `Filter` | Processes requests/responses around downstream processing |
| `DispatcherServlet` | Central Spring MVC servlet |
| `HandlerMapping` | Finds the handler for a request |
| `HandlerAdapter` | Invokes the selected handler |
| Controller | Handles application-level HTTP operations |
| Service | Contains business/application logic |
| Repository | Performs data access |
| `HttpMessageConverter` | Converts HTTP bodies to/from Java objects |
| Jackson | Common JSON serialization/deserialization library |

---

# 19. Mental Model to Remember

If you remember only one diagram, remember this:

```text
                    OUTSIDE WORLD
                         │
                         ▼
                 HTTP / TCP / Network
                         │
                         ▼
                  ┌─────────────┐
                  │   Tomcat    │
                  │  Container  │
                  └──────┬──────┘
                         │
                         ▼
                  HttpServletRequest
                  HttpServletResponse
                         │
                         ▼
                  ┌─────────────┐
                  │   Filters   │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │ Dispatcher  │
                  │   Servlet   │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │ Handler     │
                  │ Mapping     │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │ Handler     │
                  │ Adapter     │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │ Controller  │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  Service    │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │ Repository  │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  Database   │
                  └─────────────┘
```

Now remember the two major questions.

### Infrastructure question

> **How does the HTTP request reach my code?**

```text
Network
 → Tomcat
 → Servlet infrastructure
 → Filters
 → DispatcherServlet
 → Spring MVC
 → Controller
```

### Application question

> **What does my application do with the request?**

```text
Controller
 → Service
 → Repository
 → Database
```

These are different questions, but they meet inside the same request lifecycle.

---

# 20. Interview Questions

## Beginner

### What is Tomcat?

Tomcat is a web server and Servlet container commonly used to run Spring Boot web applications.

### What is a servlet?

A servlet is a Java component managed by a Servlet container that participates in request/response processing.

### What is DispatcherServlet?

It is Spring MVC's central/front-controller servlet responsible for coordinating request processing and dispatching requests to appropriate handlers.

### Is a Spring controller a servlet?

No. A controller is a Spring MVC component. `DispatcherServlet` is the servlet that coordinates requests to controllers.

### What is HttpServletRequest?

It is the Servlet API representation of the current incoming HTTP request.

## Intermediate

### What does HandlerMapping do?

It determines which handler should process an incoming request.

### What does HandlerAdapter do?

It provides the mechanism for invoking the selected handler.

### How does JSON become a Java object?

Spring MVC selects an appropriate `HttpMessageConverter`, commonly using Jackson for JSON deserialization.

### How does a Java object become JSON?

Spring MVC uses an `HttpMessageConverter`, commonly backed by Jackson, to serialize the object into JSON.

### Why are filters important?

Filters allow request/response processing before the target servlet and can implement cross-cutting concerns such as security, logging, CORS, and tracing.

---

# 21. Summary

The goal is not to memorize class names. It is to understand the layers.

A Spring Boot HTTP request can be mentally modeled as:

```text
CLIENT
  ↓
NETWORK
  ↓
TOMCAT
  ↓
SERVLET INFRASTRUCTURE
  ↓
FILTER CHAIN
  ↓
DISPATCHERSERVLET
  ↓
HANDLERMAPPING
  ↓
HANDLERADAPTER
  ↓
CONTROLLER
  ↓
SERVICE
  ↓
REPOSITORY
  ↓
DATABASE
```

The response is processed back toward the client:

```text
DATABASE
  ↓
REPOSITORY
  ↓
SERVICE
  ↓
CONTROLLER
  ↓
MESSAGE CONVERSION
  ↓
HTTP RESPONSE
  ↓
TOMCAT
  ↓
NETWORK
  ↓
CLIENT
```

The most important conceptual distinction is:

> **Tomcat/Servlet infrastructure explains how the request enters and moves through the Java web application. Spring MVC explains how that request is mapped to and processed by your controller. Your controller/service/repository code explains what the application actually does.**

Once this foundation is clear, topics such as **Spring Security, JWT authentication, exception handling, REST APIs, CORS, interceptors, and observability/Actuator** become much easier to place in the architecture.

---

## Quick Revision Card

```text
HTTP request
     ↓
Tomcat receives it
     ↓
Servlet request/response objects
     ↓
Filters
     ↓
DispatcherServlet
     ↓
HandlerMapping → "Who handles this?"
     ↓
HandlerAdapter → "How do I invoke it?"
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
Database
     ↓
Result
     ↓
Jackson / HttpMessageConverter
     ↓
HTTP response
     ↓
Tomcat
     ↓
Client
```

**Core idea:**

> **The controller is not where the request starts. It is one component in a much larger request-processing pipeline.**
