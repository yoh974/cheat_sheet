Parfait 💪
Voici ta **fiche synthèse Markdown** des **design patterns essentiels pour un développeur sénior PHP/Symfony**, claire, compacte et prête à imprimer avant ton entretien 👇

---

````markdown
# 🧱 Design Patterns – Cheat Sheet (PHP / Symfony)

## 🧩 1. CREATIONAL PATTERNS (Création)

### 🏭 Factory Method
➡ Délègue la création d’objet à une méthode spécialisée.

**Usage :**
- Créer des objets selon un type sans exposer le constructeur.

```php
interface TransportFactory { public function create(): Transport; }
class CarFactory implements TransportFactory { public function create(): Transport { return new Car(); } }
````

**Symfony :** FormFactory, ValidatorFactory

---

### 🏗️ Builder

➡ Construit un objet complexe étape par étape.

```php
$email = (new EmailBuilder())
    ->to('user@mail.com')
    ->subject('Hello')
    ->body('Welcome!')
    ->build();
```

**Symfony :** QueryBuilder, EmailBuilder

---

### 🧍 Singleton

➡ Assure qu’une seule instance existe (⚠️ à éviter sauf cas précis).

```php
class Config {
    private static ?Config $instance = null;
    public static function getInstance(): Config {
        return self::$instance ??= new Config();
    }
}
```

**Symfony :** Kernel, container unique

---

### 🏭 Abstract Factory

➡ Crée des familles d’objets compatibles.

```php
interface PaymentFactory { public function createGateway(): Gateway; }
class StripeFactory implements PaymentFactory { public function createGateway(): Gateway { return new StripeGateway(); } }
```

**Symfony :** Services configurés par environnement

---

## ⚙️ 2. STRUCTURAL PATTERNS (Structure)

### 🔌 Adapter

➡ Rend deux interfaces incompatibles compatibles.

```php
class SlackAdapter implements NotifierInterface {
    public function send(string $message) { /* adapte API Slack */ }
}
```

**Symfony :** Adapters d’API (Mailer, Logger)

---

### 🎁 Decorator

➡ Ajoute dynamiquement un comportement sans modifier la classe.

```php
class CacheMailer implements MailerInterface {
    public function __construct(private MailerInterface $mailer) {}
    public function send($email) { /* cache */ $this->mailer->send($email); }
}
```

**Symfony :** `decorates:` dans services.yaml

---

### 🧱 Facade

➡ Simplifie l’accès à un sous-système complexe.

```php
class OrderFacade {
    public function processOrder() {
        $this->payment->pay();
        $this->inventory->reserve();
    }
}
```

**Symfony :** Services qui encapsulent plusieurs composants

---

### 🪞 Proxy

➡ Contrôle l’accès à un objet (lazy loading, sécurité, cache).

**Symfony :** Doctrine Proxies, VirtualProxy

---

## 🔄 3. BEHAVIORAL PATTERNS (Comportement)

### 🧠 Strategy

➡ Permet de changer une logique à la volée.

```php
interface PaymentStrategy { public function pay(float $amount); }
class PaypalStrategy implements PaymentStrategy { /* ... */ }
class PaymentService { public function __construct(private PaymentStrategy $strategy) {} }
```

**Symfony :** Plusieurs implémentations d’une interface injectées par tag

---

### 🔔 Observer

➡ Réagit à des événements déclenchés ailleurs.

```php
class UserRegisteredEvent {}
class SendWelcomeEmailListener { public function onUserRegistered() { /* ... */ } }
```

**Symfony :** EventDispatcher, EventSubscriber

---

### ⛓️ Chain of Responsibility

➡ Passe une requête à travers une chaîne jusqu’à traitement.

```php
abstract class Handler {
    private ?Handler $next = null;
    public function setNext(Handler $handler): Handler { return $this->next = $handler; }
    public function handle($request) { return $this->next?->handle($request); }
}
```

**Symfony :** Middleware HTTP, Event propagation

---

### 💬 Command

➡ Encapsule une action dans un objet (annulable, queueable).

```php
class SendEmailCommand implements CommandInterface { public function execute() { /* ... */ } }
```

**Symfony :** Console Commands, Messenger CQRS

---

### 🧩 Template Method

➡ Définit un squelette d’algorithme avec étapes personnalisables.

```php
abstract class AbstractReport {
    public function generate() {
        $this->fetchData();
        $this->format();
        $this->export();
    }
    abstract protected function fetchData();
}
```

**Symfony :** AbstractController, AbstractFixture

---

### ⚙️ State

➡ Change le comportement selon l’état interne.

```php
interface OrderState { public function next(Order $order); }
class PaidState implements OrderState { public function next(Order $order) { $order->setState(new ShippedState()); } }
```

**Symfony :** Workflow component

---

## 🧠 4. LES PLUS COURANTS EN SYMFONY

| **Pattern**             | **Usage typique Symfony**                       |
| ----------------------- | ----------------------------------------------- |
| Factory                 | Création de Form / Validator / Service          |
| Builder                 | QueryBuilder, EmailBuilder                      |
| Strategy                | Services par tag (ex: différents algos)         |
| Decorator               | Override d’un service (`decorates:`)            |
| Observer                | Events, Listeners, Subscribers                  |
| Chain of Responsibility | Middleware HTTP                                 |
| Facade                  | Services métier regroupant plusieurs composants |
| Dependency Injection    | Cœur du container Symfony                       |

---

## 🎯 BONNES PRATIQUES

✅ Utiliser un pattern **quand il résout un vrai problème de flexibilité ou de testabilité**
❌ Ne pas “placer un pattern pour le style” (over-engineering)
💡 Penser en **SOLID** avant de penser en “pattern”

---

**Résumé ultra-court :**

> * **Factory / Builder** → créer proprement
> * **Adapter / Decorator** → rendre compatible ou ajouter une couche
> * **Strategy / Observer / Chain** → découpler la logique
> * **Command / State / Template** → structurer le comportement
> * **DI / Facade** → simplifier et centraliser

---

```

---

