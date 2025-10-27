Bien sûr 💪 voici un rappel clair et concis des **principes SOLID**, essentiels pour tout développeur orienté objet (et particulièrement valorisés en entretien sénior PHP/Symfony).

---

## 🧱 Les 5 principes SOLID

| Acronyme | Nom complet                         | En français                            | Objectif principal                                                                                                         |
| -------- | ----------------------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **S**    | **Single Responsibility Principle** | Principe de responsabilité unique      | Une classe ne doit avoir **qu’une seule raison de changer**                                                                |
| **O**    | **Open/Closed Principle**           | Principe ouvert/fermé                  | Le code doit être **ouvert à l’extension, mais fermé à la modification**                                                   |
| **L**    | **Liskov Substitution Principle**   | Principe de substitution de Liskov     | Une sous-classe doit pouvoir **remplacer sa super-classe sans altérer le comportement attendu**                            |
| **I**    | **Interface Segregation Principle** | Principe de ségrégation des interfaces | Mieux vaut **plusieurs petites interfaces spécifiques** qu’une seule interface générale et lourde                          |
| **D**    | **Dependency Inversion Principle**  | Principe d’inversion des dépendances   | Les modules haut niveau ne doivent pas dépendre des modules bas niveau, mais **tous deux doivent dépendre d’abstractions** |

---

## 🧩 Détail avec exemples (PHP)

### 1️⃣ **Single Responsibility Principle (SRP)**

> Une classe = une seule responsabilité.

❌ Mauvais :

```php
class UserManager {
    public function createUser($data) { /* ... */ }
    public function sendWelcomeEmail($user) { /* ... */ } // <-- autre responsabilité
}
```

✅ Bon :

```php
class UserManager {
    public function createUser($data) { /* ... */ }
}

class Mailer {
    public function sendWelcomeEmail($user) { /* ... */ }
}
```

---

### 2️⃣ **Open/Closed Principle (OCP)**

> On doit pouvoir **ajouter des comportements sans modifier le code existant**.

❌ Mauvais :

```php
class PaymentProcessor {
    public function pay($method) {
        if ($method === 'paypal') { /* ... */ }
        elseif ($method === 'credit_card') { /* ... */ }
    }
}
```

✅ Bon :

```php
interface PaymentMethod {
    public function pay();
}

class PaypalPayment implements PaymentMethod { /* ... */ }
class CreditCardPayment implements PaymentMethod { /* ... */ }

class PaymentProcessor {
    public function process(PaymentMethod $payment) {
        $payment->pay();
    }
}
```

---

### 3️⃣ **Liskov Substitution Principle (LSP)**

> Les sous-classes doivent pouvoir être utilisées à la place de leur classe parente **sans modifier le comportement attendu**.

❌ Mauvais :

```php
class Bird {
    public function fly() { /* ... */ }
}

class Penguin extends Bird {
    public function fly() {
        throw new Exception("I can't fly");
    }
}
```

✅ Bon :

```php
class Bird {}
class FlyingBird extends Bird {
    public function fly() { /* ... */ }
}
class Penguin extends Bird {}
```

---

### 4️⃣ **Interface Segregation Principle (ISP)**

> Une classe ne doit **pas être forcée d’implémenter des méthodes dont elle n’a pas besoin.**

❌ Mauvais :

```php
interface Worker {
    public function work();
    public function eat();
}

class Robot implements Worker {
    public function work() {}
    public function eat() { /* inutile */ }
}
```

✅ Bon :

```php
interface Workable {
    public function work();
}

interface Eatable {
    public function eat();
}

class Robot implements Workable {
    public function work() {}
}
```

---

### 5️⃣ **Dependency Inversion Principle (DIP)**

> Les classes doivent dépendre **d’interfaces ou d’abstractions**, pas d’implémentations concrètes.

❌ Mauvais :

```php
class UserController {
    private $mailer;
    public function __construct() {
        $this->mailer = new Mailer(); // dépendance forte
    }
}
```

✅ Bon :

```php
class UserController {
    private MailerInterface $mailer;

    public function __construct(MailerInterface $mailer) {
        $this->mailer = $mailer;
    }
}
```

🧠 En Symfony, c’est le **Container de services (autowiring)** qui respecte ce principe automatiquement.

---

## 🎯 En résumé

> **SOLID** permet un code :

* plus **maintenable**
* plus **testable**
* plus **extensible**
* moins **couplé**

---
