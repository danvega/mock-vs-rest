# MockMvcTester vs RestTestClient: Controller Testing in Spring Framework 7

A practical guide to choosing between Spring Framework 7's two controller testing approaches.

## The Question

> "I'm testing controllers in Spring Framework 7 and I have a couple of choices. I can use MockMvcTester or RestTestClient. Both work. Is there a preferred method or when to use one over the other?"

Great question! I asked the Spring team, and here's what I learned.

## TL;DR - Quick Decision Guide

| If you... | Choose |
|-----------|--------|
| Prefer AssertJ-style assertions | **MockMvcTester** |
| Need server-side inspection (handlers, exceptions) | **MockMvcTester** |
| Need multipart/file upload testing | **MockMvcTester** |
| Want one API for mock AND real HTTP tests | **RestTestClient** |
| Need non-JSON deserialization (XML, Protobuf) | **RestTestClient** |
| Already know WebTestClient | **RestTestClient** |

**Bottom line:** Both use mock infrastructure under the hood (RestTestClient wraps MockMvc), so it's largely a matter of personal preference and which API you prefer.

## Deep Dive Comparison

### MockMvcTester

**Setup:** Just autowire it!
```java
@WebMvcTest(BookController.class)
class BookControllerTest {

    @Autowired
    MockMvcTester mockMvcTester;

    @MockitoBean
    BookRepository bookRepository;
}
```

**API Style:** AssertJ-focused
```java
assertThat(mockMvcTester.get().uri("/api/books"))
    .hasStatusOk()
    .bodyJson()
    .extractingPath("$")
    .asArray()
    .hasSize(2);
```

**Strengths:**
- Deep AssertJ integration - natural `assertThat()` wrapping
- Server-side inspection - access to handlers, exceptions, model attributes
- Request customization - fine-grained HttpServletRequest setup
- Multipart support - file uploads work out of the box
- JSON path chaining - fluent `extractingPath()` navigation

### RestTestClient

**Setup:** Add `@AutoConfigureRestTestClient` for direct autowiring!
```java
@WebMvcTest(BookController.class)
@AutoConfigureRestTestClient
class BookControllerTest {

    @Autowired
    RestTestClient client;

    @MockitoBean
    BookRepository bookRepository;
}
```

**API Style:** WebTestClient-style fluent
```java
client.get().uri("/api/books")
    .exchange()
    .expectStatus().isOk()
    .expectBody()
    .jsonPath("$.length()").isEqualTo(2);
```

**Strengths:**
- API consistency - same API across all test types
- Broader deserialization - any HttpMessageConverter (XML, Protobuf, etc.)
- Typed response bodies - `expectBody(Book.class)` and `expectBodyList(Book.class)`
- Auto-switching - uses mock or real server based on test configuration

## The Binding Options (RestTestClient Superpower)

RestTestClient's killer feature is **binding flexibility**. The same API works across:

| Binding | Use Case | Spring Context? |
|---------|----------|-----------------|
| `bindToController(new BookController())` | Pure unit test | No |
| `bindTo(mockMvc)` | Slice test with validation | Partial |
| `bindToApplicationContext(ctx)` | Full integration test | Full |
| `bindToServer()` | Real HTTP (CORS, compression) | Full + Server |

With `@AutoConfigureRestTestClient`:
- In `@WebMvcTest` → automatically binds to MockMvc
- In `@SpringBootTest` → auto-switches based on your web environment config

## Side-by-Side Examples

### Basic GET Request

**MockMvcTester:**
```java
assertThat(mockMvcTester.get().uri("/api/books/1"))
    .hasStatusOk()
    .bodyJson()
    .extractingPath("$.title")
    .isEqualTo("Clean Code");
```

**RestTestClient:**
```java
client.get().uri("/api/books/1")
    .exchange()
    .expectStatus().isOk()
    .expectBody()
    .jsonPath("$.title").isEqualTo("Clean Code");
```

### POST with JSON Body

**MockMvcTester:**
```java
assertThat(mockMvcTester.post()
    .uri("/api/books")
    .contentType(MediaType.APPLICATION_JSON)
    .content("""
        {"title": "Clean Code", "author": "Robert Martin"}
        """))
    .hasStatus(HttpStatus.CREATED);
```

**RestTestClient:**
```java
client.post().uri("/api/books")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue("""
        {"title": "Clean Code", "author": "Robert Martin"}
        """)
    .exchange()
    .expectStatus().isCreated();
```

### Typed Object Deserialization

**MockMvcTester:**
```java
// Supports JSON deserialization
assertThat(mockMvcTester.get().uri("/api/books"))
    .bodyJson()
    .extractingPath("$[0]")
    .convertTo(Book.class)
    .satisfies(book -> assertThat(book.title()).isEqualTo("Clean Code"));
```

**RestTestClient:**
```java
// Supports ANY HttpMessageConverter
client.get().uri("/api/books/1")
    .exchange()
    .expectBody(Book.class)
    .value(book -> {
        assertThat(book.title()).isEqualTo("Clean Code");
    });
```

## Current Limitations

### RestTestClient
- **No multipart support yet** (Spring issue #35569)
- Use MockMvcTester for file upload tests

### MockMvcTester
- **JSON-focused deserialization**
- Less suitable for XML/Protobuf responses

## My Recommendation

**Start with MockMvcTester** if:
- Your team uses AssertJ extensively
- You're testing primarily JSON APIs
- You need file upload testing

**Start with RestTestClient** if:
- You want consistency with WebTestClient
- You're building tests that might later need real HTTP
- You're working with multiple content types

**Either way:** Both are excellent choices. Pick the one whose API feels more natural to you.

## Running This Demo

```bash
# Run all tests
./mvnw test

# Run MockMvcTester tests only
./mvnw test -Dtest=BookControllerMockMvcTesterTest

# Run RestTestClient tests only
./mvnw test -Dtest=BookControllerRestTestClientTest

# Run the application
./mvnw spring-boot:run
```

## Technology Stack

- **Spring Boot 4.0** - Latest with unified testing support
- **Spring Framework 7** - MockMvcTester and RestTestClient
- **JUnit 5** - Test framework
- **AssertJ** - Fluent assertions
- **In-memory repository** - No database required

## Resources

- [Spring Framework 7 RestTestClient](https://www.danvega.dev/blog/spring-framework-7-rest-test-client)
- [MockMvcTester Documentation](https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html)
- [RestTestClient Issue #35569](https://github.com/spring-projects/spring-framework/issues/35569) - Multipart support