# Java Version Features — Java 8 to Java 24

## Category

Java, Language Evolution, JVM

## Context

Java follows a **6-month release cadence** (since Java 9) with Long-Term Support (LTS) releases every two years. LTS versions receive extended security and bug-fix updates, making them the default choice for production workloads.

### LTS Timeline

| Version | Release | LTS | Support Until |
|---------|---------|-----|--------------|
| Java 8  | Mar 2014 | ✅ | Mar 2030 (Oracle) |
| Java 11 | Sep 2018 | ✅ | Sep 2026 |
| Java 17 | Sep 2021 | ✅ | Sep 2029 |
| Java 21 | Sep 2023 | ✅ | Sep 2031 |
| Java 25 | Sep 2025 | ✅ (upcoming) | TBD |

### Quick Upgrade Guide

```
Legacy app on Java 8 → migrate to Java 21 (LTS)
  Must handle: module system (Java 9), removed APIs (javax → jakarta), record types

Greenfield service → start on Java 21 or 24
  Use: records, sealed classes, pattern matching, virtual threads
```

---

## Java 8 (LTS) — March 2014

### Key Features

| Feature | JEP / API |
|---------|----------|
| Lambda expressions | Language |
| Stream API | `java.util.stream` |
| `Optional<T>` | `java.util.Optional` |
| Default & static interface methods | Language |
| Method references | Language |
| New Date/Time API | `java.time` (JSR-310) |
| `CompletableFuture` | `java.util.concurrent` |
| Nashorn JavaScript engine | `javax.script` |
| `Base64` encode/decode | `java.util.Base64` |
| Repeating annotations | Language |

```java
// ── Lambda & Stream API ───────────────────────────────────────────────────────
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Dave");

List<String> filtered = names.stream()
    .filter(n -> n.startsWith("C") || n.startsWith("D"))
    .map(String::toUpperCase)               // method reference
    .sorted()
    .collect(Collectors.toList());
// [CHARLIE, DAVE]

// ── Optional ─────────────────────────────────────────────────────────────────
import java.util.Optional;

Optional<String> maybeName = Optional.ofNullable(getUserName());
String display = maybeName
    .filter(n -> !n.isBlank())
    .map(n -> "Hello, " + n)
    .orElse("Hello, Guest");

// ── Default interface method ──────────────────────────────────────────────────
interface Validator<T> {
    boolean isValid(T value);

    default Validator<T> and(Validator<T> other) {
        return value -> this.isValid(value) && other.isValid(value);
    }
}

// ── New Date/Time API ─────────────────────────────────────────────────────────
import java.time.*;
import java.time.format.DateTimeFormatter;

LocalDate today     = LocalDate.now();
LocalDate tomorrow  = today.plusDays(1);
ZonedDateTime zonedNow = ZonedDateTime.now(ZoneId.of("Europe/London"));
String formatted    = zonedNow.format(DateTimeFormatter.ISO_OFFSET_DATE_TIME);

// ── CompletableFuture ─────────────────────────────────────────────────────────
import java.util.concurrent.CompletableFuture;

CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchFromApi())          // runs in ForkJoinPool
    .thenApply(String::toUpperCase)
    .thenApply(s -> "Result: " + s)
    .exceptionally(ex -> "Error: " + ex.getMessage());

String result = future.join();  // block until complete
```

---

## Java 9 — September 2017

### Key Features

| Feature | JEP |
|---------|-----|
| Java Platform Module System (JPMS) | JEP 261 |
| JShell REPL | JEP 222 |
| Collection factory methods | JEP 269 |
| `Stream` improvements (`takeWhile`, `dropWhile`, `iterate`) | JEP 269 |
| `Optional` improvements (`ifPresentOrElse`, `stream`) | — |
| Private interface methods | JEP 213 |
| Process API improvements | JEP 102 |
| HTTP/2 Client (incubator) | JEP 110 |

```java
// ── Collection factory methods (immutable) ────────────────────────────────────
import java.util.List;
import java.util.Map;
import java.util.Set;

List<String> list = List.of("a", "b", "c");           // immutable, no nulls
Set<Integer> set  = Set.of(1, 2, 3);
Map<String, Integer> map = Map.of("one", 1, "two", 2);

// ── Stream improvements ───────────────────────────────────────────────────────
import java.util.stream.Stream;

// takeWhile — stop at first non-matching element (ordered stream)
Stream.of(1, 2, 3, 4, 5, 1)
    .takeWhile(n -> n < 4)
    .forEach(System.out::println);   // 1, 2, 3

// dropWhile
Stream.of(1, 2, 3, 4, 5)
    .dropWhile(n -> n < 3)
    .forEach(System.out::println);   // 3, 4, 5

// iterate with predicate (like a for loop)
Stream.iterate(0, n -> n < 10, n -> n + 2)
    .forEach(System.out::println);   // 0 2 4 6 8

// ── Optional.ifPresentOrElse ──────────────────────────────────────────────────
Optional.ofNullable(getValue())
    .ifPresentOrElse(
        v  -> System.out.println("Found: " + v),
        () -> System.out.println("Not found")
    );

// ── Private interface methods ─────────────────────────────────────────────────
interface PaymentProcessor {
    default void process(double amount) {
        validate(amount);
        executeTransfer(amount);
    }

    private void validate(double amount) {        // private — not part of API
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
    }

    private void executeTransfer(double amount) { /* ... */ }
}

// ── Module declaration (module-info.java) ─────────────────────────────────────
/*
module com.example.payments {
    requires java.net.http;
    requires com.example.core;
    exports com.example.payments.api;
    opens   com.example.payments.model to com.fasterxml.jackson.databind;
}
*/
```

---

## Java 10 — March 2018

### Key Features

| Feature | JEP |
|---------|-----|
| Local variable type inference (`var`) | JEP 286 |
| Unmodifiable collection copies | `List.copyOf`, `Map.copyOf` |
| `Optional.orElseThrow()` | — |
| Parallel full GC for G1 | JEP 307 |
| Application class-data sharing | JEP 310 |

```java
// ── var — local variable type inference ──────────────────────────────────────
var list = new ArrayList<String>();      // inferred: ArrayList<String>
var map  = new HashMap<String, List<Integer>>();

// ✅ Good use of var — obvious type from context
var payments = paymentRepository.findAll();
for (var payment : payments) {
    System.out.println(payment.getId());
}

// ✅ Good with complex generic types (less verbosity)
var result = Map.of("payments", List.of(1, 2, 3));

// ❌ Bad use of var — type not obvious without IDE
var x = compute();   // reader must look up compute()'s return type

// ── Unmodifiable copies ───────────────────────────────────────────────────────
var original = new ArrayList<>(List.of("a", "b", "c"));
original.add("d");

var copy = List.copyOf(original);   // immutable snapshot; throws on null elements
// copy.add("e");                   // UnsupportedOperationException

// ── Optional.orElseThrow ──────────────────────────────────────────────────────
Payment payment = paymentRepository.findById(id)
    .orElseThrow();   // throws NoSuchElementException (prefer orElseThrow(supplier))

Payment payment2 = paymentRepository.findById(id)
    .orElseThrow(() -> new PaymentNotFoundException(id));
```

---

## Java 11 (LTS) — September 2018

### Key Features

| Feature | JEP |
|---------|-----|
| `String` new methods | `isBlank`, `lines`, `strip`, `repeat` |
| `Files.readString/writeString` | — |
| `var` in lambda parameters | JEP 323 |
| Standard HTTP Client | JEP 321 |
| Running single-file source programs | JEP 330 |
| `Collection.toArray(IntFunction)` | — |
| Removed: Java EE and CORBA modules | JEP 320 |

```java
// ── String improvements ───────────────────────────────────────────────────────
String text = "  hello world  ";

text.strip();          // "hello world"   — Unicode-aware (vs trim())
text.stripLeading();   // "hello world  "
text.stripTrailing();  // "  hello world"
text.isBlank();        // false
"  ".isBlank();        // true
"ha".repeat(3);        // "hahaha"

"line1\nline2\nline3"
    .lines()
    .map(String::strip)
    .forEach(System.out::println);

// ── Files.readString / writeString ────────────────────────────────────────────
import java.nio.file.Files;
import java.nio.file.Path;

String content = Files.readString(Path.of("payments.json"));
Files.writeString(Path.of("output.json"), content);

// ── var in lambda parameters (enables annotations) ────────────────────────────
import java.util.function.Function;

Function<String, String> processor =
    (@NotNull var input) -> input.strip().toUpperCase();

// ── Standard HTTP Client ──────────────────────────────────────────────────────
import java.net.http.*;
import java.net.URI;

var client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .build();

var request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/payments"))
    .header("Authorization", "Bearer " + token)
    .GET()
    .timeout(Duration.ofSeconds(5))
    .build();

// Synchronous
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.statusCode() + ": " + response.body());

// Asynchronous
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
```

---

## Java 12 — March 2019

### Key Features

| Feature | JEP |
|---------|-----|
| Switch expressions (preview) | JEP 325 |
| Teeing collector | `Collectors.teeing` |
| `String.indent`, `String.transform` | — |
| Compact number formatting | `NumberFormat.getCompactNumberInstance` |
| Shenandoah GC (experimental) | JEP 189 |

```java
// ── Switch expression (preview — became standard in Java 14) ──────────────────
// Old switch statement (fall-through, verbose)
int numLetters;
switch (day) {
    case MONDAY: case FRIDAY: case SUNDAY: numLetters = 6; break;
    case TUESDAY:                          numLetters = 7; break;
    default:                               numLetters = 8;
}

// New switch expression (no fall-through, returns value)
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY               -> 7;
    default                    -> 8;
};

// ── Teeing collector ──────────────────────────────────────────────────────────
import java.util.stream.Collectors;

record PaymentStats(long count, double total) {}

PaymentStats stats = payments.stream()
    .collect(Collectors.teeing(
        Collectors.counting(),
        Collectors.summingDouble(Payment::amount),
        PaymentStats::new             // merge both results
    ));

// ── Compact number format ─────────────────────────────────────────────────────
import java.text.NumberFormat;
import java.util.Locale;

var fmt = NumberFormat.getCompactNumberInstance(Locale.UK, NumberFormat.Style.SHORT);
fmt.setMinimumFractionDigits(1);
System.out.println(fmt.format(1_500_000));   // "1.5M"
System.out.println(fmt.format(2_300));        // "2.3K"
```

---

## Java 13 — September 2019

### Key Features

| Feature | JEP |
|---------|-----|
| Text blocks (preview) | JEP 355 |
| Switch expressions (second preview) | JEP 354 |
| `String.formatted` / `stripIndent` / `translateEscapes` | — |
| FileSystems API improvements | — |

```java
// ── Text blocks (preview — standard in Java 15) ───────────────────────────────
// Old
String json = "{\n  \"id\": \"pay_001\",\n  \"amount\": 150.00\n}";

// Text block — preserves indentation, no escape noise
String json = """
        {
          "id": "pay_001",
          "amount": 150.00,
          "currency": "GBP"
        }
        """;

// Useful for SQL, HTML, JSON, XML in tests
String sql = """
        SELECT id, amount, currency
        FROM   payments
        WHERE  account_id = ?
          AND  status     = 'pending'
        ORDER  BY created_at DESC
        """;

// String.formatted (like String.format but instance method)
String message = "Payment %s of %.2f %s".formatted("pay_001", 150.00, "GBP");
```

---

## Java 14 — March 2020

### Key Features

| Feature | JEP |
|---------|-----|
| Switch expressions (standard) | JEP 361 |
| Records (preview) | JEP 359 |
| Pattern matching for `instanceof` (preview) | JEP 305 |
| `NullPointerException` helpful messages | JEP 358 |
| `jpackage` tool | JEP 343 |

```java
// ── Switch expressions (now standard) ────────────────────────────────────────
PaymentStatus status = switch (code) {
    case "00" -> PaymentStatus.APPROVED;
    case "05" -> PaymentStatus.DECLINED;
    case "96" -> {
        log.warn("System error for payment {}", paymentId);
        yield PaymentStatus.FAILED;   // yield returns from block
    }
    default   -> PaymentStatus.UNKNOWN;
};

// ── Records (preview → standard in Java 16) ───────────────────────────────────
// Compact, immutable data carriers — auto-generates constructor, getters, equals, hashCode, toString
record Money(double amount, String currency) {
    // Compact constructor for validation
    Money {
        if (amount < 0)              throw new IllegalArgumentException("Amount must be >= 0");
        if (currency == null || currency.isBlank()) throw new IllegalArgumentException("Currency required");
        currency = currency.toUpperCase();   // normalise in compact constructor
    }

    // Custom instance method
    Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new ArithmeticException("Currency mismatch");
        return new Money(this.amount + other.amount, this.currency);
    }
}

Money price = new Money(150.00, "gbp");   // currency normalised to "GBP"
System.out.println(price);                // Money[amount=150.0, currency=GBP]

// ── Pattern matching for instanceof (preview) ─────────────────────────────────
// Old
if (obj instanceof Payment) {
    Payment p = (Payment) obj;
    System.out.println(p.getId());
}

// New — binding variable
if (obj instanceof Payment p) {
    System.out.println(p.getId());   // p is already cast
}
```

---

## Java 15 — September 2020

### Key Features

| Feature | JEP |
|---------|-----|
| Text blocks (standard) | JEP 378 |
| Sealed classes (preview) | JEP 360 |
| Hidden classes | JEP 371 |
| `EdDSA` cryptography | JEP 339 |
| Removed: Nashorn JavaScript engine | JEP 372 |

```java
// ── Text blocks (now standard) ────────────────────────────────────────────────
String html = """
        <html>
            <body>
                <p>Payment confirmed: %s</p>
            </body>
        </html>
        """.formatted(paymentId);

// ── Sealed classes (preview → standard in Java 17) ────────────────────────────
// Only listed subclasses may extend/implement a sealed type
sealed interface PaymentResult
    permits ApprovedResult, DeclinedResult, FailedResult {}

record ApprovedResult(String transactionId, Instant processedAt) implements PaymentResult {}
record DeclinedResult(String reason)                              implements PaymentResult {}
record FailedResult  (String errorCode, String message)          implements PaymentResult {}

// Exhaustive matching becomes possible
PaymentResult result = processPayment(request);
String summary = switch (result) {
    case ApprovedResult a -> "Approved: " + a.transactionId();
    case DeclinedResult d -> "Declined: " + d.reason();
    case FailedResult   f -> "Failed ["  + f.errorCode() + "]: " + f.message();
    // No default needed — compiler knows all cases are covered
};
```

---

## Java 16 — March 2021

### Key Features

| Feature | JEP |
|---------|-----|
| Records (standard) | JEP 395 |
| Pattern matching for `instanceof` (standard) | JEP 394 |
| `Stream.toList()` | — |
| `Stream.mapMulti` | — |
| Unix-Domain Socket Channels | JEP 380 |
| Warnings for illegal reflective access | JEP 396 |

```java
// ── Records (now standard) ────────────────────────────────────────────────────
record PaymentRequest(String accountId, double amount, String currency, String reference) {
    // Static factory
    static PaymentRequest of(String accountId, double amount) {
        return new PaymentRequest(accountId, amount, "GBP", UUID.randomUUID().toString());
    }
}

// ── Pattern matching for instanceof (now standard) ────────────────────────────
void describe(Object shape) {
    if      (shape instanceof Circle c)    System.out.println("Circle r=" + c.radius());
    else if (shape instanceof Rectangle r) System.out.println("Rect " + r.width() + "×" + r.height());
    else                                   System.out.println("Unknown shape");
}

// ── Stream.toList() — shorthand for Collectors.toUnmodifiableList() ────────────
List<String> names = payments.stream()
    .filter(p -> p.status() == Status.PENDING)
    .map(Payment::reference)
    .toList();    // unmodifiable, replaces .collect(Collectors.toList())

// ── Stream.mapMulti — flatMap alternative ────────────────────────────────────
List<Integer> result = Stream.of(1, 2, 3)
    .<Integer>mapMulti((n, consumer) -> {
        consumer.accept(n);
        consumer.accept(n * n);
    })
    .toList();
// [1, 1, 2, 4, 3, 9]
```

---

## Java 17 (LTS) — September 2021

### Key Features

| Feature | JEP |
|---------|-----|
| Sealed classes (standard) | JEP 409 |
| Pattern matching for `switch` (preview) | JEP 406 |
| Restore always-strict floating-point | JEP 306 |
| Enhanced pseudo-random number generators | JEP 356 |
| New macOS rendering pipeline | JEP 382 |
| Removed: Applet API | JEP 398 |
| Strong encapsulation of JDK internals | JEP 403 |

```java
// ── Sealed classes (now standard) ────────────────────────────────────────────
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle   (double radius)          implements Shape {}
record Rectangle(double width, double height) implements Shape {}
record Triangle (double base, double height) implements Shape {}

// ── Pattern matching for switch (preview → standard in Java 21) ───────────────
double area = switch (shape) {
    case Circle    c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle  t -> 0.5 * t.base() * t.height();
};

// Guarded patterns (when clause)
String describe = switch (payment) {
    case Payment p when p.amount() > 10_000 -> "High-value payment: " + p.id();
    case Payment p when p.status() == PENDING -> "Pending: " + p.id();
    case Payment p -> "Standard: " + p.id();
};

// ── Enhanced PRNG ─────────────────────────────────────────────────────────────
import java.util.random.RandomGenerator;
import java.util.random.RandomGeneratorFactory;

RandomGenerator rng = RandomGeneratorFactory.of("Xoshiro256PlusPlus").create();
long token = rng.nextLong();
```

---

## Java 18 — March 2022

### Key Features

| Feature | JEP |
|---------|-----|
| Simple web server | JEP 408 |
| Code snippets in Javadoc | JEP 413 |
| `UTF-8` as default charset | JEP 400 |
| `@Deprecated` for finalization | JEP 421 |
| Vector API (incubator 3rd) | JEP 417 |
| Pattern matching for `switch` (2nd preview) | JEP 420 |

```java
// ── Simple web server (for local dev / testing) ───────────────────────────────
// jwebserver --port 8080 --directory ./build/docs

// Or programmatically:
import com.sun.net.httpserver.*;
import java.net.InetSocketAddress;
import java.nio.file.Path;

var server = SimpleFileServer.createFileServer(
    new InetSocketAddress(8080),
    Path.of("./public"),
    SimpleFileServer.OutputLevel.VERBOSE
);
server.start();

// ── UTF-8 default (Java 18 onwards — previously platform-dependent) ───────────
// Before Java 18: files.write() used platform charset (e.g. windows-1252 on Windows)
// Now always UTF-8 unless explicitly overridden:
//   -Dfile.encoding=UTF-8   (was needed before; now the default)

// Always specify charset explicitly for portability:
import java.nio.charset.StandardCharsets;
Files.writeString(Path.of("output.txt"), content, StandardCharsets.UTF_8);
```

---

## Java 19 — September 2022

### Key Features

| Feature | JEP |
|---------|-----|
| Virtual threads (preview) | JEP 425 |
| Structured concurrency (incubator) | JEP 428 |
| Record patterns (preview) | JEP 405 |
| Pattern matching for `switch` (3rd preview) | JEP 427 |
| Foreign Function & Memory API (preview) | JEP 424 |

```java
// ── Virtual threads (preview → standard in Java 21) ───────────────────────────
// Virtual threads = lightweight threads managed by the JVM (not OS threads)
// Ideal for I/O-bound tasks (HTTP calls, DB queries, file reads)

// Create one virtual thread per request — no thread pool needed
Thread.ofVirtual().start(() -> {
    var response = httpClient.send(request, BodyHandlers.ofString());
    processResponse(response);
});

// ExecutorService with virtual threads per task
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var payment : payments) {
        executor.submit(() -> processPayment(payment));  // thousands of tasks, low overhead
    }
}  // auto-close waits for all tasks

// ── Record patterns (preview → standard in Java 21) ───────────────────────────
// Deconstruct records in pattern matching
if (result instanceof ApprovedResult(var txnId, var processedAt)) {
    System.out.println("TXN: " + txnId + " at " + processedAt);
}

// Nested record deconstruction
record Order(Payment payment, Address address) {}
record Payment(String id, Money amount) {}
record Money(double value, String currency) {}

if (order instanceof Order(Payment(var id, Money(var value, var currency)), var address)) {
    System.out.printf("Order %s: %.2f %s to %s%n", id, value, currency, address);
}
```

---

## Java 20 — March 2023

### Key Features

| Feature | JEP |
|---------|-----|
| Virtual threads (2nd preview) | JEP 436 |
| Structured concurrency (2nd incubator) | JEP 437 |
| Record patterns (2nd preview) | JEP 432 |
| Pattern matching for `switch` (4th preview) | JEP 433 |
| Scoped values (incubator) | JEP 429 |
| Foreign Function & Memory API (2nd preview) | JEP 434 |

```java
// ── Scoped values (incubator → standard in Java 21) ───────────────────────────
// Like ThreadLocal but designed for virtual threads: immutable, fork-safe
import jdk.incubator.concurrent.ScopedValue;

static final ScopedValue<String> CORRELATION_ID = ScopedValue.newInstance();
static final ScopedValue<User>   CURRENT_USER   = ScopedValue.newInstance();

// Bind in request handler
ScopedValue.where(CORRELATION_ID, "corr-123")
    .where(CURRENT_USER, user)
    .run(() -> {
        processPayment(request);     // any code here can read CORRELATION_ID
        auditLog();
    });

// Read anywhere in the call tree — no parameter threading needed
void auditLog() {
    String corrId = CORRELATION_ID.get();   // always set within the where() scope
    logger.info("[{}] audit event", corrId);
}
```

---

## Java 21 (LTS) — September 2023

### Key Features

| Feature | JEP |
|---------|-----|
| Virtual threads (standard) | JEP 444 |
| Pattern matching for `switch` (standard) | JEP 441 |
| Record patterns (standard) | JEP 440 |
| Sequenced collections | JEP 431 |
| String templates (preview) | JEP 430 |
| Unnamed classes & instance main (preview) | JEP 445 |
| Structured concurrency (preview) | JEP 453 |
| Scoped values (preview) | JEP 446 |

```java
// ── Virtual threads (now standard) ───────────────────────────────────────────
// Replace traditional thread pool with virtual thread executor for I/O-bound services
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<CompletableFuture<PaymentResult>> futures = payments.stream()
        .map(p -> CompletableFuture.supplyAsync(() -> processPayment(p), executor))
        .toList();

    CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new)).join();
}

// ── Pattern matching for switch (standard) ────────────────────────────────────
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0  -> "Positive int: " + i;
        case Integer i             -> "Non-positive int: " + i;
        case String s  when !s.isBlank() -> "Non-blank string: " + s;
        case String s              -> "Blank string";
        case null                  -> "null value";
        default                    -> "Other: " + obj.getClass().getSimpleName();
    };
}

// ── Record patterns (standard) ────────────────────────────────────────────────
void processResult(PaymentResult result) {
    switch (result) {
        case ApprovedResult(var txnId, var time) ->
            System.out.println("Approved txn=" + txnId + " at " + time);
        case DeclinedResult(var reason) ->
            System.out.println("Declined: " + reason);
        case FailedResult(var code, var msg) ->
            System.out.println("Failed [" + code + "]: " + msg);
    }
}

// ── Sequenced collections ─────────────────────────────────────────────────────
// New interfaces: SequencedCollection, SequencedSet, SequencedMap
import java.util.SequencedCollection;

SequencedCollection<String> sc = new ArrayList<>(List.of("a", "b", "c"));
sc.addFirst("z");       // "z", "a", "b", "c"
sc.addLast("y");        // "z", "a", "b", "c", "y"
String first = sc.getFirst();
String last  = sc.getLast();
SequencedCollection<String> reversed = sc.reversed();

// LinkedHashMap now exposes sequenced access:
var map = new LinkedHashMap<String, Integer>();
map.put("one", 1);
map.put("two", 2);
map.firstEntry();        // Map.Entry<"one", 1>
map.lastEntry();         // Map.Entry<"two", 2>

// ── Structured concurrency (preview) ─────────────────────────────────────────
import java.util.concurrent.StructuredTaskScope;

try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask    = scope.fork(() -> fetchUser(userId));
    var paymentTask = scope.fork(() -> fetchPayments(userId));

    scope.join()           // wait for both
         .throwIfFailed(); // propagate first failure

    return new Dashboard(userTask.get(), paymentTask.get());
}  // cancel any still-running tasks on exit
```

---

## Java 22 — March 2024

### Key Features

| Feature | JEP |
|---------|-----|
| Unnamed variables & patterns (`_`) | JEP 456 |
| Statements before `super()` (preview) | JEP 447 |
| String templates (2nd preview) | JEP 459 |
| Structured concurrency (2nd preview) | JEP 462 |
| Scoped values (2nd preview) | JEP 464 |
| Foreign Function & Memory API (standard) | JEP 454 |
| Launch multi-file source programs | JEP 458 |

```java
// ── Unnamed variables & patterns ──────────────────────────────────────────────
// Use _ to indicate a variable is intentionally unused

// In catch clauses — when you only want the side effect
try {
    processPayment(request);
} catch (PaymentException _) {
    // error already logged inside processPayment — just returning null
}

// In for-each — when you only care about the count
int count = 0;
for (var _ : payments) count++;

// In pattern matching — ignore irrelevant components
switch (result) {
    case ApprovedResult(var txnId, _) ->
        System.out.println("Approved: " + txnId);  // ignore processedAt
    case DeclinedResult(var reason) ->
        System.out.println("Declined: " + reason);
    case FailedResult(_, var msg) ->
        System.out.println("Failed: " + msg);       // ignore error code
}

// In lambda — unused parameter
payments.forEach((_ ) -> totalCount.incrementAndGet());

// ── Foreign Function & Memory API (standard) ──────────────────────────────────
// Call native libraries (C, Rust) without JNI — safe, GC-aware
import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;

try (var arena = Arena.ofConfined()) {
    MemorySegment str  = arena.allocateFrom("hello");
    MethodHandle strlen = Linker.nativeLinker()
        .downcallHandle(
            Linker.nativeLinker().defaultLookup().find("strlen").orElseThrow(),
            FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS)
        );
    long len = (long) strlen.invoke(str);  // 5
}
```

---

## Java 23 — September 2024

### Key Features

| Feature | JEP |
|---------|-----|
| Primitive types in patterns (preview) | JEP 455 |
| Markdown in Javadoc comments | JEP 467 |
| Structured concurrency (3rd preview) | JEP 480 |
| Scoped values (3rd preview) | JEP 481 |
| Module import declarations (preview) | JEP 476 |
| Implicitly declared classes (2nd preview) | JEP 477 |
| Remove deprecated thread stop/destroy | JEP 479 |

```java
// ── Primitive types in patterns (preview) ─────────────────────────────────────
// Pattern matching now works with primitive types directly
Object value = 42;

switch (value) {
    case int i when i < 0  -> System.out.println("Negative int: " + i);
    case int i             -> System.out.println("Non-negative int: " + i);
    case long l            -> System.out.println("Long: " + l);
    case double d          -> System.out.println("Double: " + d);
    case String s          -> System.out.println("String: " + s);
    default                -> System.out.println("Other");
}

// ── Module import declarations (preview) ──────────────────────────────────────
// Import all exported packages of a module in one line
import module java.base;    // imports java.lang, java.util, java.io, etc. from java.base
import module java.net.http; // imports java.net.http

// Before (verbose):
// import java.util.List;
// import java.util.Map;
// import java.util.Optional;
// ...

// ── Markdown Javadoc ──────────────────────────────────────────────────────────
/**
 * Processes a payment request.
 *
 * ## Parameters
 * - `request` — the payment to process; must not be null
 *
 * ## Returns
 * A `PaymentResult` indicating success or failure.
 *
 * ## Throws
 * - `PaymentException` if the request is invalid
 *
 * ## Example
 * ```java
 * var result = processor.process(PaymentRequest.of("acc_001", 150.0));
 * ```
 */
PaymentResult process(PaymentRequest request);
```

---

## Java 24 — March 2025

### Key Features

| Feature | JEP |
|---------|-----|
| Ahead-of-time class loading & linking | JEP 483 |
| Compact object headers (experimental) | JEP 450 |
| Quantum-resistant key encapsulation (`KEM`) | JEP 496 |
| Quantum-resistant digital signatures (`ML-DSA`) | JEP 497 |
| Scoped values (standard) | JEP 487 |
| Structured concurrency (standard) | JEP 499 |
| Stream gatherers (standard) | JEP 485 |
| Primitive types in patterns (2nd preview) | JEP 488 |
| `Class-File` API (standard) | JEP 484 |
| Remove old weak GC root behavior | JEP 491 |

```java
// ── Stream Gatherers (standard) ───────────────────────────────────────────────
// Custom intermediate stream operations — the missing piece after Collectors
import java.util.stream.Gatherers;

// sliding window
List<List<Integer>> windows = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.windowSliding(3))
    .toList();
// [[1,2,3], [2,3,4], [3,4,5]]

// fixed window (non-overlapping)
List<List<Integer>> chunks = Stream.of(1, 2, 3, 4, 5, 6)
    .gather(Gatherers.windowFixed(2))
    .toList();
// [[1,2], [3,4], [5,6]]

// scan — running accumulation
List<Integer> running = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.scan(() -> 0, Integer::sum))
    .toList();
// [1, 3, 6, 10, 15]

// fold (terminal-like but as gatherer)
Optional<Integer> total = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.fold(() -> 0, Integer::sum))
    .findFirst();
// Optional[15]

// Custom gatherer — take until sum exceeds a limit
Gatherer<Integer, ?, Integer> takeUntilSumExceeds(int limit) {
    return Gatherer.ofSequential(
        () -> new int[]{0},          // mutable state: running sum
        (state, element, downstream) -> {
            state[0] += element;
            if (state[0] > limit) return false;  // signal stop
            return downstream.push(element);
        }
    );
}

List<Integer> capped = Stream.of(1, 2, 3, 4, 5, 6)
    .gather(takeUntilSumExceeds(10))
    .toList();
// [1, 2, 3, 4]  (sum=10; adding 5 would exceed 10)

// ── Scoped Values (standard) ──────────────────────────────────────────────────
static final ScopedValue<RequestContext> REQUEST_CTX = ScopedValue.newInstance();

// Bind per-request context — safe with virtual threads (unlike ThreadLocal)
ScopedValue.runWhere(REQUEST_CTX, new RequestContext(correlationId, userId), () -> {
    paymentService.process(request);
    auditService.log();               // reads REQUEST_CTX.get() internally
});

// ── Structured Concurrency (standard) ────────────────────────────────────────
import java.util.concurrent.StructuredTaskScope;

record DashboardData(User user, List<Payment> payments, List<Notification> notifications) {}

DashboardData loadDashboard(String userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var userTask    = scope.fork(() -> userService.findById(userId));
        var payTask     = scope.fork(() -> paymentService.findByUser(userId));
        var notifTask   = scope.fork(() -> notifService.findByUser(userId));

        scope.join().throwIfFailed();

        return new DashboardData(userTask.get(), payTask.get(), notifTask.get());
    }
    // All forked tasks are cancelled if any one fails — clean lifecycle
}

// ── Quantum-Resistant Cryptography ───────────────────────────────────────────
import javax.crypto.KEM;
import java.security.KeyPairGenerator;

// ML-KEM (Module-Lattice Key Encapsulation Mechanism) — JEP 496
var kpg = KeyPairGenerator.getInstance("ML-KEM-768");
kpg.initialize(768);
var keyPair = kpg.generateKeyPair();

var kem = KEM.getInstance("ML-KEM-768");

// Sender: encapsulate a shared secret for the recipient's public key
var encapsulator = kem.newEncapsulator(keyPair.getPublic());
var encapsulated = encapsulator.encapsulate();
byte[] sharedSecretSender = encapsulated.key().getEncoded();
byte[] ciphertext         = encapsulated.encapsulation();

// Recipient: decapsulate using private key
var decapsulator = kem.newDecapsulator(keyPair.getPrivate());
byte[] sharedSecretRecipient = decapsulator.decapsulate(ciphertext).getEncoded();
// sharedSecretSender == sharedSecretRecipient — cryptographically secure key exchange

// ML-DSA (Module-Lattice Digital Signature Algorithm) — JEP 497
import java.security.*;
var signer = KeyPairGenerator.getInstance("ML-DSA-65");
var sigKeyPair = signer.generateKeyPair();

var sig = Signature.getInstance("ML-DSA-65");
sig.initSign(sigKeyPair.getPrivate());
sig.update("Payment payload".getBytes());
byte[] signature = sig.sign();

sig.initVerify(sigKeyPair.getPublic());
sig.update("Payment payload".getBytes());
boolean valid = sig.verify(signature);   // true
```

---

## Quick Feature Reference

| Feature | Introduced | Standard |
|---------|-----------|---------|
| Lambda / Stream API | Java 8 | Java 8 |
| `Optional` | Java 8 | Java 8 |
| `var` (local inference) | Java 10 | Java 10 |
| HTTP Client | Java 9 (incub) | Java 11 |
| Text blocks | Java 13 (prev) | Java 15 |
| Records | Java 14 (prev) | Java 16 |
| `instanceof` pattern matching | Java 14 (prev) | Java 16 |
| Sealed classes | Java 15 (prev) | Java 17 |
| `switch` pattern matching | Java 17 (prev) | Java 21 |
| Record patterns | Java 19 (prev) | Java 21 |
| Virtual threads | Java 19 (prev) | Java 21 |
| Sequenced collections | Java 21 | Java 21 |
| Structured concurrency | Java 19 (incub) | Java 24 |
| Scoped values | Java 20 (incub) | Java 24 |
| Unnamed variables (`_`) | Java 22 (prev) | Java 22 |
| Stream gatherers | Java 22 (prev) | Java 24 |
| Foreign Function & Memory API | Java 14 (incub) | Java 22 |
| Quantum-resistant crypto (ML-KEM, ML-DSA) | — | Java 24 |

## References

- [OpenJDK JEP Index](https://openjdk.org/jeps/0)
- [Java Version History — Wikipedia](https://en.wikipedia.org/wiki/Java_version_history)
- [Inside Java — Oracle Developer Blog](https://inside.java/)
- [Java Almanac — java-almanac.ch](https://javaalmanac.io/)
- [Nicolai Parlog — Java Module System in Action](https://www.manning.com/books/the-java-module-system)
- [José Paumard — Java Stream API](https://dev.java/learn/api/streams/)
