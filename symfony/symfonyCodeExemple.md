Parfait. Voici un **pack d’exemples PHP/Symfony** “vite utiles à connaître” en entretien et au quotidien. Courts, idiomatiques, prêts à copier-coller.

---

# 1) Service + Injection de dépendances

```php
// src/Service/StockAvailabilityService.php
namespace App\Service;

use Psr\Log\LoggerInterface;

final class StockAvailabilityService
{
    public function __construct(private LoggerInterface $logger) {}

    public function isAvailable(int $productId, int $qty): bool
    {
        // ... logique métier
        $available = $qty <= 10;
        $this->logger->info('Stock check', compact('productId','qty','available'));
        return $available;
    }
}
```

```yaml
# config/services.yaml
services:
  App\Service\StockAvailabilityService: ~  # autowire/ autoconfigure
```

---

# 2) Strategy (plusieurs algos interchangeables)

```php
// src/Price/PriceStrategyInterface.php
namespace App\Price;
interface PriceStrategyInterface { public function compute(float $base): float; }

// src/Price/NormalPriceStrategy.php
final class NormalPriceStrategy implements PriceStrategyInterface {
    public function compute(float $base): float { return $base; }
}

// src/Price/SalePriceStrategy.php
final class SalePriceStrategy implements PriceStrategyInterface {
    public function compute(float $base): float { return $base * 0.9; }
}

// src/Service/PriceService.php
namespace App\Service;
use App\Price\PriceStrategyInterface;

final class PriceService {
    public function __construct(private PriceStrategyInterface $strategy) {}
    public function finalPrice(float $base): float { return $this->strategy->compute($base); }
}
```

```yaml
# services.yaml – choix par alias/env/tag
services:
  App\Price\PriceStrategyInterface: '@App\Price\SalePriceStrategy'
```

---

# 3) Repository Doctrine (lecture ciblée + pagination)

```php
// src/Repository/OrderRepository.php
namespace App\Repository;

use App\Entity\Order;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

final class OrderRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $r) { parent::__construct($r, Order::class); }

    /** @return Order[] */
    public function search(string $term, int $page=1, int $limit=20): array
    {
        return $this->createQueryBuilder('o')
            ->andWhere('o.reference LIKE :q')->setParameter('q', "%$term%")
            ->orderBy('o.createdAt','DESC')
            ->setFirstResult(($page-1)*$limit)
            ->setMaxResults($limit)
            ->getQuery()->getResult();
    }
}
```

---

# 4) Transaction + verrou “pessimiste”

```php
// Dans un service
$em = $this->getEntityManager();
$em->wrapInTransaction(function($em) use ($orderId) {
    $order = $em->getRepository(Order::class)
        ->createQueryBuilder('o')
        ->andWhere('o.id = :id')->setParameter('id',$orderId)
        ->getQuery()->setLockMode(\Doctrine\DBAL\LockMode::PESSIMISTIC_WRITE)
        ->getSingleResult();

    $order->confirm(); // modifs
});
```

---

# 5) Events Symfony (Observer)

```php
// src/Event/UserRegistered.php
namespace App\Event;
use App\Entity\User;
final class UserRegistered { public function __construct(public User $user) {} }

// src/EventSubscriber/SendWelcomeEmailSubscriber.php
namespace App\EventSubscriber;
use App\Event\UserRegistered;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class SendWelcomeEmailSubscriber implements EventSubscriberInterface {
    public static function getSubscribedEvents(): array { return [UserRegistered::class => 'onRegistered']; }
    public function onRegistered(UserRegistered $event): void {
        // envoyer email…
    }
}
```

---

# 6) Middleware / Kernel Event (Chain of Responsibility)

```php
// src/EventSubscriber/RequestIdSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class RequestIdSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array { return [KernelEvents::REQUEST => 'onRequest']; }

    public function onRequest(RequestEvent $event): void
    {
        $request = $event->getRequest();
        $request->attributes->set('req_id', bin2hex(random_bytes(6)));
    }
}
```

---

# 7) Command Bus simple (Command + Handler)

```php
// src/Application/Command/CreateBooking.php
namespace App\Application\Command;
final class CreateBooking { public function __construct(public string $email, public int $pax) {} }

// src/Application/Handler/CreateBookingHandler.php
namespace App\Application\Handler;
use App\Application\Command\CreateBooking;

final class CreateBookingHandler {
    public function __invoke(CreateBooking $cmd): void {
        // persister la réservation…
    }
}
```

---

# 8) DTO + Validation

```php
// src/DTO/BookingRequest.php
namespace App\DTO;

use Symfony\Component\Validator\Constraints as Assert;

final class BookingRequest
{
    #[Assert\NotBlank, Assert\Email] public string $email;
    #[Assert\Positive] public int $pax;

    public function __construct(string $email, int $pax) { $this->email=$email; $this->pax=$pax; }
}
```

Validation dans un contrôleur :

```php
$dto = new BookingRequest($email, $pax);
$errors = $validator->validate($dto);
if (count($errors) > 0) { /* return 400 avec messages */ }
```

---

# 9) Value Object (immutabilité + règles)

```php
// src/Domain/Email.php
namespace App\Domain;

final class Email
{
    public function __construct(private string $value) {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException('Invalid email');
        }
    }
    public function value(): string { return $this->value; }
    public function __toString(): string { return $this->value; }
}
```

---

# 10) HttpClient + Retry (backoff simple)

```php
// src/Service/ApiClient.php
namespace App\Service;

use Symfony\Contracts\HttpClient\HttpClientInterface;

final class ApiClient
{
    public function __construct(private HttpClientInterface $http) {}

    public function get(string $url): array
    {
        $delay = 200; // ms
        for ($i=0; $i<3; $i++) {
            $res = $this->http->request('GET', $url);
            $code = $res->getStatusCode();
            if ($code < 500 && $code !== 429) {
                return $res->toArray(false);
            }
            usleep($delay * 1000);
            $delay *= 2; // backoff exponentiel
        }
        throw new \RuntimeException("API unavailable");
    }
}
```

---

# 11) Cache rapide (PSR-6) autour d’un service

```php
// src/Service/CachedCatalog.php
namespace App\Service;

use Psr\Cache\CacheItemPoolInterface;

final class CachedCatalog
{
    public function __construct(private CacheItemPoolInterface $cache, private Catalog $catalog) {}

    public function list(): array
    {
        $item = $this->cache->getItem('catalog:list');
        if ($item->isHit()) return $item->get();

        $data = $this->catalog->list();
        $item->set($data)->expiresAfter(3600);
        $this->cache->save($item);
        return $data;
    }
}
```

---

# 12) Décorateur de service (logs/metrics)

```php
// src/Mailer/LoggingMailer.php
namespace App\Mailer;

class LoggingMailer implements MailerInterface
{
    public function __construct(private MailerInterface $inner, private \Psr\Log\LoggerInterface $logger) {}

    public function send(Email $email): void
    {
        $this->logger->info('Sending email', ['to' => $email->getTo()]);
        $this->inner->send($email);
    }
}
```

```yaml
# services.yaml
services:
  App\Mailer\MailerInterface: '@App\Mailer\SmtpMailer'

  App\Mailer\LoggingMailer:
    decorates: App\Mailer\MailerInterface
    arguments: ['@App\Mailer\LoggingMailer.inner', '@logger']
```

---

# 13) Console Command (squelette propre)

```php
// src/Command/ReindexSearchCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(name: 'app:search:reindex', description: 'Reindex search data')]
final class ReindexSearchCommand extends Command
{
    protected function execute(InputInterface $in, OutputInterface $out): int
    {
        // … reindex logic
        $out->writeln('<info>Done.</info>');
        return Command::SUCCESS;
    }
}
```

---

# 14) Test unitaire + DataProvider + Mock

```php
// tests/Service/PriceServiceTest.php
namespace App\Tests\Service;

use App\Price\NormalPriceStrategy;
use App\Service\PriceService;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

final class PriceServiceTest extends TestCase
{
    public static function prices(): array
    {
        return [[100.0, 100.0], [0.0, 0.0], [19.99, 19.99]];
    }

    #[DataProvider('prices')]
    public function testFinalPrice(float $base, float $expected): void
    {
        $service = new PriceService(new NormalPriceStrategy());
        self::assertSame($expected, $service->finalPrice($base));
    }
}
```

---

# 15) Contrôleur API propre (validation + DTO + réponse JSON)

```php
// src/Controller/Api/BookingController.php
namespace App\Controller\Api;

use App\DTO\BookingRequest;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Validator\Validator\ValidatorInterface;

final class BookingController extends AbstractController
{
    #[Route('/api/bookings', name: 'api_bookings_create', methods: ['POST'])]
    public function create(Request $req, ValidatorInterface $validator): JsonResponse
    {
        $data = json_decode($req->getContent(), true, 512, JSON_THROW_ON_ERROR);
        $dto  = new BookingRequest($data['email'] ?? '', (int)($data['pax'] ?? 0));

        $errors = $validator->validate($dto);
        if (count($errors) > 0) {
            return $this->json(['errors' => (string) $errors], 422);
        }

        // … créer la réservation
        return $this->json(['status' => 'ok'], 201);
    }
}
```

---

# 16) Filtre/Pagination “propre” (tri whitelist)

```php
$allowedSort = ['createdAt','reference'];
$sort = in_array($req->query->get('sort'), $allowedSort, true) ? $req->query->get('sort') : 'createdAt';
$order= $req->query->get('order') === 'asc' ? 'ASC' : 'DESC';
$page = max(1, (int)$req->query->get('page', 1));
$limit= min(100, max(1, (int)$req->query->get('limit', 20)));
```

---

# 17) Regex robuste (ex : `num-md5(num."lgbop")-int`)

```php
// ^([1-9]\d{5})-([a-f0-9]{32})-(-?\d+)$  (i pour insensible casse)
$re = '/^([1-9]\d{5})-([a-f0-9]{32})-(-?\d+)$/i';
```

---

# 18) Sécurité rapide (Voter)

```php
// src/Security/Voter/OrderVoter.php
namespace App\Security\Voter;

use App\Entity\Order;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

final class OrderVoter extends Voter
{
    public const VIEW = 'ORDER_VIEW';

    protected function supports(string $attr, mixed $subj): bool
    { return $attr === self::VIEW && $subj instanceof Order; }

    protected function voteOnAttribute(string $attr, mixed $order, TokenInterface $token): bool
    {
        $user = $token->getUser();
        return $order->getCustomer() === $user || in_array('ROLE_ADMIN', $user->getRoles(), true);
    }
}
```

---


