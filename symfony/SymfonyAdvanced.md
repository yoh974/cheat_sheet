# ⚙️ Symfony – Cheat Sheet Avancé (Développeur Sénior)

> **Version ciblée : Symfony 6.4+ LTS**
> Ce guide condense les concepts, patterns et outils avancés utilisés dans les projets complexes et scalables.
> Idéal pour architectes, tech leads ou développeurs confirmés.

---

## 🧩 Architecture & Structure

### 🔸 Principes fondamentaux

* **DIC (Dependency Injection Container)** : tout est un service. Favorise l’autowiring, le typage strict et la testabilité.
* **Convention over configuration** : tout est autoconfiguré via attributs.
* **Découplage fort** : préférer interfaces, contrats, et event dispatchers aux appels directs.
* **Domain Driven Design (DDD)** compatible :

  ```
  src/
   ├─ Domain/
   │   ├─ Model/
   │   ├─ Service/
   │   └─ Repository/
   ├─ Application/
   │   ├─ Command/
   │   ├─ Query/
   │   ├─ DTO/
   │   └─ Handler/
   └─ Infrastructure/
       ├─ Persistence/
       └─ Http/
  ```

---

## ⚙️ Services & Injection

### 🧠 Advanced Autowiring

```yaml
services:
  _defaults:
    autowire: true
    autoconfigure: true
    bind:
      $env: '%kernel.environment%' # injection d’un paramètre global

  App\:
    resource: '../src/*'
    exclude: '../src/{DependencyInjection,Entity,Tests,Kernel.php}'
```

### 💡 Service alias & decoration

```yaml
services:
  # Remplacer un service existant
  App\Http\DecoratedClient:
    decorates: 'http_client'
    arguments:
      - '@.inner'
```

### 💾 Lazy services

* Les services marqués `lazy: true` ne sont instanciés qu’à la première utilisation.
* Exemple :

  ```yaml
  App\Service\HeavyService:
    lazy: true
  ```

---

## 🧠 ParamConverter & Attributes Custom

### Exemple : Validation custom d’un token

```php
#[Attribute(Attribute::TARGET_PARAMETER)]
final class BookingToken {}

#[ValueResolver]
final class BookingTokenResolver implements ValueResolverInterface {
    public function resolve(Request $request, ArgumentMetadata $metadata): iterable {
        yield $request->query->get('bookingtoken');
    }
}
```

---

## 🗺️ Routing avancé

### Préfixes & Groupes

```php
#[Route('/admin', name: 'admin_')]
final class AdminController
{
    #[Route('/dashboard', name: 'dashboard')]
    public function dashboard() { /* ... */ }
}
```

### Requirements dynamiques

```php
#[Route('/user/{id<\d+>}', name: 'user_show')]
```

### Redirections & SubRequest

```php
$sub = $this->forward('App\Controller\APIController::index', ['id' => 42]);
```

---

## 🧩 Doctrine Avancé

### Repositories custom

```php
#[ORM\Entity(repositoryClass: PostRepository::class)]
final class Post {}

final class PostRepository extends ServiceEntityRepository
{
    public function findRecent(int $days): array {
        return $this->createQueryBuilder('p')
            ->where('p.createdAt > :d')
            ->setParameter('d', new \DateTime("-$days days"))
            ->getQuery()
            ->getResult();
    }
}
```

### QueryHints / Caching

```php
$query = $em->createQuery('SELECT u FROM App:User u');
$query->setHint(Query::HINT_FORCE_PARTIAL_LOAD, true);
```

### DQL dans les repositories

```php
return $this->_em->createQuery(
  'SELECT COUNT(u.id) FROM App\Entity\User u WHERE u.active = true'
)->getSingleScalarResult();
```

### Lifecycle callbacks

```php
#[ORM\PrePersist]
public function setCreatedAt(): void { $this->createdAt = new \DateTimeImmutable(); }
```

---

## 🔄 Événements & Subscribers

### EventSubscriber global

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Doctrine\ORM\Events;
use Doctrine\ORM\Event\PrePersistEventArgs;

final class AuditSubscriber implements EventSubscriberInterface
{
    public function getSubscribedEvents(): array {
        return [Events::prePersist];
    }

    public function prePersist(PrePersistEventArgs $args): void {
        $entity = $args->getObject();
        if (method_exists($entity, 'touch')) $entity->touch();
    }
}
```

### EventListener custom

```php
#[AsEventListener(event: KernelEvents::REQUEST, priority: 10)]
public function onRequest(RequestEvent $e): void { /* ... */ }
```

---

## 📬 Messenger & CQRS

### Message + Handler + Bus

```php
final class CreateOrderCommand {
    public function __construct(public string $orderId) {}
}

#[AsMessageHandler(bus: 'command.bus')]
final class CreateOrderHandler {
    public function __invoke(CreateOrderCommand $cmd): void {
        // logique applicative
    }
}
```

### Buses multiples

```yaml
framework:
  messenger:
    default_bus: command.bus
    buses:
      command.bus: ~
      event.bus: ~
```

### Transport async

```yaml
framework:
  messenger:
    transports:
      async: '%env(MESSENGER_TRANSPORT_DSN)%'
    routing:
      'App\Message\*': async
```

---

## 🧰 HttpClient avancé

### Middleware personnalisé

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;
use Symfony\Component\HttpClient\Response\TraceableResponse;

final class LoggedHttpClient
{
    public function __construct(private HttpClientInterface $client) {}

    public function request(string $method, string $url, array $options = []): TraceableResponse
    {
        $start = microtime(true);
        $response = $this->client->request($method, $url, $options);
        $duration = round(microtime(true) - $start, 3);
        dump("Request to $url in {$duration}s");
        return $response;
    }
}
```

---

## 🧩 Serializer & DTOs

### Normalizer & Groupes

```php
use Symfony\Component\Serializer\Annotation\Groups;

final class UserDTO
{
    #[Groups(['read'])]
    public string $name;

    #[Groups(['write'])]
    public string $password;
}
```

### Config service

```yaml
serializer:
  mapping:
    paths: ['%kernel.project_dir%/config/serialization']
```

---

## 🧪 Tests avancés

### Tests fonctionnels

```php
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class BookingTest extends WebTestCase
{
    public function testBookingContext(): void
    {
        $client = static::createClient();
        $client->request('GET', '/context/booking/12345-abcdef...');
        self::assertResponseStatusCodeSame(200);
        self::assertStringContainsString('booking', $client->getResponse()->getContent());
    }
}
```

### Tests unitaires avec Mocks

```php
$logger = $this->createMock(LoggerInterface::class);
$logger->expects($this->once())->method('info');
```

### Data Providers

```php
#[DataProvider('provideTokens')]
public function testTokenValid(string $token): void {}

public static function provideTokens(): iterable {
    yield ['123456-abcdef1234567890abcdef1234567890-lgbop-42'];
}
```

---

## 🧱 Cache & Performance

### Cache PSR-6 / PSR-16

```php
use Psr\Cache\CacheItemPoolInterface;
public function __construct(private CacheItemPoolInterface $cache) {}

$item = $this->cache->getItem('key');
if (!$item->isHit()) {
  $item->set($data);
  $this->cache->save($item);
}
```

### HTTP Cache (reverse proxy)

* Activer :

  ```bash
  composer require symfony/http-cache
  ```
* Dans le contrôleur :

  ```php
  $response->setPublic();
  $response->setMaxAge(3600);
  $response->setEtag(md5($content));
  ```

---

## 🧩 Security & Auth avancé

### Custom Authenticator

```php
final class ApiTokenAuthenticator extends AbstractAuthenticator
{
    public function authenticate(Request $request): Passport {
        $token = $request->headers->get('X-API-TOKEN');
        return new SelfValidatingPassport(new UserBadge($token));
    }

    public function onAuthenticationFailure(Request $r, AuthenticationException $e): ?Response {
        return new JsonResponse(['error' => 'Unauthorized'], 401);
    }
}
```

### Access Control avancé

```yaml
security:
  access_control:
    - { path: ^/admin, roles: ROLE_ADMIN, requires_channel: https }
```

### Voter

```php
final class BookingVoter extends Voter
{
    protected function supports(string $attribute, $subject): bool
    {
        return $attribute === 'BOOKING_EDIT' && $subject instanceof Booking;
    }
    protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
    {
        return $subject->getOwner() === $token->getUser();
    }
}
```

---

## 🧾 Commandes & Console avancées

### Custom Command

```php
#[AsCommand(name: 'app:sync:orders', description: 'Sync orders from API')]
final class SyncOrdersCommand extends Command
{
    protected function execute(InputInterface $i, OutputInterface $o): int {
        $o->writeln('<info>Syncing...</info>');
        return Command::SUCCESS;
    }
}
```

### Scheduled commands (CRON)

```bash
* * * * * php /app/bin/console app:sync:orders --env=prod >> /var/log/sync.log 2>&1
```

---

## 🧰 Monitoring, Logging, Profiling

### Monolog channels

```yaml
monolog:
  channels: ['booking', 'api']
  handlers:
    booking:
      type: stream
      path: '%kernel.logs_dir%/booking.log'
      channels: ['booking']
```

### Profiler custom data collector

```php
#[AsDataCollector(name: 'app.stats')]
final class StatsCollector extends DataCollector {
    public function collect(Request $r, Response $res, \Throwable $e = null): void {
        $this->data['mem'] = memory_get_usage();
    }
}
```

---

## ⚡ API Platform intégration

```bash
composer require api
```

### Ressource

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;

#[ApiResource(operations: [new Get()])]
final class Product
{
    public int $id;
    public string $name;
}
```

### Filtrage, Pagination, Sécurité

```php
#[ApiResource(
  security: "is_granted('ROLE_USER')",
  paginationItemsPerPage: 20
)]
```

---

## 🧠 Bundles utiles (production-level)

| Bundle                            | Usage                         |
| --------------------------------- | ----------------------------- |
| `symfony/mercure-bundle`          | Live updates temps réel       |
| `symfony/ux-turbo`                | Rechargement partiel frontend |
| `friendsofphp/proxy-manager-lts`  | Lazy services                 |
| `doctrine/doctrine-bundle`        | ORM                           |
| `nelmio/cors-bundle`              | API CORS                      |
| `lexik/jwt-authentication-bundle` | Auth API Token                |
| `vich/uploader-bundle`            | Uploads & mapping fichiers    |

---

## 🧱 Patterns conseillés

* **DTO + Handler** pour séparer le domaine et le framework
* **Command / Query pattern** avec Messenger
* **Event sourcing** (Domain events via Messenger)
* **Factory & ValueObject** pour immuabilité
* **Service Layer** : éviter la logique métier dans les contrôleurs

---

## 🧩 Rappels pour sénior

* Toujours **typer** tes propriétés & méthodes (`readonly`, `mixed` interdit).
* Favoriser **immutabilité** & **Value Objects** (`Email`, `Money`, etc.).
* Structurer le projet autour du **domaine**, pas du framework.
* Utiliser **Event Subscribers** plutôt que des hooks implicites.
* **Tests intégration** > unitaires simples sur projet métier.
* **CI/CD** : inclure `lint:container`, `lint:twig`, `phpstan`, `phpunit`.

---

