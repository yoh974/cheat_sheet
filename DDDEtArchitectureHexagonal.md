Parfait 👌 on passe dans le **haut du panier de l’architecture logicielle** — ce que tout bon développeur senior ou tech lead doit maîtriser :
➡ **DDD (Domain-Driven Design)**
➡ **Architecture hexagonale** (ou **Ports & Adapters**).

Ces deux approches sont souvent complémentaires et constituent le cœur des architectures modernes en PHP/Symfony, surtout dans des applications métiers complexes (réservations, facturation, logistique, etc.).

---

# 🧠 1. DDD — *Domain-Driven Design*

## 🎯 Objectif

Organiser ton code autour du **métier (le “domaine”)**, pas autour de la technologie.

> Le code reflète le vocabulaire et la logique des experts métier, pas celle de la base de données ni du framework.

---

## 🧩 Les blocs fondamentaux

| Concept            | Rôle                                                    | Exemple                            |
| ------------------ | ------------------------------------------------------- | ---------------------------------- |
| **Domain**         | Cœur métier (entités, règles, invariants).              | `Booking`, `Invoice`, `Customer`   |
| **Entity**         | Objet identifié de manière unique (par ID).             | `Booking#1234`                     |
| **Value Object**   | Objet sans identité, défini uniquement par ses valeurs. | `Email`, `Money`, `DateRange`      |
| **Aggregate**      | Regroupe entités + règles internes + racine d’accès.    | `Order` + `OrderLine`              |
| **Repository**     | Accès aux agrégats comme s’ils étaient en mémoire.      | `OrderRepository->save($order)`    |
| **Domain Event**   | Fait métier significatif (“booking confirmed”).         | `BookingConfirmed`                 |
| **Service métier** | Opération métier ne dépendant pas d’un objet précis.    | `PaymentProcessor`, `StockChecker` |

---

## 🗂️ Organisation typique

```
src/
 ├── Domain/
 │   ├── Booking/
 │   │   ├── Booking.php              (Entity)
 │   │   ├── BookingId.php            (Value Object)
 │   │   ├── BookingRepository.php    (Interface)
 │   │   └── Event/
 │   │       └── BookingCreated.php   (Domain Event)
 │   └── Service/
 │       └── BookingValidator.php
 ├── Application/
 │   ├── Command/
 │   │   └── CreateBooking.php
 │   ├── Handler/
 │   │   └── CreateBookingHandler.php
 │   └── DTO/
 ├── Infrastructure/
 │   ├── Persistence/
 │   │   └── DoctrineBookingRepository.php
 │   ├── Http/
 │   └── Mail/
 └── UI/
     └── Controller/
         └── BookingController.php
```

---

## 🧩 Exemple simplifié

```php
// Domain/Booking/Booking.php
final class Booking
{
    private bool $confirmed = false;

    public function __construct(private BookingId $id, private DateTimeImmutable $date) {}

    public function confirm(): void {
        if ($this->confirmed) {
            throw new DomainException('Booking already confirmed');
        }
        $this->confirmed = true;
        // Lever un Domain Event
        DomainEvents::raise(new BookingConfirmed($this->id));
    }
}
```

```php
// Domain/Booking/BookingRepository.php
interface BookingRepository {
    public function save(Booking $booking): void;
    public function get(BookingId $id): ?Booking;
}
```

```php
// Application/Handler/CreateBookingHandler.php
final class CreateBookingHandler
{
    public function __construct(private BookingRepository $repo) {}

    public function __invoke(CreateBooking $cmd): void {
        $booking = new Booking(new BookingId(), new DateTimeImmutable($cmd->date));
        $this->repo->save($booking);
    }
}
```

> 💡 DDD = *séparation stricte entre cœur métier (Domain), application (use cases), infrastructure (implémentations techniques).*

---

# 🧱 2. Architecture hexagonale (*Ports & Adapters*)

## 🎯 Objectif

Isoler le **cœur métier** du **monde extérieur** (base de données, framework, APIs…).

> Le domaine ne “sait pas” qu’il tourne dans Symfony, ni quelle base il utilise.

---

## 🔶 Schéma mental

```
           ┌──────────────────────┐
           │      Controllers     │  ← UI (Web, CLI, API)
           └──────────┬───────────┘
                      │
         Ports (Interfaces du domaine)
                      │
           ┌──────────┴───────────┐
           │     Application      │  ← Use Cases
           └──────────┬───────────┘
                      │
         Adapters (Implémentations techniques)
                      │
      ┌───────────────┴──────────────┐
      │   Infrastructure (DB, API)   │
      └──────────────────────────────┘
```

> * **Ports** = contrats côté application (ex: `BookingRepository`, `MailerInterface`)
> * **Adapters** = implémentations concrètes côté infra (`DoctrineBookingRepository`, `SymfonyMailerAdapter`)

---

## 🔧 Exemple : port / adapter

```php
// Domain/Port/PaymentGateway.php
interface PaymentGateway {
    public function pay(Money $amount, string $customerId): PaymentResult;
}

// Infrastructure/Adapter/StripePaymentGateway.php
final class StripePaymentGateway implements PaymentGateway {
    public function pay(Money $amount, string $customerId): PaymentResult {
        // appel API Stripe
        return new PaymentResult(true, 'STRIPE123');
    }
}
```

L’application utilise seulement l’interface :

```php
final class CheckoutService
{
    public function __construct(private PaymentGateway $gateway) {}

    public function process(Order $order): void {
        $result = $this->gateway->pay($order->total(), $order->customerId());
        if (!$result->success) throw new PaymentFailed();
    }
}
```

**L’avantage :**

* Tu peux changer de Stripe à PayPal sans toucher au code métier.
* Tu peux tester ton use case avec un `FakePaymentGateway`.

---

# ⚙️ 3. DDD + Hexagonal = Architecture “Clean”

* Le **Domain** = stable, pur PHP, aucune dépendance externe.
* Le **Application layer** = orchestre les cas d’usage.
* Le **Infrastructure layer** = détail technique (Doctrine, API, Symfony).
* Le **UI layer** = contrôleurs, CLI, GraphQL, etc.

Chaque couche dépend **vers l’intérieur seulement**, jamais l’inverse.

```
UI (Controllers)
   ↓
Application (Use cases)
   ↓
Domain (Entities, Rules)
   ↑
Infrastructure (DB, API, Mailers)
```

---

# 🧪 4. Avantages concrets

| ✅               | Description                                            |
| --------------- | ------------------------------------------------------ |
| **Testabilité** | le domaine se teste sans base de données ni framework  |
| **Évolutivité** | tu peux changer de tech (ex: passer de Doctrine à API) |
| **Lisibilité**  | code métier clair, séparé du technique                 |
| **Cohérence**   | langage commun entre devs et experts métier            |
| **Résilience**  | chaque couche est indépendante et remplaçable          |

---

# ⚡ 5. En Symfony, ça donne :

* **Domain** = `src/Domain/...`
* **Application** = services / use cases injectés
* **Infrastructure** = adapters (Doctrine repo, HTTP client, filesystem, mailer)
* **Entrées** = contrôleurs REST, commandes console, events
* **Sorties** = repository, clients externes, bus

Et les dépendances :

```yaml
services:
  App\Domain\*: ~           # interfaces, value objects
  App\Application\*: ~      # use cases
  App\Infrastructure\*: ~   # implémentations techniques
```

---

# 🧭 6. En résumé

| Concept       | Rôle                                                                         |
| ------------- | ---------------------------------------------------------------------------- |
| **DDD**       | Organise le code selon le métier, pas la technique.                          |
| **Hexagonal** | Sépare métier et infrastructure (Ports & Adapters).                          |
| **Résultat**  | Une base de code claire, testable, et résistante aux changements techniques. |

> 🧠 Retenir cette phrase :
> **“Le domaine est le roi, tout le reste est à son service.”**

---


