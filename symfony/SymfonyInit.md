# 🧭 Symfony – Cheat Sheet (Installation & Fonctionnalités de base)

---

## 🚀 Installation & Démarrage

### Prérequis

* PHP ≥ 8.1 (extensions courantes : `ctype`, `iconv`, `intl`, `pdo`, `openssl`, `json`, `mbstring`, `xml`, `zip`)
* Composer
* Base de données (MySQL/MariaDB, PostgreSQL, SQLite)
* Node.js (si front), mais **AssetMapper** peut éviter un bundler

### Installer le **Symfony CLI** (recommandé)

```bash
# macOS (Homebrew)
brew install symfony-cli/tap/symfony

# Linux (script officiel)
curl -sS https://get.symfony.com/cli/installer | bash
# puis ajoute le binaire à ton PATH si nécessaire

# Windows
winget install SymfonySymfonyCLI
```

### Créer un projet

```bash
# Site web full-stack (Twig, Doctrine, etc.)
symfony new my_app --webapp

# Ou via Composer (même résultat)
composer create-project symfony/website-skeleton my_app
```

### Lancer le serveur local

```bash
cd my_app
symfony serve -d       # -d = détaché
symfony open:local     # ouvre le navigateur
```

### Variables d’environnement

* Fichier : `.env` (défaut), `.env.local` (privé, sur ta machine)
* Exemple DSN Doctrine :

  ```
  DATABASE_URL="mysql://user:pass@127.0.0.1:3306/my_db?serverVersion=8.0"
  ```

---

## 🗂️ Arborescence utile (mini)

```
config/           # YAML/PHP configs (bundles, routes, services)
migrations/       # migrations Doctrine
public/           # document root (index.php, assets publics)
src/              # code applicatif
 ├─ Controller/   # contrôleurs
 ├─ Entity/       # entités Doctrine
 ├─ Repository/   # dépôts Doctrine
 └─ ...
templates/        # vues Twig
translations/     # fichiers de traduction
var/              # cache & logs
```

---

## 🧭 Routing (routes)

### Déclaration par attribut (recommandée)

```php
// src/Controller/HelloController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class HelloController
{
    #[Route('/hello/{name}', name: 'hello', methods: ['GET'])]
    public function __invoke(string $name = 'World'): Response
    {
        return new Response("Hello $name");
    }
}
```

### Fichier `config/routes.yaml` (alternative)

```yaml
hello:
  path: /hello/{name}
  controller: App\Controller\HelloController
  methods: [GET]
```

**Lister les routes**

```bash
php bin/console debug:router
```

---

## 🧠 Contrôleurs, Requête & Réponse

```php
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class DemoController extends AbstractController
{
    #[Route('/demo', name: 'demo', methods: ['POST'])]
    public function demo(Request $request): Response
    {
        $data = $request->request->all();         // POST form
        $json = json_decode($request->getContent(), true); // JSON
        return $this->json(['ok' => true, 'data' => $data, 'json' => $json]);
    }
}
```

**Redirections & flash**

```php
$this->addFlash('success', 'Saved!');
return $this->redirectToRoute('hello', ['name' => 'Symfony']);
```

---

## 🪄 Twig (vues)

* Fichier : `templates/*.html.twig`

```twig
{# templates/hello.html.twig #}
{% extends 'base.html.twig' %}

{% block title %}Hello{% endblock %}
{% block body %}
  <h1>Hello {{ name|e }}</h1>
  <a href="{{ path('hello', {name: 'World'}) }}">World</a>
{% endblock %}
```

**Rendu dans un contrôleur**

```php
return $this->render('hello.html.twig', ['name' => 'Symfony']);
```

---

## 🗃️ Doctrine ORM (basics)

### Config & connexion

* `DATABASE_URL` dans `.env`
* Vérifier :

```bash
php bin/console doctrine:database:create      # crée la DB (si possible)
php bin/console doctrine:migrations:diff      # génère migration
php bin/console doctrine:migrations:migrate   # applique migration
```

### Entité & Repository

```php
// src/Entity/Post.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Post
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 180)]
    private string $title;

    #[ORM\Column(type: 'text', nullable: true)]
    private ?string $content = null;

    // getters/setters...
}
```

**CRUD rapide avec Maker**

```bash
composer require symfony/maker-bundle --dev
php bin/console make:entity
php bin/console make:migration
php bin/console make:crud Post
```

**Requêtes**

```php
$post = $repo->find($id);
$all  = $repo->findBy(['title' => 'Hello'], ['id' => 'DESC'], 10);
```

---

## 🧰 Services & DI (Dependency Injection)

### Déclaration automatique (autowire/autoconfigure)

* Par défaut, `src/` est scanné. Un service = une classe.
* Injection par constructeur :

```php
final class Greeter
{
    public function __construct(private readonly \Psr\Log\LoggerInterface $logger) {}
    public function hello(string $name): string { $this->logger->info('Hi'); return "Hi $name"; }
}
```

**Utilisation**

```php
public function __construct(private Greeter $greeter) {}
```

### Paramètres (config/services.yaml)

```yaml
parameters:
  app.name: 'My App'
```

---

## 🧪 Console & Outils

### Commandes utiles

```bash
php bin/console                            # liste des commandes
php bin/console cache:clear                # clear cache
php bin/console about                      # infos projet
php bin/console debug:container Greeter    # inspecter DI
php bin/console debug:config framework     # voir config
php bin/console messenger:consume async    # worker Messenger
```

---

## ✅ Validation (constraints)

```php
use Symfony\Component\Validator\Constraints as Assert;

#[Assert\Length(min: 3)]
public string $title;

#[Assert\NotBlank]
public ?string $content = null;
```

**Valider**

```php
$errors = $validator->validate($dto);
if (count($errors)) { /* gérer erreurs */ }
```

---

## 🧾 Formulaires (rapide)

```php
// src/Form/PostType.php
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\OptionsResolver\OptionsResolver;
use App\Entity\Post;

final class PostType extends AbstractType
{
    public function buildForm(FormBuilderInterface $b, array $o): void
    {
        $b->add('title', TextType::class);
    }
    public function configureOptions(OptionsResolver $r): void
    {
        $r->setDefaults(['data_class' => Post::class]);
    }
}
```

**Contrôleur**

```php
$form = $this->createForm(PostType::class, new Post());
$form->handleRequest($request);
if ($form->isSubmitted() && $form->isValid()) { /* persist */ }
return $this->renderForm('post/form.html.twig', ['form' => $form]);
```

---

## 🎧 Événements & Subscribers

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\KernelEvents;
use Symfony\Component\HttpKernel\Event\RequestEvent;

final class RequestLogger implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array {
        return [KernelEvents::REQUEST => 'onRequest'];
    }
    public function onRequest(RequestEvent $event): void {
        // ...
    }
}
```

---

## ✉️ Messenger (asynchrone)

### Message + Handler

```php
// Message
final class SendEmail { public function __construct(public string $to) {} }

// Handler
use Symfony\Component\Messenger\Attribute\AsMessageHandler;
#[AsMessageHandler]
final class SendEmailHandler
{
    public function __invoke(SendEmail $m): void { /* envoyer */ }
}
```

**Envoi**

```php
$bus->dispatch(new SendEmail('user@example.com'));
```

**Consommer**

```bash
php bin/console messenger:consume async -vv
```

---

## 🔐 Sécurité (ultra-condensé)

### Installer & Maker

```bash
composer require symfony/security-bundle
php bin/console make:user
php bin/console make:auth
```

### Rôles & accès Twig

```twig
{% if is_granted('ROLE_ADMIN') %} ... {% endif %}
```

---

## ⚡ Assets

### AssetMapper (sans bundler)

```bash
composer require symfony/asset-mapper
# Place tes fichiers dans assets/ et référence-les :
# <link rel="stylesheet" href="{{ asset('styles/app.css') }}">
# <script type="module" src="{{ asset('app.js') }}"></script>
```

---

## 🧱 HTTP Client (rapide)

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;

public function __construct(private HttpClientInterface $http) {}
$response = $this->http->request('GET', 'https://api.example.com');
$data = $response->toArray();
```

---

## 🧊 Cache & Logs

```php
# Cache PSR-6
public function __construct(private \Psr\Cache\CacheItemPoolInterface $cache) {}
$item = $this->cache->getItem('key'); /* ... */

# Logs
$this->logger->info('message');  // var/log/dev.log | prod.log
```

---

## 🧪 Tests (basics)

```bash
composer require --dev symfony/test-pack
php bin/console make:test
```

```php
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class SmokeTest extends WebTestCase
{
    public function testHome(): void {
        $client = static::createClient();
        $client->request('GET', '/');
        self::assertResponseIsSuccessful();
    }
}
```

---

## 🛠️ Maker Bundle (générateurs)

```bash
composer require --dev symfony/maker-bundle
php bin/console make:controller
php bin/console make:entity
php bin/console make:migration
php bin/console make:crud
php bin/console make:form
php bin/console make:subscriber
```

---

## 🧾 Config rapide (fichiers clés)

* `config/packages/*.yaml` – config des bundles (framework, twig, doctrine, security…)
* `config/routes.yaml` ou attributs `#[Route]`
* `config/services.yaml` – paramètres & services custom
* `config/secrets/` – secrets chiffrés (`bin/console secrets:set KEY`)

---

## 🚚 Déploiement (checklist express)

* `APP_ENV=prod`, `APP_DEBUG=0`
* `composer install --no-dev --optimize-autoloader`
* `php bin/console cache:clear --env=prod`
* Migrations : `php bin/console doctrine:migrations:migrate --no-interaction`
* Assets (si AssetMapper) : `php bin/console asset-map:compile`

---

## 🧩 Rappels flash

* **Attributs** `#[Route]`, `#[AsMessageHandler]` → modernes & lisibles
* **Autowire** partout : pas besoin de config verbosée
* **.env.local** pour secrets locaux, **secrets** pour prod
* **AssetMapper** si tu veux éviter Webpack/Vite
* **Maker** pour accélérer la prod, **debug:*** pour introspecter

---
