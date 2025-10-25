# 🐘 PHP Advanced Cheat Sheet (Senior Developer Edition)

> **Target version:** PHP 8.3+
> This cheat sheet summarizes **advanced PHP concepts** for senior developers: clean code, performance, design patterns, modern syntax, and professional tooling.

---

## ⚙️ Modern Language Features

### 🧠 Strict typing

```php
declare(strict_types=1);
```

Forces type validation at runtime — prevents implicit coercion (e.g., `"3"` ≠ `3`).

---

### 🧩 Union & Nullable Types

```php
function process(int|float $value): string {}
function maybe(?string $name): ?int {}
```

Combines multiple types (`int|float`) or allows `null`.

---

### 🚀 Constructor Property Promotion

```php
class User {
    public function __construct(
        public readonly string $id,
        private string $name
    ) {}
}
```

Defines and initializes properties directly in the constructor.

---

### ⚡ Match Expression

```php
$status = match($code) {
    200, 201 => 'Success',
    404 => 'Not Found',
    default => 'Error',
};
```

Expression-based alternative to `switch`, returning values safely.

---

### 🧱 Enumerations

```php
enum Status: string {
    case Draft = 'draft';
    case Published = 'published';
}
```

Enums replace constants with type-safe, self-documenting structures.

---

### 🧬 Attributes

```php
#[ORM\Entity]
class Post {
    #[ORM\Column(length: 255)]
    public string $title;
}
```

Native metadata annotations used by frameworks (e.g., Doctrine, Symfony).

---

### 🧰 Named Arguments

```php
sendEmail(to: 'admin@example.com', subject: 'Hi', body: 'Hello!');
```

Improves readability and avoids argument order issues.

---

### 🧩 First-Class Callables

```php
function greet($name) { echo "Hi $name"; }
$fn = greet(...);
$fn('Lionel'); // Hi Lionel
```

Turns any function into a callable reference.

---

## 🧠 OOP & Design Patterns

### 🧱 Immutability

```php
final class Money {
    public function __construct(
        public readonly float $amount,
        public readonly string $currency
    ) {}
}
```

Objects are state-safe once created — improves testability and predictability.

---

### 🧩 Value Object

```php
final class Email {
    public function __construct(private string $value) {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL))
            throw new InvalidArgumentException('Invalid email');
    }
    public function __toString(): string { return $this->value; }
}
```

Encapsulates validation and domain logic inside immutable objects.

---

### 🏭 Factory Method

```php
final class User {
    private function __construct(private string $name) {}
    public static function fromArray(array $data): self {
        return new self($data['name']);
    }
}
```

Creates controlled object instances instead of using `new`.

---

### 🧱 Repository Pattern

```php
interface UserRepository {
    public function findByEmail(string $email): ?User;
}

final class DoctrineUserRepository implements UserRepository {
    public function __construct(private EntityManager $em) {}
    public function findByEmail(string $email): ?User {
        return $this->em->getRepository(User::class)->findOneBy(['email' => $email]);
    }
}
```

Separates persistence logic from the domain.

---

### ⚙️ Strategy Pattern

```php
interface PaymentStrategy { public function pay(float $amount): void; }

final class Paypal implements PaymentStrategy { public function pay(float $a): void {} }
final class Stripe implements PaymentStrategy { public function pay(float $a): void {} }

final class PaymentProcessor {
    public function __construct(private PaymentStrategy $strategy) {}
    public function process(float $a): void { $this->strategy->pay($a); }
}
```

Allows swapping algorithms dynamically.

---

## ⚡ Performance & Memory

### 🔥 Opcache

```ini
opcache.enable=1
opcache.validate_timestamps=0
opcache.memory_consumption=128
```

Caches compiled bytecode → reduces runtime parsing.

---

### 🧩 JIT (Just-In-Time)

```ini
opcache.jit_buffer_size=128M
opcache.jit=1255
```

Accelerates CPU-heavy workloads (math, loops).

---

### ⚙️ Micro-optimizations

* Use `isset()` instead of `array_key_exists()` for faster lookups.
* Avoid `count()` inside loops.
* Use `foreach` over arrays; prefer `$array[] = $x` over `array_push()`.
* Prefer `SplFixedArray` or generators for large datasets.

---

## 🧾 Error Handling

### 🧱 Try/Catch with Multi-exception

```php
try {
    run();
} catch (InvalidArgumentException|RuntimeException $e) {
    log($e->getMessage());
} finally {
    cleanup();
}
```

### 💡 Custom Exceptions

```php
class DomainException extends RuntimeException {}
throw new DomainException('Invalid operation');
```

---

## 🧮 Generators & Iterators

### 🔁 Generator (lazy iteration)

```php
function rangeGen(int $max): Generator {
    for ($i = 0; $i < $max; $i++) yield $i;
}
foreach (rangeGen(3) as $i) echo $i; // 0 1 2
```

Efficient memory usage — processes data streams without full allocation.

---

### 🧱 SPL Structures

```php
$stack = new SplStack();
$stack->push('A');
$stack->push('B');
echo $stack->pop(); // B
```

PHP’s built-in data structures: `SplStack`, `SplQueue`, `SplHeap`, etc.

---

## 🧩 Reflection & Meta Programming

### 🔍 Reflection

```php
$ref = new ReflectionClass(App\Service\UserService::class);
foreach ($ref->getMethods() as $m) echo $m->getName();
```

Used for autowiring, serialization, and testing internals.

### 🔐 Access private methods

```php
$m = $ref->getMethod('privateMethod');
$m->setAccessible(true);
$m->invoke($instance);
```

---

## 🔐 Security Best Practices

### Password Hashing

```php
$hash = password_hash('secret', PASSWORD_ARGON2ID);
password_verify('secret', $hash);
```

### Input Sanitization

```php
$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);
```

### Encryption

```php
$key = random_bytes(SODIUM_CRYPTO_SECRETBOX_KEYBYTES);
$nonce = random_bytes(SODIUM_CRYPTO_SECRETBOX_NONCEBYTES);
$cipher = sodium_crypto_secretbox("message", $nonce, $key);
```

---

## 🧰 JSON, I/O & Streams

### JSON

```php
$data = ['user' => 'Lionel'];
$json = json_encode($data, JSON_PRETTY_PRINT);
$array = json_decode($json, true);
```

### Streams

```php
$ctx = stream_context_create(['http' => ['timeout' => 3]]);
$content = file_get_contents('https://api.example.com', false, $ctx);
```

---

## 🧩 Namespaces & Autoloading

### Namespace

```php
namespace App\Service;
class Mailer {}
```

### Composer Autoload

```json
"autoload": { "psr-4": { "App\\": "src/" } }
```

---

## 🧪 Testing & Quality Tools

### PHPUnit

```php
use PHPUnit\Framework\TestCase;

final class UserTest extends TestCase {
    public function testName(): void {
        $u = new User('John');
        self::assertSame('John', $u->getName());
    }
}
```

### PHPStan / Psalm (Static Analysis)

```bash
vendor/bin/phpstan analyse src --level=max
vendor/bin/psalm
```

### CodeSniffer

```bash
vendor/bin/phpcs --standard=PSR12 src/
```

---

## 🧠 Advanced Topics

### Async & Parallel PHP

* Use **Fibers** for cooperative concurrency.
* Use **Amp** or **ReactPHP** for event-driven I/O.
* Use **pthreads** or **parallel** for true multithreading.

```php
$fiber = new Fiber(fn() => yield 'pause');
```

---

### Dependency Injection

```php
interface Logger { public function log(string $msg): void; }

final class FileLogger implements Logger {
    public function log(string $msg): void { file_put_contents('log.txt', $msg); }
}

final class App {
    public function __construct(private Logger $logger) {}
}
```

---

### Middleware Example

```php
$middleware = fn($req, $next) => tap($next($req), fn() => log('request done'));
```

Functional-style request interceptors (common in HTTP frameworks).

---

## 🧱 Common Architecture (DDD-style)

```
src/
├─ Domain/
│   ├─ Model/
│   ├─ ValueObject/
│   ├─ Event/
│   └─ Service/
├─ Application/
│   ├─ Command/
│   ├─ Handler/
│   └─ DTO/
└─ Infrastructure/
    ├─ Repository/
    ├─ Http/
    └─ Persistence/
```

* **Domain** → pure logic
* **Application** → orchestration
* **Infrastructure** → technical glue

---

## 🧩 Recommended Patterns for Senior Devs

| Pattern            | Use Case                         |
| ------------------ | -------------------------------- |
| **CQRS**           | Separate read/write logic        |
| **Event Sourcing** | Track state changes              |
| **Strategy**       | Runtime behavior switching       |
| **Decorator**      | Extend class without inheritance |
| **Observer**       | Event-driven architecture        |
| **Factory**        | Controlled object creation       |
| **Repository**     | Persistence abstraction          |
| **Value Object**   | Encapsulate domain data          |

---

## ⚙️ Tooling for Professionals

| Tool                | Purpose               |
| ------------------- | --------------------- |
| **Xdebug**          | Debugger & profiler   |
| **Blackfire**       | Production profiling  |
| **PHPStan / Psalm** | Static analysis       |
| **Infection**       | Mutation testing      |
| **PHP-CS-Fixer**    | Code formatting       |
| **Composer**        | Dependency management |

---

## ✅ Senior-Level Guidelines

✔ Always enable `declare(strict_types=1);`
✔ Use **typed properties** and **immutability**
✔ Follow **PSR-12**, **SOLID**, and **DDD** principles
✔ Prefer **composition over inheritance**
✔ No business logic in controllers
✔ Write **tests for every critical path**
✔ Profile performance before optimizing

---

