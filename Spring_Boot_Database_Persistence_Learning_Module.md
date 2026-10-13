# Spring Boot Database Persistence: From Transaction to Hibernate

> **Learning Module**
>
> This module follows a real learning conversation: each concept is introduced when the previous concept creates a natural doubt. The goal is not to memorize definitions, but to build a mental model of what Spring Boot, JPA, Hibernate, and the database are actually doing.

---

## 1. Starting Point: What Is a Persistence Layer?

Before understanding transactions and Hibernate, we need to understand where database-related work belongs in a Spring Boot application.

A typical application can be viewed as:

```text
Client
   ↓
Controller
   ↓
Service
   ↓
Persistence Layer
   ↓
Database
```

The **persistence layer** is the part of the application responsible for interacting with persistent data.

It handles operations such as:

- Saving data
- Retrieving data
- Updating data
- Deleting data
- Executing database queries
- Managing the application's interaction with the database

A common Spring Boot implementation looks like:

```text
Entity
   ↓
Repository
   ↓
JPA
   ↓
Hibernate
   ↓
JDBC
   ↓
Database
```

### A useful distinction

The persistence layer is not simply "the repository."

A repository is one entry point into the persistence layer. The persistence layer can involve:

- Entity mappings
- Repositories / DAOs
- JPA
- Hibernate
- EntityManager
- Transaction management
- Database configuration

---

# 2. What Is an Entity?

A natural next question is:

> **If the persistence layer works with database data, what exactly is an entity?**

Consider:

```java
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String username;
    private String password;
}
```

This class represents persistent data.

Conceptually:

```text
Java                          Database

User class       ───────→     user table

id               ───────→     id
username         ───────→     username
password         ───────→     password
```

A precise way to express it is:

> An entity is a Java class mapped to persistent database data, and its instances can be managed by a JPA provider such as Hibernate.

Be careful with the wording:

- `User` → entity class
- `new User()` → an instance/object of that class
- A managed entity instance → an object currently being tracked by the persistence context

---

# 3. The First Big Question: What Is a Transaction?

Instead of starting with a textbook definition, imagine an Order Management System.

A customer places an order.

The application might need to:

```text
1. Create the order
2. Reduce inventory
3. Create a payment record
4. Create order history
```

Now imagine:

```text
Create order       ✅
Reduce inventory   ✅
Create payment     ❌
```

We don't want the database to end up with:

```text
Order created
Stock reduced
Payment missing
```

The operations should behave as one unit of database work.

This is where a **transaction** comes in.

Think of a transaction as a controlled database work session:

```text
START TRANSACTION
      ↓
Database operations
      ↓
Everything successful?
   ↙              ↘
 YES              NO
  ↓                ↓
COMMIT          ROLLBACK
  ↓                ↓
END              END
```

### The mental model

A transaction says:

> "I'm performing this unit of database work. If it succeeds, finish it successfully. If it fails according to the transaction's rollback rules, undo its database changes."

---

# 4. What Does a Transaction Actually Do?

Suppose:

```java
@Transactional
public void placeOrder(Order order) {

    orderRepository.save(order);

    productRepository.reduceStock(order.getProductId());

    paymentRepository.createPayment(order);
}
```

Visualize:

```text
START TRANSACTION
       ↓
Create order
       ↓
Reduce stock
       ↓
Create payment
       ↓
Everything successful?
       ↓
     COMMIT
```

If payment creation fails:

```text
START TRANSACTION
       ↓
Create order       ✅
       ↓
Reduce stock       ✅
       ↓
Create payment     ❌
       ↓
ROLLBACK
```

The database changes made by that transaction are undone.

---

# 5. When Does a Transaction End?

A transaction has two important successful/failed endings.

### Successful completion

```text
START
  ↓
Operations
  ↓
Success
  ↓
COMMIT
  ↓
TRANSACTION ENDS
```

### Failed completion

```text
START
  ↓
Operations
  ↓
Transaction failure
  ↓
ROLLBACK
  ↓
TRANSACTION ENDS
```

So the core mental model is:

> **A transaction ends when it is successfully committed or rolled back.**

---

# 6. `@Transactional` in Spring Boot

When we write:

```java
@Transactional
public void placeOrder() {
    ...
}
```

Spring's transaction management infrastructure creates the transaction boundary around that method.

Conceptually:

```text
Spring
  ↓
START TRANSACTION
  ↓
execute placeOrder()
  ↓
success?
 ↙       ↘
YES      NO
 ↓        ↓
COMMIT  ROLLBACK
 ↓        ↓
END      END
```

We don't normally write the transaction-start/commit/rollback code manually.

---

# 7. A Common Doubt: Is One HTTP Request One Transaction?

No.

This is an important distinction.

An HTTP request and a database transaction are different concepts.

Think:

```text
HTTP REQUEST
     ↓
Controller
     ↓
Service
     ↓
@Transactional
     ↓
TRANSACTION
```

A request **may trigger** a transaction.

But:

```text
1 HTTP request ≠ automatically 1 transaction
```

A request can involve:

### Zero transactions

```text
GET /hello
   ↓
Controller
   ↓
"Hello"
   ↓
HTTP response
```

No database work may be involved.

### One transaction

```text
HTTP request
    ↓
Controller
    ↓
@Transactional service
    ↓
ONE transaction
    ↓
Many database operations
    ↓
COMMIT
    ↓
Response
```

### Multiple transactions

A single request can also involve more than one transaction if the application is structured that way.

Therefore:

> **HTTP request is a web/application concept. Transaction is a database-work concept. They can overlap, but they are not the same thing.**

---

# 8. The Next Question: What Is a Persistence Context?

Once transactions are understood, another term starts appearing everywhere:

- "in the current persistence context"
- "loaded into the persistence context"
- "managed by the persistence context"
- "initial state when loaded"
- "changes made before the persistence context ends"

Instead of memorizing a definition, visualize Hibernate having a workspace.

Suppose the database contains:

```text
USER TABLE

id   username   age
1    Ravi       22
```

Your application executes:

```java
User user = userRepository.findById(1L).get();
```

Hibernate retrieves the data.

But it does not simply hand the object to your application and forget about it.

The entity becomes associated with a **persistence context**.

Visualize:

```text
             HIBERNATE / JPA

       ┌────────────────────────┐
       │   PERSISTENCE CONTEXT  │
       │                        │
       │   User #1              │
       │   username = Ravi      │
       │   age = 22             │
       │                        │
       └────────────────────────┘
                  ↓
              DATABASE
```

The persistence context is the workspace in which entity instances are managed.

---

# 9. Why Does Hibernate Need This Workspace?

Suppose:

```java
User user = userRepository.findById(1L).get();

user.setAge(23);
```

Hibernate knows that the `User` object it loaded belongs to the current persistence context.

Conceptually, it can keep track of:

```text
Original state:
age = 22

Current state:
age = 23
```

It can therefore detect:

```text
22 → 23
```

Something changed.

This is called **dirty checking**.

---

# 10. Managed Entity and Dirty Checking

A managed entity is an entity instance currently being tracked by the persistence context.

Visualize:

```text
Database
   ↓
Hibernate
   ↓
Persistence Context
   ↓
User object
   ↓
MANAGED
```

Then:

```java
user.setAge(25);
```

The persistence context conceptually contains:

```text
Original state:
age = 22

Current state:
age = 25
```

Hibernate can detect that the entity is dirty.

This is why you often don't need to write:

```java
userRepository.update(user);
```

You can simply change the managed entity:

```java
user.setAge(25);
```

and Hibernate can later synchronize that change with the database.

---

# 11. "Current Persistence Context" — What Does That Mean?

When someone says:

> "The entity is managed in the current persistence context."

Think:

```text
HIBERNATE'S CURRENT WORKSPACE

┌─────────────────────────────┐
│     PERSISTENCE CONTEXT     │
│                             │
│  User #1                    │
│  Order #50                  │
│  Product #10                │
│                             │
└─────────────────────────────┘
```

The entity is currently inside that workspace and being tracked.

If you change a managed entity:

```java
user.setAge(25);
```

Hibernate can detect the change.

---

# 12. "Initial State When Loaded Into the Persistence Context"

Suppose the database says:

```text
User #1
age = 22
```

After loading:

```text
Persistence Context

User #1

Original state:
age = 22

Current state:
age = 22
```

Then:

```java
user.setAge(25);
```

Conceptually:

```text
Persistence Context

User #1

Original state:
age = 22

Current state:
age = 25
```

Hibernate can compare these states and determine that the entity changed.

That is the mental model behind dirty checking.

---

# 13. A Critical Distinction: Flush Is Not Commit

Once Hibernate detects changes, it eventually needs to synchronize them with the database.

That is where **flush** comes in.

Visualize:

```text
Managed Entity
      ↓
Change detected
      ↓
Entity is dirty
      ↓
FLUSH
      ↓
Hibernate generates SQL
      ↓
Database
```

For example:

```sql
UPDATE user
SET age = 25
WHERE id = 1;
```

But:

```text
FLUSH ≠ COMMIT
```

After a flush:

```text
Transaction       → still active
Persistence Context → still active
Entity             → still managed
```

Flush means:

> Synchronize the changes tracked by the persistence context with the database.

Commit means:

> Successfully finish the transaction.

---

# 14. Visualizing Flush

Suppose:

```text
Persistence Context

User #1
age = 25
```

Hibernate flushes:

```text
Persistence Context
       ↓
      FLUSH
       ↓
Hibernate generates SQL
       ↓
UPDATE user SET age = 25 WHERE id = 1
       ↓
Database
```

But the workspace is still there:

```text
Persistence Context

User #1
age = 25
       ↓
STILL MANAGED
```

The transaction can continue.

---

# 15. Transaction and Persistence Context Are Related — But Not the Same

This is one of the most important concepts.

### Transaction

Think:

> "What unit of database work am I performing?"

```text
START
  ↓
database operations
  ↓
COMMIT / ROLLBACK
  ↓
END
```

### Persistence Context

Think:

> "Which entity instances is JPA/Hibernate currently managing and tracking?"

```text
OPEN / ASSOCIATED
  ↓
User
Order
Product
  ↓
track entity state
  ↓
dirty checking
  ↓
context ends
```

So:

```text
TRANSACTION
    ↓
controls database work

PERSISTENCE CONTEXT
    ↓
manages entity instances
```

They are closely related in the common Spring Boot setup, but they are not the same thing.

---

# 16. How Are They Connected?

In the typical transaction-scoped Spring Boot + JPA setup:

```text
@Transactional method
       ↓
Transaction starts
       ↓
Persistence Context is associated
       ↓
Entities are loaded
       ↓
Entities become managed
       ↓
Changes are tracked
       ↓
Flush
       ↓
Commit / Rollback
       ↓
Transaction ends
       ↓
Persistence Context is no longer associated / ends
       ↓
Entities become detached
```

Do not turn this into an absolute rule that every transaction always creates one brand-new persistence context. Persistence-context lifecycle depends on how it is configured.

---

# 17. When Does a Persistence Context Close?

This was an important doubt.

A persistence context does not simply close because:

```text
"the HTTP request ended"
```

Its lifecycle depends on how it is managed.

For the common **transaction-scoped** setup, visualize:

```text
Transaction starts
      ↓
Persistence Context associated
      ↓
Entities managed
      ↓
COMMIT / ROLLBACK
      ↓
Transaction ends
      ↓
Transaction-scoped Persistence Context
no longer remains associated
      ↓
Managed entities become detached
```

Commit and closing are not literally the same operation:

```text
COMMIT ≠ explicit close()
```

They are separate lifecycle concepts that are commonly coordinated by Spring.

A persistence context can also be explicitly closed if an `EntityManager` was manually created:

```java
EntityManager em =
    entityManagerFactory.createEntityManager();

...

em.close();
```

In normal Spring Boot applications, Spring generally manages this lifecycle for you.

There are also configurations such as **Open EntityManager in View (OSIV)** where the persistence context can remain available beyond the service transaction and through more of the web request lifecycle.

---

# 18. Managed vs Detached

Before the persistence context ends:

```text
Persistence Context
       ↓
User object
       ↓
MANAGED
```

After the context ends:

```text
User object
       ↓
DETACHED
```

Detached does not mean deleted.

The Java object can still exist.

It simply means:

> The entity is no longer being managed by that persistence context.

---

# 19. Rollback and the Java Object

A subtle but important point:

Rollback affects the **database transaction**.

It does not necessarily rewind the fields of your Java object.

Suppose:

```text
Database:
age = 22
```

Then:

```java
user.setAge(25);
```

Then the transaction rolls back.

The database can return to:

```text
age = 22
```

But the Java object may still have:

```text
user.age = 25
```

Think of two worlds:

```text
JAVA WORLD

User object
age = 25


DATABASE WORLD

User row
age = 22
```

Rollback rolls back the database transaction. It does not mean Java automatically rewinds every object field.

---

# 20. JPA, EntityManager, and Hibernate

Now we can connect the technologies.

A common confusion is:

> "Are JPA, EntityManager, and Hibernate three different things that do the same job?"

No.

They exist at different levels.

## JPA

**JPA is a specification.**

It defines standard concepts and APIs for persistence, including:

- Entities
- Persistence contexts
- EntityManager
- Entity lifecycle
- JPQL

Think:

```text
JPA
↓
Rules + standards + interfaces
```

---

## EntityManager

JPA defines an interface called `EntityManager`.

It provides standard operations such as:

```java
persist()
find()
merge()
remove()
flush()
```

The EntityManager provides the application with operations for interacting with entities and the persistence context.

Think:

```text
EntityManager
       ↓
Operations on entities
       ↓
Persistence Context
```

---

## Hibernate

Hibernate is a **JPA implementation**.

JPA says:

> "Here is the standard."

Hibernate says:

> "I will implement that standard and perform the actual persistence work."

Conceptually:

```text
JPA
 ↓
defines EntityManager API
 ↓
Hibernate provides implementation
 ↓
Hibernate performs persistence work
```

A precise statement is:

> **JPA defines the persistence model and APIs; Hibernate provides an implementation of those JPA APIs.**

---

# 21. What Is Spring Data JPA?

Now another layer appears.

In a Spring Boot application, instead of manually using:

```java
entityManager.find(User.class, 1L);
```

we often write:

```java
userRepository.findById(1L);
```

For example:

```java
public interface UserRepository
        extends JpaRepository<User, Long> {
}
```

Spring Data JPA provides repository methods such as:

```text
findById()
findAll()
save()
delete()
```

So the layers look like:

```text
OUR CODE
   ↓
Spring Data JPA Repository
   ↓
EntityManager
   ↓
Hibernate
   ↓
JDBC
   ↓
Database
```

---

# 22. Who Provides `findById()`?

A common student doubt is:

> "Are `findById()` and `findAll()` methods provided by EntityManager?"

Not directly.

`findById()` and `findAll()` are provided by the **Spring Data JPA repository abstraction**.

Spring Data JPA internally uses JPA APIs such as the EntityManager to perform the underlying persistence operations.

So:

```text
userRepo.findById(1L)
        ↓
Spring Data JPA
        ↓
EntityManager
        ↓
Hibernate
        ↓
Database
```

Do not say:

> "`findById()` is an EntityManager method."

Instead:

> **"`findById()` is a Spring Data JPA repository method that internally uses JPA's EntityManager and Hibernate to perform the persistence operation."**

---

# 23. `save()` Is an Interesting Example

When we write:

```java
userRepository.save(user);
```

Spring Data JPA determines how to handle the entity.

Conceptually, it may end up using:

```java
entityManager.persist(user);
```

for a new entity, or:

```java
entityManager.merge(user);
```

when the entity is not considered new.

So:

```text
userRepository.save(user)
        ↓
Spring Data JPA
        ↓
decides how to handle the entity
        ↓
EntityManager
        ↓
persist() / merge()
        ↓
Hibernate
        ↓
Persistence Context
        ↓
Database
```

The exact internal behavior depends on how Spring Data JPA determines whether the entity is new.

---

# 24. HQL / JPQL and SQL

Another common question:

> "Hibernate uses Hibernate Query Language, right? How does that become PostgreSQL SQL?"

There are two related query languages:

- **JPQL** — standardized by JPA
- **HQL** — Hibernate's query language, with Hibernate-specific capabilities

For example:

```java
SELECT u
FROM User u
WHERE u.username = :username
```

Notice that we are talking about:

```text
User
username
```

rather than:

```text
users
user_name
```

Those are Java/entity concepts.

Hibernate uses the entity mapping to understand how these concepts correspond to database tables and columns.

For example:

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "user_name")
    private String username;
}
```

Hibernate knows:

```text
Java                         Database

User              ───────→   users

User.id           ───────→   id

User.username     ───────→   user_name
```

So a query such as:

```text
SELECT u
FROM User u
WHERE u.username = :username
```

can ultimately become SQL similar to:

```sql
SELECT id, user_name
FROM users
WHERE user_name = ?;
```

The exact SQL depends on Hibernate, mappings, configuration, and the target database.

---

# 25. What Is a Hibernate Dialect?

A database dialect can be thought of as:

> **The rules Hibernate uses to express database operations appropriately for a particular database.**

Different databases have differences in:

- SQL syntax
- Pagination
- Sequences
- Identity generation
- Data types
- Functions
- Locking
- Other database-specific features

Conceptually:

```text
Hibernate
   ↓
Database information / dialect
   ↓
PostgreSQL-compatible SQL
   ↓
JDBC
   ↓
PostgreSQL
```

Modern Hibernate can often determine the database dialect automatically from the database connection, so explicit dialect configuration is not always necessary.

---

# 26. How Does Hibernate Map Objects to Rows?

This is the core of **ORM — Object-Relational Mapping**.

There are two worlds:

```text
OBJECT WORLD                 RELATIONAL WORLD

Java                         Database

User object       ←────→     users row

user.id           ←────→     id

user.username     ←────→     user_name
```

The mapping is defined using entity metadata:

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "user_name")
    private String username;
}
```

Hibernate reads this metadata and understands the mapping.

Conceptually:

```text
ENTITY METADATA

User
│
├── table → users
│
├── id
│    ├── Java field → id
│    └── DB column → id
│
└── username
     ├── Java field → username
     └── DB column → user_name
```

---

# 27. Database Row → Java Object

Suppose PostgreSQL returns:

```text
id = 1
user_name = Ravi
```

Hibernate uses the mapping:

```text
users.id
     ↓
User.id

users.user_name
     ↓
User.username
```

and creates/populates a Java entity:

```java
User user = new User();

user.setId(1L);
user.setUsername("Ravi");
```

Conceptually:

```text
Database row
     ↓
JDBC
     ↓
Hibernate
     ↓
ORM mapping
     ↓
Java Entity
     ↓
Persistence Context
     ↓
MANAGED
```

---

# 28. Java Object → Database Row

The process works in the other direction too.

Suppose:

```java
User user = new User();
user.setUsername("Ravi");
```

Hibernate knows:

```text
user.username
      ↓
users.user_name
```

When the entity is persisted and the changes are flushed, Hibernate can generate SQL such as:

```sql
INSERT INTO users (user_name)
VALUES (?);
```

So ORM works in both directions:

```text
Java object
    ↓
ORM mapping
    ↓
Database row
```

and:

```text
Database row
    ↓
ORM mapping
    ↓
Java object
```

---

# 29. How Does the Persistence Context Fit Into Object Mapping?

Suppose User #10 is already managed:

```text
Persistence Context

User #10
username = Ravi
```

Another operation requests User #10.

Hibernate can recognize that the same database identity is already represented by a managed entity in the current persistence context.

So the persistence context is not just a place to store random objects.

It also helps Hibernate manage entity identity and state during the persistence context's lifetime.

---

# 30. The Complete Database Request → Response Cycle

Now all the concepts can be combined.

Suppose the client sends:

```http
PUT /users/1
Content-Type: application/json

{
    "name": "Ravi Kumar",
    "age": 25
}
```

The controller may look like:

```java
@PutMapping("/users/{id}")
public User updateUser(
        @PathVariable Long id,
        @RequestBody UserRequest request) {

    return userService.updateUser(id, request);
}
```

The service:

```java
@Transactional
public User updateUser(Long id, UserRequest request) {

    User user = userRepository.findById(id)
            .orElseThrow();

    user.setName(request.getName());
    user.setAge(request.getAge());

    return user;
}
```

The conceptual cycle is:

```text
HTTP REQUEST
      ↓
Controller
      ↓
Service
      ↓
@Transactional
      ↓
Transaction starts
      ↓
Persistence Context is associated
      ↓
Spring Data JPA Repository
      ↓
EntityManager
      ↓
Hibernate
      ↓
Persistence Context lookup
      ↓
If necessary: SQL generation
      ↓
JDBC
      ↓
Database
      ↓
Database row
      ↓
Hibernate ORM mapping
      ↓
Java Entity
      ↓
Persistence Context
      ↓
Entity is MANAGED
      ↓
Java code changes entity
      ↓
Dirty checking
      ↓
Entity becomes dirty
      ↓
FLUSH
      ↓
Hibernate generates SQL
      ↓
JDBC
      ↓
Database
      ↓
COMMIT
      ↓
Transaction ends
      ↓
Transaction-scoped Persistence Context ends
      ↓
Entity becomes DETACHED
      ↓
Controller receives result
      ↓
Spring/Jackson converts Java object to JSON
      ↓
HTTP RESPONSE
```

---

# 31. Putting the Architecture Together

The complete mental model is:

```text
                         CLIENT
                           │
                           │ HTTP Request
                           ↓
                    SPRING BOOT APP
                           │
                           ↓
                      CONTROLLER
                           │
                           ↓
                        SERVICE
                           │
                     @Transactional
                           │
                           ↓
                     TRANSACTION
                           │
                           ↓
                 PERSISTENCE CONTEXT
                           │
                           ↓
                  SPRING DATA JPA
                           │
                           ↓
                    ENTITY MANAGER
                           │
                           ↓
                      HIBERNATE
                    ↙             ↘
             ORM Mapping       Query Handling
                    │             │
                    └──────┬──────┘
                           ↓
                   SQL / Dialect
                           ↓
                         JDBC
                           ↓
                    JDBC DRIVER
                           ↓
                      DATABASE
                           │
                           ↓
                     Result Rows
                           │
                           ↓
                      HIBERNATE
                           │
                     ORM Mapping
                           ↓
                    Java Entities
                           │
                           ↓
                 Persistence Context
                           │
                    Dirty Checking
                           │
                         Flush
                           │
                         Commit
                           ↓
                    Transaction End
                           ↓
                 Persistence Context Ends
                           ↓
                     Entity Detached
                           ↓
                      Controller
                           ↓
                        Jackson
                           ↓
                      JSON Response
                           ↓
                         CLIENT
```

---

# 32. The Mental Models to Keep

Instead of memorizing definitions, remember these relationships.

### Transaction

```text
"What unit of database work am I performing?"

START
 ↓
database work
 ↓
COMMIT / ROLLBACK
 ↓
END
```

### Persistence Context

```text
"Which entity instances is JPA/Hibernate currently managing?"

Entity
 ↓
Persistence Context
 ↓
MANAGED
 ↓
State tracked
 ↓
Dirty checking
```

### JPA

```text
"The standard/specification for persistence."
```

### EntityManager

```text
"The standard JPA API through which entities and the persistence context are interacted with."
```

### Spring Data JPA

```text
"The repository abstraction that gives convenient methods such as
findById(), findAll(), save(), and delete()."
```

### Hibernate

```text
"The JPA implementation/ORM engine that performs the actual persistence work."
```

### JDBC

```text
"The standard Java API used to communicate with the database."
```

### JDBC Driver

```text
"The database-specific implementation that allows JDBC to communicate with a particular database."
```

### ORM

```text
Java Object
     ↕
Database Row
```

### Dirty Checking

```text
Original state
     ↓
Current state
     ↓
Difference detected
     ↓
Entity is dirty
```

### Flush

```text
Persistence Context
     ↓
Synchronize changes
     ↓
SQL
```

### Commit

```text
Successful transaction completion
```

### Rollback

```text
Undo database changes made by the failed transaction
```

---

# 33. Final Mental Picture

The entire module can be reduced to this:

```text
HTTP REQUEST
     ↓
Controller
     ↓
Service
     ↓
@Transactional
     ↓
TRANSACTION
     ↓
PERSISTENCE CONTEXT
     ↓
Spring Data JPA
     ↓
EntityManager
     ↓
Hibernate
     ↓
ORM + Query Processing
     ↓
Dialect / SQL
     ↓
JDBC
     ↓
JDBC Driver
     ↓
DATABASE
     ↓
Rows
     ↓
Hibernate
     ↓
Java Entity
     ↓
Persistence Context
     ↓
Managed Entity
     ↓
Dirty Checking
     ↓
Flush
     ↓
SQL
     ↓
Database
     ↓
Commit / Rollback
     ↓
Transaction Ends
     ↓
Persistence Context Ends
     ↓
Entity becomes Detached
     ↓
Controller
     ↓
Jackson
     ↓
JSON
     ↓
HTTP RESPONSE
```

The most important distinction of the entire module is:

> **A transaction controls a unit of database work, while a persistence context manages entity instances during that work. JPA defines the standard persistence model and APIs, EntityManager provides the standard operations, Spring Data JPA gives us convenient repository abstractions, and Hibernate implements the JPA behavior and translates the object-oriented persistence operations into database operations through JDBC.**

---

## What This Module Builds Toward

Once this mental model is clear, the next Hibernate concepts become much easier to understand:

- Entity lifecycle: **Transient → Managed → Detached → Removed**
- `EntityManager.persist()`
- `EntityManager.merge()`
- `detach()`
- `clear()`
- `flush()`
- Lazy vs eager loading
- `LazyInitializationException`
- Entity relationships
- Cascade operations
- `orphanRemoval`
- N+1 query problem
- Fetch joins
- Transaction propagation
- Isolation levels
- Optimistic and pessimistic locking

These are not isolated topics. They are extensions of the transaction + persistence-context model developed throughout this module.
