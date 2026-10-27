##  1. Introduction to Spring, Servlets and the Ecosystem



#### The Production Scenario

It is a Big Billion Days-style sale on Flipkart. Millions of clients (Android app, iOS app, React web, internal inventory service) hit `GET /products/9812` and `POST /orders` at the same time. Your Core Java skills let you write `ProductService.getProduct(long id)`. But:

- Who listens on a port, 24x7, without the process exiting?
- Who turns raw bytes from the network into "GET /products/9812"?
- Who calls `getProduct(9812)` for that URL, and builds the HTTP response?
- Who makes sure request #2 does not wait for request #1?

This lecture traces how the Java world answered these four questions: **raw sockets → servlets → Spring → Spring Boot**.



### 1. Client–Server and HTTP Basics

```
 Client                                   Server (a computer hosting the app)
 (browser / mobile app /        HTTP Request
  Postman / another service) ──────────────────────▶  process (auth, DB, logic)
                            ◀──────────────────────
                                 HTTP Response
```

- **Client** = whoever sends the request. Browser, mobile app, React app, Postman, or *another backend service*.
- **Server** = whoever receives, processes and responds. A server is just a computer running your app.
- In microservices, `OrderService` calling `PaymentService` makes OrderService a **client** for that call.
- **HTTP** = the rulebook: request structure, response structure, methods, status codes. **HTTPS** = HTTP carried over TLS (encrypted in transit).

#### Request anatomy (4 parts)

```
POST /login HTTP/1.1                 <- method + path (endpoint) + version
Host: api.example.in                 <- header
Content-Type: application/json       <- header
                                     <- blank line
{"email":"a@b.com","password":"..."} <- body
```

| Part | Purpose |
|---|---|
| Method | What you want to do (GET, POST, PUT, PATCH, DELETE) |
| URL / path | Which resource (`/courses`), plus host |
| Headers | Key-value metadata (`Accept`, `Content-Type`, `Authorization`) |
| Body | Payload (usually JSON) for POST/PUT/PATCH |

#### Response anatomy (3 parts)

```
HTTP/1.1 200 OK                      <- status line (status code)
Content-Type: application/json       <- headers
                                     <- blank line
{"message":"Login successful"}       <- body
```

#### Methods

| Method | Intent | Safe | Idempotent |
|---|---|---|---|
| GET | Retrieve | Yes | Yes |
| POST | Create / trigger action | No | No |
| PUT | **Replace** the whole resource | No | Yes |
| PATCH | **Partially** update | No | Not guaranteed |
| DELETE | Remove | No | Yes |

> **Precision note:** The lecture says PUT "updates" and PATCH updates "a specific piece". Correct version: PUT replaces the full representation, PATCH sends only the changed fields. Idempotency (same request repeated = same end state) is what matters for retries, for example a UPI app retrying after a timeout. GET carries no body by convention; the spec does not forbid one, but its semantics are undefined, so do not rely on it.

#### Status codes (the ones the lecture mentioned, corrected)

| Code | Meaning |
|---|---|
| 200 | OK |
| 201 | Created (a resource was created) |
| 404 | Not Found |
| 500 | Internal Server Error |
| 503 | Service Unavailable (overloaded / down / in maintenance) |

> **Precision note:** The lecture (transcript) says "2001" for Created and "503" for internal server error. Correct: **201** is Created, **500** is Internal Server Error, **503** is Service Unavailable. Interviewers do check 500 vs 503.



### 2. Why Core Java Alone Does Not Make a Web App

#### 2.1 Program flow vs server flow

```
Normal Java program:   start ──▶ run instructions ──▶ exit

Web server process:    start ──▶ wait for request ──▶ respond ──┐
                                        ▲                       │
                                        └───────────────────────┘   (never exits)
```

A website is a program that **stays up** and loops on "wait → handle → respond". Faking this with `while (true)` works for the lifetime part only. It does not solve the mapping problem below.

#### 2.2 Core Java *can* do networking: `java.net`

`java.net` has existed since JDK 1.0. `ServerSocket` binds to a port and accepts TCP connections.

- **Port** = how the OS routes incoming traffic to the right process (8080 for your app, 5432 for Postgres, etc.). IP identifies the machine, port identifies the process.
- `localhost` = `127.0.0.1` = your own machine. `localhost:8080` = "my own machine, port 8080".

#### 2.3 The problem: JVM sees bytes, not HTTP

```
Network ──▶ bytes ──▶ Socket.getInputStream() ──▶ your code
                       "GET /courses HTTP/1.1\r\nHost: ...\r\n\r\n"
                       (just a stream of bytes to the JVM)
```

If you use raw sockets, **you** must do all of this:

| Chore | Done by you with raw sockets |
|---|---|
| Bind port, accept connections | `ServerSocket`, `accept()` loop |
| Read bytes | `InputStream` / `BufferedReader` |
| Parse the HTTP request | Manual string splitting |
| Map URL → method | `if/else` chain on path |
| Build a valid HTTP response | Manual status line, headers, body |
| Handle many users at once | Manual threads / thread pool |

This is the "boilerplate" the lecture refers to. Every line here is infrastructure, not business logic.



### 3. Servlets and the Servlet Container

- **Servlet** = a Java class (extends `HttpServlet`) that handles requests for a URL mapping.
- **Servlet container** = the runtime that hosts servlets. Tomcat, Jetty and Undertow are the common ones. Colloquially "the server".
- The container runs *inside a JVM process*. Servlets run *inside the container*.
- Servlet API arrived in the late 1990s (the lecture says 1997) as part of what became Java EE. It is now **Jakarta EE**.

#### What the container does for you

```
 Client
   │  HTTP bytes
   ▼
┌──────────────────────── Tomcat (servlet container) ───────────────────────┐
│ 1. Opens port (8080), accepts connection                                   │
│ 2. Reads bytes, parses HTTP → HttpServletRequest                           │
│ 3. Picks a worker thread from its pool                                     │
│ 4. Matches URL → servlet                                                   │
│ 5. Calls servlet.service() → doGet()/doPost()                              │
│ 6. Takes HttpServletResponse, serialises status + headers + body           │
│ 7. Writes bytes back to client, returns thread to pool                     │
└────────────────────────────────────────────────────────────────────────────┘
        │ calls                      ▲ returns
        ▼                            │
   Your Servlet (HttpServletRequest in, HttpServletResponse out)
```

You write only step 5's body: business logic.

> **Gotcha (skipped in lecture):** The container, not you, creates servlet instances and calls them, typically one shared instance serving many threads concurrently. So **servlets must not hold per-request mutable state in fields**. This is the same thread-safety issue you will meet again with Spring singleton beans.

> **Precision note:** Tomcat's request handling is usually described as an acceptor + NIO poller feeding a **worker thread pool** (Spring Boot default max 200 threads, `server.tomcat.threads.max`). It is not "one new thread per request" and not unlimited. Slow downstream calls can exhaust the pool, a real production failure mode (requests queue, then time out and you see 503s).



### 4. Why Spring Came After Servlets

At enterprise scale, servlet-oriented code tends to become **tightly coupled**: classes create their own dependencies with `new`, so everything is wired to everything.

| Tight coupling | Loose coupling (Spring's goal) |
|---|---|
| `OrderServlet` does `new PaymentService()` | `OrderService` *receives* a `PaymentService` from outside |
| Hard to swap/test (cannot mock) | Swap implementations, inject mocks |
| Scaling the codebase = scaling the pain | Components change independently |

Spring's core ideas (details in upcoming videos):

- **Inversion of Control (IoC):** the framework creates and manages objects and controls their lifecycle, not your code.
- **Dependency Injection (DI):** the framework hands each object the collaborators it needs.
- **Beans:** the objects the Spring container manages.



### 5. The Spring Ecosystem

Spring is an **ecosystem of projects**, not "one framework". Spring.io lists use cases such as web apps, microservices, reactive apps, cloud, event-driven systems, serverless, batch and GenAI.

```
┌──────────────────────────────────────────────────────────────────┐
│                          SPRING BOOT                             │  auto-config, starters,
│        (opinionated defaults + embedded server + packaging)      │  embedded Tomcat
├────────────┬──────────────┬────────────┬──────────────┬──────────┤
│ Spring MVC │ Spring Data  │ Spring AOP │   Spring     │ Spring   │
│ (web)      │ (persistence)│ (cross-    │   Security   │ AI, ...  │
│            │              │  cutting)  │ (authn/z)    │          │
├────────────┴──────────────┴────────────┴──────────────┴──────────┤
│                          SPRING CORE                             │
│                  IoC container · DI · Beans                      │
└──────────────────────────────────────────────────────────────────┘
```

| Layer | Role |
|---|---|
| Spring Core | IoC container, DI, bean lifecycle: the foundation of everything |
| Spring MVC | Build web apps/REST APIs; runs on the Servlet API |
| Spring Data | Data-access abstraction (JPA, MongoDB, Redis, ...) |
| Spring AOP | Cross-cutting concerns (logging, transactions) via aspects |
| Spring Security | Authentication and authorisation |
| Spring Boot | Auto-configuration + starters + embedded server on top of the above |

> **Precision note:** The lecture calls the middle modules "skills" (transcription noise for *modules*) and says Boot is "based on Spring MVC". More precisely, **Spring Boot is built on the Spring Framework as a whole**. It auto-configures whichever modules are on your classpath. Spring MVC is one of them, and the reactive stack (WebFlux) is another. Boot does not replace Spring; you still need Core, MVC, Data, AOP and Security concepts to debug it.

### Spring MVC and servlets

Spring MVC does not eliminate servlets. It hides them behind **one front-controller servlet, `DispatcherServlet`**, which receives every request and routes it to your `@Controller` methods. That is the "URL → method mapping" problem from section 2.3, solved once and generically.

### Spring Boot: what "opinionated" means

- Boot **assumes sensible defaults** (embedded Tomcat on 8080, JSON serialisation, a datasource if a driver is on the classpath).
- You override only what you disagree with.
- Name = *boot up quickly*.

> **Added (not in lecture):** Boot ships an **embedded Tomcat** inside the app JAR, so you run `java -jar app.jar` and the container starts with it. You do not deploy WARs into a separately installed Tomcat. This is why Spring Boot apps fit Docker and Kubernetes so naturally.



### 6. The Persistence Stack: JDBC → JPA → Hibernate → Spring Data JPA

```
 Your code
    │ calls repository methods (no SQL)
    ▼
 Spring Data JPA   ← Spring's abstraction/boilerplate remover
    │
    ▼
 JPA               ← a *specification* (rulebook / interfaces)
    │ implemented by
    ▼
 Hibernate         ← a JPA *implementation* (ORM)
    │ generates SQL, uses
    ▼
 JDBC              ← low-level API: connections, SQL strings, ResultSets
    │
    ▼
 Database
```

| Layer | What it is | You write |
|---|---|---|
| JDBC | Java's low-level DB API | SQL strings by hand |
| JPA | Spec for object-relational mapping | Entities + annotations |
| Hibernate | Most popular JPA provider | (usually configured, not called directly) |
| Spring Data JPA | Repository abstraction over JPA | Interfaces, not implementations |

> **Precision note:** "Spring JPA" in the lecture means **Spring Data JPA**. Spring Data is wider than JPA (MongoDB, Redis, and others). Hibernate is the *default* JPA provider, not the only one (EclipseLink is another). Since you come from MongoDB: Spring Data MongoDB gives you the same repository style without JPA/JDBC underneath.



### 7. Monolith vs Microservices

- **Microservices is an architecture style, not a Spring module.**
- Spring Boot can build either.

```
Monolith:                         Microservices:
┌───────────────────────┐         ┌────────┐ HTTP ┌─────────┐
│ users + orders +      │         │ Order  │─────▶│ Payment │
│ payments in ONE app   │         │ Service│      │ Service │
└───────────────────────┘         └───┬────┘      └─────────┘
                                      │ HTTP
                                  ┌───▼────┐
                                  │  User  │
                                  │ Service│
                                  └────────┘
```

Each service is itself a client and server (ties back to section 1).



### 8. Trace: One Request, End to End (Spring Boot, preview)

| Step | What happens |
|---|---|
| 1 | Client sends `GET /courses` to `host:8080` |
| 2 | Embedded Tomcat accepts the connection, a worker thread is assigned |
| 3 | Bytes are parsed into `HttpServletRequest` |
| 4 | `DispatcherServlet` finds the controller method mapped to `GET /courses` |
| 5 | Controller calls service → repository → (Spring Data JPA → Hibernate → JDBC) → DB |
| 6 | Return value is serialised to JSON, Tomcat writes the HTTP response |
| 7 | Worker thread is released to the pool |



### Code

#### WRONG: raw `ServerSocket`, serial handling

```java
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;

public class RawServer {
    public static void main(String[] args) throws IOException {
        try (ServerSocket server = new ServerSocket(8080)) {
            while (true) {
                Socket s = server.accept();   // blocks until a client connects
                handle(s);                    // WRONG: runs on main thread, one client at a time
            }
        }
    }

    static void handle(Socket s) throws IOException {
        try (s;
             BufferedReader in = new BufferedReader(new InputStreamReader(s.getInputStream()));
             OutputStream out = s.getOutputStream()) {
            String requestLine = in.readLine();            // "GET /courses HTTP/1.1"
            if (requestLine == null) return;
            String path = requestLine.split(" ")[1];       // manual parsing
            boolean found = path.equals("/courses");       // manual routing
            String body = found ? "[\"Java\",\"Spring\"]" : "{\"error\":\"not found\"}";
            byte[] bytes = body.getBytes(StandardCharsets.UTF_8);
            String head = "HTTP/1.1 " + (found ? "200 OK" : "404 Not Found") + "\r\n"
                        + "Content-Type: application/json\r\n"
                        + "Content-Length: " + bytes.length + "\r\n"
                        + "Connection: close\r\n\r\n";    // manual response building
            out.write(head.getBytes(StandardCharsets.UTF_8));
            out.write(bytes);
        }
    }
}
```

#### BETTER (but still manual): hand off to a thread pool

```java
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class PooledRawServer {
    public static void main(String[] args) throws IOException {
        ExecutorService pool = Executors.newFixedThreadPool(50);   // bounded, not unbounded threads
        Runtime.getRuntime().addShutdownHook(new Thread(pool::shutdown));
        try (ServerSocket server = new ServerSocket(8080)) {
            while (true) {
                Socket s = server.accept();
                pool.submit(() -> {
                    try { RawServer.handle(s); }               // same handle() as above
                    catch (IOException e) { e.printStackTrace(); }
                });
            }
        }
    }
}
```

#### RIGHT (servlet): you only write the handler

```java
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebServlet("/courses")                       // container maps URL -> this class
public class CourseServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.setStatus(HttpServletResponse.SC_OK);
        resp.setContentType("application/json");
        resp.getWriter().write("[\"Java\",\"Spring\"]");   // no parsing, no threads, no raw HTTP
    }
}
```

> **Note:** Servlet 5+/Jakarta EE 9+ (and therefore Spring Framework 6 / Spring Boot 3) use the `jakarta.servlet.*` package. Older tutorials show `javax.servlet.*`. Mixing them is a classic "class not found" error.

#### Preview (next videos): same endpoint in Spring

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;

@RestController
class CourseController {
    @GetMapping("/courses")
    List<String> courses() {
        return List.of("Java", "Spring");
    }
}
```



#### Quick Self-Check

1. **Why can't a plain `main()` program serve a website out of the box?**
   *Answer:* It exits when its instructions finish, and nothing maps incoming HTTP to your methods. A server must stay alive, listen on a port, parse HTTP, route to code and serve many users concurrently.

2. **What does `ServerSocket` give you, and what does it not?**
   *Answer:* It gives you TCP connections on a port and byte streams. It does not understand HTTP, so parsing, routing, response building and concurrency are all manual.

3. **Servlet vs servlet container?**
   *Answer:* A servlet is your class handling a URL. The container (Tomcat/Jetty/Undertow) listens on the port, parses HTTP, manages threads, creates and calls servlets, and writes the response.

4. **PUT vs PATCH? Which are idempotent?**
   *Answer:* PUT replaces the whole resource, PATCH applies a partial change. PUT (and GET, DELETE) are idempotent. PATCH is not guaranteed to be.

5. **Is Spring Boot a replacement for Spring Framework?**
   *Answer:* No. It is an opinionated layer (auto-configuration, starters, embedded server) on top of the Spring Framework. The underlying Core/MVC/Data/AOP/Security concepts still apply.

6. **How do JPA, Hibernate, JDBC and Spring Data JPA relate?**
   *Answer:* JPA is the spec, Hibernate is an implementation, Hibernate uses JDBC underneath, and Spring Data JPA is a repository abstraction on top of JPA.

7. **Difference between 500 and 503? What is 201?**
   *Answer:* 500 means the server hit an unexpected error. 503 means the service is temporarily unavailable (overloaded or down). 201 means a resource was created.

8. **Is "microservices" a Spring module?**
   *Answer:* No. It is an architectural style. Spring Boot can build monolithic or microservice apps.



#### Golden Rules / Checklist

- [ ] Client = whoever sends the request, and a backend service can be one
- [ ] Server process stays alive; it is "wait → handle → respond" in a loop
- [ ] JVM sees bytes, not HTTP; a container or framework does the parsing
- [ ] Servlet = your class; servlet container = the thing running it
- [ ] Never keep per-request mutable state in servlet or singleton-bean fields
- [ ] Thread pool is bounded, so slow dependencies can exhaust it
- [ ] Spring Core (IoC/DI) is the base; every other Spring module sits on it
- [ ] Spring Boot adds defaults, it does not replace Spring
- [ ] JPA is a spec; Hibernate implements it; JDBC sits underneath
- [ ] 201 Created, 500 server error, 503 unavailable
- [ ] Boot 3 / Spring 6 means `jakarta.*`, not `javax.*`



### Practice Questions

#### Basic

1. Draw the client–server flow and label the HTTP request and response.

2. List the four parts of an HTTP request and the three parts of a response.

3. What is the difference between `localhost`, `127.0.0.1` and a port number?

4. Name three Spring modules and one sentence on what each does.

#### Intermediate

5. Write a raw `ServerSocket` loop that returns 404 for unknown paths. Which of your lines are infrastructure vs business logic?

6. Explain why `while (true)` alone does not make a program a web server.

7. Why does a servlet container need a thread pool? What happens when every worker thread is blocked on a slow DB call?

8. Explain tight vs loose coupling with an `OrderService` / `PaymentService` example.

#### Advanced / Interview-style

9. Spring Boot's role: what would you have to configure manually if you built a Spring MVC app *without* Boot?

10. Explain the full path from `GET /courses` arriving at the NIC to a JSON response, naming Tomcat, `DispatcherServlet`, controller and (optionally) Hibernate/JDBC.

11. A retry after timeout creates two orders in your Razorpay-style checkout. Which HTTP method semantics are involved, and how would you design for idempotency?

12. Why is "Spring Boot is Spring" a better mental model than "Spring Boot replaced Spring"? Give two examples of debugging that require knowing the underlying modules.

## 2. Your First Spring Boot App (Hello World)



#### The Production Scenario

You join a Zomato-style backend team. Your first ticket: a new `restaurant-search` service must expose `GET /ping` so the load balancer's health check can mark the instance healthy before it is registered.

The ALB listens on **443 (HTTPS)**. Your app listens on **8080**. A developer who does not understand DNS, ports and reverse proxies will ask "why does `curl localhost:8080/ping` work but the public URL times out?". A developer who does not understand Initializr will spend a day on folder layout and dependency versions before writing one line of logic.

This lecture covers both: the network path to your app, and the fastest way to get an app to answer.



### 1. How a Request Actually Reaches Your App

#### 1.1 The lecture's simplification vs reality

Lecture 01 wrote `GET www.example.in/hello` straight to the server. Real flow:

```
 Browser (client)
    │ 1. "What is the IP of shop.example.in?"
    ▼
   DNS ──────────▶ returns 203.0.113.10
    │
    │ 2. HTTPS request to 203.0.113.10:443   (browser adds :443 automatically)
    ▼
 ┌──────────────────────────── Server side ─────────────────────────────┐
 │  Reverse proxy (nginx / ALB / ingress)   listens on 443              │
 │        │ 3. forwards to 127.0.0.1:8080 (or pod IP:8080)              │
 │        ▼                                                             │
 │  Spring Boot app (embedded Tomcat) listens on 8080 → GET /hello      │
 └──────────────────────────────────────────────────────────────────────┘
```

#### 1.2 Terms

| Term | Meaning |
|---|---|
| IP address | Identifies a device on a network (IPv4 / IPv6) |
| DNS | Resolves a domain name to an IP address |
| Port | Identifies *which process* on that machine gets the traffic |
| `localhost` / `127.0.0.1` | Loopback: "this machine"; traffic never leaves it |
| Reverse proxy | Sits in front of the app, accepts public traffic and forwards to the app's internal port |

#### 1.3 Well-known ports

| Scheme | Default port |
|---|---|
| HTTP | 80 |
| HTTPS | 443 |
| Spring Boot (Tomcat) default | 8080 |

The browser fills in 80/443 itself. That is why users never type a port. Your app typically listens on 8080 because binding to ports below 1024 normally needs elevated privileges, so a proxy maps 443 → 8080.

> **Precision note:** The lecture says anyone in the world can reach your laptop via your public IP. In practice home/office machines sit behind NAT and firewalls, so inbound traffic does not reach them. That is why you test with `localhost`. Also, **TLS termination** (decrypting HTTPS) usually happens at the reverse proxy, so the app behind it often speaks plain HTTP on 8080.

> **Added (DevOps tie-in):** Docker's `-p 9090:8080` is the same idea as a reverse proxy: map an outer port to the app's inner port. Inside a container the app must listen on all interfaces (Boot's default), not just loopback.



### 2. Spring Initializr (start.spring.io)

### 2.1 Why it exists

Building a Java web app from scratch means deciding:

- folder structure
- build file and dependency list
- Spring Boot version and Java version
- package structure

None of that is business logic. Initializr takes your choices and gives you a ready **zip skeleton**: unzip, open in IDE, write logic.

### 2.2 Options chosen in the lecture

| Option | Choice | Notes |
|---|---|---|
| Project | **Maven** | Alternatives: Gradle (Groovy/Kotlin DSL) |
| Language | Java | Kotlin/Groovy also supported |
| Spring Boot version | **4.0.x** (stable) | Avoid SNAPSHOT and RC |
| Packaging | **Jar** | War only for deploying into an external servlet container |
| Configuration | `properties` | `yaml` is the alternative |
| Java version | 21 | Must be at least the minimum Boot requires |
| Dependency | **Spring Web** | Spring MVC + REST + embedded Tomcat |

### 2.3 Version labels

| Label | Meaning | Use? |
|---|---|---|
| `SNAPSHOT` | Development build, still changing | No |
| `M1`, `M2` (Milestone) | Early preview | No |
| `RC` | Release Candidate: almost final, not released | No |
| (no suffix) | General Availability, stable | Yes |

Pick the latest stable (GA) version of the line you want.

### 2.4 Metadata fields (deferred by the lecturer)

Group / Artifact / Name / Package name. Defaults are fine for now. In short:

- **Group:** reverse domain of the organisation (`com.example`)
- **Artifact:** project name (`demo`)
- Together with a version they identify the project in Maven: `group:artifact:version`

### 2.5 Dependencies

- A **dependency** = an external library your code uses (a MySQL connector, a JSON library, and so on).
- Build tools **download them for you** and manage the versions.
- **Spring Web** is a *starter*: one dependency that pulls in a compatible set of libraries (Spring MVC, JSON support, embedded Tomcat, logging).

> **Precision note:** The lecture calls Maven/Gradle "project management tools". More exact: they are **build automation and dependency management tools** (compile, test, package, resolve dependencies). The next lecture covers them.

> **Gotcha:** Starter names changed across Boot generations (Boot 4 modularised and renamed several starters). Do not copy dependency names from older tutorials; check the `pom.xml` Initializr generated for you.



### 3. Project Anatomy

```
demo/
├── pom.xml                         <- Maven build file (dependencies, versions, plugins)
├── mvnw, mvnw.cmd, .mvn/           <- Maven wrapper (build without installing Maven)
├── src/
│   ├── main/
│   │   ├── java/com/example/demo/
│   │   │   └── DemoApplication.java    <- entry point (has main())
│   │   └── resources/
│   │       └── application.properties  <- app configuration
│   └── test/java/com/example/demo/     <- tests
└── target/                         <- build output (generated, not committed)
```

| Path | Purpose |
|---|---|
| `src/main/java` | Production code |
| `src/main/resources` | Config files, static assets, templates |
| `src/test/java` | Tests |
| `pom.xml` | What the project is and what it depends on |

#### Entry point

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

It is still plain Java: `main()` is where execution starts. `SpringApplication.run(...)` boots Spring. `@SpringBootApplication` is explained in upcoming videos.



### 4. What Happens When You Run It

```
main()
  └─ SpringApplication.run()
        ├─ creates the Spring container
        ├─ sees Spring Web on the classpath → starts embedded Tomcat
        └─ Tomcat binds port 8080, starts listening
main() returns, but the JVM stays alive (Tomcat's non-daemon threads keep it running)
```

Proof in the logs:

```
Tomcat initialized with port 8080 (http)
Starting Service [Tomcat]
Tomcat started on port 8080 (http)
```

#### Contrast with Lecture 01 flow

| Plain Java program | Spring Boot web app |
|---|---|
| `main()` finishes → JVM exits | `main()` finishes, but the server threads keep the JVM alive |
| You call your methods | The framework calls your methods |

#### Visiting an unmapped URL

Before any endpoint exists, `http://localhost:8080/` shows the **Whitelabel Error Page**.

- The server is up and reachable.
- No handler matches the URL, so Boot's default error page appears with status **404**.
- It proves the server runs; it is not a startup failure.

> **Precision note:** The lecture reads the message as "no explicit mapping for /error". The practical meaning is simply **404 Not Found: nothing is mapped to this URL**.



### 5. First Endpoint

#### 5.1 Steps

1. Create a class in the same package (or a sub-package) as `DemoApplication`.
2. Annotate with `@RestController`.
3. Write a normal method returning `String`.
4. Annotate with `@GetMapping("<path>")`.

#### 5.2 What each annotation means (preview level)

| Annotation | Meaning (for now) |
|---|---|
| `@RestController` | This class handles web requests; return values go straight into the HTTP response body |
| `@GetMapping("hello")` | Call this method for `GET /hello` |

> **Precision note:** The lecture calls a controller a "gateway". More accurate: a controller is the web-layer class that Spring MVC routes HTTP requests to. `@RestController` is `@Controller` + `@ResponseBody`: the returned object is written directly as the response body instead of being treated as a view name.

#### 5.3 Request trace

| Step | What happens |
|---|---|
| 1 | Browser sends `GET /hello` to `localhost:8080` |
| 2 | Tomcat accepts the connection, parses HTTP, assigns a worker thread |
| 3 | Spring MVC looks up which method is mapped to `GET /hello` |
| 4 | Spring invokes `HelloController.hello()` on the instance it manages (you never call `new`) |
| 5 | Returned `String` is written as the response body, status 200 |
| 6 | Browser renders it |

You never called `hello()` from `main()`. This is the URL → method mapping problem from Lecture 01, solved by the framework.

#### 5.4 The HTML tag trick

Returning `"<h1>Hello World</h1>"` renders as a heading in the browser. It works because the browser's `Accept` header asks for HTML and Spring writes the string back as-is.

> **Gotcha:** Do not rely on this for APIs. For machine clients, return JSON (objects/records), which comes in later videos.



### 6. Configuration: Changing the Port

`src/main/resources/application.properties`:

```properties
server.port=9090
```

- Restart, then the logs show `Tomcat started on port 9090`; `localhost:8080` is now unreachable.
- It is **Tomcat** that listens on the port, not "your Spring Boot code".

> **Added (DevOps tie-in):** Spring Boot reads configuration from several sources, and an environment variable (`SERVER_PORT=9090`) overrides the file. That is how you change the port per environment in Docker/Kubernetes without rebuilding.

> **Gotcha:** `Port 8080 was already in use` means another process (or a previous run of your app) still owns the port. Stop it or change `server.port`.



### 7. Why You Cannot Skip the Spring Fundamentals

| Situation | What you need |
|---|---|
| `/bye` returns a 404 despite your method | Knowing how mapping and component scanning work |
| Wondering what `@RestController` really does | Spring Core (beans) + Spring MVC |
| Reading `SpringApplication.run()` internals | Spring Core / Boot auto-configuration |

Boot hides configuration, it does not remove it. Debugging means understanding what was hidden.

#### Questions this lecture leaves open (to be answered in the series)

| Question | Where it leads |
|---|---|
| Who started Tomcat? | `SpringApplication.run()` + auto-configuration |
| How did the URL map to the method? | Spring MVC internals (`DispatcherServlet`) |
| What are `@SpringBootApplication`, `@RestController`, `@GetMapping`? | Spring Core and MVC |
| What do Group/Artifact/Package mean, and what is `pom.xml`? | Maven |



### Code

#### WRONG: calling the method yourself

```java
@RestController
public class HelloController {
    @GetMapping("hello")
    public String hello() { return "Hello World"; }
}

// in main()
System.out.println(new HelloController().hello()); // plain Java call, no HTTP involved
```

#### RIGHT: let the framework call it

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello World";
    }

    @GetMapping("/bye")
    public String greetBye() {   // method name need not match the path
        return "Bye";
    }
}
```

#### WRONG: `void` return

```java
@GetMapping("/hello")
public void hello() {            // nothing is written to the response body
    System.out.println("Hello World");   // goes to the server console, not the browser
}
```

#### WRONG: controller outside the scanned package

```java
package com.other.web;           // outside com.example.demo

@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() { return "Hello World"; }   // never registered → 404
}
```

#### RIGHT: same package or a sub-package

```java
package com.example.demo.web;    // sub-package of the @SpringBootApplication class

@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() { return "Hello World"; }
}
```

> **Added (not in the lecture):** Boot discovers your classes by **component scanning**, starting from the package of the `@SpringBootApplication` class. Controllers placed outside that package are silently ignored.



#### Quick Self-Check

1. **A client hits `https://shop.example.in/hello`. Trace what happens before the app sees the request.**
   *Answer:* DNS resolves the domain to an IP. The browser adds port 443 (HTTPS default). The reverse proxy on 443 forwards to the app's port (for example 8080).

2. **Why do users not type 8080 or 443?**
   *Answer:* Browsers use standard ports (80 for HTTP, 443 for HTTPS) by default. A reverse proxy maps the public port to the app's internal port.

3. **What does Spring Initializr give you, and why not set up by hand?**
   *Answer:* A pre-configured project (build file, folder layout, versions, dependencies). Manual setup is boilerplate with no business value and is easy to get wrong.

4. **Why avoid SNAPSHOT and RC versions?**
   *Answer:* SNAPSHOT is a moving development build. RC is not yet an official release. Both can have bugs or change behaviour. Use GA releases.

5. **Jar vs War?**
   *Answer:* Jar is a Java archive; Boot's executable jar embeds the server (`java -jar app.jar`). War (web archive) is for deploying into an external servlet container.

6. **`main()` returns immediately. Why does the app keep running?**
   *Answer:* Embedded Tomcat's non-daemon threads keep the JVM alive. The JVM exits only when all non-daemon threads finish.

7. **You see the Whitelabel Error Page at `/`. Is the app broken?**
   *Answer:* No. The server is running; no handler is mapped to `/`, so it returns a 404 error page.

8. **Your `@GetMapping` method never fires. Name two likely causes.**
   *Answer:* The class sits outside the component-scanned package; the class is missing `@RestController`. Also check the path, HTTP method and port, and restart the app.



#### Golden Rules / Checklist

- [ ] DNS resolves name → IP; the port picks the process
- [ ] HTTP defaults to 80, HTTPS to 443, Boot/Tomcat to 8080
- [ ] Use stable (GA) Spring Boot versions, never SNAPSHOT or RC for real work
- [ ] Generate projects from start.spring.io rather than hand-building them
- [ ] Put code under `src/main/java`, config under `src/main/resources`
- [ ] Keep controllers in the same package or a sub-package of the `@SpringBootApplication` class
- [ ] Return `String` (not `void`) when you want the text in the response body
- [ ] Restart the app after changes unless you use dev tooling that reloads automatically
- [ ] Set `server.port` in properties, or override with `SERVER_PORT` per environment
- [ ] Do not call controller methods yourself; the framework does it
- [ ] Read the startup logs: they show the port and what started



### Practice Questions

#### Basic

1. Create a Boot project from Initializr (Maven, Java, Jar, Spring Web) and run it. What do the logs say about Tomcat and the port?

2. Add `GET /hello` returning "Hello World" and `GET /bye` returning "Bye".

3. Change the port to 9090 with `application.properties` and verify the old port stops working.

4. Explain SNAPSHOT, RC and GA in one line each.

#### Intermediate

5. Make a controller method name different from its path (`greetBye` for `/bye`). Why does it still work?

6. Move the controller to a package outside `com.example.demo`. What happens and why?

7. Override the port with an environment variable (`SERVER_PORT`) without editing the properties file. Which one wins?

8. Draw the DNS → reverse proxy → app flow for `https://api.example.in/ping` with the app on 8080.

#### Advanced / Interview-style

9. Explain why `main()` can return while the JVM keeps running. What would change if Tomcat used only daemon threads?

10. A request to `/hello` returns 404 though the method exists. Give a debugging checklist (package, annotation, path, HTTP method, port, restart, context path).

11. Explain "Spring Boot hides configuration" with two concrete things Boot configured for you in this lecture. How would you find out what exactly it configured?

12. Why would a team package a Boot service as a Jar rather than a War in a Docker/Kubernetes world?

## 3. Maven (Build Tool, Dependencies, Lifecycle)



#### The Production Scenario

At a Razorpay-style company, 12 microservices share one internal library, `payments-common` (money types, signature verification). Without a build tool:

- someone zips the library and posts it on Slack, and every service has a different copy
- Service A uses `payments-common` v1.4 with an old MySQL driver, while Service B has v1.6 with a new one
- the app compiles locally, then crashes in production with `NoClassDefFoundError` because a *transitive* dependency was never copied

Maven's answer: publish `payments-common:1.6` once to the company repository; every service declares one `<dependency>` block; Maven fetches the exact version and everything it needs.



### 1. JARs, Libraries and Applications

#### 1.1 What a JAR is

```
javac: Order.java, User.java, Payment.java  ──▶  Order.class, User.class, Payment.class
                                                    │ package
                                                    ▼
                                              app.jar  (a ZIP)
                                              ├── org/example/Order.class
                                              ├── org/example/User.class
                                              ├── application.properties, images ...
                                              └── META-INF/MANIFEST.MF
```

- JAR = **Java ARchive**: a ZIP of `.class` files, resources and metadata.
- Sending 100+ loose `.class` files, or a hand-made ZIP, loses folder structure and resources. A JAR is the standard unit for sharing.

#### 1.2 Library vs application

| | Library | Application |
|---|---|---|
| Has `main()` | Usually no | Yes |
| Runnable on its own | No | Yes |
| Used by | Other code (as a dependency) | End users / servers |
| Example | MySQL connector, Hibernate | Your Spring Boot service |

Any library you use inside your app becomes a **dependency** of that app.

#### 1.3 Thin JAR vs executable ("fat") JAR

| | Thin JAR | Executable (fat) JAR |
|---|---|---|
| Contains | Only your classes | Your classes + all dependency JARs |
| Run with | Needs a classpath | `java -jar app.jar` |
| Typical use | Libraries | Spring Boot applications |
| Dependency info | Listed in the `pom.xml` bundled with the JAR | Nested inside the JAR |

> **Precision note:** The lecture says an application's JAR automatically contains its dependencies. Plain `mvn package` does **not** do that; it produces a thin JAR. Spring Boot gets an executable JAR because of the **`spring-boot-maven-plugin`** in your `pom.xml`, which repackages the JAR with its dependencies nested inside (Boot's own loader reads them). Without that plugin, `java -jar` fails with missing classes.

#### 1.4 Classpath

- The **classpath** is the ordered list of directories and JARs the JVM searches for classes.
- Your compiled code is on it, along with each dependency JAR.
- **First match in order wins.** Two versions of the same class on the classpath cause hard-to-debug behaviour ("JAR hell").

> **Precision note:** The lecture says the JVM checks "your code first, then the JARs". Precisely: it searches in classpath order, and class loaders delegate to their parent first. Tools like Maven build the classpath for you.



### 2. The Problem Maven Solves

```
Manual approach                           Problems
─────────────                             ────────────────────────────────────────
Google "mysql connector jar" → download   1. Manual effort per JAR (100s in real apps)
Copy into project, add to classpath       2. Version mismatches (Spring 7 ↔ Boot 4)
Library needs protobuf → download it too  3. Transitive dependencies must be found by hand
Share your lib → write a README of deps   4. No standard way to tell consumers what you need
```

A **transitive dependency** is a dependency of your dependency: you need A, A needs B, so B comes along.

> **Precision note:** The lecture's version-mismatch example garbles the numbers. The relevant fact: **Spring Boot 4 is built on Spring Framework 7; Boot 3 on Spring 6**. Mixing generations breaks at startup or runtime.



### 3. What Maven Is

**Maven = build automation and dependency management tool** (the lecture says "project management tool"). Based on **convention over configuration**: follow the standard layout and Maven needs little configuration.

| Job | Details |
|---|---|
| 1. Standard folder structure | Same layout in every project and IDE |
| 2. Compile | All sources in one command |
| 3. Package | Builds the JAR (or WAR) |
| 4. Resolve and download dependencies | Direct and transitive, from repositories |
| 5. Run the build lifecycle | Test, verify, install, deploy |

Alternatives: Gradle, and the IDE's own build system. A team should **standardise on one** so the layout, build and dependency handling are identical on every machine and in CI.



### 4. Standard Directory Layout

```
maven-demo/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/org/example/Main.java        <- production code
│   │   └── resources/                        <- non-Java files (config, static, templates)
│   └── test/
│       ├── java/org/example/MainTest.java    <- tests mirror the main package layout
│       └── resources/
└── target/                                   <- build output (generated, git-ignored)
    ├── classes/                              <- compiled .class files, same package layout
    └── maven-demo-1.0-SNAPSHOT.jar           <- built artifact
```

| Path | Rule |
|---|---|
| `src/main/java` | Production `.java` files |
| `src/main/resources` | Properties, images, templates |
| `src/test/java` | Test code, **same package structure** as the class under test |
| `target/` | Disposable; `mvn clean` deletes it |

**Package names become folders.** `org.example` → `org/example`. The IDE shows `org.example` on one line only while each folder has a single child.



### 5. `pom.xml` (Project Object Model)

The single most important file: it tells Maven what the project is, what it depends on and how to build it.

#### 5.1 Anatomy

| Element | Meaning |
|---|---|
| `<project>` | Root element |
| `xmlns`, `xsi:schemaLocation` | Schema: which tags are legal (an unknown tag is flagged by the IDE) |
| `<modelVersion>` | Version of the **POM format** (4.0.0), not the Maven version |
| `<groupId>` | Organisation / namespace |
| `<artifactId>` | Project (artifact) name |
| `<version>` | Project version |
| `<packaging>` | `jar` (default) or `war` |
| `<properties>` | Key-value settings, reusable as `${key}` |
| `<dependencies>` | External libraries |
| `<build><plugins>` | Plugins that do lifecycle work |
| `<parent>` | POM to inherit from |
| `<repositories>` | Extra places to download from |

#### 5.2 Coordinates: GAV

```
groupId     : org.example           <- reverse domain (in.coderarmy for coderarmy.in)
artifactId  : maven-demo            <- project name
version     : 1.0-SNAPSHOT
                │
                └─ file name:  artifactId-version.jar  →  maven-demo-1.0-SNAPSHOT.jar
```

- **G + A + V uniquely identify** an artifact worldwide. Reverse-domain `groupId` is the convention for uniqueness.
- Every dependency you declare is identified the same way.
- Libraries are therefore also called **artifacts**.

#### 5.3 `-SNAPSHOT`

| Version | Meaning |
|---|---|
| `1.0-SNAPSHOT` | Work in progress; may change under the same version |
| `1.0` | Release; **immutable** once published |

Drop `-SNAPSHOT` when you cut a release, so consumers get a stable, reproducible artifact.

#### 5.4 Properties

```xml
<properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <author.name>Aditya</author.name>
</properties>
```

- Custom keys are allowed; reuse with `${author.name}`.
- Reserved keys are read by plugins (`maven.compiler.release`, `project.build.sourceEncoding`).

> **Gotcha:** The lecture's IDE template sets `maven.compiler.source/target`. Prefer **`maven.compiler.release`** (it also checks the JDK API level). A Spring Boot parent already exposes `java.version` for this.

#### 5.5 Dependencies and scope

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>9.5.0</version>
</dependency>
```

| Scope | Available at | Typical use |
|---|---|---|
| `compile` (default) | Compile, test, runtime | Most libraries |
| `runtime` | Test, runtime (not compile) | JDBC drivers |
| `test` | Test only | JUnit, Mockito |
| `provided` | Compile + test, not packaged | Servlet API supplied by a container |

> **Precision note:** The lecture only mentions `compile` and `test`. The JDBC driver is normally `runtime`, since your code compiles against the JDBC interfaces, not the driver classes.

#### 5.6 Transitive dependencies

- You declare `mysql-connector-j`; Maven also pulls what it needs (for example protobuf).
- The IDE's Maven tool window shows the tree. On the command line use `mvn dependency:tree`.
- If two paths want different versions of the same artifact, Maven picks the **nearest** one in the dependency tree (first declared wins ties). Use `<exclusions>` to cut an unwanted transitive dependency.

#### 5.7 How to find coordinates and choose a version

- Search the Maven repository index, copy the `<dependency>` snippet, paste it into `pom.xml`, then reload.
- The lecture uses **mvnrepository.com**. Note it is a *third-party index*; the official repository is **Maven Central** (search.maven.org).
- **Version choice:** latest stable release is fine in general. The lecture advises "a few versions back, with more usages". The better checks are: not a pre-release, **no known vulnerabilities**, compatible with your framework version.
- Hibernate 6+ coordinates are `org.hibernate.orm:hibernate-core` (older tutorials show `org.hibernate`).

> **Added (Spring Boot):** With the Spring Boot parent you usually **omit the version**. The parent's managed dependency list (a BOM) supplies tested, compatible versions. Overriding it casually is how version mismatches get back in.

#### 5.8 Parent POM, Super POM, Effective POM

```
Super POM (built into Maven, implicit parent of every POM)
   ▲
spring-boot-dependencies  (version management)
   ▲
spring-boot-starter-parent
   ▲
your pom.xml                    <- inherits, can override
```

| Term | Meaning |
|---|---|
| Super POM | Maven's built-in root POM; default repositories, plugin versions, build defaults |
| Parent POM | POM you inherit from through `<parent>` |
| **Effective POM** | The merged result Maven actually uses: Super POM + parent chain + your POM + active profiles |

Inheritance works like Java: a child overrides what it wants. In IntelliJ: right-click `pom.xml` → Maven → **Show Effective POM**. On the command line: `mvn help:effective-pom`.

> **Precision note:** The lecture treats "super POM" and "effective POM" almost interchangeably. They are different: the Super POM is the top of the chain; the effective POM is the **final merged** model.

#### 5.9 Plugins

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

Maven itself does little; **plugins do the work** at each lifecycle phase (compile, test, package, ...). Spring Boot projects include `spring-boot-maven-plugin` to create the executable JAR.



### 6. Repositories and Dependency Resolution

#### 6.1 Types

| Repository | Where | Purpose |
|---|---|---|
| Local | `~/.m2/repository` | Cache of everything downloaded **and** artifacts you `install` |
| Central | `repo.maven.apache.org` | Public default remote repo |
| Other remote | Company Nexus / Artifactory | Private libraries, vetted copies |

#### 6.2 Resolution flow

```
mvn reads pom.xml
      │
      ▼
 needed artifact (G:A:V) in ~/.m2/repository? ──yes──▶ use it
      │ no
      ▼
 download from remote (Central or company repo)
      │
      ▼
 store in ~/.m2/repository (cache) ──▶ show under External Libraries
```

- First build is slow, since everything is downloaded. Later builds are fast, because the local cache is used.
- Deleting `~/.m2/repository` just forces a re-download (the lecture demonstrates this; use **Reload All Maven Projects** in the IDE).

#### 6.3 Local repository layout

```
~/.m2/repository/
└── org/example/maven-demo/1.0-SNAPSHOT/maven-demo-1.0-SNAPSHOT.jar
    └─ groupId as folders / artifactId / version / artifactId-version.jar
```

#### 6.4 Company repositories

Reasons to use a private repository:

- **private internal libraries** that must not go to Maven Central
- **security/quality**: only vetted artifacts are allowed

Declare one in `<repositories>` (with `id` and `url`).

> **Added (DevOps tie-in):** In real organisations the company repository is usually configured as a **mirror in `~/.m2/settings.xml`** (credentials too), not copied into every `pom.xml`. In CI, cache `~/.m2` between runs to avoid re-downloading on every build.



### 7. Build Lifecycle

#### 7.1 Three built-in lifecycles

| Lifecycle | Main phase(s) | Purpose |
|---|---|---|
| `clean` | `clean` | Delete `target/` |
| `default` | validate → ... → deploy | Build, test, package, publish |
| `site` | `site` | Generate project documentation/reports (rarely used) |

#### 7.2 Default lifecycle phases (the main ones)

| # | Phase | What happens |
|---|---|---|
| 1 | `validate` | Check the project is correct (valid `pom.xml`, required info present) |
| 2 | `compile` | `.java` → `.class` into `target/classes` |
| 3 | `test` | Run unit tests (JUnit, Mockito) |
| 4 | `package` | Build the JAR/WAR into `target/` |
| 5 | `verify` | Run checks on the packaged result (integration tests, code-quality/coverage plugins) |
| 6 | `install` | Copy the artifact into the **local** repository (`~/.m2`) |
| 7 | `deploy` | Upload the artifact to a **remote** repository |

> **Precision note:** The lecture mentions hidden in-between phases. The real default lifecycle has about 23 phases (`generate-sources`, `process-resources`, `test-compile`, `prepare-package`, `integration-test` and others). Unit tests run in `test`, integration tests in `integration-test`/`verify` (usually via the Failsafe plugin).

#### 7.3 Key rule: phases are cumulative

```
mvn package   →  validate → compile → test → package        (stops here)
mvn install   →  validate → compile → test → package → verify → install
mvn deploy    →  ... everything ... → deploy
```

Running a phase runs **all earlier phases** first. So `mvn package` will not produce a JAR if tests fail.

#### 7.4 Phases vs plugin goals

```
phase (a stage name)   ←binds→   plugin goal (actual work)
compile                          maven-compiler-plugin : compile
test                             maven-surefire-plugin : test
package                          maven-jar-plugin : jar
```

A phase on its own does nothing. The plugins bound to it do the work. (This is why the `<build><plugins>` section matters.)

> **Added:** Skip tests only when you know why: `mvn package -DskipTests` (compiles tests but does not run them) or `-Dmaven.test.skip=true` (does not even compile them).

#### 7.5 `install` vs `deploy` vs Git

| | Target | Audience |
|---|---|---|
| `install` | Local `~/.m2` | Your own other projects on this machine |
| `deploy` | Remote artifact repository (Nexus/Artifactory) | Your team and CI |
| `git push` | Source repository | Source code, not built artifacts |

> **Precision note:** The lecture says deploy puts "your code" somewhere for production. More precisely, `deploy` publishes the **built artifact** to an artifact repository. It does not run your app in production.

#### 7.6 Commands

| Command | Effect |
|---|---|
| `mvn validate` | Validate only |
| `mvn compile` | validate + compile |
| `mvn test` | ... + run tests |
| `mvn package` | ... + build JAR |
| `mvn clean` | Delete `target/` (not `~/.m2`) |
| `mvn clean install` | Fresh build, then install to `~/.m2`; the most common command |
| `mvn clean package` | Typical CI build (no need to install locally) |
| `mvn site` | Generate reports |

You can combine phases from **different lifecycles**: `mvn clean install` runs `clean`, then everything up to `install`.



### 8. IDE vs Command Line

- IntelliJ's green **Run** button uses **IntelliJ's own builder** unless Maven delegation is enabled. It writes to the same `target/classes` layout but is **not** the same as running Maven.
- The Maven tool window and the terminal run real Maven phases.
- The build in CI uses Maven, so verify with `mvn clean install` before you push.
- After editing `pom.xml`, **reload Maven** (the IDE's reload icon) so the dependencies are resolved. Until you do, the new classes show in red.

#### Installing Maven

- Install Maven yourself (Homebrew `brew install maven`, Chocolatey `choco install maven`, or the Apache binary with `PATH` set), or
- use the **Maven Wrapper** (`mvnw` / `mvnw.cmd`) that Spring Initializr already put in your project; it downloads the right Maven version for you. Recommended for teams and CI.



### 9. Archetypes

- An **archetype** is a project template (`mvn archetype:generate`; in IntelliJ: New Project → Archetype).
- Example: `maven-archetype-quickstart` creates a ready `pom.xml`, `App.java` and a test.
- Catalogs: IDE internal and Maven Central.
- For Spring Boot, **Spring Initializr** is the usual template source.



### Code

#### WRONG: manual classpath

```bash
# downloaded by hand; transitive deps missing, versions unknown
javac -cp lib/mysql-connector-j-9.5.0.jar src/Main.java
java  -cp lib/mysql-connector-j-9.5.0.jar:. Main   # NoClassDefFoundError at runtime
```

#### RIGHT: declare it and let Maven resolve it

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>maven-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>9.5.0</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</project>
```

#### WRONG: overriding versions the Boot parent already manages

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.0</version>   <!-- old, may clash with Boot-managed libraries -->
</dependency>
```

#### RIGHT: Spring Boot parent manages versions

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.6</version>
    <relativePath/>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc</artifactId>   <!-- no version -->
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>   <!-- builds the executable JAR -->
        </plugin>
    </plugins>
</build>
```

#### Commands

```bash
mvn clean install
mvn clean package -DskipTests
mvn dependency:tree
mvn help:effective-pom
java -jar target/demo-0.0.1-SNAPSHOT.jar
```



### Quick Self-Check

1. **What is a JAR, and why is it better than sharing `.class` files or a hand-made ZIP?**
   *Answer:* A JAR is a standard ZIP of classes, resources and metadata. It keeps the package structure and resources intact and is the format tools and the JVM understand.

2. **Library vs application, and what ends up in each JAR?**
   *Answer:* A library has no entry point and is used by other code; it is shipped as a thin JAR and describes its dependencies in its `pom.xml`. An application has `main()`; Spring Boot packages it as an executable JAR with dependencies nested, via `spring-boot-maven-plugin`.

3. **What does GAV stand for, and how does it map to a file name?**
   *Answer:* groupId, artifactId, version. The file is `artifactId-version.jar`. It lives under `groupId-as-folders/artifactId/version/` in the repository.

4. **What is a transitive dependency? Which tools show you the tree?**
   *Answer:* A dependency of a dependency. Maven resolves it automatically. Use the IDE's Maven tool window or `mvn dependency:tree`.

5. **Where does Maven look for an artifact, in order?**
   *Answer:* The local repository (`~/.m2/repository`), then the configured remote repositories (Maven Central by default, or a company repo/mirror). Downloads are cached locally.

6. **You run `mvn package` and a test fails. Is a JAR produced? Why?**
   *Answer:* No. Phases are cumulative, so `test` runs before `package`. Fix the test, or skip with `-DskipTests` knowingly.

7. **`install` vs `deploy`?**
   *Answer:* `install` copies the artifact into the local `~/.m2`; `deploy` uploads it to a remote artifact repository for the team.

8. **What is the effective POM?**
   *Answer:* The fully merged POM Maven uses: Super POM + parent chain + your POM (+ profiles). View it with `mvn help:effective-pom`.



### Golden Rules / Checklist

- [ ] One build tool per team; Maven here
- [ ] Follow the standard layout: `src/main/java`, `src/main/resources`, `src/test/java`
- [ ] Mirror the main package structure in tests
- [ ] Every artifact has unique GAV; groupId is a reversed domain
- [ ] Use `-SNAPSHOT` for work in progress and plain versions for releases
- [ ] Declare dependencies in `pom.xml`, never copy JARs by hand
- [ ] Reload Maven after editing `pom.xml`
- [ ] Use the right scope (`test` for JUnit, `runtime` for JDBC drivers)
- [ ] Omit versions that the Spring Boot parent manages
- [ ] Keep `spring-boot-maven-plugin` for executable JARs
- [ ] Prefer `mvn clean install` before pushing, because the IDE's run button does not exercise the full lifecycle
- [ ] Cache `~/.m2` in CI; use a mirror for company repositories
- [ ] Remember: running a phase runs all earlier phases
- [ ] Don't commit `target/`



### Practice Questions

#### Basic

1. Create a Maven project in IntelliJ and identify `src/main/java`, `src/test/java` and `pom.xml`. What is `target/` and when does it appear?

2. Compile with `mvn compile`; look inside `target/classes`. How does it mirror the source layout?

3. Run `mvn package`. What is the JAR's name and how is it derived from the POM?

4. Add `mysql-connector-j` as a dependency. Where does the JAR land locally?

#### Intermediate

5. Add Hibernate Core and expand its transitive dependencies in the Maven tool window. Run `mvn dependency:tree` and compare.

6. Define a custom property and use it as `${...}` inside `<version>`. What does the artifact name become? Name a risk of doing this.

7. Delete `~/.m2/repository` and rebuild. What happens, in what order, and why is the first build slow?

8. Explain what `mvn clean install` does, step by step, for your project. Which phases are skipped when you have no tests?

#### Advanced / Interview-style

9. Why does a plain `mvn package` JAR of a Spring Boot app fail with `java -jar`, and what in the POM fixes it?

10. Two dependencies need different versions of the same library. How does Maven decide, and how would you force or exclude a version?

11. Your company mandates that all artifacts come from an internal Nexus. Where do you configure that (pom vs settings), and why?

12. Explain the difference between a lifecycle phase and a plugin goal, with the `test` phase as an example. What does `-DskipTests` change?


## 4. Tight Coupling, Dependency Injection and Inversion of Control



### The Production Scenario

At Swiggy, `OrderService.placeOrder()` must notify the customer: first by email, later by SMS, then by app push. A developer writes:

```java
private final EmailService notification = new EmailService();
```

Consequences within a quarter:

- Adding SMS means editing and redeploying `OrderService`, a class that is about orders, not notifications.
- Every unit test of `placeOrder()` sends **real emails** (cost, spam, flakiness), because the service cannot be given a fake.
- Product wants WhatsApp for premium users. Another edit, another risk to the order flow.

The fix is not "more interfaces" alone. It is changing *who creates the dependency*.



### 1. Dependency: Two Different Meanings

| | Build-level dependency (Lecture 03) | Class-level dependency (this lecture) |
|---|---|---|
| What | An external library (`mysql-connector-j`) | Another class this class needs |
| Declared in | `pom.xml` | Fields / constructor of the class |
| Resolved by | Maven | You (manually), or Spring (IoC container) |

`OrderService` **depends on** a notification service because `placeOrder()` cannot finish its job without calling it. Dependencies are normal and unavoidable. In a real system they form a graph (A → B → C).

> **The problem is not the dependency. It is who creates it.**



### 2. Tight Coupling

**Tight coupling** means a class knows too much about *which concrete thing* it uses and *how it is built*. Analogy from the lecture: "I need to reach Chandigarh" vs "I need *this* bus, *this* driver, *this* company".

#### 2.1 The tightly coupled design

```
Main ──creates──▶ OrderService ──creates──▶ EmailService
                       │
                       └─ placeOrder(): print, then emailService.sendNotification()
```

#### 2.2 What breaks

| Principle (SOLID) | How the design violates it |
|---|---|
| **S**ingle Responsibility | `OrderService` handles orders **and** constructs/chooses notification objects |
| **O**pen/Closed | Switching email → SMS requires *modifying* `OrderService` |
| **D**ependency Inversion (principle) | A high-level class depends directly on a concrete low-level class |

Practical effects:

- cannot swap implementations without editing the class
- cannot unit test in isolation (the real email sender runs)
- changes ripple through the codebase as the dependency graph grows



### 3. Fixing It in Two Steps

#### Step 1: Program to an interface

```
              «interface» NotificationService
                    + sendNotification()
                   ▲        ▲          ▲
      EmailService   SmsService   PopupNotificationService
```

`OrderService` holds a `NotificationService` reference. This removes the compile-time tie to `EmailService` **but is not enough**: if `OrderService` still does `new EmailService()` inside, the concrete class is still hard-wired.

> **Precision note:** The lecture calls the interface step "coding to interfaces". That corresponds to the **Dependency Inversion Principle** (depend on abstractions). Do not confuse it with Dependency *Injection*, the technique in step 2. See section 6.

#### Step 2: Stop creating, start receiving

`OrderService` **asks** for what it needs; someone else supplies it.

```
Before DI                         After DI
─────────                         ────────
Main                              Main
 └─ new OrderService()             ├─ new EmailService()          (creates dependency)
     └─ new EmailService()         ├─ new OrderService(email)     (creates + injects)
                                   └─ order.placeOrder()
```

> **Rule of thumb:** *Don't create your own dependencies; get them.* A class should **ask for what it needs** rather than build everything itself.



### 4. Dependency Injection (DI)

**DI = a class receives its dependencies from the outside instead of creating them.**

Where the outside is, in this lecture, `main()`. The class (here `OrderService`) neither knows nor cares which implementation it got.

#### 4.1 Injection styles

| Style | How | Notes |
|---|---|---|
| **Constructor injection** | Dependency is a constructor parameter | Dependency is mandatory; object is never half-built; field can be `final`. **Preferred default.** |
| **Setter injection** | Dependency passed via `setX()` | For optional dependencies / reconfiguration; object can exist without it (NPE risk) |
| **Field injection** | Container sets the field directly (via reflection) | Not possible in plain Java; only with a container like Spring. Discouraged: hidden dependencies, hard to test |

> **Precision note:** The lecture shows constructor and setter, adds an empty constructor to make setter injection work, and defers field injection to Spring. Correct and complete. The added guidance: prefer **constructor injection**. With setter injection you can call `placeOrder()` before `setNotification()` and get a `NullPointerException`.

#### 4.2 Benefits

| Benefit | Why |
|---|---|
| Swap behaviour without editing `OrderService` | Open/Closed satisfied |
| `OrderService` has one job | Single Responsibility satisfied |
| Easy unit tests | Inject a **fake**/stub instead of the real sender |
| Clear dependencies | Constructor signature shows what the class needs |

#### 4.3 Testing with a fake

Real `placeOrder()` would trigger a real email in every test run. With DI, the test provides a harmless double:

```
Production wiring:  OrderService ◀── EmailService (really sends)
Test wiring:        OrderService ◀── FakeNotification (just records)
```

The lecture's `FakeEmailService` prints "dummy email sent". A better fake **records** that it was called so the test can assert it (see code).



### 5. Inversion of Control (IoC)

#### 5.1 Control = who creates and wires objects

```
BEFORE DI                                AFTER DI
(control flows downward through          (control sits in one place, main,
 each class)                              and flows into the classes)

Main ──▶ OrderService ──▶ EmailService   Main ──creates──▶ EmailService
        (creates)         (creates)       Main ──creates──▶ OrderService
                                          Main ──provides EmailService──▶ OrderService
```

Previously `OrderService` controlled the creation of `EmailService`. Now that **control is inverted**: `OrderService` no longer controls its dependency's creation; an outside party does.

#### 5.2 IoC vs DI vs DIP

| Term | What it is |
|---|---|
| **IoC** (Inversion of Control) | A **principle**: a framework/container controls creation, wiring and flow rather than your classes doing it |
| **DI** (Dependency Injection) | A **technique** to achieve IoC: supply dependencies from outside |
| **DIP** (Dependency Inversion Principle) | A SOLID **design principle**: high-level modules should depend on abstractions (interfaces), not concretes |

```
IoC (principle)
  └── achieved by ──▶ DI (constructor / setter / field)
DIP (separate principle) ── pushes you to use interfaces, which makes DI useful
```

> **Precision note:** The lecture (in the transcript) sometimes says "Dependency Inversion" or "Dependency Rejection" when it means **Dependency Injection**. The three terms above are different, and interviewers like asking about the difference. Also, DI is the most common way to get IoC, but not the only one (service locator, template-method callbacks, event-driven designs).

> **Added:** IoC is a broader idea than DI: *"don't call us, we'll call you"*. A servlet container calling your servlet (Lecture 01) and Spring MVC calling your `@GetMapping` method (Lecture 02) are also inversion of control: the framework drives, your code is called.



### 6. Why a Container Is the Next Step

Wiring by hand in `main()` works for 5 classes. For 100s of classes, `main()` becomes a huge wiring file:

```
main() {
   new A(); new B(a); new C(a, b); new D(c); ...   // order matters, errors multiply
}
```

**Spring's IoC container** takes this job:

| Container job | Detail |
|---|---|
| Create objects | You don't write `new` for managed objects |
| Manage them | Lifecycle (when created, when destroyed) |
| Connect them | Inject the right dependencies into the right objects |

In the three diagrams above, **`main` is replaced by the IoC container**. Result: you will see very little `new` for application classes in Spring code.

### Beans

- A **bean** is an object that is created, assembled and managed by the Spring IoC container.
- Every bean is an object; **not every object is a bean** (only the ones the container manages).

### Composition root

The place where objects get created and wired (`main()` here, the Spring container later) is called the **composition root**. Keep object creation there, not scattered in business classes.

> **Gotcha (added):** With constructor injection, circular dependencies (A needs B, B needs A) cannot be satisfied: neither can be constructed first. That is a design smell; Spring reports it at startup.



### 7. Trace: Who Creates What

| Stage | Creates `EmailService` | Creates `OrderService` | Decides which notification is used |
|---|---|---|---|
| Tightly coupled | `OrderService` | `main` | `OrderService` code (hard-coded) |
| After DI (plain Java) | `main` | `main` | `main` (one line to change) |
| With Spring | IoC container | IoC container | Configuration / annotations |



### Code

Package layout: `in.coderarmy` (Main, OrderService) and `in.coderarmy.notification` (the rest).

#### WRONG: `OrderService` creates its own dependency

```java
// OrderService.java
package in.coderarmy;

import in.coderarmy.notification.EmailService;

public class OrderService {
    private final EmailService notification = new EmailService(); // hard-wired

    public void placeOrder() {
        System.out.println("Order placed");
        notification.sendNotification();
    }
}
```

#### RIGHT: interface + constructor injection

```java
// NotificationService.java
package in.coderarmy.notification;

public interface NotificationService {
    void sendNotification();
}
```

```java
// EmailService.java
package in.coderarmy.notification;

public class EmailService implements NotificationService {
    @Override
    public void sendNotification() {
        System.out.println("Email notification sent");
    }
}
```

```java
// SmsService.java
package in.coderarmy.notification;

public class SmsService implements NotificationService {
    @Override
    public void sendNotification() {
        System.out.println("SMS notification sent");
    }
}
```

```java
// OrderService.java
package in.coderarmy;

import in.coderarmy.notification.NotificationService;
import java.util.Objects;

public class OrderService {
    private final NotificationService notification;

    public OrderService(NotificationService notification) {
        this.notification = Objects.requireNonNull(notification);
    }

    public void placeOrder() {
        System.out.println("Order placed");
        notification.sendNotification();
    }
}
```

```java
// Main.java (composition root)
package in.coderarmy;

import in.coderarmy.notification.EmailService;
import in.coderarmy.notification.NotificationService;

public class Main {
    public static void main(String[] args) {
        NotificationService notification = new EmailService(); // swap to new SmsService() here only
        OrderService order = new OrderService(notification);
        order.placeOrder();
    }
}
```

#### WRONG: setter injection that is easy to misuse

```java
public class OrderService {
    private NotificationService notification;

    public OrderService() {}                       // allows a half-built object

    public void setNotification(NotificationService notification) {
        this.notification = notification;
    }

    public void placeOrder() {
        System.out.println("Order placed");
        notification.sendNotification();           // NPE if setter was never called
    }
}

// usage
OrderService order = new OrderService();
order.placeOrder();                                // NullPointerException
```

#### RIGHT: when you do use a setter, call it before use

```java
OrderService order = new OrderService();
order.setNotification(new EmailService());         // wire first
order.placeOrder();
```

#### Testing with a recording fake

```java
// RecordingNotification.java (test code)
package in.coderarmy.notification;

public class RecordingNotification implements NotificationService {
    public int sent;

    @Override
    public void sendNotification() {
        sent++;
    }
}
```

```java
// check (plain Java; later replaced by JUnit/Mockito)
RecordingNotification fake = new RecordingNotification();
new OrderService(fake).placeOrder();
if (fake.sent != 1) throw new AssertionError("expected exactly 1 notification");
```



#### Quick Self-Check

1. **What exactly is wrong with `private EmailService n = new EmailService();` inside `OrderService`?**
   *Answer:* The class creates and chooses its own dependency. It is coupled to a concrete class, has two responsibilities, cannot be switched without editing it, and cannot be unit tested without the real sender.

2. **Why is using an interface alone not enough to remove tight coupling?**
   *Answer:* If the class still does `new EmailService()` inside, the concrete class is still hard-wired. The interface only helps once the implementation is supplied from outside.

3. **Define Dependency Injection in one sentence. Name three injection styles.**
   *Answer:* A class gets its dependencies from outside instead of creating them. Constructor, setter and field injection.

4. **Which injection style is preferred and why?**
   *Answer:* Constructor injection: dependencies are mandatory and visible, objects are never partially constructed, fields can be `final`, and tests are simple.

5. **IoC vs DI?**
   *Answer:* IoC is the principle that something else controls object creation and wiring. DI is the technique that achieves it. IoC also covers frameworks calling your code.

6. **How is the Dependency Inversion Principle different from Dependency Injection?**
   *Answer:* DIP is a SOLID guideline: depend on abstractions, not concretes. DI is the mechanism for supplying the concrete instance at runtime.

7. **Why does DI make unit testing easier? Give an example.**
   *Answer:* You can inject a fake or mock `NotificationService` so tests do not send real emails and can assert that it was called.

8. **What is a bean?**
   *Answer:* An object created, assembled and managed by the Spring IoC container. All beans are objects; not all objects are beans.



#### Golden Rules / Checklist

- [ ] Dependencies are fine; creating your own dependencies is the problem
- [ ] Don't use `new` for collaborators inside business classes
- [ ] Depend on interfaces, not concrete implementations
- [ ] Prefer constructor injection; make the field `final`
- [ ] Use setter injection only for genuinely optional dependencies
- [ ] Avoid field injection (it hides dependencies and hurts testing)
- [ ] Keep object creation and wiring in one place (composition root)
- [ ] Inject fakes/mocks in tests; never send real emails/SMS from a unit test
- [ ] Do not confuse IoC (principle), DI (technique) and DIP (SOLID principle)
- [ ] Avoid circular dependencies between constructor-injected classes
- [ ] In Spring, let the IoC container create and wire beans, not `main()`



### Practice Questions

#### Basic

1. Write the tightly coupled `OrderService` + `EmailService` version and run it. List two things you cannot change without editing `OrderService`.

2. Create the `NotificationService` interface with three implementations. Which line in `Main` do you change to switch the notification type?

3. Convert `OrderService` to constructor injection. What happens if you pass `null`?

4. Explain a "dependency" in two different contexts: `pom.xml` and a Java class.

#### Intermediate

5. Add setter injection next to constructor injection. Show the `NullPointerException` scenario and fix it.

6. Write a `RecordingNotification` and a check (or JUnit test) that asserts `placeOrder()` sends exactly one notification.

7. Draw before/after diagrams of "who controls creation". Label where control is inverted.

8. Add `PaymentService` as a second dependency of `OrderService`. How does the wiring in `main` grow? Why does this motivate a container?

#### Advanced / Interview-style

9. Explain IoC, DI and DIP with one example each. Which one is a SOLID principle?

10. Why is field injection impossible in plain Java without reflection, and why do many teams avoid it even in Spring?

11. Two classes depend on each other through constructors. What happens, why, and how would you redesign?

12. "Spring is just DI." Argue for and against, using what a container does beyond injection (lifecycle management, wiring many objects, replacing `main`).

## 5. Spring IoC Container, Beans and Annotation-Based Configuration



### The Production Scenario

At Zomato, `OrderService.placeOrder()` must charge the customer before confirming the order. Three real constraints show up in the first week:

- Two payment implementations exist: `UpiPayment` and `CardPayment`. Both implement `PaymentService`. Spring now has to choose one for `OrderService`.
- Card processing uses a vendor SDK class from a JAR. You cannot put `@Component` on it, because you do not own the source.
- Staging must use a fake gateway, production the real one, **without touching `OrderService`**.

Lecture 04 wired this by hand in `main()`. This lecture hands the wiring to Spring, and shows what to do when Spring cannot decide or cannot create an object by itself.



### 1. Recap and Setup

| Lecture 04 (plain Java) | Lecture 05 (Spring Core) |
|---|---|
| `main()` creates objects and passes dependencies | `ApplicationContext` creates and injects them |
| Objects | **Beans** (objects managed by the container) |
| Wiring code in `main()` | Annotations + a configuration class |

#### Dependency needed

Only **`spring-context`** (the IoC container). It pulls in `spring-core`, `spring-beans`, `spring-aop`, `spring-expression` and others as transitive dependencies. You do **not** need Spring Boot or Spring MVC for a console app.

> **Precision note:** Spring Boot 4 uses **Spring Framework 7**; Boot 3 used Spring 6. Match the `spring-context` major version to the Spring/Boot generation you will use later. Pick a current, stable 7.0.x release from Maven Central.



### 2. Reflection Primer (why Spring needs it)

A `java.lang.Class<T>` object holds **metadata about a class**: its name, fields, constructors, methods and annotations. It is **not** an instance of that class.

```
Student s1 = new Student();          // an object (instance)
Class<Student> c = Student.class;    // metadata about the Student class
```

| Class metadata | Examples |
|---|---|
| Name / package | `Student`, `in.coderarmy.Student` |
| Fields and types | `name : String`, `age : int` |
| Constructors | `Student()` |
| Methods | `getAttendance()`, `print()` |
| Annotations | `@Component`, `@Configuration`, ... |

Spring uses the **Reflection API** to read this metadata (which classes carry which annotations, which constructors need which parameters) and to **invoke constructors, setters and `@Bean` methods** without compile-time knowledge of your classes. That is how "magic" injection works.



### 3. Annotation-Based Configuration: The Pieces

| Piece | Role |
|---|---|
| `@Component` (on a class) | "Spring, manage an object of this class" |
| `@Configuration` (on a class) | "This class holds container configuration" |
| `@ComponentScan("pkg")` | "Search this package (and sub-packages) for `@Component` classes" |
| `AnnotationConfigApplicationContext` | The container implementation that reads annotation-based config |
| `ctx.getBean(X.class)` | Ask the container for a bean of type `X` |

#### Start-up wiring

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// AppConfig.class = metadata handed to the container: "read the rules here"
```

```
AppConfig.class ──reflection──▶ container reads:
                                  @Configuration  → config class
                                  @ComponentScan  → scan in.coderarmy
                                         │
                                         ▼
                           finds OrderService, PaymentService (@Component)
                                         │
                                         ▼
                           creates beans and wires dependencies
```

#### Component scan rules

- Scans the given package **and all sub-packages**.
- **No package given** (`@ComponentScan` with no value) means: scan the package of the annotated config class (and its sub-packages).
- A class **without `@Component`** (or another stereotype) is invisible to the scan.
- Classes in a package **outside** the scanned tree (a sibling package such as `in.other`) are not found. Widen the base package or scan it explicitly.

> **Precision note (Boot link):** `@SpringBootApplication` on `DemoApplication` (Lecture 02) is a combination of `@Configuration` + `@ComponentScan` + `@EnableAutoConfiguration`. That answers why your controller worked when it was in the same package, and why a controller outside that package gave a 404.



### 4. Dependency Injection with `@Autowired`

```java
@Component
public class OrderService {
    private final PaymentService payment;

    public OrderService(PaymentService payment) {   // @Autowired optional here
        this.payment = payment;
    }
}
```

| Style | `@Autowired` needed? | Notes |
|---|---|---|
| Constructor | **Not needed** if the class has a single constructor (Spring 4.3+) | Preferred |
| Setter | Yes | Called by the container after construction |
| Field | Yes | Works via reflection; IDE warns it is not recommended |

> **Gotcha:** With **multiple constructors**, Spring does not know which to use; mark one with `@Autowired`.

#### Why constructor injection is recommended

| Reason | Detail |
|---|---|
| Wired at creation | The object is complete the moment it exists |
| Fields can be `final` | Dependencies cannot be reassigned |
| Easy unit tests | `new OrderService(fakePayment)`: no Spring, no reflection |
| Fails fast | A missing dependency fails at startup, not mid-request |

Field injection counter-example: the field is `private` and not `final`, there is no constructor or setter, so a plain unit test cannot supply a dependency and `placeOrder()` throws `NullPointerException`. You need Spring (or reflection hacks) just to test.



### 5. What the Container Does at Start-up

| Step | What happens |
|---|---|
| 1 | `new AnnotationConfigApplicationContext(AppConfig.class)` starts the container |
| 2 | Container reads `AppConfig` (reflection) |
| 3 | It sees `@Configuration` and `@ComponentScan` |
| 4 | Scans the packages for `@Component` classes |
| 5 | For each found class (and each `@Bean` method) it registers a **BeanDefinition** |
| 6 | It instantiates beans, resolving each constructor's dependencies first |
| 7 | Your code uses the beans (`ctx.getBean(...)`) |

```
class metadata ──▶ BeanDefinition ──▶ (container instantiates) ──▶ bean in container
                   (name, class,                                    (wired, ready)
                    scope, dependencies)
```

#### BeanDefinition

- A recipe for a bean, **not** the bean: bean name, class, scope, dependencies and more.
- Built **before** any object exists. The container needs the whole picture so it can order creation and wire dependencies correctly.
- It is an internal infrastructure interface (`BeanDefinition`); you rarely use it directly.

#### Creation order

For `OrderService` that needs `PaymentService`: the container sees the dependency while creating `OrderService`, and creates `PaymentService` first.

> **Precision note:** The lecture says "assume `PaymentService` was created first". The accurate rule: **dependencies are always created before their dependents**; for unrelated beans the order is not something to rely on.

> **Added (essential):**
> - Beans are **singletons by default**: one instance per container. `getBean(X.class)` returns the *same* object each time. (Scopes come later.)
> - `ApplicationContext` creates singleton beans **eagerly at startup**, so wiring mistakes fail immediately, before you call any business method.
> - Close the context when done (`try-with-resources`), so beans can run shutdown callbacks.



### 6. Multiple Candidates: `@Primary` and `@Qualifier`

When `OrderService` asks for a `PaymentService` and two beans qualify, Spring cannot guess.

```
PaymentService (interface)
   ├── CardPayment  (@Component)
   └── UpiPayment   (@Component)

OrderService(PaymentService payment)   ← which one?
```

| Problem | Exception |
|---|---|
| No bean of the required type | `NoSuchBeanDefinitionException` / "No qualifying bean" (shown as `UnsatisfiedDependencyException` during startup) |
| More than one bean, no way to choose | `NoUniqueBeanDefinitionException` ("expected single matching bean but found 2") |

#### Fixes

| Tool | Where | Effect |
|---|---|---|
| `@Primary` | On the preferred bean class / `@Bean` method | "Use this one by default" |
| `@Qualifier("beanName")` | At the **injection point** (constructor parameter, setter, field) | "Use exactly this named bean" |

- `@Qualifier` is more specific and **wins over `@Primary`**.
- Use `@Primary` for a sensible default; use `@Qualifier` where a particular consumer needs a particular implementation.

#### Bean names

| How the bean is declared | Default bean name |
|---|---|
| `@Component` on `CardPayment` | `cardPayment` (class name, first letter lower-case) |
| `@Bean` method `createUpiPayment()` | `createUpiPayment` (the method name) |
| Explicit | `@Component("cp")` or `@Bean("cp")` |



### 7. When `@Component` Is Not Enough: `@Bean`

`@Component` needs Spring to create the object itself, which works when the constructor's parameters are all beans. It fails or is impossible when:

| Situation | Why `@Component` fails |
|---|---|
| Constructor needs plain values (`String name`, `int age`) | Spring has no value to supply (no bean of type `String`/`int`) |
| Class comes from a JAR/library | You cannot edit the `.class`, so you cannot add the annotation |
| Creation logic is complex | Needs custom setup, conditions, calls |

#### `@Bean`: you create it, Spring manages it

```java
@Configuration
public class AppConfig {

    @Bean
    public CardService cardService() {      // class from a library
        return new CardService();
    }
}
```

- Put `@Bean` on a **method** inside a `@Configuration` class.
- Spring calls the method, takes the returned object and registers it as a bean (named after the method).
- Everything after that is normal: Spring manages it, injects it, returns it from `getBean`.

#### `@Component` vs `@Bean`

| | `@Component` | `@Bean` |
|---|---|---|
| Placed on | Class | Method (in a `@Configuration` class) |
| Who creates the object | Spring | You (the method body) |
| Needs source access | Yes | No (works for library classes) |
| Typical use | Your own services | Third-party classes, custom construction |
| Found by | `@ComponentScan` | `@Configuration` class processing |

#### DI inside `@Bean` methods

Dependencies come in as **method parameters**; Spring resolves them just like constructor parameters (no `@Autowired` needed):

```java
@Bean
public OrderService orderService(PaymentService payment) {
    return new OrderService(payment);
}
```

`@Primary` can be placed on a `@Bean` method; `@Qualifier` on the parameter.

#### Same type declared both ways

> **Precision note:** The lecture says that if a class is both `@Component` and has a `@Bean` method, "the `@Bean` wins". That is only true when both end up with the **same bean name** (a `@Bean` definition silently replaces a scanned definition of that name). With **different names** (for example `cardPayment` from the scan and `createCardPayment` from the method) you get **two** beans and an ambiguity error. Declare each bean one way only.

> **Gotcha (added):** In a `@Configuration` class, one `@Bean` method calling another returns the existing singleton. In a plain `@Component` class ("lite mode") the same call creates a **new** object. Prefer method **parameters** for dependencies and you avoid the difference.

> **Added:** Plain data classes like a `User` (name, age) are normally **not** beans. You create them with `new` when needed. The lecture uses `User` to show Spring's limit with primitive constructor arguments; real beans are services, repositories, clients.



### 8. Why a Separate Configuration Class?

- Wiring in `main()` made the entry point complicated (Lecture 04).
- A `@Configuration` class keeps setup out of `main()`; its name is arbitrary, but it must be annotated `@Configuration`.
- Without `@Configuration`, the container does not treat it as a configuration source; `@ComponentScan` and `@Bean` methods on it are ignored.



### 9. `ApplicationContext` vs `BeanFactory`

```
BeanFactory                    <- basic container: getBean, containsBean, ...
   ▲
ListableBeanFactory / HierarchicalBeanFactory
   ▲
ApplicationContext             <- what you use; adds events, i18n, resource loading, environment
   ▲
AnnotationConfigApplicationContext  (annotation config; the Boot web variant extends this idea)
```

| | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Role | Basic container interface | Full-featured container (extends `BeanFactory`) |
| Singletons | Typically created lazily | Created eagerly at startup |
| Extras | None | Event publishing, i18n, resource loading, automatic post-processors |
| Used directly? | Rarely | Yes |

> **Precision note:** The lecture implies `BeanFactory` is old and mostly removed. In fact it is still the **root interface**; you simply work with `ApplicationContext`, which includes everything in it.

> **Link to Lecture 02:** `SpringApplication.run(...)` in a Boot app creates an `ApplicationContext` (a web-aware variant), scans your packages, registers beans, and starts the embedded Tomcat. That answers "who started Tomcat?".



### Code

Layout: package `in.coderarmy`; `payment` sub-package for payment classes.

#### Reflection demo

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class ReflectionDemo {
    static class Student {
        private String name;
        private int age;
        public Student() {}
        public void getAttendance() {}
        public void print() {}
    }

    public static void main(String[] args) {
        Class<Student> meta = Student.class;
        System.out.println(meta.getSimpleName());
        for (Field f : meta.getDeclaredFields())
            System.out.println(f.getName() + " : " + f.getType().getSimpleName());
        for (Constructor<?> c : meta.getDeclaredConstructors())
            System.out.println(c);
        for (Method m : meta.getDeclaredMethods())
            System.out.println(m.getName());
    }
}
```

#### `pom.xml` dependency

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>7.0.0</version> <!-- use a current 7.0.x -->
    </dependency>
</dependencies>
```

#### RIGHT: `@Component` + constructor injection + container

```java
// PaymentService.java
package in.coderarmy.payment;

public interface PaymentService {
    void pay();
}
```

```java
// CardPayment.java
package in.coderarmy.payment;

import org.springframework.stereotype.Component;

@Component
public class CardPayment implements PaymentService {
    @Override
    public void pay() {
        System.out.println("Paying via card");
    }
}
```

```java
// OrderService.java
package in.coderarmy;

import in.coderarmy.payment.PaymentService;
import org.springframework.stereotype.Component;

@Component
public class OrderService {
    private final PaymentService payment;

    public OrderService(PaymentService payment) {   // single constructor: no @Autowired needed
        this.payment = payment;
    }

    public void placeOrder() {
        payment.pay();
        System.out.println("Order placed");
    }
}
```

```java
// AppConfig.java
package in.coderarmy;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("in.coderarmy")
public class AppConfig {}
```

```java
// Main.java
package in.coderarmy;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Main {
    public static void main(String[] args) {
        try (AnnotationConfigApplicationContext ctx =
                     new AnnotationConfigApplicationContext(AppConfig.class)) {
            OrderService order = ctx.getBean(OrderService.class);   // no 'new'
            order.placeOrder();
        }
    }
}
```

#### WRONG: forgot `@Component` on the dependency

```java
public class CardPayment implements PaymentService {   // not scanned, no bean
    @Override
    public void pay() { System.out.println("Paying via card"); }
}
// startup: no qualifying bean of type PaymentService
```

#### WRONG: two implementations, no way to choose

```java
@Component public class CardPayment implements PaymentService { /* ... */ }
@Component public class UpiPayment  implements PaymentService { /* ... */ }
// startup: NoUniqueBeanDefinitionException (expected single matching bean but found 2)
```

#### RIGHT: `@Primary` for a default

```java
@Component
@Primary
public class CardPayment implements PaymentService {
    @Override
    public void pay() { System.out.println("Paying via card"); }
}
```

#### RIGHT: `@Qualifier` at the injection point

```java
@Component
public class OrderService {
    private final PaymentService payment;

    public OrderService(@Qualifier("upiPayment") PaymentService payment) {
        this.payment = payment;
    }
}
```

#### WRONG: field injection

```java
@Component
public class OrderService {
    @Autowired
    private PaymentService payment;      // hidden dependency, cannot be final,
                                         // plain unit test cannot set it
}
```

#### RIGHT: `@Bean` for library classes and custom construction

```java
package in.coderarmy;

import in.coderarmy.payment.CardPayment;
import in.coderarmy.payment.PaymentService;
import in.coderarmy.payment.UpiPayment;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class AppConfig {

    @Bean
    public CardService cardService() {              // class from a JAR; cannot annotate
        return new CardService();
    }

    @Bean
    @Primary
    public PaymentService cardPayment() {
        return new CardPayment();
    }

    @Bean("upi")
    public PaymentService upiPayment() {
        return new UpiPayment();
    }

    @Bean
    public OrderService orderService(@Qualifier("upi") PaymentService payment) {
        return new OrderService(payment);           // dependency arrives as a parameter
    }
}
```

(No `@ComponentScan` and no `@Component` on these classes, so each bean is declared exactly once.)



#### Quick Self-Check

1. **What is an IoC container in Spring, and what is a bean?**
   *Answer:* The container (`ApplicationContext`) creates, wires and manages objects. A bean is an object the container manages; every bean is an object, but not every object is a bean.

2. **What do `@Configuration`, `@ComponentScan` and `@Component` each do?**
   *Answer:* `@Configuration` marks a configuration class. `@ComponentScan` says which packages (and sub-packages) to search. `@Component` marks a class whose object Spring should create and manage.

3. **Why does Spring build bean definitions before objects?**
   *Answer:* It needs complete metadata (name, class, dependencies, scope) for all beans to order creation and wire dependencies correctly. Definitions are built via reflection before instantiation.

4. **When is `@Autowired` required?**
   *Answer:* For setter and field injection, and for choosing among multiple constructors. With a single constructor it is optional.

5. **Two beans implement `PaymentService`. What happens and how do you fix it?**
   *Answer:* `NoUniqueBeanDefinitionException`. Mark one `@Primary`, or use `@Qualifier("beanName")` at the injection point (it overrides `@Primary`).

6. **Why can't `@Component` create a `User(String name, int age)` bean automatically?**
   *Answer:* There is no bean of type `String` or `int` to pass. Create it with a `@Bean` method (and note data objects usually are not beans at all).

7. **`@Component` vs `@Bean`?**
   *Answer:* `@Component` is on a class and Spring creates it; `@Bean` is on a method in a `@Configuration` class and you create the object. Use `@Bean` for third-party classes and custom construction.

8. **Why is constructor injection preferred? Why is field injection discouraged?**
   *Answer:* Constructor injection wires dependencies when the object is created, allows `final` fields, and makes unit tests simple. Field injection hides dependencies, cannot be `final`, and needs a container or reflection to test.



#### Golden Rules / Checklist

- [ ] Add only `spring-context` for core-container-only practice
- [ ] Annotate the config class with `@Configuration`
- [ ] Put `@ComponentScan` on the config class; remember it scans sub-packages
- [ ] Put `@Component` only on classes you own; use `@Bean` for third-party classes
- [ ] Never `new` a bean yourself and expect Spring to manage it
- [ ] Prefer constructor injection; make fields `final`
- [ ] Declare each bean in exactly one way (scan or `@Bean`, not both)
- [ ] Use `@Primary` for a default, `@Qualifier` for a specific choice
- [ ] Remember: default bean name = class name (lower camel case) or `@Bean` method name
- [ ] Keep wiring out of `main()` and in `@Configuration` classes
- [ ] Don't turn plain data objects (DTOs/entities) into beans without a reason
- [ ] Close the context (try-with-resources) in standalone apps
- [ ] Interpret errors by name: no bean (missing annotation/scan) vs more than one bean (add `@Primary`/`@Qualifier`)



### Practice Questions

#### Basic

1. Create a Maven project with `spring-context`. Write `PaymentService`, `OrderService`, `AppConfig` and run `placeOrder()` via `getBean`.

2. Remove `@Component` from the payment class. What exception do you get, and why?

3. What is the default bean name of `CardPayment`? Of a `@Bean` method called `createUpiPayment`?

4. Write a `Student` class and print its fields, constructors and methods using reflection.

#### Intermediate

5. Add `UpiPayment` as a second `PaymentService`. Reproduce the "two beans" error, then fix it once with `@Primary` and once with `@Qualifier`.

6. Move a payment class to a package outside the `@ComponentScan` base package. What happens? Fix it two ways.

7. Convert your `AppConfig` to pure `@Bean` methods and delete all `@Component` annotations. Which dependencies are now passed as method parameters?

8. Convert constructor injection to setter, then field injection. For each, what changes in the annotations and in how you would unit test `OrderService`?

#### Advanced / Interview-style

9. Explain, step by step, what `new AnnotationConfigApplicationContext(AppConfig.class)` does until `getBean` returns. Where do bean definitions fit?

10. Why can `@Bean` handle `new User("Aditya", 28)` and a library class where `@Component` cannot? What are the trade-offs of declaring beans manually?

11. A class is declared both with `@Component` and via a `@Bean` method. Describe what happens when the names match and when they differ. How would you avoid the situation?

12. Compare `BeanFactory` and `ApplicationContext`. Which one does `SpringApplication.run` use, and what does that imply for startup failures?


## 6. Circular Dependencies, Bean Scopes and Lazy/Eager Initialization

> **Source:** Spring Framework & Spring Boot series, Lecture 06 (Spring Core, annotation-based config).
> **Depth note:** Three linked ideas. (1) A **circular dependency** (A needs B, B needs A) is a *design* problem, not a Spring problem. Constructor cycles can never be satisfied; setter/field cycles can be patched by the container, but should be refactored away. (2) **Scope** decides *how many* instances a bean definition produces: `singleton` (one per bean definition per container, the default) vs `prototype` (new instance on every request). (3) **Initialization** decides *when* a bean is created: singletons are **eager** by default (fail fast at startup), `@Lazy` defers creation, prototypes are always on demand. `@Lazy` on an injection point injects a **proxy**, which is one way to break a cycle.



### The Production Scenario

At Swiggy, `OrderService.placeOrder()` calls `PaymentService.pay()`. A well-meaning developer lets `PaymentService` call back `OrderService.getOrderDetails()` to print a receipt. The app now refuses to start:

```
Requested bean is currently in creation: Is there an unresolvable circular reference?
```

Three more things surface in the same sprint:

- A per-customer `CartSession` object accidentally shared by *every* customer, because the bean was a singleton.
- Pod start-up takes 40 s, because a heavy `RecommendationModelLoader` is built at boot. Someone sets global lazy initialization, and a misconfigured bean now fails on the *first live request* instead of at deploy time.
- Two `@Bean` methods return the same class, and "singleton" suddenly gives two objects.

All of this comes from the three topics below.



### 1. Circular Dependency

#### 1.1 What it is

```
Normal (acyclic):               Circular:
Order ─▶ Payment ─▶ Gateway     A ─▶ B
                                ▲    │
                                └────┘
```

- Normal graphs have a clear starting point: create `Gateway`, then `Payment`, then `Order`. Spring orders creation by dependencies.
- In a cycle, nothing can be created first.

#### 1.2 It exists in plain Java too

```
new A() ─▶ A() runs: new B() ─▶ B() runs: new A() ─▶ A() runs: new B() ─▶ ...
                                                          StackOverflowError
```

Each constructor creates the other, so the call stack grows until the JVM throws `StackOverflowError`. Spring did not cause this; the **design** did.

#### 1.3 What Spring does, by injection style

Setup: `OrderService` ↔ `PaymentService`, both singleton `@Component`s.

| Injection style in the cycle | Plain Spring (Framework) | Spring Boot (2.6+) |
|---|---|---|
| Constructor ↔ constructor | **Fails**: `BeanCurrentlyInCreationException` ("unresolvable circular reference") | Fails |
| Setter / field | **Works** (container resolves it) | **Fails by default**, because Boot sets `spring.main.allow-circular-references=false` |
| Constructor on one side, setter/field on the other | Depends on which bean is created first | Fails by default |

> **Precision note:** The lecture says "Boot 2.6 and above" in one place and "2.4" in another. It is **Spring Boot 2.6** that disabled circular references by default; this remains the default in Boot 3 and 4. Plain Spring (without Boot) allows setter/field cycles for singletons by default.

#### 1.4 Why constructor cycles fail

```
create OrderService ─needs─▶ PaymentService
        ▲                           │
        │                           ▼
        └──────needs─────── create PaymentService
(OrderService is "currently in creation", so the container throws)
```

`BeanCurrentlyInCreationException` means: "this bean is already being built, and something asked for it again".

#### 1.5 Why setter/field cycles "work"

Creation and injection are **separate steps** for setter/field injection, so each object can exist before its dependencies are filled in.

| Step | Container action |
|---|---|
| 1 | Instantiate `OrderService` using its default constructor (a raw object) and expose an *early reference* to it |
| 2 | Start populating `OrderService`: it needs `PaymentService`, which does not exist yet |
| 3 | Instantiate `PaymentService` (raw) |
| 4 | Populate `PaymentService`: it needs `OrderService`, so it gets the **early reference** (still incomplete) |
| 5 | `PaymentService` is finished and registered |
| 6 | Back in `OrderService`, inject the finished `PaymentService`; it is finished too |

Internally this uses a **three-level singleton cache** (fully built singletons, early singleton references, singleton factories for early references). It works only for singletons.

> **Added:** Field injection works on private fields because the container uses **reflection** (Lecture 05) to set them, with no setter needed.

#### 1.6 Do not rely on it. Avoid it.

| Reason | Detail |
|---|---|
| It violates the **Single Responsibility Principle** | The two classes are tangled: each does work that belongs to the other |
| Tight coupling | Neither class can be understood, tested or changed alone |
| Fragile | Works only for singletons, fails if Boot defaults apply, depends on injection style |
| Hidden half-built objects | A bean can see its partner before it is fully initialized |

#### 1.7 Fixing it: preferred to workarounds

| Option | When | Notes |
|---|---|---|
| **Refactor the design** (preferred) | Always, first | See below |
| Extract the shared logic into a third class used by both | When both really need the same behaviour | Breaks the cycle with a one-way graph |
| Events / callbacks | When one side only needs to be notified | Decouples the direction |
| Switch to setter/field injection | Last resort, plain Spring only | Hides the smell; Boot disables it by default |
| `@Lazy` on an injection point | Last resort | Injects a proxy (section 4.4) |
| `spring.main.allow-circular-references=true` | Boot legacy code only | Turns the safety net off |

> **Precision note:** The lecture lists `@PostConstruct` as another fix. It is a **lifecycle callback** that runs *after* injection, not a resolver for cycles. It can help if you defer a lookup until later, but it does not itself break a construction cycle. (Lifecycle comes in a later video.)

#### 1.8 Refactoring the example

Cause of the cycle: `PaymentService.pay()` calls back `OrderService.getOrderDetails()`.

```
BEFORE                                   AFTER
OrderService.placeOrder()                OrderService.placeOrder()
   └─▶ PaymentService.pay()                 ├─▶ PaymentService.pay()      (payment only)
          └─▶ OrderService.getOrderDetails()  ├─▶ getOrderDetails()         (order's own job)
                                              └─▶ "Order placed"
```

- `PaymentService`'s job is **payments**. Fetching order details is not.
- `OrderService` orchestrates: pay, then print details, then confirm. The dependency is now **one-way** (Order → Payment).
- Output is identical (`Payment done`, `Order details`, `Order placed`), and the cycle is gone.



### 2. `@Configuration` Is Itself a Bean

- `@Configuration` is meta-annotated with `@Component`, so the config class is registered as a bean too.
- That is how Spring can call your `@Bean` methods: it has an `AppConfig` instance to call them on.
- It is special because it declares other beans (`@Bean` methods) and carries `@ComponentScan`.

> **Added (gotcha):** In a `@Configuration` class, a `@Bean` method calling another `@Bean` method gets the **existing singleton** (the class is proxied to guarantee this). In a plain `@Component` class it creates a **new** object. Prefer passing dependencies as method parameters.



#### 3. Bean Scopes

A **scope** defines the lifecycle and visibility of the instances created from one bean definition.

#### 3.1 Singleton (default)

- **One instance per bean definition, per container.**
- Every `getBean(...)` and every injection of that bean gets the **same** object.
- Proof from the lecture: `ctx.getBean(OrderService.class) == ctx.getBean(OrderService.class)` is `true`, and the constructor prints once.

```
          IoC container
   ┌──────────────────────────┐
   │  orderService  (1 object)│◀── getBean (order)
   │                          │◀── getBean (order2)   same reference
   │                          │◀── injected into A
   │                          │◀── injected into B
   └──────────────────────────┘
```

> **Precision note: "singleton" in Spring is not the GoF Singleton pattern.**
> - GoF Singleton: *one instance of the class per JVM (classloader)*, enforced by the class itself.
> - Spring singleton: *one instance per bean definition per container*.
> - You can still call `new OrderService()` yourself and get another object.
> - Two `@Bean` methods returning `OrderService` are **two bean definitions → two instances**, each a singleton. `getBean(OrderService.class)` then fails with `NoUniqueBeanDefinitionException` (fix with `@Primary`/`@Qualifier`).

#### 3.2 Prototype

- **A new instance on every request**: every `getBean(...)` and every injection point.
- Declared with `@Scope("prototype")` on the class or on a `@Bean` method.

```
getBean(Order) ─▶ new object #1
getBean(Order) ─▶ new object #2
A injects it   ─▶ new object #3
B injects it   ─▶ new object #4        (4 constructor calls; `o1 == o2` is false)
```

> **Gotchas (added):**
> - The container does **not** manage the full lifecycle of prototypes. It creates and hands them over; it does **not** call destroy callbacks. You own cleanup.
> - **Prototype injected into a singleton is injected once**, at the singleton's creation, so that singleton keeps *one* prototype instance forever. If you need a fresh one per use, inject an `ObjectProvider<T>` and call `getObject()`.

#### 3.3 Other scopes (web only; later)

| Scope | New instance per |
|---|---|
| `request` | HTTP request |
| `session` | HTTP session (user session) |
| `application` | Servlet application (like a singleton for the web app) |

They exist only in web-aware application contexts (Spring MVC/Boot web).

#### 3.4 Choosing a scope

| Bean type | Scope | Why |
|---|---|---|
| Services, managers, repositories, clients (`OrderService`, `PaymentService`) | **Singleton** | Behaviour only; **no per-user mutable state**; one shared instance is cheap |
| Objects holding per-use mutable state (a cart for one user) | Prototype, or a web scope | Each user/use needs its own state |

> **Precision notes:**
> - "Stateless" really means **no mutable state that varies per user/request**. A singleton service holding *references to its dependencies* (immutable) is fine.
> - Singleton beans are shared across threads (Lecture 01: servlets/Tomcat worker threads). A mutable field in a singleton is a **thread-safety bug**.
> - Plain data classes (`User` with name/age) are usually **not beans at all**; create them with `new`. The lecture uses `User` to illustrate state.



### 4. Initialization: Eager vs Lazy

#### 4.1 Definitions

| | When the object is created |
|---|---|
| **Eager** | At container startup, as part of building the `ApplicationContext` |
| **Lazy** | When first needed (a `getBean(...)`, or when another bean needs it injected) |

#### 4.2 Scope and initialization

| Scope | Default | Can it be changed? |
|---|---|---|
| Singleton | **Eager** | Yes: add `@Lazy` |
| Prototype | **Lazy** (always on demand) | No eager option (creating one up front would be pointless) |

Proof from the lecture: with a prototype bean, starting the context prints nothing; with a singleton it prints the constructor message immediately.

#### 4.3 `@Lazy` on a class or `@Bean` method

```java
@Component
@Lazy
public class PaymentService { ... }
```

- The bean is **not created at startup**.
- It **is** created when someone asks for it (`getBean`) **or when a non-lazy bean needs it injected**, because the dependent bean cannot be built without it.

| Case | Result at startup |
|---|---|
| `OrderService` lazy, `PaymentService` eager | Only `PaymentService` is created |
| `PaymentService` lazy, `OrderService` eager and **needs it** | Both are created (Order forces Payment) |
| Both lazy | Neither is created until requested |

#### 4.4 `@Lazy` at an injection point (proxy)

```java
public OrderService(@Lazy PaymentService payment) { ... }
```

- Spring injects a **lazy-resolution proxy**: an object that *looks like* `PaymentService` and fetches the real bean on **first method call**.
- Nothing needs to be created to build `OrderService`, so a **constructor cycle is broken**.
- Sequence for a cycle fixed this way:

| Step | Event |
|---|---|
| 1 | Create `OrderService`; the `@Lazy` parameter receives a proxy |
| 2 | `OrderService` is complete |
| 3 | The real `PaymentService` is created when needed/at startup; its constructor receives the existing `OrderService` |
| 4 | On the first `payment.pay()` call the proxy resolves to the real bean |

> **Gotchas (added):**
> - The lecture's printed order ("PaymentService not started yet" before its constructor runs) holds only if `PaymentService` is **also** lazy (class level) *and* the injection point is `@Lazy`. With an eager `PaymentService`, it is created at startup anyway.
> - A proxy of a class type uses a generated subclass, so the injected class **cannot be `final`** (for interface types, a JDK proxy is used).

#### 4.5 Global lazy initialization

```properties
spring.main.lazy-initialization=true
```

- Spring Boot property: **all** beans become lazy.
- Opt a bean out with `@Lazy(false)`.
- `@Lazy(false)` is only meaningful when lazy init is on globally or inherited from a config class.

#### 4.6 Why eager is the default: fail fast

| | Eager | Lazy |
|---|---|---|
| Wiring mistakes (missing bean, two candidates, cycle) | **Found at startup**, before traffic | Found **at first use**, possibly in production |
| Startup time | Slower (everything is built) | Faster |
| First-request latency | None | The first caller pays creation cost |

Use `@Lazy` deliberately for genuinely heavy, rarely-used beans. Default eager is the safe choice.

> **Added (DevOps tie-in):** A bad bean config found at startup makes the pod fail its readiness/startup probe and the rollout stops. With lazy init, the same bug passes the health check and fails on a customer request. Test lazy configurations the same way you test eager ones.



### 5. Summary Matrix

| | Singleton | Prototype |
|---|---|---|
| Instances | One per bean definition (per container) | One per request/injection |
| Default init | Eager | Lazy (always) |
| `@Lazy` | Allowed (defers to first use) | Redundant |
| Container manages destruction | Yes | No |
| Typical use | Services, repositories | Stateful, short-lived objects |



### Code

#### Plain-Java cycle (no Spring)

```java
class A {
    private final B b;
    A() {
        System.out.println("A created");
        this.b = new B();
    }
}

class B {
    private final A a;
    B() {
        System.out.println("B created");
        this.a = new A();
    }
}

public class CycleDemo {
    public static void main(String[] args) {
        new A();      // A created, B created, A created ... StackOverflowError
    }
}
```

#### Spring setup used below

```java
// AppConfig.java
package in.strikes;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
public class AppConfig {}
```

```java
// Main.java
package in.strikes;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Main {
    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {
            ctx.getBean(OrderService.class).placeOrder();
        }
    }
}
```

#### WRONG: constructor cycle

```java
@Component
public class OrderService {
    private final PaymentService payment;

    public OrderService(PaymentService payment) {
        this.payment = payment;
    }

    public void placeOrder() {
        payment.pay();
        System.out.println("Order placed");
    }

    public void getOrderDetails() {
        System.out.println("Order details");
    }
}
```

```java
@Component
public class PaymentService {
    private final OrderService order;      // cycle: Payment -> Order -> Payment

    public PaymentService(OrderService order) {
        this.order = order;
    }

    public void pay() {
        System.out.println("Payment done");
        order.getOrderDetails();           // not Payment's job
    }
}
// startup: BeanCurrentlyInCreationException
```

#### WRONG (works in plain Spring, rejected by Boot): field injection

```java
@Component
public class PaymentService {
    @Autowired
    private OrderService order;            // hides the cycle
    // ...
}
```

#### RIGHT: refactor to a one-way dependency

```java
@Component
public class OrderService {
    private final PaymentService payment;

    public OrderService(PaymentService payment) {
        this.payment = payment;
    }

    public void placeOrder() {
        payment.pay();
        getOrderDetails();                 // Order's own responsibility
        System.out.println("Order placed");
    }

    public void getOrderDetails() {
        System.out.println("Order details");
    }
}
```

```java
@Component
public class PaymentService {
    public void pay() {
        System.out.println("Payment done");
    }
}
// output: Payment done / Order details / Order placed
```

#### Last resort: `@Lazy` on the injection point

```java
@Component
public class OrderService {
    private final PaymentService payment;

    public OrderService(@Lazy PaymentService payment) {   // proxy injected
        this.payment = payment;
    }
    // ...
}
// PaymentService keeps its OrderService constructor parameter
```

#### Scope: singleton vs prototype

```java
@Component
@Scope("prototype")
public class OrderService {
    public OrderService() {
        System.out.println("OrderService created");
    }

    public void placeOrder() {
        System.out.println("Order placed");
    }
}
```

```java
OrderService o1 = ctx.getBean(OrderService.class);
OrderService o2 = ctx.getBean(OrderService.class);
System.out.println(o1 == o2);   // prototype: false (2 constructor calls); singleton: true (1 call)
```

#### Singleton is per bean definition

```java
@Configuration
public class AppConfig {
    @Bean
    public OrderService orderService1() { return new OrderService(); }

    @Bean
    public OrderService orderService2() { return new OrderService(); }
}
// startup prints "OrderService created" twice: two bean definitions, two singletons
// ctx.getBean(OrderService.class): NoUniqueBeanDefinitionException
```

#### Prototype inside a singleton: use `ObjectProvider`

```java
@Component
public class ReportJob {
    private final ObjectProvider<CartSession> sessions;

    public ReportJob(ObjectProvider<CartSession> sessions) {
        this.sessions = sessions;
    }

    public void run() {
        CartSession fresh = sessions.getObject();   // new prototype each call
    }
}
```

#### Eager vs lazy

```java
@Component
@Lazy
public class ReportService {
    public ReportService() {
        System.out.println("ReportService created");
    }
}
// startup prints nothing; created on first getBean or first injection into an eager bean
```

```properties
# application.properties (Spring Boot only)
spring.main.lazy-initialization=true
```



#### Quick Self-Check

1. **What is a circular dependency, and is it a Spring problem?**
   *Answer:* A needs B and B needs A. It is a design problem; even plain Java recurses into `StackOverflowError` if both constructors create each other.

2. **Why does a constructor cycle fail in Spring but a setter/field cycle can work?**
   *Answer:* Constructors need the other bean before the object exists, so neither can be created first (`BeanCurrentlyInCreationException`). With setter/field injection, objects are created first, then populated; the container hands out early references to singletons.

3. **What does Spring Boot do by default with setter/field cycles?**
   *Answer:* Boot 2.6+ sets `spring.main.allow-circular-references=false`, so they fail at startup.

4. **How would you remove a cycle between `OrderService` and `PaymentService`?**
   *Answer:* Fix responsibilities: payment does payment only; `OrderService` orchestrates and fetches its own details. Or extract shared logic into a third class.

5. **What does singleton scope mean in Spring, and how is it different from the GoF Singleton?**
   *Answer:* One instance per bean definition per container. GoF Singleton is one instance per JVM/classloader enforced by the class. Two `@Bean` methods of the same class give two Spring singletons.

6. **What happens with prototype scope on `getBean` and on injection? What does the container not do?**
   *Answer:* A new instance each time. The container does not manage prototype destruction (no destroy callbacks).

7. **Eager vs lazy? Which is default for singleton and for prototype?**
   *Answer:* Eager creates at startup; lazy on first need. Singleton is eager by default (can be made lazy with `@Lazy`); prototype is always created on demand.

8. **Why is eager initialization the safer default?**
   *Answer:* Fail fast: wiring errors surface at startup instead of during live requests, and the first request does not pay the creation cost.



#### Golden Rules / Checklist

- [ ] Treat a circular dependency as a design bug and fix the responsibilities
- [ ] Do not "fix" cycles with setter/field injection or `allow-circular-references=true`
- [ ] Keep dependencies one-way; the orchestrator calls the helpers, not the reverse
- [ ] Use `@Lazy` on an injection point only as a documented last resort
- [ ] Make services singletons and keep them free of per-user mutable state
- [ ] Use prototype (or web scopes) only for genuinely per-use state
- [ ] Inject `ObjectProvider<T>` when a singleton needs a fresh prototype each time
- [ ] Remember: you must clean up prototype beans yourself
- [ ] Remember: singleton = per bean definition, not per class
- [ ] Prefer eager initialization; use `@Lazy` for genuinely heavy, rarely used beans
- [ ] Test with the same laziness setting you deploy with
- [ ] Do not make `@Lazy`-proxied classes `final`



### Practice Questions

#### Basic

1. Reproduce the plain-Java `A`/`B` cycle and observe the `StackOverflowError`. Why does the stack overflow?

2. Create `OrderService` ↔ `PaymentService` with constructor injection. Read the exception message and identify the exception type.

3. Print `ctx.getBean(OrderService.class) == ctx.getBean(OrderService.class)` for singleton and prototype. Count constructor calls.

4. Add `@Lazy` to a class and show that nothing prints at startup.

#### Intermediate

5. Convert the constructor cycle to field injection in plain Spring: does it run? Then run it in Spring Boot 4: what happens, and which property controls it?

6. Refactor the cycle so `PaymentService` no longer references `OrderService`, keeping the same output.

7. Two beans `A` and `B` both depend on a prototype `OrderService`. How many `OrderService` objects exist after startup and two extra `getBean` calls? Why?

8. Make `PaymentService` lazy (class level) while `OrderService` is eager and depends on it. Predict the startup output. Then add `@Lazy` on the constructor parameter and predict again.

#### Advanced / Interview-style

9. Explain how the container resolves a setter-injection cycle using early references. Why does it work only for singletons?

10. Two `@Bean` methods return the same class and the scope is singleton. How many instances, why, and how do you disambiguate `getBean`?

11. A singleton `ReportJob` injects a prototype `CartSession`. What is the bug, and how do you fix it?

12. Your team sets `spring.main.lazy-initialization=true` to speed up pod start-up. What can go wrong in production, how would you detect it earlier, and which beans would you exempt?


## 7. Spring Bean Lifecycle


### The Production Scenario

At Hotstar, during an IPL match, a `MatchScoreCache` bean must (a) preload team and player data from a file before serving the first request, and (b) flush pending counters to Redis when the pod shuts down.

What goes wrong without lifecycle knowledge:

- The preload is done **in the constructor**, which runs before the injected `FileLoader` exists, so it hits `NullPointerException` (or loads an expensive file while the object is still half-built).
- The flush is never run, because on deploy Kubernetes sends `SIGTERM` and nobody closes the Spring context; the last few seconds of counters vanish.
- A teammate puts cleanup in `@PreDestroy` on a **prototype** bean and wonders why it never fires.

All three come from the lifecycle below.



### 1. Who Manages What

| Object kind | Created by | Lifecycle managed by |
|---|---|---|
| Plain Java object (`new X()`) | You | You (garbage collection frees memory once unreferenced) |
| Singleton bean | IoC container | **Container**: creation, wiring, initialization, **destruction** |
| Prototype bean | IoC container (on request) | Container up to **hand-over**; after that it is yours |

"Destroy" for a bean means: the container removes it and calls its cleanup callbacks. It is separate from Java garbage collection of memory.



### 2. The Lifecycle at a Glance (singleton, eager)

```
 ┌──────────────────────── container start ───────────────────────────┐
 │ 1  IoC container starts                                              │
 │ 2  Read configuration (@Configuration, @ComponentScan, @Bean)       │
 │ 3  Create BeanDefinitions for ALL beans (metadata)                   │
 │ 4  Instantiate the object (constructor)                              │
 │ 5  Inject dependencies (setter/field; constructor already did it)   │
 │ 6  Aware callbacks (BeanNameAware, ApplicationContextAware, ...)    │
 │ 7  Initialization callbacks (@PostConstruct, afterPropertiesSet,    │
 │    init-method)                                                      │
 └──────────────────────────────┬──────────────────────────────────────┘
                                ▼
                  8  BEAN IS READY: your code uses it
                                │
 ┌──────────────────────── container close ───────────────────────────┐
 │ 9  Destruction callbacks (@PreDestroy, destroy(), destroy-method)  │
 │ 10 Bean destroyed / removed from the container                      │
 └─────────────────────────────────────────────────────────────────────┘
```

| # | Step | What happens | Notes |
|---|---|---|---|
| 1 | Container starts | `new AnnotationConfigApplicationContext(AppConfig.class)` | Nothing can be managed before this |
| 2 | Read configuration | Reflection on `AppConfig` reveals `@Configuration`, `@ComponentScan`, `@Bean` methods | `AppConfig` is itself a bean, so this lifecycle applies to it too |
| 3 | Bean definitions | Metadata (name, class, scope, lazy flag, dependencies) for **every** bean is built first | Lets the container decide ordering and which beans to create now |
| 4 | Instantiate | Constructor runs | With constructor injection, dependencies are resolved here, so 4 and 5 merge |
| 5 | Dependency injection | Setter/field injection after the object exists | Dependencies are created on demand if missing |
| 6 | Aware callbacks | Spring calls methods on beans that implement `*Aware` interfaces | Only if the bean implements them |
| 7 | Initialization callbacks | `@PostConstruct` etc. | Dependencies are all present now |
| 8 | Ready | Your code calls methods | Beans live until the container closes |
| 9 | Destruction callbacks | `@PreDestroy` etc. | Triggered by closing the context |
| 10 | Destroyed | Removed from the container | |

> **Precision note:** The lecture does not mention **`BeanPostProcessor`**. It is the extension point around steps 6–8: `postProcessBeforeInitialization` runs just before init callbacks (this is where `@PostConstruct` is actually processed) and `postProcessAfterInitialization` just after, which is where **AOP proxies** (for `@Transactional`, etc.) are created. So the object you inject may be a proxy wrapping your bean.



### 3. Steps 1–5 in Practice

- The container starts, reads `AppConfig`, builds definitions for all beans, then instantiates them in dependency order (Lectures 05 and 06).
- `OrderService` depends on `PaymentService`: creating `OrderService` forces `PaymentService` to be created first.
- **Constructor injection:** object creation and dependency injection happen together. The object is born complete.
- **Setter/field injection:** the object is created first, *then* injected. Between these two moments the object exists without its dependencies.



### 4. Aware Interfaces (Callback Interfaces)

A bean sometimes needs infrastructure information about itself. Spring provides `*Aware` interfaces; implement one and Spring **calls your method back** with the data.

| Interface | Method Spring calls | Gives you |
|---|---|---|
| `BeanNameAware` | `setBeanName(String name)` | The bean's name in the container |
| `ApplicationContextAware` | `setApplicationContext(ApplicationContext ctx)` | The container this bean lives in |
| (others: `BeanFactoryAware`, `EnvironmentAware`, ...) | | |

#### Callback methods

- A **callback method** is one that *the framework* calls, not you.
- Each Aware interface is a functional interface with one setter-style method.
- If your bean does **not** implement the interface, Spring simply skips that step.

#### Name and effect

- The method is named `setBeanName` because Spring **sets** (supplies) the value for you. You normally just store or log it.
- **Calling the method yourself changes nothing**: `userBean.setBeanName("x")` only runs your method body; the real bean name in the container stays the same.
- Default bean name is the class name with a lower-case first letter (`userService`); override with `@Component("userBean")`.

#### When are they useful?

| Use | Example |
|---|---|
| Logging | Prefix each log line with the bean name |
| Multiple containers | Know which container (annotation-based vs XML-based) a bean came from, for debugging |

Business code rarely needs them; they are mostly framework plumbing.

> **Gotcha (added):** Using `ApplicationContextAware` just to call `getBean(...)` inside your classes is the **service-locator** pattern. It hides dependencies and defeats DI. Prefer constructor injection.



### 5. Initialization Callbacks

#### 5.1 Why they exist

Moment: **the object is created and its dependencies are injected, but nobody is using it yet.** This is the place for work that needs the bean fully wired:

| Typical use | Example |
|---|---|
| Load a large or expensive resource | Preload reference data, open a connection pool |
| Pre-fill or clear in-memory structures | Fill a map, clear a cache |
| Validate configuration | Fail early if a required value is missing |
| Start background work | Warm-up |

#### 5.2 Three ways

| Way | How | Today |
|---|---|---|
| `@PostConstruct` | Annotate a no-arg method | **Most used.** No interface, no Spring dependency |
| `InitializingBean` | Implement `afterPropertiesSet()` | Older style; couples the class to Spring |
| `init-method` | Name a method in `@Bean(initMethod = "start")` | For classes you **cannot annotate** (third-party) |

```java
@Component
public class CardService {
    @PostConstruct
    public void init() { /* runs after injection */ }
}
```

#### 5.3 Order when several are present

```
constructor
  → dependency injection
    → Aware callbacks
      → @PostConstruct
        → InitializingBean.afterPropertiesSet()
          → custom init-method
```

> **Precision note:** The lecture presents the three as alternatives and does not give their relative order. If a class uses more than one, this is the order.

#### 5.4 The jakarta annotations dependency

- `@PostConstruct` and `@PreDestroy` are **standard annotations (JSR-250 / Jakarta Annotations)**, not Spring's own. They are not inside `spring-context`.
- A bare Spring Core project needs `jakarta.annotation:jakarta.annotation-api`. **Spring Boot's starters already include it.**
- Since Spring 6, use `jakarta.annotation.*`, not `javax.annotation.*`.

#### 5.5 Why not just use the constructor?

| Constructor | `@PostConstruct` |
|---|---|
| Runs very early; with setter/field injection, **dependencies are not injected yet** (you would get `NullPointerException`) | Runs after **all** dependencies are injected |
| Should stay **light**: assign fields only | Fine for heavy work (file loading, warm-up) |
| Slow constructors slow down object creation | Keeps creation fast and initialization explicit |

> **Gotcha (added):** `@PostConstruct` runs **before** the proxy that provides `@Transactional`/AOP is wrapped around the bean, so those annotations do not apply to calls made from inside `@PostConstruct`.



### 6. The Bean Is Ready

After initialization the bean is handed to whoever needs it (`getBean`, or injection). Your business logic calls its methods for as long as the container lives.



### 7. Destruction Callbacks

#### 7.1 Why

Mirror image of initialization: **undo what init did**, before the bean disappears.

| Typical use | Example |
|---|---|
| Release resources | Close connections, files, thread pools |
| Flush pending data | Write counters/buffers to Redis or disk |
| Clear/invalidate | Empty caches, deregister listeners |

#### 7.2 Three ways (mirror of initialization)

| Initialization | Destruction |
|---|---|
| `@PostConstruct` | `@PreDestroy` (**most used**) |
| `InitializingBean.afterPropertiesSet()` | `DisposableBean.destroy()` |
| `@Bean(initMethod = "start")` | `@Bean(destroyMethod = "stop")` |

Order at shutdown: `@PreDestroy` → `DisposableBean.destroy()` → custom `destroy-method`.

> **Gotcha (added):** For `@Bean` methods Spring **infers** a public no-arg `close()` or `shutdown()` method as the destroy method automatically. Set `destroyMethod = ""` to disable that.

#### 7.3 When do destruction callbacks run?

Only when the **container is closed**. In a plain Java program that means calling `close()`:

```java
ConfigurableApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// ... use beans ...
ctx.close();      // triggers @PreDestroy / destroy() / destroy-method
```

- `close()` is on `ConfigurableApplicationContext`, not on the plain `ApplicationContext` interface (which is why the lecture changes the variable's type). `AnnotationConfigApplicationContext` implements it, so `try-with-resources` works.
- If your program just ends with no `close()`, **destruction callbacks never run**. `ctx.registerShutdownHook()` makes the JVM close the context on normal exit and on `SIGTERM`.
- **Spring Boot registers the shutdown hook for you.** In Kubernetes: pod gets `SIGTERM` → JVM shutdown hook → context closes → `@PreDestroy` runs. A hard kill (`SIGKILL`) runs none of it.



### 8. Lifecycle Variants

#### 8.1 Singleton + `@Lazy`

| Step | Singleton eager | Singleton lazy |
|---|---|---|
| Container start, config read, bean definition | At startup | At startup |
| Instantiate → inject → Aware → init | At startup | **On first use only** (`getBean` or injection into another bean) |
| Ready, used | After startup | After first use |
| Destroy callbacks | On close | On close, **only if the bean was ever created** |

The bean definition exists either way; only creation is deferred.

#### 8.2 Prototype

```
container start → read config → bean definition       (same as singleton)
on each getBean / injection:
   instantiate → inject → Aware → init callbacks → HAND OVER to caller
                                                          │
                       container forgets the object  ◀────┘
   (no @PreDestroy / destroy() / destroy-method, ever)
```

| Question | Answer |
|---|---|
| Are init callbacks (`@PostConstruct`) called? | **Yes**, on every new instance |
| Are destroy callbacks called? | **No**, not even on `context.close()` |
| Who frees memory? | **Garbage collection**, once the caller drops its reference |
| Who closes resources inside it? | **You** (`try-with-resources`, `finally`, an explicit `close()`) |

Why the container lets go: if it held every prototype instance it ever created, each `getBean` would add one more strong reference. The garbage collector could not reclaim any of them, and the heap would fill (memory leak).



### 9. Hack Demonstrated: Breaking a Cycle with `@PostConstruct`

The lecture shows `A` (needs `B` in its constructor) and `B` (no constructor dependency, plain setter):

```
A(B b)              ← constructor injection (Spring creates B first)
B.setA(A a)         ← no injection annotation
A.@PostConstruct: b.setA(this)    ← A wires the back-reference itself
```

- It works because `B` no longer *needs* `A` to be created, so the constructor cycle disappears; `A` then **manually** hands itself to `B` once it is ready.
- It is still a circular dependency, with manual wiring on top.
- Boot disables circular references by default; this pattern avoids that check, but the design problem stays.

> **Rule from Lecture 06 stands:** refactor responsibilities so no cycle exists.



### Code

#### Dependency for `@PostConstruct` / `@PreDestroy` in plain Spring

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>7.0.0</version> <!-- use a current 7.0.x -->
    </dependency>
    <dependency>
        <groupId>jakarta.annotation</groupId>
        <artifactId>jakarta.annotation-api</artifactId>
        <version>3.0.0</version>
    </dependency>
</dependencies>
```

#### Full lifecycle demo

```java
package in.strikes;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.util.HashMap;
import java.util.Map;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

@Component
public class CardService implements BeanNameAware, ApplicationContextAware {
    private final Map<Integer, String> map;

    public CardService() {
        this.map = new HashMap<>();
        System.out.println("CardService constructor called");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("Bean name is " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("Context type is " + ctx.getClass().getSimpleName());
    }

    @PostConstruct
    public void init() {
        map.put(1, "Aditya");
        map.put(2, "Rohit");
        System.out.println("Bean is ready");
    }

    public String getValue(int key) {
        return map.get(key);
    }

    @PreDestroy
    public void stop() {
        map.clear();
        System.out.println("Bean is getting destroyed");
    }
}
```

```java
package in.strikes;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Main {
    public static void main(String[] args) {
        try (AnnotationConfigApplicationContext ctx =
                     new AnnotationConfigApplicationContext(AppConfig.class)) {
            CardService card = ctx.getBean(CardService.class);
            System.out.println(card.getValue(1));
        }   // close() runs here → @PreDestroy
    }
}
// output: constructor / Bean name is cardService / Context type is ... /
//         Bean is ready / Aditya / Bean is getting destroyed
```

#### WRONG: heavy work in the constructor that uses a dependency

```java
@Component
public class MatchScoreCache {
    @Autowired
    private FileLoader loader;               // field injection

    public MatchScoreCache() {
        loader.loadAll();                    // NullPointerException: loader not injected yet
    }
}
```

#### RIGHT: initialize after injection

```java
@Component
public class MatchScoreCache {
    private final FileLoader loader;

    public MatchScoreCache(FileLoader loader) {
        this.loader = loader;                // constructor only assigns
    }

    @PostConstruct
    public void warmUp() {
        loader.loadAll();                    // heavy work after wiring
    }

    @PreDestroy
    public void flush() {
        loader.closeQuietly();               // release/flush before shutdown
    }
}
```

#### Alternative styles

```java
@Component
public class OldStyle implements InitializingBean, DisposableBean {
    @Override
    public void afterPropertiesSet() { /* init */ }

    @Override
    public void destroy() { /* cleanup */ }
}
```

```java
@Configuration
public class AppConfig {

    @Bean(initMethod = "start", destroyMethod = "stop")
    public ThirdPartyClient client() {       // class you cannot annotate
        return new ThirdPartyClient();
    }
}
```

#### WRONG: expecting destruction callbacks for a prototype, or without `close()`

```java
@Component
@Scope("prototype")
public class Report {
    @PreDestroy
    public void cleanup() {                  // NEVER called by the container
        System.out.println("closing resources");
    }
}

ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// ... program ends, no ctx.close(): even singleton @PreDestroy never runs
```

#### RIGHT: close what you own

```java
Report report = ctx.getBean(Report.class);
try {
    // use report
} finally {
    report.closeResources();                 // your cleanup for prototypes
}
```

```java
ctx.registerShutdownHook();                  // close the context on JVM exit / SIGTERM
```



#### Quick Self-Check

1. **List the lifecycle of a singleton bean in order.**
   *Answer:* Container starts → configuration read → bean definitions → instantiate → inject dependencies → Aware callbacks → initialization callbacks → ready → destruction callbacks (on close) → destroyed.

2. **When are instantiation and dependency injection one step, and when two?**
   *Answer:* With constructor injection they are one step (the object cannot be created without its dependencies). With setter/field injection the object is created first and injected afterwards.

3. **What is an Aware interface? Does calling `setBeanName("x")` yourself rename the bean?**
   *Answer:* A callback interface through which Spring hands the bean infrastructure data (its name, its context). Calling it yourself only runs your method body; the container's bean name stays unchanged.

4. **Why use `@PostConstruct` instead of doing the same work in the constructor?**
   *Answer:* `@PostConstruct` runs after all dependencies (including setter/field) are injected; the constructor runs earlier and should stay light.

5. **Name the three initialization mechanisms and their order.**
   *Answer:* `@PostConstruct`, then `InitializingBean.afterPropertiesSet()`, then the custom `init-method`.

6. **When do `@PreDestroy` callbacks run, and what makes them run in a plain Java program?**
   *Answer:* When the container is closed (`close()`, `try-with-resources`, or a shutdown hook). Without it, they never run.

7. **What is different for prototype beans?**
   *Answer:* The container runs creation, injection, Aware and init callbacks, then hands the object over and forgets it: no destroy callbacks, not even on `close()`. You clean up; GC frees memory.

8. **Why does the container not keep prototype instances?**
   *Answer:* It would hold references to every instance ever handed out, preventing garbage collection and leaking memory.



#### Golden Rules / Checklist

- [ ] Keep constructors light; assign fields only
- [ ] Do dependency-dependent or heavy setup in `@PostConstruct`
- [ ] Mirror every acquire in init with a release in `@PreDestroy`
- [ ] Prefer `@PostConstruct`/`@PreDestroy` over `InitializingBean`/`DisposableBean`
- [ ] Use `initMethod`/`destroyMethod` on `@Bean` for third-party classes
- [ ] Add `jakarta.annotation-api` in plain Spring projects; Boot includes it
- [ ] Use `jakarta.annotation`, not `javax.annotation`
- [ ] Close the context (try-with-resources) or register a shutdown hook
- [ ] Never expect destroy callbacks for prototype beans; clean up yourself
- [ ] Don't call Aware setters yourself; they are callbacks
- [ ] Don't use `ApplicationContextAware` as a shortcut to `getBean`
- [ ] Remember `@PostConstruct` runs before AOP proxies wrap the bean
- [ ] Remember the order: `@PostConstruct` → `afterPropertiesSet` → init-method (and the reverse family for destroy)



### Practice Questions

#### Basic

1. Create `CardService` with a constructor, `@PostConstruct` and `@PreDestroy` that each print a line. Run it and note the order. What happens if you remove `ctx.close()`?

2. Implement `BeanNameAware` and print the bean name. Rename the bean with `@Component("cardBean")` and rerun.

3. Why does a plain Spring project fail to compile `@PostConstruct`, and what do you add?

4. Draw the singleton lifecycle as a numbered list from container start to destroyed.

#### Intermediate

5. Make `CardService` use all three init styles together. Predict the printed order, then verify. Do the same for destroy.

6. Convert a `@Component` to a `@Bean` with `initMethod`/`destroyMethod` for a class you "cannot modify".

7. Mark the bean `@Lazy` and predict which lifecycle steps run at startup. Then call `getBean` and re-predict.

8. Make the bean `@Scope("prototype")` and show: `@PostConstruct` runs on each `getBean`, `@PreDestroy` never runs, even after `ctx.close()`.

#### Advanced / Interview-style

9. Why can't you put the file-loading logic in the constructor when dependencies are injected into fields? Show the `NullPointerException`.

10. Where do `BeanPostProcessor`s fit in the lifecycle, and why might the injected bean be a proxy? What does that mean for `@Transactional` inside `@PostConstruct`?

11. In Kubernetes, a pod receives `SIGTERM` on rolling deploy. Trace how that results in your `@PreDestroy` running in a Spring Boot app, and what changes with `SIGKILL`.

12. Explain why the container does not manage the destruction of prototype beans. How would you safely handle a prototype that holds a database connection?


## 8. XML-Based Configuration



## The Production Scenario

You join a team maintaining a 10-year-old Flipkart-style order system. Its `applicationContext.xml` is 2,500 lines. Three situations show up:

- A bean fails at startup with `ClassNotFoundException`, because someone renamed a package and the class name inside an XML string was not updated (the IDE refactor did not catch it).
- The team wants to migrate to annotations, but not all at once: the new code uses `@Component`, the old modules stay in XML, and both must live in **one** container.
- A new joiner asks, "Where is `OrderService` wired to `PaymentService`?" There is no `@Autowired` anywhere. The answer is in the XML.

This lecture teaches you to read that XML and to bridge it with annotations.



### 1. Same Container, Different Metadata Source

```
                 ┌─────────────────────────────┐
 Annotated   ──▶ │                             │
 @Configuration  │   IoC container            │──▶ BeanDefinitions ──▶ beans
                 │   (ApplicationContext)      │      (same lifecycle,
 XML file    ──▶ │                             │       scopes, DI)
 (beans.xml)     └─────────────────────────────┘
```

| | Annotation-based | XML-based |
|---|---|---|
| Container class | `AnnotationConfigApplicationContext` | `ClassPathXmlApplicationContext` |
| Metadata source | A `@Configuration` class (read via reflection) | An XML file on the classpath |
| What you declare | `@Component`, `@Bean`, `@ComponentScan` | `<bean>` elements |
| After the definitions are built | Same | Same |

The container's job is unchanged: read metadata, build bean definitions, create beans, wire dependencies, run lifecycle callbacks.

#### When you need this

| Situation | Use |
|---|---|
| New Spring Boot projects | Annotations only |
| Legacy codebases | Read/maintain XML |
| Gradual migration | **Hybrid**: XML + annotations in one container |
| Interview curiosity | Know it exists and how it maps |



### 2. Setting Up an XML Context

#### 2.1 Where the file lives

`src/main/resources/` is on the **classpath**, so a file placed there is found by `ClassPathXmlApplicationContext("beans.xml")`.

```
src/main/
├── java/in/strikes/ ...          <- plain classes, no Spring annotations needed
└── resources/
    └── beans.xml                  <- configuration metadata
```

> **Added:** Variants exist: `FileSystemXmlApplicationContext` (path on disk), `GenericXmlApplicationContext`, and you can pass **several files** to the constructor.

#### 2.2 The XML header

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           https://www.springframework.org/schema/beans/spring-beans.xsd">
    <!-- bean definitions go here -->
</beans>
```

- The `xmlns`/`schemaLocation` lines define the **schema** (which tags are legal). The root element is `<beans>`.
- **Copy it** from the Spring reference documentation; nobody types it from memory.
- Use the version-less `spring-beans.xsd` so it tracks your Spring version.

#### 2.3 Starting the container

```java
try (ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml")) {
    OrderService order = ctx.getBean("orderService", OrderService.class);
    order.placeOrder();
}
```

- `ClassPathXmlApplicationContext` implements `ConfigurableApplicationContext`, so it has `close()` (and works with `try-with-resources`) and triggers destroy callbacks (Lecture 07).
- Beans are **singleton and eager by default**, exactly as before: a constructor `println` fires at startup.



### 3. Declaring Beans

```xml
<bean id="orderService" class="in.strikes.OrderService"/>
```

| Attribute | Meaning |
|---|---|
| `id` | Unique identifier of the bean |
| `class` | **Fully qualified** class name (package + class) |

- Convention: `id` = class name in camelCase.
- Plain Java classes: **no `@Component` needed**. The class carries no Spring annotations.
- Equivalent to a `@Bean` method returning `new OrderService()`.

#### 3.1 `id` vs `name`

| | `id` | `name` |
|---|---|---|
| How many | Exactly one | One **or several** aliases (separated by comma, space or semicolon) |
| Uniqueness | Must be unique | Must be unique across names |
| Lookup | `getBean("orderService")` | `getBean("alias1")` works too |

```xml
<bean id="orderService" name="orderBean, orderSvc" class="in.strikes.OrderService"/>
```

> **Precision note:** The lecture says that without an `id` the bean has no name. More precisely, Spring generates one (the class name plus `#0`, for example `in.strikes.OrderService#0`). You cannot sensibly look it up by name, but `getBean(OrderService.class)` still works if there is exactly one bean of that type. Note also the difference from annotations: for `@Component` the default name is the decapitalized class name, and for `@Bean` it is the method name.

> **Gotcha:** Two beans with the same `id` fail at startup (`BeanDefinitionParsingException`: name already used). This is the XML equivalent of a duplicate bean name.

#### 3.2 Fetching beans

| Call | Returns | Notes |
|---|---|---|
| `getBean("orderService")` | `Object` | Needs a cast |
| `getBean(OrderService.class)` | `OrderService` | Fails with `NoUniqueBeanDefinitionException` if two beans of that type exist |
| `getBean("orderService", OrderService.class)` | `OrderService` | **Best**: no cast, no ambiguity |

An unknown id throws `NoSuchBeanDefinitionException`.



#### 4. Dependency Injection in XML

Your bean classes are plain Java:

```java
public class OrderService {
    private final PaymentService payment;
    public OrderService(PaymentService payment) { this.payment = payment; }
    public void placeOrder() { payment.pay(); System.out.println("Order placed"); }
}
```

#### 4.1 Constructor injection: `<constructor-arg>`

```xml
<bean id="upiPaymentService"  class="in.strikes.payment.UpiPaymentService"/>
<bean id="cardPaymentService" class="in.strikes.payment.CardPaymentService"/>

<bean id="orderService" class="in.strikes.OrderService">
    <constructor-arg ref="cardPaymentService"/>
</bean>
```

| Attribute | Use |
|---|---|
| `ref="beanId"` | Inject another **bean** |
| `value="..."` | Inject a literal (String, number; converted to the parameter type) |
| `index="0"` | Match by constructor position (0-based) |
| `name="type"` | Match by constructor parameter name |
| `type="java.lang.String"` | Match by type |

```xml
<bean id="upiPaymentService" class="in.strikes.payment.UpiPaymentService">
    <constructor-arg name="type" value="UPI"/>
    <constructor-arg name="retryCount" value="3"/>
</bean>
```

> **Gotcha (added):** Matching by `name` needs parameter names present at runtime. In Spring 6.1+ that means compiling with `-parameters` (set `maven.compiler.parameters` to `true` for plain Maven projects). Positional (`index`) or `type` matching has no such requirement.

#### 4.2 Setter injection: `<property>`

```xml
<bean id="orderService" class="in.strikes.OrderService">
    <property name="paymentService" ref="cardPaymentService"/>
</bean>
```

- `name="paymentService"` means "call `setPaymentService(...)`". The property name is derived from the **setter** (JavaBeans convention), not from a field. If the setter is `setPaymentServiceBean`, the property name is `paymentServiceBean`.
- `ref` = which bean to pass; `value` for literals.
- The class needs a no-arg constructor and the setter.

#### 4.3 Field injection: not possible

XML only calls constructors and setters; it has no way to write to a `private` field. That is why `@Autowired` field injection exists only with annotation processing.

#### 4.4 Several candidates: no confusion in XML

With annotations, two `PaymentService` beans give `NoUniqueBeanDefinitionException`, resolved with `@Primary`/`@Qualifier`. With XML, `ref="cardPaymentService"` **names the exact bean**, so there is nothing to resolve (the `ref` plays the role of `@Qualifier`). Equivalent flags exist (`primary="true"` on a `<bean>`), but are rarely needed.



### 5. Collections and Values

```xml
<bean id="userService" class="in.strikes.UserService">
    <constructor-arg name="userNames">
        <list>
            <value>Aditya</value>
            <value>Rohit</value>
            <value>Rohan</value>
        </list>
    </constructor-arg>
</bean>
```

| Java type | XML element |
|---|---|
| `List<T>` | `<list><value>..</value></list>` |
| `Set<T>` | `<set><value>..</value></set>` |
| `Map<K,V>` | `<map><entry key=".." value=".."/></map>` |
| `Properties` | `<props><prop key="..">..</prop></props>` |

Use `<ref bean="id"/>` instead of `<value>` to put **beans** into the collection.



### 6. Scope, Laziness and Lifecycle Callbacks in XML

| Concept | Annotation | XML attribute on `<bean>` |
|---|---|---|
| Prototype scope | `@Scope("prototype")` | `scope="prototype"` |
| Lazy initialization | `@Lazy` | `lazy-init="true"` |
| Init callback | `@PostConstruct` | `init-method="init"` |
| Destroy callback | `@PreDestroy` | `destroy-method="cleanup"` |

```xml
<bean id="userService" class="in.strikes.UserService"
      scope="prototype"
      init-method="init"
      destroy-method="cleanup"/>
```

- `init`/`cleanup` are ordinary methods on the class; **no annotations and no interfaces** needed.
- The rules from Lectures 06 and 07 still apply: a prototype is created on request, its init method runs, but **its destroy method is never called**; singleton destroy callbacks run only when the context is **closed**.
- A root-level `<beans lazy-init="true">` or `default-init-method` applies to all beans in the file.



### 7. Splitting Configuration Across Files

One large XML file becomes unreadable, so split it (for example `beans.xml` for orders/payments, `beans2.xml` for users) and combine with `<import>`:

```xml
<!-- app-config.xml -->
<beans ...>
    <import resource="beans.xml"/>
    <import resource="beans2.xml"/>
</beans>
```

```java
new ClassPathXmlApplicationContext("app-config.xml");
```

Alternative: `new ClassPathXmlApplicationContext("beans.xml", "beans2.xml")`. Each file needs its own header and root `<beans>` element.



### 8. Hybrid Setups: XML + Annotations in One Container

You do **not** run two containers. One container can read both.

| Direction | Mechanism |
|---|---|
| XML context, also pick up annotated classes | `<context:annotation-config/>` and/or `<context:component-scan base-package="in.strikes"/>` in the XML (requires the `context` namespace in the header) |
| Annotation context, also load an old XML file | `@ImportResource("classpath:beans.xml")` on a `@Configuration` class |

```java
@Configuration
@ImportResource("classpath:beans.xml")
@ComponentScan("in.strikes")
public class AppConfig {}
```

> **Added:** `@ImportResource` is how real migrations proceed (and how a Spring Boot app can still load legacy XML). Move bean definitions from XML to annotations module by module; the container does not care where each definition came from.



### 9. Annotation ↔ XML Mapping

| Annotation style | XML equivalent |
|---|---|
| `@Component` / `@Bean` | `<bean id class/>` |
| `@Autowired` constructor | `<constructor-arg ref="..."/>` |
| `@Autowired` setter | `<property name ref/>` |
| `@Autowired` field | **Not possible** |
| `@Qualifier("x")` | `ref="x"` |
| `@Primary` | `primary="true"` |
| `@Scope("prototype")` | `scope="prototype"` |
| `@Lazy` | `lazy-init="true"` |
| `@PostConstruct` / `@PreDestroy` | `init-method` / `destroy-method` |
| `@ComponentScan` | `<context:component-scan base-package=.../>` |
| `@Configuration` class passed to the context | XML file passed to `ClassPathXmlApplicationContext` |
| Literal value (`@Value`) | `value="..."` |



### 10. Pros and Cons

| Aspect | XML | Annotations |
|---|---|---|
| Java classes | Clean POJOs, no Spring imports | Carry Spring annotations |
| Where wiring is visible | One file | Spread across classes |
| Type safety | **Strings**: class names/ids checked only at startup | Compiler/IDE-checked, refactor-safe |
| Verbosity | High; files grow large | Low |
| Changing wiring | Edit config (still packaged in the app) | Edit code |
| Today | Legacy and hybrid | Standard; **all of Spring Boot** |

> **Precision note:** The lecture frames "everything in one file" as the main advantage. The practical weakness is the flip side: names live inside strings, so renaming a class or package silently breaks the XML until the context starts. Compile-time safety is the main reason annotations won.

> **Gotcha (added):** IDEs sometimes show warnings such as "could not autowire, no beans of String type found" on XML-configured classes. They come from static analysis not seeing the XML values; if the app starts and behaves correctly, they are harmless.



### Code

#### Beans (plain Java, no Spring annotations)

```java
// PaymentService.java
package in.strikes.payment;

public interface PaymentService {
    void pay();
}
```

```java
// UpiPaymentService.java
package in.strikes.payment;

public class UpiPaymentService implements PaymentService {
    private final String type;
    private final int retryCount;

    public UpiPaymentService(String type, int retryCount) {
        this.type = type;
        this.retryCount = retryCount;
    }

    @Override
    public void pay() {
        System.out.println("Paying via " + type + " with " + retryCount + " retries");
    }
}
```

```java
// CardPaymentService.java
package in.strikes.payment;

public class CardPaymentService implements PaymentService {
    @Override
    public void pay() {
        System.out.println("Paying via card");
    }
}
```

```java
// OrderService.java
package in.strikes;

import in.strikes.payment.PaymentService;

public class OrderService {
    private PaymentService payment;

    public OrderService() {}                        // needed for setter injection

    public void setPaymentServiceBean(PaymentService payment) {   // property name: paymentServiceBean
        this.payment = payment;
    }

    public void placeOrder() {
        payment.pay();
        System.out.println("Order placed");
    }
}
```

#### `src/main/resources/beans.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="upiPaymentService" class="in.strikes.payment.UpiPaymentService">
        <constructor-arg index="0" value="UPI"/>
        <constructor-arg index="1" value="3"/>
    </bean>

    <bean id="cardPaymentService" class="in.strikes.payment.CardPaymentService"/>

    <bean id="orderService" class="in.strikes.OrderService">
        <property name="paymentServiceBean" ref="cardPaymentService"/>   <!-- picks the exact bean -->
    </bean>
</beans>
```

#### Main

```java
package in.strikes;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {
    public static void main(String[] args) {
        try (ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml")) {
            OrderService order = ctx.getBean("orderService", OrderService.class);
            order.placeOrder();
        }
    }
}
// output: Paying via card / Order placed
```

#### WRONG: mistakes the container reports

```xml
<!-- 1. wrong class name (typo / renamed package): ClassNotFoundException at startup -->
<bean id="orderService" class="in.strike.OrderService"/>

<!-- 2. duplicate id: BeanDefinitionParsingException -->
<bean id="orderService" class="in.strikes.OrderService"/>
<bean id="orderService" class="in.strikes.OrderService"/>
```

```java
// 3. two beans of type PaymentService: NoUniqueBeanDefinitionException
ctx.getBean(PaymentService.class);

// 4. bean declared without id, looked up by name: NoSuchBeanDefinitionException
ctx.getBean("orderService");
```

#### RIGHT

```java
ctx.getBean("cardPaymentService", PaymentService.class);   // id + type: exact bean, no cast
```

#### Scope and lifecycle in XML

```java
public class UserService {
    public UserService() { System.out.println("UserService created"); }
    public void init()    { System.out.println("init"); }
    public void cleanup() { System.out.println("cleanup"); }
}
```

```xml
<bean id="userService" class="in.strikes.UserService"
      init-method="init" destroy-method="cleanup"/>
<!-- add scope="prototype": init still runs per instance, cleanup NEVER runs -->
```

#### Collections

```xml
<bean id="userService" class="in.strikes.UserService">
    <constructor-arg name="userNames">
        <list>
            <value>Aditya</value>
            <value>Rohit</value>
        </list>
    </constructor-arg>
</bean>
```

#### Combining files, and hybrid

```xml
<!-- app-config.xml -->
<beans ...>
    <import resource="beans.xml"/>
    <import resource="beans2.xml"/>
</beans>
```

```java
@Configuration
@ImportResource("classpath:beans.xml")   // annotation config loading the legacy XML
@ComponentScan("in.strikes")
public class AppConfig {}
```



#### Quick Self-Check

1. **Does the IoC container behave differently for XML vs annotation metadata?**
   *Answer:* No. Only the metadata source and the context class differ. Bean definitions, scopes, DI and lifecycle behave the same.

2. **What do `id` and `class` mean on a `<bean>`?**
   *Answer:* `id` is the unique identifier; `class` is the fully qualified class name Spring instantiates.

3. **Difference between `id` and `name`?**
   *Answer:* A bean has one `id` but may have multiple `name` aliases. Either can be used in `getBean`.

4. **How do you inject a dependency by constructor and by setter in XML?**
   *Answer:* `<constructor-arg ref/value .../>` for constructors; `<property name="x" ref/value .../>` for setters (calls `setX`).

5. **Why is field injection impossible in XML?**
   *Answer:* XML can only call constructors and setters; it has no mechanism to assign a private field.

6. **How does XML avoid the "two candidates" ambiguity?**
   *Answer:* `ref="beanId"` names the exact bean to inject, so no `@Primary`/`@Qualifier` is needed.

7. **How do you set prototype scope, lazy init, and init/destroy callbacks in XML?**
   *Answer:* `scope="prototype"`, `lazy-init="true"`, `init-method="..."`, `destroy-method="..."` on the `<bean>`.

8. **How can one container use both XML and annotations?**
   *Answer:* `<context:component-scan>` (and `<context:annotation-config/>`) in the XML, or `@ImportResource` on a `@Configuration` class. It is one container with definitions from both.



#### Golden Rules / Checklist

- [ ] New code: annotations. XML only for legacy or migration
- [ ] Copy the XML header from the Spring docs; do not memorize it
- [ ] Put the XML on the classpath (`src/main/resources`)
- [ ] Use `ClassPathXmlApplicationContext` and close it (`try-with-resources`)
- [ ] Use fully qualified class names; double-check them after refactors
- [ ] Give every bean a unique `id`
- [ ] Prefer `getBean("id", Type.class)`
- [ ] Use `<constructor-arg>` for required dependencies, `<property>` for optional ones
- [ ] Use `ref` for beans and `value` for literals
- [ ] Remember the setter naming rule for `<property name=...>`
- [ ] Remember that prototype destroy callbacks never run (same as before)
- [ ] Split big configurations with `<import>`
- [ ] Migrate gradually with hybrid setups (`@ImportResource`, `<context:component-scan>`)



### Practice Questions

#### Basic

1. Convert your annotation-based `OrderService` + `PaymentService` example to XML and run it with `ClassPathXmlApplicationContext`.

2. Add a second `PaymentService` implementation. Inject each one in turn using only `ref`.

3. Break the XML on purpose (wrong class name, duplicate id, unknown id in `getBean`). Match each failure to its exception.

4. Add `name="a, b"` aliases to a bean and fetch it through each alias.

#### Intermediate

5. Inject two constructor arguments, `String type` and `int retryCount`, by `index`, then by `name`. What do you need to compile with for `name` to work?

6. Switch an injection from constructor to setter. Which class changes are needed? What is the XML property name if the setter is `setPaymentServiceBean`?

7. Inject a `List<String>` and a `Map<String,String>` through XML.

8. Give a bean `scope="prototype"` and `init-method`/`destroy-method`. Predict and verify which methods run when you call `ctx.close()`.

#### Advanced / Interview-style

9. Explain why the XML and annotation setups produce the same lifecycle. What does the container see in both cases?

10. Design a migration plan for a 2,500-line `applicationContext.xml` to annotations with no downtime. Which mechanisms let both styles coexist?

11. Compare XML and annotations on type safety and refactoring. How would a package rename bite each?

12. Given two `PaymentService` beans, show how the ambiguity is resolved in annotation config and in XML. Why does XML not need `@Primary`?

## 9. Spring Boot and Auto-Configuration


### The Production Scenario

A teammate builds a nightly settlement job as a plain Spring Boot app: it reads files, writes to a DB, and exits. Months later someone adds `spring-boot-starter-web` "for a health endpoint". The next deployment, the Kubernetes **Job** never finishes: the pod stays `Running` forever.

Why: the web starter on the classpath triggered auto-configuration, which started **embedded Tomcat on 8080**. Tomcat's non-daemon threads keep the JVM alive after `main()` returns, so the job never exits (Lectures 02 and 07).

You change one dependency and the application's behaviour changes. That is auto-configuration, and it is why this lecture matters in operations as much as in coding.



### 1. Recap: What Spring Core Made You Do by Hand

| Step in Spring Core | Who did it |
|---|---|
| Create the container: `new AnnotationConfigApplicationContext(AppConfig.class)` | You |
| Write a `@Configuration` class | You |
| Add `@ComponentScan("pkg")` | You |
| Mark beans with `@Component` / `@Bean` | You |
| Fetch beans: `ctx.getBean(OrderService.class)` and call a method | You |
| Add dependencies and pick compatible versions | You |

> Everything above still works exactly the same underneath. Boot automates the plumbing.



### 2. Spring Boot Is Not "Web"

- Boot's goal: **automate configuration** of the Spring ecosystem.
- Boot is independent of Spring MVC. It can run a plain console-style application (this lecture) and can also host Spring MVC when you add a web dependency.
- Rule: *what Boot sets up depends on what is on your classpath.* No web library, no web server.



### 3. Project Setup

#### 3.1 The starter

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```

- A **starter** is a curated bundle of dependencies for one purpose, so you declare one thing and get a compatible set.
- `spring-boot-starter` is the **core starter**: Spring Core, logging, auto-configuration support and more, via transitive dependencies (visible in the Maven tool window as a tree: `starter-logging`, `starter-autoconfigure`, ...).
- It replaces the lone `spring-context` dependency from Spring Core, and gives you a lot more.
- Adding the **web starter** brings Spring MVC and embedded Tomcat.

> **Precision note:** The lecture adds the web starter as `spring-boot-starter-web` in its examples. Starter artifact names changed in Boot 4 (the generated project from Initializr used `spring-boot-starter-webmvc` in Lecture 03). Take the exact name from Spring Initializr, not from memory.

#### 3.2 Two ways to start a Boot project

| Way | Notes |
|---|---|
| Hand-written Maven project | You add the starter and, ideally, the Boot parent |
| **Spring Initializr** (start.spring.io) | Generates `pom.xml` with the parent, starters, main class and layout (the lecture's second project) |

Initializr also adds `spring-boot-starter-test`; you may ignore it while not writing tests.

> **Gotcha (DevOps):** The IDE's "vulnerability found in dependency" warning is real information. It is fine to ignore on a throw-away practice project, but for real services bump to a patched release and run dependency scanning in CI.



### 4. The Parent POM: Why There Are No Versions

Generated `pom.xml` (Initializr) contains:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.x.y</version>
    <relativePath/>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>   <!-- no <version> -->
    </dependency>
</dependencies>
```

```
Super POM
   ▲
spring-boot-dependencies          (version table / BOM for hundreds of libraries)
   ▲
spring-boot-starter-parent        (plus plugin config, Java/encoding defaults)
   ▲
your pom.xml
```

| Problem without it | What the parent gives you |
|---|---|
| You must pick a compatible version for every library | One tested, compatible version per library, from the parent's list |
| Mismatched versions break at runtime | Consistency across dependencies |
| Repeating plugin/encoding/Java settings | Sensible defaults |

- This is the `<parent>` tag from Lecture 03: anything you do not declare comes from the parent.
- A dependency declared **without `<version>`** takes its version from the parent. Add a version yourself only to override deliberately.

> **Added:** If you cannot use `spring-boot-starter-parent` (your project already has a different parent), import the **`spring-boot-dependencies` BOM** in `<dependencyManagement>` to get the same version management.



### 5. The Entry Point

```java
@SpringBootApplication
public class SpringBootCoreDemo2Application {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootCoreDemo2Application.class, args);
    }
}
```

#### 5.1 `SpringApplication.run` creates the container

- It builds and **returns a `ConfigurableApplicationContext`** (a subtype of `ApplicationContext`).
- That is the IoC container you used to create by hand. You can capture it: `ApplicationContext ctx = SpringApplication.run(...)`.
- You did **not** create it, choose the config class (it is the main class), or choose the context type (Boot decides).

> **Precision note:** Boot also decides the *kind* of application from the classpath: servlet web app, reactive web app, or plain (no web server). That is why the same `main` starts Tomcat only when the web starter is present.

#### 5.2 Why `getBean` in `main` is the wrong pattern

The lecture's first demo captures the context, calls `getBean(OrderService.class)` and then `placeOrder()`. It works, but:

- It is the Spring Core way of working, not how Boot apps are written.
- In a real web app, `placeOrder()` would be called by Boot/MVC when a request hits an endpoint (Lecture 02).
- For a non-web app, the idiom is a startup callback such as `CommandLineRunner`, which Boot calls for you after the context is ready. (Next lecture, per the lecturer.)



### 6. `@SpringBootApplication` = Three Annotations

```
@SpringBootApplication
   ├── @SpringBootConfiguration   ← this class is the main configuration class
   ├── @EnableAutoConfiguration   ← register beans for libraries on the classpath
   └── @ComponentScan             ← scan this package and below for @Component etc.
```

| Annotation | Plays the role of... | Why it exists |
|---|---|---|
| `@SpringBootConfiguration` | `@Configuration` (it is a specialised version) | Marks the main configuration class; `@Bean` methods allowed in your main class |
| `@ComponentScan` | Your old `@ComponentScan("pkg")` | Finds `@Component` classes without you writing it |
| `@EnableAutoConfiguration` | Nothing in Spring Core | Switches on Boot's auto-configuration |

You can replace `@SpringBootApplication` with these three annotations and get the same behaviour. It exists so you do not have to write all three.

(It also carries ordinary meta-annotations such as `@Inherited`, `@Documented`, `@Retention`, `@Target`, which you can ignore for now.)

#### 6.1 `@SpringBootConfiguration`: the main class is a config class

- A `@Configuration` class is itself a bean (Lecture 06), so your main class is a bean too.
- So you can put `@Bean` methods directly in the main class, and you **do not need a separate `AppConfig`**.

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) { SpringApplication.run(DemoApplication.class, args); }

    @Bean
    public UserService userService() { return new UserService(); }
}
```

> **Added:** It works, but keep the main class slim and put beans in dedicated `@Configuration` classes as the project grows. You can have as many `@Configuration` classes as you like; there is just one *main* one.

#### 6.2 `@ComponentScan` by convention

- No package given → Boot scans **the package of the class that carries `@SpringBootApplication`, and all sub-packages**.
- `@Component` classes in a **sibling or parent** package are not scanned, even with `@Component` on them.

```
in.strikes.demo                  ← main class lives here → scan root
in.strikes.demo.OrderService     ✓ scanned
in.strikes.demo.payment.X        ✓ scanned (sub-package)
in.strikes.other.Y               ✗ NOT scanned (sibling package)
```

Two ways to deal with it:

| Option | How |
|---|---|
| Follow Boot's convention (preferred) | Keep everything under the main class's package |
| Override | `@SpringBootApplication(scanBasePackages = "in.strikes")` (widen the scan to the common parent) |

This is Boot being an **opinionated framework**: follow the convention and you configure nothing; deviate and you tell Boot.



### 7. `@EnableAutoConfiguration`

#### 7.1 The problem it solves

You can declare beans two ways: `@Component` on your own classes, and `@Bean` methods in configuration classes. For a **library** class (for example a JSON parser, a Tomcat server, a DataSource) you would have to write a `@Bean` method yourself, repeating the same boilerplate in every project.

Boot's answer: **the library authors (Boot itself, and third-party starters) ship those `@Bean` definitions for you**, and `@EnableAutoConfiguration` activates them.

| How beans get registered | Who writes it | Typical use |
|---|---|---|
| `@Component` on a class | You | Your own classes |
| `@Bean` in a `@Configuration` class | You | Library classes you configure yourself |
| **Auto-configuration** | **Boot / the library** | Standard infrastructure (web server, JSON, DataSource, ...) |

#### 7.2 What an auto-configuration class looks like

It is an ordinary configuration class with conditions:

```java
@AutoConfiguration
@ConditionalOnClass(PaymentGatewayClient.class)          // only if the library is on the classpath
public class PaymentGatewayAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                            // only if the user has not defined one
    public PaymentGatewayClient paymentGatewayClient() {
        return new PaymentGatewayClient();
    }
}
```

> **Precision note:** The lecture shows `@AutoConfiguration` and `@Bean` and adds the two conditions. It does not say **how Boot finds these classes**. Boot reads a list of auto-configuration class names from each JAR on the classpath, from the file `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (one fully-qualified class name per line). `@EnableAutoConfiguration` loads that list and registers each class (subject to its conditions). Your own library can ship such a file too.

#### 7.3 The conditions

| Condition | Meaning | Example |
|---|---|---|
| `@ConditionalOnClass(X.class)` | Apply only if `X` is on the classpath (that dependency is present) | Web-server beans only if Tomcat/Spring MVC classes are present |
| `@ConditionalOnMissingBean` | Create this bean only if **you have not already defined one** | Your own `ObjectMapper`/`DataSource` replaces Boot's |
| `@ConditionalOnBean` | Only if another bean exists | |
| `@ConditionalOnProperty` | Only if a property has a given value | `spring.xyz.enabled=true` |
| `@ConditionalOnWebApplication` | Only in a web app | |

#### 7.4 The back-off principle

```
your @Component / @Bean definitions are registered first
          │
          ▼
auto-configuration runs after them
          │  @ConditionalOnMissingBean: "is a bean of this type already there?"
          ├─ yes → back off (your bean wins)
          └─ no  → create the default bean
```

So Boot gives you working defaults, and you can override any piece by defining your own bean.

#### 7.5 Demo: adding a dependency changes behaviour

| State | Result |
|---|---|
| `spring-boot-starter` only | App prints output and exits; **no server** |
| Add the web starter, change nothing else | Logs show **Tomcat started on port 8080**; the process keeps running |

Nothing in your code changed. The classes Boot's web auto-configuration needs (`@ConditionalOnClass`) now exist on the classpath, so it registered a Tomcat-based web server bean.

> **Added:** Boot decides "web or not" from the classpath, and also lets you force it: `spring.main.web-application-type=none` stops a web server from starting even when web classes are present. Use it for batch jobs.

#### 7.6 Seeing what happened

| Tool | How |
|---|---|
| Conditions report at startup | Run with `--debug` (or `debug=true` in `application.properties`): lists auto-configurations that applied (**positive matches**) and those skipped and why (**negative matches**) |
| Actuator | `/actuator/conditions` when Actuator is present |
| Exclude an auto-configuration | `@SpringBootApplication(exclude = SomeAutoConfiguration.class)` or `spring.autoconfigure.exclude=...` |



### 8. Putting It Together: What Happens on `SpringApplication.run`

| Step | What Boot does |
|---|---|
| 1 | Decides the application type from the classpath |
| 2 | Creates the matching `ApplicationContext` (the IoC container) |
| 3 | Treats the main class as the main configuration (`@SpringBootConfiguration`) |
| 4 | Scans the main class's package and sub-packages (`@ComponentScan`) → bean definitions for your `@Component`s and `@Bean`s |
| 5 | Loads auto-configuration classes (`@EnableAutoConfiguration`), keeps those whose conditions match |
| 6 | Instantiates beans, wires dependencies, runs lifecycle callbacks (Lecture 07) |
| 7 | Starts the web server only if web auto-configuration applied |



### Code

#### Spring Core style (for comparison)

```java
@Component
public class PaymentService {
    public void pay() { System.out.println("Payment done"); }
}

@Component
public class OrderService {
    private final PaymentService payment;
    public OrderService(PaymentService payment) { this.payment = payment; }
    public void placeOrder() { payment.pay(); System.out.println("Order placed"); }
}

@Configuration
@ComponentScan("in.strikes")
public class AppConfig {}

public class Main {
    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {
            ctx.getBean(OrderService.class).placeOrder();
        }
    }
}
```

#### Spring Boot style

```java
// SpringBootCoreDemo2Application.java
package in.strikes.demo;

@SpringBootApplication
public class SpringBootCoreDemo2Application {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootCoreDemo2Application.class, args);
    }
}
```

```java
// in the same package or a sub-package
@Component
public class PaymentService {
    public void pay() { System.out.println("Payment done"); }
}

@Component
public class OrderService {
    private final PaymentService payment;
    public OrderService(PaymentService payment) { this.payment = payment; }
    public void placeOrder() { payment.pay(); System.out.println("Order placed"); }
}
```

No `AppConfig`, no `@ComponentScan`, no `new ...ApplicationContext`.

#### The three annotations spelled out (equivalent)

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
public class SpringBootCoreDemo2Application {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootCoreDemo2Application.class, args);
    }
}
```

#### WRONG: a component outside the scanned package

```java
package in.strikes.other;                  // sibling of in.strikes.demo (main class package)

@Component
public class ReportService {}              // never registered; injecting it fails at startup
```

#### RIGHT

```java
package in.strikes.demo.report;            // sub-package of the main class's package

@Component
public class ReportService {}
```

```java
@SpringBootApplication(scanBasePackages = "in.strikes")   // or widen the scan explicitly
public class SpringBootCoreDemo2Application { /* ... */ }
```

#### WRONG: using `getBean` as the application logic in Boot

```java
public static void main(String[] args) {
    ApplicationContext ctx = SpringApplication.run(SpringBootCoreDemo2Application.class, args);
    ctx.getBean(OrderService.class).placeOrder();     // Spring Core habit
}
```

#### Parent and starter (no versions)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.5</version>
    <relativePath/>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

#### A library's own auto-configuration

```java
// PaymentGatewayAutoConfiguration.java (inside the library JAR)
@AutoConfiguration
@ConditionalOnClass(PaymentGatewayClient.class)
public class PaymentGatewayAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public PaymentGatewayClient paymentGatewayClient() {
        return new PaymentGatewayClient();
    }
}
```

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.pg.PaymentGatewayAutoConfiguration
```

#### User override (auto-config backs off)

```java
@SpringBootApplication
public class App {
    @Bean
    public PaymentGatewayClient paymentGatewayClient() {   // yours wins over the auto-configured default
        return new PaymentGatewayClient("sandbox");
    }
}
```

#### Batch job that must exit even if web classes are on the classpath

```properties
# application.properties
spring.main.web-application-type=none
```



#### Quick Self-Check

1. **Is Spring Boot only for web applications?**
   *Answer:* No. Boot automates configuration of the Spring ecosystem. It starts a web server only when web dependencies are present.

2. **What does `spring-boot-starter-parent` give you, and why do starters have no `<version>`?**
   *Answer:* A tested table of compatible versions (plus plugin defaults). A dependency without a version takes it from the parent.

3. **What three annotations does `@SpringBootApplication` combine?**
   *Answer:* `@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`.

4. **Which packages does Boot scan by default, and how do you scan elsewhere?**
   *Answer:* The package of the `@SpringBootApplication` class and its sub-packages. Use `scanBasePackages` (or move the classes).

5. **What does `SpringApplication.run` return?**
   *Answer:* The `ConfigurableApplicationContext`, the IoC container Boot created.

6. **What is auto-configuration, and who writes it?**
   *Answer:* `@Configuration` classes with `@Bean` methods shipped by Boot or by libraries and activated by `@EnableAutoConfiguration`, applied only when their conditions match.

7. **Explain `@ConditionalOnClass` and `@ConditionalOnMissingBean`.**
   *Answer:* The first applies the configuration only if a class is on the classpath (the dependency is present). The second creates the bean only if you have not defined one of that type, so your bean overrides Boot's default.

8. **Why did adding a web dependency make the app start Tomcat on 8080 with no code change?**
   *Answer:* The web classes became available, so conditions on Boot's web auto-configuration matched and it registered the embedded server beans.



#### Golden Rules / Checklist

- [ ] Start Boot projects from Spring Initializr (or use the Boot parent)
- [ ] Declare starters without versions; let the parent manage them
- [ ] Remember: starters bundle dependencies; the dependencies decide behaviour
- [ ] Put `@SpringBootApplication` on the main class in the **root** package of your app
- [ ] Keep every `@Component` inside that package tree (or set `scanBasePackages`)
- [ ] Do not write `new ...ApplicationContext` or `@ComponentScan` in a Boot app
- [ ] Do not use `getBean` as your program logic in Boot; use runners/endpoints
- [ ] Define your own bean to override a Boot default (auto-config backs off)
- [ ] Debug surprises with `--debug` (conditions report)
- [ ] For batch jobs, check that no web starter silently starts a server (or set `web-application-type=none`)
- [ ] Take exact starter names from Initializr; do not rely on older tutorials
- [ ] Treat vulnerability warnings seriously outside practice projects



#### Practice Questions

#### Basic

1. Re-create `OrderService` + `PaymentService` in Spring Core and then in Boot. List every line you no longer write.

2. Replace `@SpringBootApplication` with its three constituent annotations. Does the app behave the same?

3. Move `PaymentService` to a package outside the main class's package tree. What happens, and how do you fix it two ways?

4. Open the generated `pom.xml`. Which tag stops starters from needing a version?

#### Intermediate

5. Add a `@Bean` method to the main class and fetch it. Then move it to a separate `@Configuration` class. What stays the same?

6. Run the app with `--debug`. Find one positive and one negative match in the conditions report and explain why each happened.

7. Add the web starter and observe the log. Then add `spring.main.web-application-type=none` and observe again.

8. Define your own bean of a type Boot auto-configures (for example a JSON `ObjectMapper`) and show that Boot's default backs off.

#### Advanced / Interview-style

9. Explain end-to-end how Boot knows which auto-configuration classes to consider (hint: the `.imports` file) and how it decides which to apply.

10. Why does `@ConditionalOnMissingBean` only work reliably if auto-configuration is processed **after** your own bean definitions?

11. A Kubernetes Job never completes after someone adds a dependency. Walk through the cause and two fixes.

12. Compare configuring a Spring MVC app by hand (Spring Core style) with Boot. Which responsibilities moved from you to Boot, and which stayed with you?

## 10. `application.properties`, `@Value`, `@ConfigurationProperties` and Startup Runners



#### The Production Scenario

At 9 pm the payment gateway your service uses (say Paytm) starts timing out during a sale. The team decides to switch traffic to Razorpay and raise the retry count from 3 to 5.

- If the gateway name and retry count are **hard-coded** in Java, that means a code change, a new build, a new image and a deployment during an incident.
- If they are **externalised**, an engineer changes one value (a Kubernetes `ConfigMap` / environment variable) and restarts the pods. **No rebuild.**

That is the whole point of `application.properties` and its relatives. It is also why DevOps people care about how Spring Boot resolves configuration (sources and precedence, section 3).



### 1. `application.properties`

- Lives in `src/main/resources/` (so it is on the classpath, Lecture 03).
- It is **not Java code**: plain text, one `key=value` per line (like a `Map<String,String>`).
- Initializr generates it with a single entry: `spring.application.name=demo`.
- Spring Boot **loads it automatically at startup**; you write no code to read the file.

```properties
payment-property.type=Paytm
payment-property.retry-count=5
payment-property.enabled=true
payment-property.timeout=3000
```

> **Precision note:** The lecture says the file reloads "when the code compiles". More precisely: the file is **read at application startup**. When packaged inside the JAR, changing it means rebuilding. To change values *without rebuilding*, supply them from **outside** the JAR (environment variables, command-line arguments, an external config file or a ConfigMap). The lecture lists these other sources; section 3 shows how they rank.

> **Added:** `.properties` and `.yml`/`.yaml` carry the same data. Initializr offers both; the Boot default for these notes is `.properties`.



### 2. Injecting Values

#### 2.1 The problem with hard-coding

```java
paymentGateway.setType("Paytm");        // change of gateway = code change + rebuild
paymentGateway.setRetryCount(5);
```

#### 2.2 `@Value("${key}")`

```java
@Component
public class PaymentGateway {

    @Value("${payment-property.type}")
    private String type;

    @Value("${payment-property.retry-count}")
    private int retryCount;
}
```

| Detail | Behaviour |
|---|---|
| Placeholder syntax | `${property.key}` with **curly braces** |
| Where it can go | A field, a constructor parameter, a setter parameter (like `@Autowired`) |
| Type conversion | Done for you (`"5"` becomes `int 5`) |
| The class must be a bean | `@Component` etc.; the container performs the injection (it is DI for values) |
| Missing key, no default | **Startup fails** (*Could not resolve placeholder*, wrapped in `BeanCreationException`) |
| Default value | `${key:default}`: used only if the key is absent; **if the key exists, the property wins** |

```java
@Value("${payment-property.type:Razorpay}")   // Razorpay only if the key is missing
private String type;

@Value("${payment-property.retry-count:3}")
private int retryCount;
```

> **Gotcha (seen in the lecture):** Writing `@Value("$(payment-property.type)")` with **parentheses** is not a placeholder, so Spring tries to inject the literal text and fails (type mismatch / `NumberFormatException` for the int, then `UnsatisfiedDependencyException`). Braces `{}`, not parentheses.

> **Added:** `@Value` can read from any property source Boot knows (YAML, environment variables, command-line arguments), not only `application.properties`.

#### 2.3 Why `@Value` stops scaling

With many settings you repeat `@Value` per field, key names are strings that are easy to mistype, and a wrong mapping fails silently or late. Use a single typed object instead.

#### 2.4 `@ConfigurationProperties(prefix = "...")`

One class holds a whole group of settings, bound by **prefix**:

```java
@Component
@ConfigurationProperties(prefix = "payment-property")
public class PaymentProperties {

    private String type;
    private int retryCount;
    private boolean enabled = true;
    private int timeout;

    // getters and setters for all four fields (binding uses the setters)
}
```

```properties
payment-property.type=Paytm
payment-property.retry-count=5
payment-property.enabled=true
payment-property.timeout=3000
```

Rules the lecture hit:

| Rule | Explanation |
|---|---|
| **Prefix must be in canonical form** | lowercase, words separated by `-` (kebab-case): `payment-property`, not `paymentProperty` (error: *Prefix must be in canonical form*) |
| **Relaxed binding** | `retry-count` in the file binds to the Java field `retryCount` automatically. Write camelCase in Java, kebab-case in the file |
| Boolean field | Use `enabled` (getter `isEnabled()`), not a field named `isEnabled`; then the property is `payment-property.enabled` |
| Setters needed | JavaBean-style binding calls setters (or use constructor binding / a record, see below) |
| Class is a bean | `@Component` makes it injectable like any other bean |

#### 2.5 The standard pattern: properties class + business class

The business bean does not read the file; it **depends on the properties bean** (constructor injection):

```java
@Component
public class PaymentGateway {

    private final PaymentProperties paymentProperties;

    public PaymentGateway(PaymentProperties paymentProperties) {
        this.paymentProperties = paymentProperties;
    }

    public void print() {
        System.out.println(paymentProperties.getType() + " "
                + paymentProperties.getRetryCount() + " "
                + paymentProperties.isEnabled() + " "
                + paymentProperties.getTimeout());
    }
}
```

Output: `Paytm 5 true 3000`.

```
application.properties ──(Boot binds by prefix)──▶ PaymentProperties (bean)
                                                        │ constructor injection
                                                        ▼
                                                  PaymentGateway (bean)
```

#### 2.6 `@Value` vs `@ConfigurationProperties`

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Best for | One or two values | A group of related settings |
| Key naming | Exact key string you write | Relaxed binding (`retry-count` ↔ `retryCount`) |
| Type safety / one place | No: scattered strings | Yes: one typed class |
| Validation | Manual | Add `@Validated` + `@NotNull`, `@Min`, ... (added) |
| Default | `${key:default}` | Field initialisers (`= true`) |

> **Added:** Modern Boot also lets you bind to an immutable **record** / constructor-bound class, and use types like `Duration` (`timeout=3s`) instead of "milliseconds as int". If you use `@ConfigurationProperties` on a class *without* `@Component`, register it with `@EnableConfigurationProperties` or `@ConfigurationPropertiesScan`.



### 3. Other Configuration Sources and Precedence

The lecture lists: `application.properties`, `application.yml`, **environment variables**, **command-line arguments**, **system properties**. They all feed the same mechanism (Spring's `Environment`), so `@Value` and `@ConfigurationProperties` read them identically.

Higher wins when the same key appears twice (simplified; highest first):

| Priority | Source |
|---|---|
| 1 | **Command-line arguments**: `--payment-property.type=Razorpay` |
| 2 | Java system properties: `-Dpayment-property.type=Razorpay` |
| 3 | **OS environment variables** |
| 4 | Profile files outside / inside the JAR: `application-prod.properties` |
| 5 | `application.properties` / `application.yml` (outside, then inside the JAR) |
| 6 | Defaults in code (`${key:default}`, field initialisers) |

> **Added (DevOps):** Environment variable names are derived from the property: uppercase, dots become `_`, dashes dropped. For example `payment-property.retry-count` becomes `PAYMENTPROPERTY_RETRYCOUNT`. This is how a Kubernetes `ConfigMap`/`Secret` or a `docker run -e` changes a value **without rebuilding the image**. Keep passwords and API keys out of `application.properties` in Git; inject them through env vars / secrets.



### 4. Starting Code Without `getBean`: Runners

#### 4.1 Why not `context.getBean(...)` in `main`

```java
ConfigurableApplicationContext context = SpringApplication.run(DemoApplication.class, args);
context.getBean(PaymentGateway.class).print();     // works, but not the Boot way
```

It couples `main` to the container. In a web app, entry points are endpoints (`/print` mapped to a method); in a console app the Boot equivalent is a **runner**.

#### 4.2 `ApplicationRunner` and `CommandLineRunner`

Both are Spring Boot interfaces (`org.springframework.boot`) with a single `run` method; Boot calls every runner **bean** once, after the context is fully started.

```java
@Component
public class DemoRunner implements ApplicationRunner {

    private final PaymentGateway paymentGateway;

    public DemoRunner(PaymentGateway paymentGateway) {   // normal constructor injection
        this.paymentGateway = paymentGateway;
    }

    @Override
    public void run(ApplicationArguments args) {
        paymentGateway.print();
    }
}
```

| | `ApplicationRunner` | `CommandLineRunner` |
|---|---|---|
| Method | `run(ApplicationArguments args)` | `run(String... args)` |
| Arguments | Parsed: options (`--name=value`) vs plain arguments | Raw strings |
| Behaviour | Same: runs once at startup | Same |

`main` shrinks to:

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);   // nothing else needed
    }
}
```

> **Precision note:** The lecture calls the runner interface "a special interface in Java"; it is a **Spring Boot** interface. The runner class must itself be a bean (`@Component`), otherwise Boot never sees it.

> **Gotchas (added):** (1) Several runners are allowed; control order with `@Order`. (2) If a runner throws, **startup fails** and the process exits with an error, which matters for Kubernetes restarts. (3) In a web app, runners execute during startup, so a slow runner delays the point at which the app is ready.

#### 4.3 Running from the terminal

```
mvn spring-boot:run
```

Same output (`Paytm 5 true 3000`). Command-line arguments become application arguments and, in the `--key=value` form, also override properties (priority 1 above), for example `java -jar app.jar --payment-property.type=Razorpay`.



### Code

#### `application.properties`

```properties
spring.application.name=demo
payment-property.type=Paytm
payment-property.retry-count=5
payment-property.enabled=true
payment-property.timeout=3000
```

#### Same configuration as `application.yml` (added)

```yaml
payment-property:
  type: Paytm
  retry-count: 5
  enabled: true
  timeout: 3000
```

#### WRONG vs RIGHT: `@Value`

```java
// WRONG: parentheses are not a placeholder
@Value("$(payment-property.type)")
private String type;

// WRONG: key missing and no default -> application fails to start
@Value("${payment-property.currency}")
private String currency;

// RIGHT
@Value("${payment-property.type}")
private String type;

// RIGHT: key may be absent
@Value("${payment-property.currency:INR}")
private String currency;
```

#### WRONG vs RIGHT: `@ConfigurationProperties` prefix

```java
@ConfigurationProperties(prefix = "paymentProperty")     // error: prefix must be in canonical form
```

```java
@ConfigurationProperties(prefix = "payment-property")    // kebab-case, lowercase
```

#### WRONG vs RIGHT: startup logic

```java
// WRONG: pulling beans out of the container in main
public static void main(String[] args) {
    ConfigurableApplicationContext ctx = SpringApplication.run(DemoApplication.class, args);
    ctx.getBean(PaymentGateway.class).print();
}
```

```java
// RIGHT: a runner bean
@Component
class DemoRunner implements CommandLineRunner {
    private final PaymentGateway gateway;
    DemoRunner(PaymentGateway gateway) { this.gateway = gateway; }

    @Override
    public void run(String... args) { gateway.print(); }
}
```

#### Overriding at deploy time (DevOps view)

```bash
# command-line argument
java -jar demo.jar --payment-property.type=Razorpay --payment-property.retry-count=3

# environment variable (no rebuild, no code change)
PAYMENTPROPERTY_TYPE=Razorpay java -jar demo.jar
```



#### Quick Self-Check

1. **Why use `application.properties` instead of hard-coding values?**
   *Answer:* Configuration can change without editing/recompiling code, and can differ per environment.

2. **Syntax and behaviour of `@Value`? What if the key is missing?**
   *Answer:* `@Value("${key}")`; missing key with no default fails startup; `${key:default}` supplies a fallback only when the key is absent.

3. **When do you prefer `@ConfigurationProperties` over `@Value`?**
   *Answer:* For a group of related settings: one typed class, relaxed binding, validation, no repeated annotations.

4. **How does `retry-count` in the file map to `retryCount` in Java? What form must the prefix have?**
   *Answer:* Relaxed binding. The prefix must be canonical: lowercase kebab-case.

5. **Why inject the properties class into `PaymentGateway` instead of reading the file there?**
   *Answer:* It keeps business code free of config plumbing; the properties object is just another bean, easy to test and reuse.

6. **Which source wins: command-line argument, environment variable or `application.properties`?**
   *Answer:* Command-line argument, then environment variable, then the properties file.

7. **How do you run code at startup without touching the context? Difference between the two interfaces?**
   *Answer:* A bean implementing `ApplicationRunner` or `CommandLineRunner`. They differ only in the argument type (parsed `ApplicationArguments` vs raw `String...`).

8. **If you change `application.properties` inside the JAR, does a running app see it?**
   *Answer:* No. It is read at startup; restart (and rebuild if it is packaged in the JAR), or use external sources.



#### Golden Rules / Checklist

- [ ] Keep environment-specific values out of Java; put them in config
- [ ] Use `${...}` (curly braces) in `@Value`
- [ ] Give optional values a default (`${key:default}`)
- [ ] Use `@ConfigurationProperties` for groups; prefix in kebab-case
- [ ] Write camelCase in Java, kebab-case in the file; rely on relaxed binding
- [ ] Inject the properties bean into business beans; do not read files directly
- [ ] Never commit secrets to `application.properties`; use env vars / secret stores
- [ ] Know the precedence: command line > system properties > env vars > files > defaults
- [ ] Use a runner bean for startup logic; keep `main` to `SpringApplication.run`
- [ ] Remember runners that throw abort startup
- [ ] Restart the app after changing packaged config



### Practice Questions

#### Basic

1. Add `payment-property.type` and `payment-property.retry-count` to `application.properties` and inject them with `@Value` into a `PaymentGateway` bean.

2. Remove one key and observe the failure. Then add a default with `${key:default}`.

3. Reproduce the `$(...)` mistake and read the error.

4. Replace `main`'s `getBean` call with an `ApplicationRunner`.

#### Intermediate

5. Convert the four `@Value` fields into a `PaymentProperties` class with `@ConfigurationProperties`. What happens with `prefix = "paymentProperty"`?

6. Rename a Java boolean field `isEnabled` to `enabled` and explain why.

7. Override `payment-property.type` through a command-line argument, then through an environment variable. Which wins when both are set?

8. Write the same configuration in `application.yml`.

#### Advanced / Interview-style

9. Compare `@Value` and `@ConfigurationProperties` on binding, type safety and validation.

10. Explain, with Kubernetes in mind, how you would change the retry count in production without rebuilding the image.

11. Two runners exist and one throws. What happens to startup, and how do you order them?

12. Why is calling `context.getBean(...)` in `main` discouraged in Spring Boot, and what replaces it in console and web apps?

## 11. First CRUD Project (Part 1): Architecture, Entity and the Create Endpoint



#### The Production Scenario

A Razorpay-style team builds a **merchant onboarding API**: `POST /api/merchants` creates a merchant, `GET /api/merchants/{id}` reads one, and so on. Two situations come up in the first week:

- A new joiner puts SQL calls inside the REST controller "to keep it simple". Two months later nobody can add validation (for example "GST number must match a pattern") without touching HTTP code, and the same logic gets copied into a batch job.
- The service starts in Kubernetes but crashes with *"Failed to configure a DataSource"* because the database URL was never supplied. (This is exactly the error this lecture hits, and it ties back to auto-configuration from Lecture 09.)

The layered structure below is what prevents the first problem; understanding auto-configuration explains the second.

---

### 1. What We Are Building

A **CRUD** application for `Student` records.

| Operation | HTTP method | Endpoint |
|---|---|---|
| **C**reate | `POST` | `/api/students` |
| **R**ead one | `GET` | `/api/students/{id}` |
| **R**ead all | `GET` | `/api/students` |
| **U**pdate | `PUT` | `/api/students/{id}` (full body) |
| **D**elete | `DELETE` | `/api/students/{id}` |
| Soft delete (demo at the start) | `PATCH` | `/api/students/{id}/soft-delete` |

```
 Postman  ──HTTP request──▶  Spring Boot app  ──SQL──▶  MySQL
 (client)  ◀─HTTP response──   (server)       ◀─rows──   (database)
```

> **Why CRUD matters:** almost every backend feature is create/read/update/delete on some data. Once this skeleton is clear, business logic is just extra code inside it.

#### 1.1 Tools used

| Tool | Role |
|---|---|
| **Postman** | Acts as the client; sends requests to `localhost:8080` |
| **MySQL server** | The relational database |
| **DBeaver** (or Workbench, etc.) | GUI to look at tables and rows |
| **Spring Initializr** | Generates the project |

---

### 2. HTTP Recap Needed for the Project

| HTTP method | Meaning | CRUD |
|---|---|---|
| `POST` | Create a new resource | Create |
| `GET` | Fetch data; no body | Read |
| `PUT` | **Replace** an existing resource (send the full object) | Update |
| `PATCH` | Change **part** of a resource (a field or state) | Partial update / soft delete |
| `DELETE` | Remove a resource | Delete |

- **Host** = `localhost:8080` (Tomcat's default port). Only the host changes on deployment (a domain name replaces `localhost`).
- **Endpoint** = the URL path (`/api/students`) **together with the HTTP method**; that pair maps to one Java method.
- A request can carry **headers** (`Content-Type: application/json` says the body is JSON; `Accept: application/json` says JSON is wanted back) and a **body** (JSON).

> **Precision note:** The lecture describes an endpoint as "a complete URL mapped to a method". Strictly, Spring maps **HTTP method + path** (plus optional conditions) to a handler method. That is why `GET /api/students` and `POST /api/students` can coexist. Two handlers with the same method and path fail at startup (*Ambiguous mapping*).

> **Precision note:** `PUT` replaces the whole resource, so the update endpoint expects the full object (the lecturer sends the full body). Use `PATCH` for partial changes.

> **Precision note:** The lecture says "2001" for the Created status code; it is **201**. Valid HTTP status codes are three-digit codes in the 100 to 599 range.

---

### 3. Project Setup

Spring Initializr choices: Maven, Java 21, Boot 4.x, packaging JAR, configuration **Properties**. Dependencies:

| Dependency | Why |
|---|---|
| Spring Web (MVC) | REST controllers + embedded Tomcat |
| Spring Data JPA | Talk to the DB through JPA/Hibernate without writing SQL |
| MySQL Driver | JDBC driver that speaks to MySQL |

> **Added:** Boot 4 starter names differ from older tutorials (for example the web starter is `spring-boot-starter-webmvc`). Take them from the generated `pom.xml`.

#### 3.1 First run fails: *"Failed to configure a DataSource"*

Adding Data JPA + the MySQL driver makes Boot's **auto-configuration** (Lecture 09) try to create a `DataSource`, but no URL, username or password was supplied.

```
Failed to configure a DataSource: 'url' attribute is not specified
and no embedded datasource could be configured.
```

| | Fix |
|---|---|
| **Temporary** (used in the lecture) | Exclude the auto-configuration: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`. Tomcat then starts on 8080. |
| **Real fix** (next lecture) | Provide the connection settings in `application.properties` (`spring.datasource.url`, `username`, `password`) and remove the exclusion. |

> **Gotcha (added):** The exclusion is only a stopgap. With the `DataSource` excluded, Boot's JPA auto-configuration also backs off (it needs a `DataSource`), so nothing will persist until you remove it. In real deployments the URL/credentials come from environment variables (`SPRING_DATASOURCE_URL`, ...), not from code.

A browser visit to `localhost:8080` shows a "Whitelabel Error Page" (404): the server works, but no endpoint is mapped yet (Lecture 02).

---

### 4. The Layered Architecture

Rule: **one class, one responsibility** (the SRP from Lecture 04). Put everything in one class and you cannot reuse or test any part.

```
Postman
   │ HTTP request (JSON body)
   ▼
┌──────────────────┐   listens to HTTP, converts JSON <-> Java object
│ StudentController│
└────────┬─────────┘
         ▼
┌──────────────────┐   business logic and validation
│  StudentService  │
└────────┬─────────┘
         ▼
┌──────────────────┐   talks to the database
│StudentRepository │
└────────┬─────────┘
         ▼
      MySQL

 Student (Entity) = the data object carried between the layers
 Response travels back: Repository → Service → Controller → Postman
```

| Layer | Responsibility | Should NOT do |
|---|---|---|
| **Controller** | Receive the HTTP request, map URL + method to a method, convert body to object, return the HTTP response (status + body) | Business rules, SQL |
| **Service** | Business logic, validation (for example "subject must be a CS subject", "roll number in a range") | HTTP details, SQL |
| **Repository** | Persist/fetch data | Business rules |
| **Entity** | Describes the data and its table mapping | Logic that belongs in services |

- Dependencies point **one way**: Controller → Service → Repository, wired by **constructor injection** (Lectures 04 and 05).
- The service looks thin in this demo (it just forwards), but it is where validation and rules will live; the layer exists so you do not restructure later.

> **Added:** Later lectures add **DTOs** (separate request/response objects), so API shapes can differ from entities. Returning entities directly is fine for learning but exposes your table structure in production APIs.

#### Packages

```
in.strikes.crudspringbootdemo/
├── CrudSpringBootDemoApplication.java     <- main class (scan root, Lecture 09)
├── controller/StudentController.java
├── service/StudentService.java
├── repository/StudentRepository.java
└── entity/Student.java                    <- sometimes called "model"
```

> **Gotcha:** All packages must be **under the main class's package**, or component scanning will not find them (Lecture 09).

---

### 5. The Entity: Mapping a Class to a Table

The table (created manually in the lecture, to show what the ORM will automate):

```sql
CREATE TABLE student (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255),
  age INT,
  email VARCHAR(255),
  roll_number INT,
  subject VARCHAR(255)
);
```

The Java side is a POJO with fields, getters and setters, annotated with **`@Entity`** and **`@Id`**:

```
 JSON  ──(controller, Jackson)──▶  Student object  ──(JPA/Hibernate)──▶  table row
 one Student object  ==  one row
 field names/types    ==  column names/types
```

#### What "ORM" means

**Object-Relational Mapping**: mapping Java **objects** to rows in **relational** tables. Instead of hand-writing `INSERT INTO student ... VALUES (...)`, the framework builds the SQL from the object. (The lecture says "Object Relation Model"; the term is *Mapping*.)

#### Rules

| Rule | Why |
|---|---|
| `@Entity` on the class | Tells JPA this class maps to a table |
| An **`@Id`** field is mandatory | The DB needs a primary key. Without it: *"Persistent entity should have a primary key"* (IDE) or an error at startup. |
| Prefer `Long` for ids | Row counts can exceed `Integer`'s range |
| Choose a **surrogate id** (separate `id`) | Name/age/subject are not unique; email or roll number could be, but a dedicated id is safest |
| Annotation package is `jakarta.persistence` | Spring Boot 3+/4 (not `javax`, Lecture 01) |

> **Added:** For auto-increment ids in MySQL, annotate the id with `@GeneratedValue(strategy = GenerationType.IDENTITY)`; otherwise the application must supply ids.

> **Added:** Hibernate does **not** create tables automatically by default for MySQL. You must set `spring.jpa.hibernate.ddl-auto` (for example `update` for development). Never rely on it in production; use migration tools (Flyway/Liquibase).

#### The entity is not a bean

`@Entity` classes are **not** annotated `@Component`, so Spring's IoC container does **not** manage them; you create them with `new` (or Jackson creates them from JSON). They are managed by **JPA** (the persistence context), not by the IoC container. Do not try to `@Autowired` an entity.

---

### 6. Controller: Receiving the Request

#### 6.1 Stereotype annotations

| Annotation | Meaning |
|---|---|
| `@Component` | Generic bean |
| `@Service` | A `@Component` marking the business layer |
| `@Repository` | A `@Component` marking the data layer (adds exception translation) |
| `@RestController` | `@Controller` (itself a `@Component`) + `@ResponseBody`: methods return data (JSON), not view names |

They all end up as beans in the IoC container; the specialised ones make the **intent** clear and enable extra behaviour.

> **Added:** For now `StudentRepository` is a plain class annotated as a component. The next lectures replace it with a Spring Data `JpaRepository` **interface**, where Spring generates the implementation (so no SQL and no implementation class).

#### 6.2 Mapping URLs to methods

```java
@RestController
@RequestMapping("/api/students")        // common prefix for every method in this class
public class StudentController {

    private final StudentService studentService;

    public StudentController(StudentService studentService) {   // constructor injection, no @Autowired
        this.studentService = studentService;
    }

    @PostMapping                         // POST /api/students
    public ResponseEntity<Student> createStudent(@RequestBody Student studentRequest) {
        Student created = studentService.createStudent(studentRequest);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

| Annotation | Role |
|---|---|
| `@RequestMapping("/api/students")` on the class | Common path prefix |
| `@PostMapping`, `@GetMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping` | HTTP method (+ optional extra path) |
| `@RequestBody` | Read the **request body** (JSON) and convert it into the parameter object |

Two styles for the lecturer's choice:

| Style | Example | Note |
|---|---|---|
| Method in the path | `POST /api/students/create`, `GET /api/students/get/{id}` | Works, but puts verbs in URLs |
| **HTTP method + noun path** (lecturer's preference) | `POST /api/students`, `GET /api/students/{id}` | Standard REST style: the **method** is the verb |

#### 6.3 JSON ↔ object conversion (automatic)

- `@RequestBody Student s` makes Spring (using the **Jackson** library via the web starter's auto-configuration) convert JSON to a `Student`, matching property names (`name`, `email`, `age`, ...).
- Returning an object makes Spring convert it **back to JSON** for the response body.
- The controller therefore never parses JSON by hand; your Java code only deals with objects.
- Returning a plain `String` sends text, not JSON.

> **Gotcha (added):** If the client does not send `Content-Type: application/json`, Spring answers **415 Unsupported Media Type**. Postman sets it when you choose "raw JSON". Boot's default Jackson settings ignore unknown JSON fields.

#### 6.4 Status codes with `ResponseEntity`

| Return type | Status | When |
|---|---|---|
| `Student` (plain object) | Always **200 OK** | Quick and simple |
| `ResponseEntity<Student>` | **Your choice** | You control status, headers and body |

```java
return ResponseEntity.status(HttpStatus.CREATED).body(created);   // 201 Created
return ResponseEntity.ok(student);                                  // 200 OK
```

- `HttpStatus` is an enum (`OK`=200, `CREATED`=201, `NOT_FOUND`=404, `UNAUTHORIZED`=401, ...). Use the constants, not magic numbers.
- Creating a resource should return **201**, and ideally a `Location` header pointing to the new resource (added; the lecture returns only the body).

---

### 7. Service and Repository (Stub Version)

```java
@Service
public class StudentService {

    private final StudentRepository studentRepository;

    public StudentService(StudentRepository studentRepository) {
        this.studentRepository = studentRepository;
    }

    public Student createStudent(Student studentRequest) {
        // later: validation / business rules go here
        return studentRepository.saveStudent(studentRequest);
    }
}
```

```java
@Repository
public class StudentRepository {

    public Student saveStudent(Student student) {
        // later: real DB call. For now return a dummy Student
        Student s1 = new Student();
        s1.setName("Aditya");
        s1.setAge(28);
        return s1;
    }
}
```

> **Gotcha:** If the repository class has no stereotype annotation, injecting it fails with *"Could not autowire; no beans of type StudentRepository found"*: the IoC container does not know it exists (Lecture 05).

---

### 8. Verifying the Flow

Adding `System.out.println("Inside StudentController")` style prints in each layer (and on exit) shows the call order:

```
Inside StudentController
  Inside StudentService
    Inside StudentRepository
    Exiting StudentRepository
  Exiting StudentService
Exiting StudentController
```

Results observed in the demo:

| Step | Result |
|---|---|
| Postman `POST /api/students` with JSON body | `@RequestBody` fills a `Student`; console prints its name/email (JSON → object works) |
| Controller returns a `String` | Postman shows plain text (not JSON) |
| Controller returns the dummy `Student` | Postman shows JSON; `id` is `null` because the stub never set it |
| Controller returns `ResponseEntity` with `CREATED` | Postman shows **201 Created** |

---

### 9. What Is Not Done Yet (Next Video)

- Database connection settings in `application.properties`
- Real persistence via Spring Data JPA (`save`, `findById`, `findAll`, ...) instead of the stub
- Read, Update, Delete endpoints
- Soft delete vs hard delete

---

### Code

#### `entity/Student.java`

```java
package in.strikes.crudspringbootdemo.entity;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;

@Entity
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // DB auto-increment (added)
    private Long id;

    private String name;
    private Integer age;
    private String email;
    private Integer rollNumber;
    private String subject;

    // getters and setters for all fields
}
```

#### Request / response (Postman)

```
POST http://localhost:8080/api/students
Headers: Content-Type: application/json
Body:
{
  "name": "Aditya",
  "email": "aditya@gmail.com",
  "age": 28,
  "rollNumber": 1001,
  "subject": "Spring Framework"
}

Response: 201 Created
{ "id": null, "name": "Aditya", ... }      // dummy stub response for now
```

#### WRONG: everything in one class

```java
@RestController
@RequestMapping("/api/students")
public class StudentController {
    @PostMapping
    public Student create(@RequestBody Student s) {
        // HTTP + validation + SQL in one place
        // jdbcTemplate.update("INSERT INTO student ...");
        return s;
    }
}
```

#### WRONG: unmapped or ambiguous routes

```java
@GetMapping("/api/students")   // in one method
@GetMapping("/api/students")   // in another: "Ambiguous mapping" at startup
```

#### Temporary workaround for the missing DataSource (remove later)

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class CrudSpringBootDemoApplication { ... }
```

#### Real configuration (next lecture, shown for context)

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/student_crud_db
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=update      # dev only
spring.jpa.show-sql=true                  # prints the generated SQL
```

---

#### Quick Self-Check

1. **Which HTTP method maps to each CRUD operation, and which is used for a partial update?**
   *Answer:* POST=Create, GET=Read, PUT=Update (full replace), DELETE=Delete; PATCH for partial updates.

2. **What is an endpoint in Spring MVC terms?**
   *Answer:* A URL path plus HTTP method that maps to a handler method in a controller.

3. **What is each layer responsible for (Controller, Service, Repository, Entity)?**
   *Answer:* Controller: HTTP in/out. Service: business logic. Repository: database access. Entity: the data class mapped to a table.

4. **Why did the app fail with "Failed to configure a DataSource", and what are the temporary and proper fixes?**
   *Answer:* Auto-configuration saw Data JPA and the MySQL driver but no connection URL. Temporary: exclude `DataSourceAutoConfiguration`. Proper: set `spring.datasource.*` properties.

5. **What does `@RequestBody` do, and what library does it rely on?**
   *Answer:* Converts the JSON request body into the parameter object (Jackson, auto-configured by Boot).

6. **Why must an entity have an `@Id`?**
   *Answer:* A relational table needs a primary key to identify rows.

7. **Is `Student` an IoC bean? Who manages it?**
   *Answer:* No. It has no `@Component`; JPA manages it, not the IoC container.

8. **How do you return 201 instead of the default 200?**
   *Answer:* Return `ResponseEntity.status(HttpStatus.CREATED).body(...)`.

---

#### Golden Rules / Checklist

- [ ] One responsibility per class: controller (HTTP), service (logic), repository (data)
- [ ] Constructor-inject downwards: controller → service → repository
- [ ] Put all packages under the main class's package
- [ ] Use `@RestController` + `@RequestMapping` for the common prefix
- [ ] Use HTTP method + noun path (`POST /api/students`), not verbs in URLs
- [ ] Use `@RequestBody` for incoming JSON; send `Content-Type: application/json`
- [ ] Return `ResponseEntity` when the status code matters (201 for creation)
- [ ] Give every entity an `@Id`; use `Long` ids with a generation strategy
- [ ] Do not annotate entities as beans or autowire them
- [ ] Remove `DataSourceAutoConfiguration` exclusions once the DB properties exist
- [ ] Keep DB credentials out of source; read them from environment variables
- [ ] Do not use `ddl-auto=update` in production; use migrations

---

### Practice Questions

#### Basic

1. Create the project with Web, Data JPA and MySQL Driver. Reproduce the "Failed to configure a DataSource" error. What causes it?

2. Create `controller`, `service`, `repository`, `entity` packages and move your classes in.

3. Build the `Student` entity with `@Entity` and `@Id`. What error appears if you remove `@Id`?

4. Create `POST /api/students` that prints the received student's name and email.

#### Intermediate

5. Make the endpoint return `String`, then `Student`, then `ResponseEntity<Student>` with 201. Compare what Postman shows.

6. Remove the repository's stereotype annotation. Read the error and explain it using IoC concepts.

7. Remove `Content-Type: application/json` from the Postman request. What status do you get and why?

8. Map two methods to the same method + path and read the startup error.

#### Advanced / Interview-style

9. Explain why the service layer exists when the controller could call the repository directly. Give two examples of logic that belongs there.

10. Explain why an entity is not a Spring bean. Who instantiates it in the create flow?

11. How do Boot's auto-configuration and the missing-DataSource error relate (Lecture 09)? How would the app be configured in Kubernetes without touching code?

12. Compare `PUT` and `PATCH`, and decide which verb fits "soft delete".

## 12. CRUD Project (Part 2): Spring Data JPA, Database Configuration and Full CRUD



#### The Production Scenario

A Swiggy-style **restaurant-catalog service** exposes `GET /api/restaurants/{id}`. In staging everything works. In production two things go wrong in the first week:

- A lookup for a non-existent restaurant returns **200 with an empty body**, so the mobile app crashes while parsing it. The right answer is **404**.
- The service fails at startup in a new cluster: *"Communications link failure"*. The MySQL host was hard-coded as `localhost` in `application.properties`; in Kubernetes the database lives at another address.

This lecture builds exactly the pieces involved: repository methods, status-code mapping, and DB connection properties (which must come from the environment in real deployments, Lecture 10).

---

### 1. Where Spring Data JPA Sits

```
StudentController      (HTTP)
      │
StudentService         (business logic)
      │
StudentRepository      (Spring Data JPA interface, no code of yours)
      │ runtime-generated implementation
JPA  (Jakarta Persistence API, the specification)
      │
Hibernate              (the JPA implementation, builds SQL)
      │
JDBC + MySQL driver
      │
MySQL
```

- **JPA** = Jakarta Persistence API (formerly "Java Persistence API"; imports are now `jakarta.persistence.*`). It is a *specification*; **Hibernate** implements it.
- **Spring Data JPA** sits on top and removes the boilerplate: you declare an interface, it supplies the code.
- The lecture deliberately treats this as a black box; Hibernate and JDBC details come in later lectures.

---

### 2. The Repository: An Interface, Not a Class

### 2.1 Before

The stub class had hand-written methods such as `saveStudent(...)`. Doing it "for real" would mean writing `INSERT INTO student ... VALUES (?, ?, ...)` and mapping fields by hand.

### 2.2 Now

```java
public interface StudentRepository extends JpaRepository<Student, Long> {
}
```

| Type parameter | Meaning |
|---|---|
| `Student` | The entity this repository manages |
| `Long` | The type of its `@Id` field |

The interface body is **empty**, yet these methods already exist (inherited):

| Method | SQL it generates (roughly) | Returns |
|---|---|---|
| `save(entity)` | `INSERT` or `UPDATE` | the saved entity |
| `findById(id)` | `SELECT ... WHERE id = ?` | `Optional<Student>` |
| `findAll()` | `SELECT * FROM student` | `List<Student>` |
| `existsById(id)` | `SELECT ... WHERE id = ?` (existence) | `boolean` |
| `deleteById(id)` | `DELETE ... WHERE id = ?` | `void` |
| `saveAll`, `count`, `flush`, `saveAndFlush`, ... | | |

```
JpaRepository → ListCrudRepository / ListPagingAndSortingRepository → CrudRepository → Repository
```

`save` lives on a parent (`CrudRepository`); you inherit everything up the chain.

### 2.3 "But I never implemented the interface!"

Normally a class implementing an interface must define every method. Here nothing is implemented in your code because **Spring Data creates the implementation at runtime** (a generated proxy backed by Spring Data's `SimpleJpaRepository`), registers it as a bean, and injects it where you declare `StudentRepository`.

| Question | Answer |
|---|---|
| Why an interface? | You have no SQL to write; Spring supplies the behaviour |
| Do I need `@Repository` / `@Component`? | **No.** Spring Data detects repository interfaces under the main class's package and registers them (Boot auto-configuration, Lecture 09) |
| What if I do need custom SQL later? | Declare extra methods (derived names or `@Query`); covered in later lectures |

> **Added:** `@Repository` is a `@Component` specialization that also translates database exceptions. Use it on a hand-written repository class, not on a Spring Data interface.

### 2.4 `save()` does both insert and update

There is no separate `update` method.

| Situation | What `save` does |
|---|---|
| Entity is **new** (its `id` is `null`) | `INSERT` |
| Entity has an existing `id` | Loads by id (`SELECT`), then `UPDATE`s (or inserts if the row is absent) |

That explains the logs in the demo: a `SELECT ... WHERE id = ?` first, then `INSERT`/`UPDATE`.

> **Gotcha (added):** For MySQL auto-increment, put `@GeneratedValue(strategy = GenerationType.IDENTITY)` on the `id` (Lecture 11). Without it Hibernate expects you to assign ids yourself. Also, if a client sends an `id` in a **create** request, `save` would update an existing row; in the service, reset it: `student.setId(null)`.

---

### 3. Connecting to MySQL

#### 3.1 Create the database (only this is manual)

```sql
CREATE DATABASE student_crud_db;
```

MySQL listens on **port 3306** by default (like Tomcat on 8080, Lecture 02). A GUI (DBeaver) connects via host `localhost`, port `3306`, user, password, and lets you run the statement above. The **table is not created by hand**; Hibernate will create it.

#### 3.2 `application.properties`

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/student_crud_db
spring.datasource.username=root
spring.datasource.password=<your-password>

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

| Property | Role |
|---|---|
| `spring.datasource.url` | JDBC URL: `jdbc:mysql://<host>:<port>/<database>` |
| `spring.datasource.username` / `password` | Credentials |
| `spring.jpa.hibernate.ddl-auto` | What Hibernate does with the schema |
| `spring.jpa.show-sql` | Print every SQL statement in the log |
| `spring.jpa.properties.hibernate.format_sql` | Pretty-print the SQL |

> **Correction:** The lecture's spoken key is garbled; the real key is `spring.datasource.url` (not `...datasource.datasource...`).

`ddl-auto` values (lecture defers details):

| Value | Effect |
|---|---|
| `none` | Do nothing (typical default for external DBs) |
| `validate` | Check that the schema matches the entities; fail if not |
| `update` | Create missing tables/columns; never drops anything |
| `create` / `create-drop` | Recreate the schema (data loss) |

`update` creates `student` with columns derived from the entity: `id` (primary key), `name` (`VARCHAR`), `age` (`INT`), `email`, `roll_number`, `subject`. It only creates the table if it is missing, which is why, after the first run, no `CREATE TABLE` appears in the log unless you drop the table.

#### 3.3 Remove the earlier workaround

Lecture 11 added `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` to get past "Failed to configure a DataSource". Now that the URL/username/password exist, **delete the exclusion** so Boot's DataSource and JPA auto-configuration (Lecture 09) run:

```java
@SpringBootApplication
public class CrudSpringBootDemoApplication { /* ... */ }
```

#### 3.4 DevOps notes

- Use **environment variables** for host and credentials (`SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, or `${DB_URL}` placeholders). Never commit real passwords; do not reuse the simple local-dev password anywhere else.
- `ddl-auto=update` is a **development** convenience. In production use migration tools (Flyway/Liquibase).
- `show-sql=true` is for debugging; turn it off in production (use proper logging).
- The app **fails to start** if the DB is unreachable. In Kubernetes/Docker Compose, make sure the database is ready first (init containers, `depends_on` with health checks).
- Boot configures a connection pool (HikariCP) behind the DataSource automatically.

---

### 4. The Five Operations

Status-code policy used by the controller:

| Outcome | Status |
|---|---|
| Created | `201 Created` |
| Found / updated / listed | `200 OK` |
| Not found | `404 Not Found` |

#### 4.1 Create: `POST /api/students`

```
Postman ─POST body─▶ Controller.createStudent ─▶ Service ─▶ repository.save(student)
                                                              (INSERT)
        ◀─201 + saved Student (JSON, with generated id)────────────────────
```

Observed: the log shows the SQL (`select ... where id=?` then `insert into student (age, email, name, roll_number, subject) values (?,?,?,?,?)`; the `?` are prepared-statement placeholders). The table row appears in DBeaver. The response body is the **saved entity**, now including its database id.

#### 4.2 Read one: `GET /api/students/{id}`

```java
@GetMapping("/{id}")
public ResponseEntity<Student> getStudent(@PathVariable Long id) { ... }
```

**Path variable vs query parameter**

| | Path variable | Query parameter |
|---|---|---|
| URL | `/api/students/1` | `/api/students?id=1&name=Aditya` |
| Annotation | `@PathVariable` | `@RequestParam` |
| Use for | Identifying one resource | Filtering, sorting, paging |
| Postman | Part of the URL | "Params" tab |

- In the mapping, the variable **must be in curly braces**: `"/{id}"`. Writing `"/get/id"` (or forgetting the braces) makes the URL match only the literal text, so `/get/1` returns **404** (the lecture's mistake).
- `@PathVariable Long id` binds by name; `{id}` and the parameter name must match (or give it explicitly: `@PathVariable("id")`).

**`findById` returns `Optional<Student>`** because the row may not exist.

| `Optional` method | Use |
|---|---|
| `isPresent()` / `isEmpty()` | Check |
| `get()` | Unwrap (throws `NoSuchElementException` if empty; avoid without a check) |
| `orElse(x)` / `orElseThrow()` | Safe unwrap |
| `map(...)` | Transform if present |

Lecture style (service returns `null` when absent, controller checks):

```java
// service
Optional<Student> found = studentRepository.findById(id);
if (found.isPresent()) return found.get();
return null;

// controller
Student s = studentService.getStudent(id);
if (s == null) return ResponseEntity.notFound().build();   // 404
return ResponseEntity.ok(s);
```

Bug demonstrated: without the null check the API returned **200 with an empty body** for a missing id. The fix is the 404 branch. `ResponseEntity.notFound().build()` is the shorthand for status 404 with no body.

The lecturer notes a cleaner approach: pass the `Optional` up and let the controller decide (see Code).

#### 4.3 Read all: `GET /api/students`

```java
List<Student> students = studentRepository.findAll();
```

Same path as create, different HTTP method: Spring maps method + path, so this is fine.

> **Precision note:** The lecture returns **404 when the list is empty**. The usual REST convention is **200 with an empty array `[]`**: the *collection* exists, it just has no items. Reserve 404 for a *specific* id that doesn't exist.

#### 4.4 Update: `PUT /api/students/{id}` + JSON body

Flow:

1. Controller receives `@PathVariable Long id` and `@RequestBody Student request`.
2. Service loads the existing row: `findById(id)`; if absent, report "not found" (404).
3. Copy the fields from the request onto the **existing** entity (name, age, email, rollNumber, subject). **Do not copy the id**: the id must stay as it was.
4. `studentRepository.save(existing)`: since the entity has an id, Spring Data issues `SELECT` then `UPDATE`.
5. Return the updated entity (200).

Why the body carries all fields: `PUT` means "replace the resource", so send the full representation (Lecture 11). Partial updates are what `PATCH` is for.

#### 4.5 Delete: `DELETE /api/students/{id}`

```java
if (!studentRepository.existsById(id)) return false;   // -> controller returns 404
studentRepository.deleteById(id);                      // returns void
return true;                                           // -> controller returns 200/204
```

> **Precision note (added):** In current Spring Data, `deleteById` on a **missing** id does nothing and does not fail. That is why the `existsById` check is needed if you want a 404. For a successful delete, **204 No Content** (no body) is the more conventional status than 200 with a text message.

#### 4.6 Quick verification table (Postman)

| Request | Result |
|---|---|
| `POST /api/students` (Aditya) | 201 + JSON; row in DB |
| `POST /api/students` (Rohit) | 201 + JSON; second row |
| `GET /api/students/1` | 200, Aditya |
| `GET /api/students/3` (absent) | **404**, empty body |
| `GET /api/students` | 200, list with both |
| `PUT /api/students/2` (age 28, roll 105) | 200; DB row updated; `UPDATE` in log |
| `DELETE /api/students/3` (absent) | 404 |
| `DELETE /api/students/2` | success; row removed |

> **Gotcha (seen in the demo):** A typo in the URL (for example `/uptade/2`) or a mismatched mapping returns 404 *from Spring* (no handler), which looks identical to your own "not found" 404. Check the mapping first.

---

### 5. Looking Ahead: Soft Delete

Hard delete removes the row. **Soft delete** keeps it and sets a flag (`is_deleted = 1`).

| Why | Effect on the code |
|---|---|
| Recover accidentally deleted data; keep audit/analytics history | Every read/update must **ignore** flagged rows |
| Business rules (for example "active users") | The default repository methods (`findById`, `findAll`) no longer fit; custom queries are needed |

The next lecture adds the flag, a `PATCH .../soft-delete` endpoint, and the modified queries.

---

### Code

#### `repository/StudentRepository.java`

```java
public interface StudentRepository extends JpaRepository<Student, Long> {
}
```

#### `service/StudentService.java`

```java
@Service
public class StudentService {

    private final StudentRepository studentRepository;

    public StudentService(StudentRepository studentRepository) {
        this.studentRepository = studentRepository;
    }

    public Student createStudent(Student student) {
        student.setId(null);                          // never let a create act as an update
        return studentRepository.save(student);
    }

    public Optional<Student> getStudent(Long id) {
        return studentRepository.findById(id);
    }

    public List<Student> getAllStudents() {
        return studentRepository.findAll();
    }

    public Optional<Student> updateStudent(Long id, Student request) {
        return studentRepository.findById(id).map(existing -> {
            existing.setName(request.getName());
            existing.setAge(request.getAge());
            existing.setEmail(request.getEmail());
            existing.setRollNumber(request.getRollNumber());
            existing.setSubject(request.getSubject());
            return studentRepository.save(existing);   // id unchanged -> UPDATE
        });
    }

    public boolean deleteStudent(Long id) {
        if (!studentRepository.existsById(id)) {
            return false;
        }
        studentRepository.deleteById(id);
        return true;
    }
}
```

#### `controller/StudentController.java`

```java
@RestController
@RequestMapping("/api/students")
public class StudentController {

    private final StudentService studentService;

    public StudentController(StudentService studentService) {
        this.studentService = studentService;
    }

    @PostMapping
    public ResponseEntity<Student> createStudent(@RequestBody Student student) {
        return ResponseEntity.status(HttpStatus.CREATED).body(studentService.createStudent(student));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Student> getStudent(@PathVariable Long id) {
        return studentService.getStudent(id)
                .map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @GetMapping
    public ResponseEntity<List<Student>> getAllStudents() {
        return ResponseEntity.ok(studentService.getAllStudents());   // [] when empty
    }

    @PutMapping("/{id}")
    public ResponseEntity<Student> updateStudent(@PathVariable Long id,
                                                 @RequestBody Student request) {
        return studentService.updateStudent(id, request)
                .map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteStudent(@PathVariable Long id) {
        return studentService.deleteStudent(id)
                ? ResponseEntity.noContent().build()      // 204
                : ResponseEntity.notFound().build();      // 404
    }
}
```

#### WRONG: path variable without braces

```java
@GetMapping("/get/id")                       // matches only the literal URL /get/id
public ResponseEntity<Student> getStudent(@PathVariable Long id) { ... }
```

#### RIGHT

```java
@GetMapping("/{id}")
public ResponseEntity<Student> getStudent(@PathVariable Long id) { ... }
```

#### WRONG: assuming `deleteById` reports a missing row

```java
studentRepository.deleteById(id);      // missing id: silently does nothing
return ResponseEntity.ok().build();    // wrongly says "deleted"
```

#### WRONG: unwrapping an Optional blindly

```java
Student s = studentRepository.findById(id).get();   // NoSuchElementException -> 500
```

```java
Student s = studentRepository.findById(id).orElseThrow();   // or map(...).orElse(...)
```

#### Properties (with environment overrides)

```properties
spring.datasource.url=${DB_URL:jdbc:mysql://localhost:3306/student_crud_db}
spring.datasource.username=${DB_USER:root}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

---

#### Quick Self-Check

1. **What does `extends JpaRepository<Student, Long>` mean, and why is the interface body empty?**
   *Answer:* It manages `Student` entities with `Long` ids; Spring Data generates the implementation of the inherited methods at runtime.

2. **How does `save()` decide between insert and update?**
   *Answer:* By the id: a null/new id means insert; an existing id means load then update.

3. **Which properties connect the app to MySQL, and what does `ddl-auto=update` do?**
   *Answer:* `spring.datasource.url/username/password`; `update` creates missing tables/columns from the entities without dropping data.

4. **Why must the `DataSourceAutoConfiguration` exclusion be removed now?**
   *Answer:* It was a stopgap; with a configured datasource, Boot's DataSource/JPA auto-configuration must run.

5. **Path variable or query parameter for "get student 5"? Which annotation?**
   *Answer:* Path variable, `@PathVariable`; the placeholder must be `{id}`.

6. **Why does `findById` return `Optional`, and how do you map it to a 404?**
   *Answer:* The row may not exist. If empty, return `ResponseEntity.notFound().build()`.

7. **What status for an empty list from `GET /api/students`?**
   *Answer:* 200 with `[]`; 404 is for a specific missing id.

8. **Why check `existsById` before `deleteById`?**
   *Answer:* `deleteById` silently ignores a missing id, so you would otherwise report success for nothing.

---

#### Golden Rules / Checklist

- [ ] Repository = interface extending `JpaRepository<Entity, IdType>`; no SQL, no implementation
- [ ] Do not annotate Spring Data interfaces with `@Repository`
- [ ] Use `ddl-auto=update` only in development; migrations in production
- [ ] Remove temporary `DataSourceAutoConfiguration` exclusions
- [ ] Take DB URL/credentials from environment variables, not from Git
- [ ] Use `show-sql` for debugging only
- [ ] `{id}` in curly braces for path variables
- [ ] Handle `Optional` safely (`map`/`orElse`/`orElseThrow`), never a bare `get()`
- [ ] 201 for create, 200 for read/update, 404 for missing id, 204 for delete
- [ ] Empty collection = 200 + `[]`
- [ ] Never copy the id when updating; reset it on create
- [ ] Verify with Postman and read the SQL in the log

---

### Practice Questions

#### Basic

1. Create the database `student_crud_db` and the properties file. Start the app and confirm the `student` table appears.

2. Replace the stub repository with `JpaRepository<Student, Long>`. Which methods can you call without writing anything?

3. Implement `POST /api/students` and check the row and the SQL in the log.

4. Implement `GET /api/students/{id}` and test an existing and a non-existing id.

#### Intermediate

5. Implement `GET /api/students`. What status do you return for an empty table and why?

6. Implement `PUT /api/students/{id}`. Why must you not copy the id from the request?

7. Implement `DELETE /api/students/{id}` with `existsById`. What changes if you omit the check?

8. Remove the braces from `{id}` in the mapping. Predict and verify the result.

#### Advanced / Interview-style

9. Explain how a Spring Data repository works without an implementation class (proxy, auto-configuration, scanning).

10. Walk through the SQL `save()` issues for a new entity versus an existing one.

11. The service starts locally but fails in Kubernetes with a connection error. List the causes and fixes (host, credentials, readiness, environment overrides).

12. Compare returning `null` from the service with returning `Optional`. Which is cleaner and why?

## 13. Soft Delete vs Hard Delete, Derived Query Methods and Request Parameters



### The Production Scenario

A Zomato-style team lets restaurant owners "delete" a menu item. A week later an owner calls support: *"I deleted a dish by mistake, please bring it back."* The team had implemented a **hard delete**; the row is gone, and the only recovery is a database backup restore.

At the same time finance wants a report of *all* items ever created, including removed ones, and the audit team wants to know who removed what. All three requests are trivial if the delete was a **flag** instead of a `DELETE FROM`.

The price: every query in the codebase must now remember `AND deleted = false`. This lecture shows how to apply that rule consistently with Spring Data.

---

### 1. Hard Delete vs Soft Delete

| | Hard delete | Soft delete |
|---|---|---|
| What happens | `DELETE FROM student WHERE id = ?` | `UPDATE student SET deleted = true WHERE id = ?` |
| Row afterwards | Gone | Still in the table, marked deleted |
| Recovery | Only from backups | Flip the flag back |
| Audit / analytics | History lost | History kept |
| Cost | Frees space | Table grows; every query needs a filter |
| Typical use | Data that must really go (see note) | Default choice in business apps |

```
id | name   | roll | deleted
 1 | Aditya | 1001 | 0        <- active
 2 | Rohit  |  102 | 1        <- "deleted" for the application, still in the DB
```

Rules the lecture sets for a soft-deleted row (it must behave as if it does not exist):

| Operation | On a soft-deleted row |
|---|---|
| Read one | Not found (404) |
| Read all | Not included |
| Update | Not found (404) |
| Soft delete again | Not found |
| **Hard delete** | **Still allowed**: a deliberate, rare "really remove it" action (a design choice) |

> **Added (compliance):** Soft delete is not "deleted" in the legal sense. If a user has a right to erasure (for example under India's DPDP Act or GDPR), you still need a real hard-delete or anonymisation path. Decide retention rules per table.

> **Added (DevOps):** Soft-deleted rows keep growing the table and backups; plan archiving. Index columns used in the filter if the table is large, and add the column through a **migration** (default `0`), not via `ddl-auto=update` in production.

---

### 2. Adding the Flag to the Entity

```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private Integer age;
    private String email;
    private Integer rollNumber;
    private String subject;

    private boolean deleted;          // false by default

    // getters and setters (the getter for a boolean is isDeleted())
}
```

- Hibernate adds a column `deleted` (a `bit` in MySQL) when `ddl-auto=update` and the table is recreated.
- Spring Data property name = **Java field name** (`deleted`), not the getter name.
- The lecture recreates the table (drops it) so the new column appears; with `update`, Hibernate adds missing columns to an existing table but never alters existing ones.

#### Who controls the flag?

The client must **never** decide `deleted`. In the lecture a client can send `"deleted": true` in a **create** or **update** body; the service overwrites it with `false`, so the request cannot create a row that is born deleted. Only the soft-delete endpoint sets it to `true`.

> **Added (security):** This is the "mass assignment" problem: binding a JSON body straight onto an entity lets clients set fields they should not touch. The robust fix is separate request DTOs (a later topic); until then, copy only the fields you allow.

---

### 3. Derived Query Methods

The standard methods (`findById`, `findAll`, `existsById`) do not know about `deleted`. Instead of writing SQL, **declare a method whose name describes the query**; Spring Data parses the name at startup and implements it.

```java
public interface StudentRepository extends JpaRepository<Student, Long> {

    Optional<Student> findByIdAndDeletedFalse(Long id);   // WHERE id = ? AND deleted = false

    List<Student> findByDeletedFalse();                   // WHERE deleted = false
}
```

Generated SQL (visible with `show-sql`):

```sql
select ... from student s where s.id=? and s.deleted=0
select ... from student s where s.deleted=0
```

#### Method-name grammar

```
find  [anything]  By  Property  [Keyword]  [And|Or  Property  [Keyword]] ...
```

| Part | Examples |
|---|---|
| Subject | `find`, `findAll`, `read`, `get`, `count`, `exists`, `delete` |
| Separator | `By` (required before the conditions) |
| Property | An entity **field name** in camelCase: `Id`, `Name`, `RollNumber`, `Deleted` |
| Keyword | `True`/`False`, `Is`/`IsNot`, `IsNull`, `Containing`, `StartingWith`, `Between`, `GreaterThan`, `In`, ... |
| Combine | `And`, `Or` |

| Method | Meaning |
|---|---|
| `findByNameContaining(String s)` | `name LIKE %s%` |
| `findByRollNumberBetween(int a, int b)` | range |
| `existsByIdAndDeletedFalse(Long id)` | boolean existence among active rows |
| `findByDeletedIsFalse()` | same as `findByDeletedFalse()` |

#### The error in the lecture

`findAllAndDeletedIsFalse()` made the **application fail at startup** (*Error creating bean ... StudentRepository*, a property-reference error): Spring Data reads the name, finds no `By`, and tries to treat `AndDeleted...` as a property.

> **Precision note:** The lecture says a method "must start with `findBy`". More exactly, text *between* `find` and `By` is free (`findAllByDeletedIsFalse`, `findStudentsByDeletedFalse` are valid); what is mandatory is the **`By`** before the first property. `findAllAndDeletedIsFalse` fails because it has no `By`.

Because names are parsed at startup, a typo (wrong property name, missing `By`) **fails fast** instead of at runtime, which is good.

> **Added:** Other approaches: `@Query("select s from Student s where ...")` for complex queries, and Hibernate's `@SQLRestriction("deleted = false")` plus `@SQLDelete(sql = "update student set deleted = true where id = ?")` on the entity to make soft delete transparent for the built-in methods (later topic).

---

### 4. Rewriting the Service Layer

| Operation | Before (Lecture 12) | After (soft-delete aware) |
|---|---|---|
| Create | `save(student)` | Force `deleted = false` (and `id = null`), then `save` |
| Read one | `findById` | `findByIdAndDeletedFalse` |
| Read all | `findAll` | `findByDeletedFalse` |
| Update | `findById` → copy → `save` | `findByIdAndDeletedFalse` → copy → keep `deleted = false` → `save` |
| Hard delete | `existsById` + `deleteById` | **Unchanged**: works on any row |
| Soft delete | n/a | `findByIdAndDeletedFalse` → `setDeleted(true)` → `save` |

Soft delete flow:

```
PATCH /soft-delete  ──▶ Controller ──▶ Service.deleteStudentSoftly(id)
                                          │ findByIdAndDeletedFalse(id)
                                          │   empty ──▶ return false ──▶ 404
                                          │ present
                                          │   student.setDeleted(true)
                                          │   studentRepository.save(student)   (UPDATE)
                                          ▼ return true ──▶ 200 "Record deleted"
```

Log output for one soft delete: a `SELECT ... WHERE id=? AND deleted=0` followed by `UPDATE student SET ... deleted=? ...`.

---

### 5. Which HTTP Method for Soft Delete?

The lecture chooses **`PATCH`**: the operation changes **one field** of an existing resource, it does not remove it. `DELETE` is reserved for real removal (hard delete). Either can be defended; the important rule is to be consistent and document the behaviour.

| Endpoint (as in the lecture) | Purpose |
|---|---|
| `POST   /api/students/create` | Create |
| `GET    /api/students/get` | Read one |
| `GET    /api/students/get-all` | Read all (active only) |
| `PUT    /api/students/update` | Update |
| `DELETE /api/students/delete` | Hard delete |
| `PATCH  /api/students/delete-soft` | Soft delete |

Cleaner REST shape of the same API (verbs in HTTP methods, nouns in paths):

| Method + path | Purpose |
|---|---|
| `POST   /api/students` | Create |
| `GET    /api/students/{id}` | Read one |
| `GET    /api/students` | Read all |
| `PUT    /api/students/{id}` | Update |
| `DELETE /api/students/{id}` | Hard delete |
| `PATCH  /api/students/{id}/soft-delete` | Soft delete |

---

### 6. Generated Ids (Fixing a Data-Corruption Bug)

Problem shown in the lecture: the client sends `"id": 2` in a **create** request. `save()` sees an existing id and runs `SELECT` then `UPDATE`, so a new student "Rohan" silently **overwrote** Rohit. Create must never act as update.

Fix:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)   // DB assigns the id (AUTO_INCREMENT)
private Long id;
```

- The table must be created with `id bigint not null auto_increment` (`CREATE TABLE ... auto_increment`).
- `ddl-auto=update` does **not** change an existing column, so the lecture drops the table and restarts; Hibernate recreates it with `auto_increment`.
- Afterwards, new rows get ids 1, 2, 3, ... automatically, and the client does not send an id.

> **Added:** Also set `student.setId(null)` in the service's create method so a stray id in the body can never trigger an update (Lecture 12).

---

### 7. Path Variables vs Request Parameters

Both carry values from the client in the URL.

| | Path variable | Request (query) parameter |
|---|---|---|
| Looks like | `/api/students/1` | `/api/students?id=1&name=Aditya` |
| Annotation | `@PathVariable` | `@RequestParam` |
| Mapping | Placeholder in the path: `"/{id}"` | None in the path |
| Number of values | Awkward for many | Natural for many (`&` separated) |
| Required? | Always (part of the URL) | Required by default; optional with `required = false` or `defaultValue` |
| Typical use | Identify **one resource** | Filtering, searching, sorting, paging, options |

```java
// path variable
@GetMapping("/{id}")
public ResponseEntity<Student> get(@PathVariable Long id) { ... }

// request parameter
@GetMapping("/get")
public ResponseEntity<Student> get(@RequestParam Long id) { ... }   // /get?id=1
```

- The parameter **name** in the URL (`id`) and the Java parameter name must match (or use `@RequestParam("id")`).
- In Postman, query parameters live in the **Params** tab; they are appended after `?` and joined by `&`.
- After switching the lecture's endpoints to `@RequestParam`, the old URL `/get/1` returns **404** (no such mapping) and `/get?id=1` works.
- A missing required parameter gives **400 Bad Request**.

> **Precision note:** The lecture converts every "by id" endpoint to request parameters to demonstrate them. In REST design the usual rule is: **path variable to identify a resource** (`/students/1`), **query parameters to filter or modify a list** (`/students?subject=Spring&page=2`).

> **Added (DevOps/security):** Query strings end up in access logs, browser history and proxy logs. Never put secrets (tokens, passwords) in them.

---

### 8. Result Checks in the Demo

| Step | Result |
|---|---|
| Create Aditya (id 1), Rohit (id 2) with `"deleted": true` sent in the body | Both stored with `deleted = false` |
| `PATCH` soft delete of id 2 | 200 "Record deleted"; DB shows `deleted = 1` |
| `GET` id 2 | **404** (filtered out) |
| `GET` all | Only Aditya |
| Create Rahul (id 3), soft delete him, then **hard delete** id 3 | Row physically removed (hard delete ignores the flag) |
| Create with a client-supplied existing id (before `@GeneratedValue`) | Existing row overwritten (bug fixed in section 6) |

---

### Code

#### Repository

```java
public interface StudentRepository extends JpaRepository<Student, Long> {

    Optional<Student> findByIdAndDeletedFalse(Long id);

    List<Student> findByDeletedFalse();
}
```

#### Service

```java
@Service
public class StudentService {

    private final StudentRepository studentRepository;

    public StudentService(StudentRepository studentRepository) {
        this.studentRepository = studentRepository;
    }

    public Student createStudent(Student student) {
        student.setId(null);                 // never an update
        student.setDeleted(false);           // client cannot create a deleted row
        return studentRepository.save(student);
    }

    public Optional<Student> getStudent(Long id) {
        return studentRepository.findByIdAndDeletedFalse(id);
    }

    public List<Student> getAllStudents() {
        return studentRepository.findByDeletedFalse();
    }

    public Optional<Student> updateStudent(Long id, Student request) {
        return studentRepository.findByIdAndDeletedFalse(id).map(existing -> {
            existing.setName(request.getName());
            existing.setAge(request.getAge());
            existing.setEmail(request.getEmail());
            existing.setRollNumber(request.getRollNumber());
            existing.setSubject(request.getSubject());
            existing.setDeleted(false);      // cannot be changed through update
            return studentRepository.save(existing);
        });
    }

    // hard delete: any row, soft-deleted or not
    public boolean deleteStudent(Long id) {
        if (!studentRepository.existsById(id)) return false;
        studentRepository.deleteById(id);
        return true;
    }

    // soft delete: only active rows
    public boolean deleteStudentSoftly(Long id) {
        return studentRepository.findByIdAndDeletedFalse(id).map(student -> {
            student.setDeleted(true);
            studentRepository.save(student);
            return true;
        }).orElse(false);
    }
}
```

#### Controller (REST-style paths)

```java
@RestController
@RequestMapping("/api/students")
public class StudentController {

    private final StudentService studentService;

    public StudentController(StudentService studentService) { this.studentService = studentService; }

    @PostMapping
    public ResponseEntity<Student> create(@RequestBody Student student) {
        return ResponseEntity.status(HttpStatus.CREATED).body(studentService.createStudent(student));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Student> get(@PathVariable Long id) {
        return studentService.getStudent(id).map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @GetMapping
    public ResponseEntity<List<Student>> getAll() {
        return ResponseEntity.ok(studentService.getAllStudents());
    }

    @PutMapping("/{id}")
    public ResponseEntity<Student> update(@PathVariable Long id, @RequestBody Student request) {
        return studentService.updateStudent(id, request).map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")                        // hard delete
    public ResponseEntity<Void> hardDelete(@PathVariable Long id) {
        return studentService.deleteStudent(id)
                ? ResponseEntity.noContent().build()
                : ResponseEntity.notFound().build();
    }

    @PatchMapping("/{id}/soft-delete")             // soft delete
    public ResponseEntity<String> softDelete(@PathVariable Long id) {
        return studentService.deleteStudentSoftly(id)
                ? ResponseEntity.ok("Record deleted")
                : ResponseEntity.notFound().build();
    }
}
```

#### Request-parameter variant (as in the lecture)

```java
@GetMapping("/get")
public ResponseEntity<Student> get(@RequestParam Long id) { ... }          // GET /api/students/get?id=1

@PatchMapping("/delete-soft")
public ResponseEntity<String> softDelete(@RequestParam Long id) { ... }    // PATCH /api/students/delete-soft?id=2
```

#### WRONG vs RIGHT

```java
// WRONG: no "By": application fails to start
List<Student> findAllAndDeletedIsFalse();

// RIGHT
List<Student> findAllByDeletedIsFalse();     // or findByDeletedFalse()
```

```java
// WRONG: built-in finder ignores the flag, so soft-deleted rows leak
studentRepository.findById(id);
studentRepository.findAll();

// RIGHT
studentRepository.findByIdAndDeletedFalse(id);
studentRepository.findByDeletedFalse();
```

```java
// WRONG: trusting the client's flag (and id)
studentRepository.save(requestBody);

// RIGHT
requestBody.setId(null);
requestBody.setDeleted(false);
studentRepository.save(requestBody);
```

---

#### Quick Self-Check

1. **Difference between hard and soft delete? Why is soft delete the default in business apps?**
   *Answer:* Hard delete removes the row; soft delete sets a flag and keeps it. It allows recovery, audit and analytics.

2. **What must change in reads and updates after adding the `deleted` flag?**
   *Answer:* Every read/update must filter on `deleted = false`; the standard `findById`/`findAll` no longer suffice.

3. **How does Spring Data build `findByIdAndDeletedFalse`? Where is the SQL?**
   *Answer:* It parses the method name at startup (`By` + properties + keywords) and generates the query; you write no SQL.

4. **Why did `findAllAndDeletedIsFalse` crash the app, and what is the precise rule?**
   *Answer:* It lacks the mandatory `By`. Text between `find` and `By` is free (`findAllBy...`).

5. **Why does the service force `deleted = false` on create and update?**
   *Answer:* The client must not control server-managed fields; only the soft-delete operation sets it to true.

6. **Which HTTP verb fits soft delete, and why `PATCH`?**
   *Answer:* It changes one field of an existing resource; `DELETE` is kept for real removal.

7. **How did a create request overwrite an existing student, and what fixes it?**
   *Answer:* A client-supplied existing id made `save()` update that row. Use `@GeneratedValue(strategy = IDENTITY)` (auto-increment) and null the id in the service.

8. **Path variable vs request parameter: when each, and what annotation?**
   *Answer:* Path variable (`@PathVariable`) to identify one resource; request parameters (`@RequestParam`) for filters/options, several values via `?a=1&b=2`.

---

#### Golden Rules / Checklist

- [ ] Default to soft delete for business data; keep a deliberate hard-delete path
- [ ] Soft delete must hide rows from read, list and update
- [ ] Never let the client set `deleted` or the `id` on create
- [ ] Generate ids in the database (`IDENTITY`) and null the id before create
- [ ] Declare derived queries (`findBy...`) instead of writing SQL; include `By`
- [ ] Property names in method names are entity field names
- [ ] Read the SQL in the log to confirm the filter is applied
- [ ] Use `PATCH` for single-field changes; `DELETE` for real removal
- [ ] Use path variables for identity, request parameters for filters
- [ ] Never put secrets in query strings
- [ ] Add schema changes through migrations, not `ddl-auto=update`, in production
- [ ] Plan retention/archiving for soft-deleted rows

---

### Practice Questions

#### Basic

1. Add `deleted` to `Student`, recreate the table, and confirm the new column in the DB.

2. Declare `findByIdAndDeletedFalse` and `findByDeletedFalse`; use them in the service. Read the generated SQL.

3. Implement the soft-delete endpoint and verify that the row stays in the table with `deleted = 1`.

4. Try to read and update a soft-deleted student. What do you get?

#### Intermediate

5. Send `"deleted": true` in a create request. What is stored, and why?

6. Make your id generated and show how a client-supplied id used to overwrite a row.

7. Convert one endpoint from `@PathVariable` to `@RequestParam`. What happens to the old URL, and what if the parameter is missing?

8. Add `findByNameContainingAndDeletedFalse(String name)` and expose it as `GET /api/students?name=...`.

#### Advanced / Interview-style

9. Compare soft and hard delete: audit, performance, storage, compliance (erasure rights). When must you still hard-delete?

10. Explain how Spring Data turns a method name into a query and why mistakes fail at startup.

11. Design `@SQLRestriction`/`@SQLDelete` as an alternative to hand-written filtered finders. What are the trade-offs?

12. Add validation in the service (for example roll number within a range, subject from an allowed list) and decide which HTTP status each violation should return.

## 14. Java Servlets, Tomcat and the Road to Spring MVC



#### The Production Scenario

A bank's internal tools team runs **five legacy web apps as five WAR files on one shared Tomcat**. Three recurring problems:

- One app's memory leak or `System.exit`-style bug takes **all five** down, because they share one JVM.
- A servlet keeps a `HashMap` of recent transactions in an **instance field**; under load two requests corrupt it (a servlet is a *single shared instance* handling many threads).
- Releases mean "copy the new WAR into `webapps/`, restart Tomcat, hope nothing stays stale".

Modern Spring Boot apps ship **an embedded Tomcat inside each app** (one process per app, one container per service). Knowing how the old model works explains why.

---

### 1. Client–Server Recap

```
Client (Postman, browser, mobile) ──HTTP request──▶ Server ──HTTP response──▶ Client
```

An HTTP request carries: method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`), URL path, headers, optional body, query parameters or path variables. A server must read all of these to decide **which Java code to run**, then write a response (status, headers, body).

---

### 2. Why Plain Java Is Painful: `java.net`

Core Java has `ServerSocket` / `Socket` in `java.net`: you can **listen on a port** (say 8080) and read/write bytes. That is all.

```
Browser ──bytes──▶ ServerSocket(8080) ──▶ your code gets a raw stream
```

What you would have to write yourself (the lecture's 13 pointers, grouped):

| Concern | You must hand-code |
|---|---|
| **Parsing the request** | Split the raw text into method, URL, query parameters, headers, body |
| **Routing** | `if (path.equals("/hello") && method.equals("GET")) callHello();` for every endpoint |
| **Building the response** | Status line, headers, content type, body, all as text |
| **Concurrency** | One thread (or pool) per client, otherwise the second user waits |
| **Keeping the server alive** | Accept loop, error handling, shutdown |

None of this is business logic. A **servlet container** removes it.

---

### 3. Servlet Container and Servlets

#### 3.1 The split of responsibilities

```
Client ─▶ ┌──────── Tomcat (servlet container) ────────┐
          │ listens on 8080                            │
          │ parses HTTP → HttpServletRequest object    │
          │ creates empty HttpServletResponse object   │
          │ picks the right servlet by URL pattern     │
          │ runs it on a pooled thread                 │
          │        ┌─────────────┐  ┌─────────────┐    │
          │        │ UserServlet │  │ OrderServlet│ ...│
          │        └─────────────┘  └─────────────┘    │
          │ sends the filled response back             │
          └────────────────────────────────────────────┘
```

| Term | Meaning |
|---|---|
| **Servlet** | A Java class (normally `extends HttpServlet`) that you write; it handles requests for some URL pattern |
| **Servlet container / web container** | The runtime that creates servlets, calls them and manages their lifecycle. **Tomcat** is the best-known one |
| **Tomcat** | A long-running Java process; listens on a port (default **8080**, set in `conf/server.xml` `Connector`); hosts web apps |
| `HttpServletRequest` | Object holding method, URL, headers, parameters, body |
| `HttpServletResponse` | Object you fill in (status, headers, body) |

#### 3.2 Request flow

1. Request `GET /users?id=1` arrives on port 8080.
2. Tomcat parses it into an `HttpServletRequest` and creates an empty `HttpServletResponse`.
3. Tomcat looks up which servlet handles `/users`.
4. It creates that servlet object (if not yet created), then calls its `service(request, response)` method.
5. `HttpServlet.service()` looks at the HTTP method and calls `doGet`, `doPost`, `doPut`, `doDelete`, ...
6. Your method **fills the response object** (it returns `void`; the container sends what you wrote).
7. Tomcat sends the response to the client.

> **Precision note:** The container passes **references** to the request and response objects, so changes your servlet makes are visible to Tomcat (the lecture explains this as ordinary Java object sharing).

#### 3.3 Container analogy with Spring

| Spring IoC | Servlet container |
|---|---|
| Container: `ApplicationContext` | Container: Tomcat |
| Managed objects: **beans** | Managed objects: **servlets** |
| You declare beans; the container creates/destroys them | You write servlets; the container creates/destroys them |
| `@PostConstruct` / `@PreDestroy` | `init()` / `destroy()` |

> **Precision note:** The lecture says Spring's container idea "came from" servlets. Treat this as a useful analogy (container-managed components with a lifecycle) rather than a historical claim. One real difference: servlets get **no dependency injection** from Tomcat.

---

### 4. Embedded vs External Tomcat; JAR vs WAR

| | External Tomcat (this lecture) | Embedded Tomcat (Spring Boot) |
|---|---|---|
| Where Tomcat lives | Installed separately on the machine | A library inside your app (starter-web) |
| Packaging | **WAR** (Web Application Archive) | Executable **JAR** |
| Run | Copy WAR to Tomcat's `webapps/`, start Tomcat | `java -jar app.jar` |
| Apps per Tomcat | Many (each WAR is a separate web app, own URL prefix) | One |
| Typical today | Legacy / shared app servers | Microservices, Docker, Kubernetes |

A **WAR** is like a JAR: compiled classes, config files, libraries (`WEB-INF/lib`), plus static files (HTML/CSS/JS). Tomcat unpacks it into `webapps/<name>/` with `WEB-INF/classes`, `WEB-INF/lib`, `META-INF`.

> **Added:** Spring Boot *can* also produce a WAR for an external Tomcat (extend `SpringBootServletInitializer`, mark the Tomcat starter `provided`), but embedded is the norm.

---

### 5. Setting Up (External Tomcat)

1. Download **Apache Tomcat 10.1.x** and unzip it. Tomcat 10+ uses the **`jakarta.*`** packages (Tomcat 9 and older use `javax.*`, Lecture 01).
2. Port is in `conf/server.xml` (`<Connector port="8080" .../>`).
3. Start it: `bin/catalina.sh run` (macOS/Linux; run `chmod +x bin/*.sh` once) or `bin\catalina.bat run` (Windows). The console shows *"Server startup in ... ms"*.
4. Browse to `http://localhost:8080`: the Tomcat welcome page proves it is listening.
5. Stop it: `bin/shutdown.sh` (or `shutdown.bat`).

> **Gotcha (added):** Match the API to the container: Tomcat 10.1 implements Servlet **6.0**; Servlet 6.1 belongs to Tomcat 11. Using a newer API than the server supports may compile but fail at runtime.

---

### 6. Project Setup (Maven, no Spring)

```xml
<packaging>war</packaging>

<dependencies>
    <dependency>
        <groupId>jakarta.servlet</groupId>
        <artifactId>jakarta.servlet-api</artifactId>
        <version>6.0.0</version>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <finalName>crud-app</finalName>
</build>
```

| Setting | Meaning |
|---|---|
| `<packaging>war</packaging>` | Build produces `target/<name>.war` instead of a JAR |
| `jakarta.servlet-api` | Provides `HttpServlet`, `HttpServletRequest`, `HttpServletResponse` (the `jakarta.servlet.*` types) |
| `<scope>provided</scope>` | **Needed to compile, supplied by the container at runtime, so it is not packed into the WAR.** Bundling a second copy risks class conflicts with Tomcat's own |
| `<finalName>` | Name of the WAR, which also becomes the **context path** (`crud-app.war` → `/crud-app`) |

> **Gotcha (added):** If the WAR build complains about a missing `web.xml`, set the `maven-war-plugin` property `failOnMissingWebXml` to `false` (annotation-based servlets need no `web.xml`).

---

### 7. Writing a Servlet

#### 7.1 Mapping with `@WebServlet`

```java
@WebServlet("/users")
public class UserServlet extends HttpServlet { ... }
```

- The annotation tells Tomcat which URL pattern this servlet handles. (The older alternative is XML in `WEB-INF/web.xml`.)
- Mapping is **per class**, not per method: **one servlet = one path**, and *all* HTTP methods for it are handled in that class by overriding `doGet`, `doPost`, `doPut`, `doDelete`.
- Compare Spring MVC, where `@RequestMapping`/`@GetMapping` map **methods**.

| HTTP method | Servlet method |
|---|---|
| GET | `doGet(req, resp)` |
| POST | `doPost(req, resp)` |
| PUT | `doPut(req, resp)` |
| DELETE | `doDelete(req, resp)` |

> **Precision note:** `HttpServlet` has `doGet/doPost/doPut/doDelete/doHead/doOptions/doTrace`. For `PATCH` there is no dedicated standard `do...` method in the traditional API; override `service()` if you need it.

#### 7.2 Reading input and writing output

```java
String idParam = request.getParameter("id");         // always a String; null if absent
int id = Integer.parseInt(idParam);                   // you convert types yourself

response.setStatus(HttpServletResponse.SC_OK);        // 200
response.setContentType("application/json");
response.getWriter().write("{\"message\":\"User added\"}");
```

- `getParameter` reads **query parameters** (and form fields). The lecture sends even POST data as query parameters because parsing a **JSON body** in a plain servlet means reading the stream and using a JSON library by hand.
- Spring Boot does JSON ↔ object conversion for you (Jackson), plus type conversion of parameters.
- `getWriter().write(...)` writes the response body; the method itself returns `void`.
- After you send an error response, **`return`**; otherwise execution continues and may write twice or crash (the lecture's 500 error came from a missing `return`).

#### 7.3 Manual JSON

The lecture builds JSON by string concatenation, with a `StringBuilder` and a loop to join items with commas (and no trailing comma):

```
[ {"id":1,...}, {"id":2,...} ]
```

> **Gotcha (added):** Hand-built JSON breaks on quotes, newlines and special characters (and can be injection-prone). In real code use Jackson or Gson. The lecture accepts the shortcut only to keep the focus on servlets.

---

### 8. Servlet Lifecycle

```
Tomcat creates servlet object      → constructor
                                   → init()          once
 each request:                     → service() → doGet / doPost / doPut / doDelete
Tomcat shuts down / undeploys      → destroy()       once
```

| Method | When | Typical use | Spring analogue |
|---|---|---|---|
| Constructor | Servlet created (by default **lazily on the first request**; can be eager with `loadOnStartup`) | | Bean constructor |
| `init()` | Once, right after creation | Load config, open expensive resources | `@PostConstruct` |
| `service(req, resp)` | **Every request** | Normally *not* overridden; `HttpServlet.service` dispatches to `doGet`, `doPost`, ... by HTTP method | Request handler call |
| `destroy()` | Once, before the servlet is removed (Tomcat shutdown, undeploy) | Close resources | `@PreDestroy` |

Observed in the demo: the constructor and `init()` messages appear in **Tomcat's console** (not IntelliJ, since the code runs inside Tomcat), when the first `/hello` request arrives; `destroy()` prints when Tomcat is shut down (`bin/shutdown.sh`).

> **Gotcha (added) — thread safety:** Tomcat creates **one servlet instance** and calls it from many threads at once. Do **not** keep mutable per-request state in instance fields. The lecture's `HashMap` in the service is acceptable for a demo only; real code needs a `ConcurrentHashMap` or a database.

---

### 9. Deploying and Calling the App

```
mvn package            →  target/crud-app.war
copy it into           →  <tomcat>/webapps/
restart Tomcat         →  Tomcat unpacks webapps/crud-app/ and starts the app
```

URL structure:

```
http://localhost:8080 / crud-app / users ?id=1
        host:port       context     servlet   query
                        path        mapping   parameters
```

- Tomcat hosts **many apps**; the **context path** (the WAR name) selects the app, then the servlet mapping selects the servlet. `ROOT.war` is served at `/`.
- Boot differs: one app per Tomcat, context path `/` by default.
- Forgetting the context path gives **404** (the lecture's first attempt).
- Redeploy flow in the demo: rebuild, delete the old WAR/folder in `webapps`, copy the new WAR, restart Tomcat. In-memory data is lost on every restart.

> **Gotcha (added):** Delete the old exploded folder (`webapps/crud-app/`) when replacing a WAR, or Tomcat may keep stale classes.

---

### 10. The Demo CRUD in Plain Servlets

| Piece | Details |
|---|---|
| `model/User` | POJO: `id` (Integer), `name`, `email`, `mobile`, getters/setters, constructor |
| `service/UserService` | `Map<Integer, User>` as an in-memory "database": `createUser`, `getUserById`, `getAllUsers` |
| `servlet/UserServlet` | `@WebServlet("/users")`; `doPost` creates, `doGet` reads one (`?id=`) or all (no `id`) |

Behaviour built:

| Request | Response |
|---|---|
| `POST /crud-app/users?id=1&name=...&email=...&mobile=...` | 201 and a JSON message; **400** if any field is missing |
| `GET /crud-app/users?id=1` | 200 + JSON user; **404** if absent |
| `GET /crud-app/users` | 200 + JSON array |
| `PUT` / `DELETE` | Left as the lecturer's assignment (update/remove from the map) |

Errors the demo ran into and fixed:

| Error | Cause |
|---|---|
| `ClassCastException` | Casting `map.values()` (a `Collection`) to `List`; fix: `new ArrayList<>(map.values())` |
| `500` on "get all" | Missing `return` after writing the response |
| `404` on the first call | Missing context path `/crud-app` in the URL |

> **Precision note:** The lecture says "2001" for the created status code; it is **201**. Also validate numbers: `Integer.parseInt("abc")` throws `NumberFormatException` (a 500 unless you catch it and return 400).

---

### 11. Pain Points → Why Spring MVC Exists

| Servlet-era pain | What Spring MVC / Boot gives |
|---|---|
| `new UserService()` inside the servlet (tight coupling, Lecture 04) | Dependency injection (`@Autowired`/constructor) |
| Manual JSON building and parsing | Jackson auto-conversion (`@RequestBody`, return object) |
| `getParameter` + `parseInt` + null checks | `@RequestParam` / `@PathVariable` with type conversion |
| Status/headers/body written by hand | `ResponseEntity`, `@ResponseStatus` |
| Routing per class (one path per servlet) | Routing per method (`@GetMapping`, ...) |
| Package, copy WAR, restart Tomcat | Embedded Tomcat: run the app |
| Every servlet repeats validation, routing, response code | One central dispatcher |

---

### 12. The Front Controller: `DispatcherServlet`

Idea: instead of many servlets each doing the same plumbing, use **one central servlet** that receives **every** request and hands it to the right handler.

```
Tomcat
  │ every request
  ▼
Central servlet  (Spring: DispatcherServlet)
  • common work: read request, convert data, write response
  │ finds the right handler by URL + HTTP method
  ▼
Controller A     Controller B     Controller C     (plain beans, not servlets)
```

- In Spring MVC the central servlet is **`DispatcherServlet`**; your small "servlets" become **controllers** (beans) with `@RequestMapping` methods.
- Spring Boot registers `DispatcherServlet` for you and starts an embedded Tomcat (Lecture 09); this is the "magic" behind `@RestController`.
- The next lecture covers Spring MVC internals.

---

### Code

#### `model/User.java`

```java
package in.strikes.model;

public class User {
    private Integer id;
    private String name;
    private String email;
    private String mobile;

    public User(Integer id, String name, String email, String mobile) {
        this.id = id; this.name = name; this.email = email; this.mobile = mobile;
    }
    // getters and setters
}
```

#### `service/UserService.java`

```java
package in.strikes.service;

import in.strikes.model.User;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class UserService {
    private final Map<Integer, User> userDb = new ConcurrentHashMap<>();   // thread-safe, unlike HashMap

    public User createUser(User user)       { userDb.put(user.getId(), user); return user; }
    public User getUserById(Integer id)     { return userDb.get(id); }          // null if absent
    public List<User> getAllUsers()         { return new ArrayList<>(userDb.values()); }
}
```

#### `servlet/UserServlet.java`

```java
package in.strikes.servlet;

@WebServlet("/users")
public class UserServlet extends HttpServlet {

    private final UserService userService = new UserService();   // no DI in plain servlets

    @Override
    public void init() {
        System.out.println("UserServlet init");
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws IOException {
        String id = request.getParameter("id");
        String name = request.getParameter("name");
        String email = request.getParameter("email");
        String mobile = request.getParameter("mobile");

        response.setContentType("application/json");
        if (id == null || name == null || email == null || mobile == null) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);          // 400
            response.getWriter().write("{\"message\":\"Some fields are missing\"}");
            return;                                                           // stop here
        }

        userService.createUser(new User(Integer.parseInt(id), name, email, mobile));
        response.setStatus(HttpServletResponse.SC_CREATED);                   // 201
        response.getWriter().write("{\"message\":\"User added successfully\"}");
    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.setContentType("application/json");
        String idParam = request.getParameter("id");

        if (idParam == null) {                                                // all users
            response.setStatus(HttpServletResponse.SC_OK);
            response.getWriter().write(usersToJson(userService.getAllUsers()));
            return;
        }

        User user = userService.getUserById(Integer.parseInt(idParam));
        if (user == null) {
            response.setStatus(HttpServletResponse.SC_NOT_FOUND);             // 404
            return;
        }
        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write(userToJson(user));
    }

    @Override
    public void destroy() {
        System.out.println("UserServlet destroy");
    }

    private String userToJson(User u) {
        return "{\"id\":" + u.getId() + ",\"name\":\"" + u.getName() + "\","
             + "\"email\":\"" + u.getEmail() + "\",\"mobile\":\"" + u.getMobile() + "\"}";
    }

    private String usersToJson(List<User> users) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < users.size(); i++) {
            sb.append(userToJson(users.get(i)));
            if (i < users.size() - 1) sb.append(",");
        }
        return sb.append("]").toString();
    }
}
```

#### WRONG vs RIGHT

```xml
<!-- WRONG: the servlet API is bundled into the WAR and clashes with Tomcat's copy -->
<scope>compile</scope>
```

```xml
<!-- RIGHT: compile against it, let Tomcat provide it -->
<scope>provided</scope>
```

```java
// WRONG: mutable shared state in a servlet instance field (one instance, many threads)
private final Map<Integer, User> users = new HashMap<>();

// RIGHT for a demo
private final Map<Integer, User> users = new ConcurrentHashMap<>();
```

```java
// WRONG: error response without return, the code continues
response.setStatus(400);
response.getWriter().write("...");
// ... continues and may write again / throw

// RIGHT
response.setStatus(400);
response.getWriter().write("...");
return;
```

#### Running

```bash
mvn package                                  # target/crud-app.war
cp target/crud-app.war $TOMCAT/webapps/
$TOMCAT/bin/catalina.sh run                  # logs print in this terminal
curl -X POST "http://localhost:8080/crud-app/users?id=1&name=Aditya&email=a@x.com&mobile=9876543210"
curl "http://localhost:8080/crud-app/users?id=1"
$TOMCAT/bin/shutdown.sh                      # destroy() runs
```

> **Added (DevOps):** In containers the same model is the official image `tomcat:10.1-jdk21` with the WAR copied to `/usr/local/tomcat/webapps/`. Tune `maxThreads` (default 200) in `server.xml` for the thread pool that serves requests.

---

#### Quick Self-Check

1. **Why can't `java.net` alone build a convenient web app?**
   *Answer:* It only delivers bytes; you would write HTTP parsing, routing, response building and multithreading yourself.

2. **What does a servlet container do for you?**
   *Answer:* Listens on the port, parses HTTP into request/response objects, maps URLs to servlets, runs them on a thread pool and manages their lifecycle.

3. **What is a servlet, and how does it return data if its methods are `void`?**
   *Answer:* A container-managed Java class (usually `extends HttpServlet`). It fills the `HttpServletResponse` object; the container sends it.

4. **Name the servlet lifecycle methods and their Spring analogues.**
   *Answer:* `init()` (like `@PostConstruct`), `service()`/`doXxx()` per request, `destroy()` (like `@PreDestroy`).

5. **What does `@WebServlet("/users")` map, and how are GET/POST/PUT/DELETE separated?**
   *Answer:* One class to one path; the HTTP method selects `doGet`/`doPost`/`doPut`/`doDelete` inside that class.

6. **Why is the servlet API `provided` scope?**
   *Answer:* It is needed to compile, but Tomcat supplies it at runtime; packing it risks conflicts.

7. **How is the URL `http://localhost:8080/crud-app/users` built?**
   *Answer:* Host and port, then the context path (WAR name), then the servlet mapping.

8. **How does `DispatcherServlet` improve on writing many servlets?**
   *Answer:* It is one front controller that does the common work and dispatches to controller beans.

---

#### Golden Rules / Checklist

- [ ] Match the servlet API version to the Tomcat version (Tomcat 10.1 ↔ Servlet 6.0, `jakarta.*`)
- [ ] Use `provided` scope for the servlet API in WAR projects
- [ ] Remember the context path (WAR name) in the URL
- [ ] Do not keep mutable request state in servlet fields; servlets are shared by threads
- [ ] `return` after writing an error response
- [ ] Use a JSON library; do not build JSON by hand in real code
- [ ] Validate and parse parameters defensively (`NumberFormatException`)
- [ ] Remove old exploded WAR folders when redeploying
- [ ] Read Tomcat's console/logs, not the IDE console, for servlet output
- [ ] Prefer embedded Tomcat (Boot JAR) for new services; one app per process

---

### Practice Questions

#### Basic

1. Install Tomcat 10.1, start it, and confirm `http://localhost:8080` responds. Where is the port configured?

2. Create a Maven WAR project with the servlet API as `provided`. What happens if you change it to `compile`?

3. Write a `HelloServlet` mapped to `/hello` that returns plain text. Where do its constructor, `init` and `destroy` messages appear?

4. Deploy it and call it with and without the context path. Explain the difference.

#### Intermediate

5. Implement `doPost` and `doGet` for `/users` as in the lecture, then add `doPut` and `doDelete` (the lecturer's assignment).

6. Add validation: reject a non-numeric id with 400 instead of letting `NumberFormatException` cause a 500.

7. Replace manual JSON with Jackson. What do you gain?

8. Show a thread-safety bug using a plain `HashMap` and concurrent requests, then fix it.

#### Advanced / Interview-style

9. Explain the full path of an HTTP request through Tomcat to `doGet`, naming the objects created.

10. Compare servlets and Spring beans: creation, lifecycle, DI, mapping granularity.

11. Compare deploying WARs to a shared Tomcat with running Boot JARs with embedded Tomcat (isolation, scaling, ops, Docker).

12. Describe the Front Controller pattern and how Spring MVC's `DispatcherServlet` implements it.


// ^###(?=\s)
// ^\s*---\s*$