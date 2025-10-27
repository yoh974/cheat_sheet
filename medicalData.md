Génial. Voici un **pack d’exemples PHP/Symfony “médical-ready”** : interop (FHIR/HL7), sécurité (consentement, audit), conformité (pseudonymisation, idempotence), et logique clinique (dosage, unités, RDV). Les commentaires sont en **anglais** pour être partageables.

---

## 1) FHIR REST client (Patient by identifier)

```php
// src/Fhir/FhirClient.php
namespace App\Fhir;

use Symfony\Contracts\HttpClient\HttpClientInterface;

final class FhirClient
{
    public function __construct(
        private HttpClientInterface $http,
        private string $baseUrl,           // e.g. https://ehr.example.com/fhir
        private string $bearerToken        // OAuth2 access token
    ) {}

    /** Fetch Patient by identifier (system|value) */
    public function getPatientByIdentifier(string $system, string $value): array
    {
        $url = sprintf('%s/Patient?identifier=%s|%s', rtrim($this->baseUrl,'/'), rawurlencode($system), rawurlencode($value));
        $res = $this->http->request('GET', $url, [
            'headers' => ['Authorization' => 'Bearer '.$this->bearerToken, 'Accept' => 'application/fhir+json'],
        ]);

        if (200 !== $res->getStatusCode()) {
            throw new \RuntimeException('FHIR error: '.$res->getStatusCode());
        }
        return $res->toArray(false); // Bundle
    }
}
```

---

## 2) Validation FHIR minimale (Patient resource shape)

```php
// src/Fhir/Validator/FhirPatientValidator.php
namespace App\Fhir\Validator;

/** Minimal FHIR Patient structural checks (not a full schema) */
final class FhirPatientValidator
{
    public static function validate(array $patient): void
    {
        if (($patient['resourceType'] ?? null) !== 'Patient') {
            throw new \InvalidArgumentException('resourceType must be Patient');
        }
        if (!isset($patient['identifier']) || !is_array($patient['identifier'])) {
            throw new \InvalidArgumentException('Patient.identifier[] required');
        }
        // Optional but common checks
        if (isset($patient['birthDate']) && !preg_match('/^\d{4}-\d{2}-\d{2}$/', $patient['birthDate'])) {
            throw new \InvalidArgumentException('birthDate must be YYYY-MM-DD');
        }
    }
}
```

---

## 3) Webhook résultats labo — **idempotence** + upsert

```php
// src/Lab/Controller/LabWebhookController.php
#[Route('/webhook/lab', methods: ['POST'])]
public function receive(Request $r, EntityManagerInterface $em): JsonResponse
{
    // Idempotency key (lab result unique id)
    $payload = $r->getContent();
    $data = json_decode($payload, true, 512, JSON_THROW_ON_ERROR);
    $key  = $data['resultId'] ?? null;
    if (!$key) return $this->json(['error' => 'missing resultId'], 400);

    // Prevent duplicate processing
    $lock = $em->getConnection()->prepare('INSERT IGNORE INTO idempotency_keys (k) VALUES (:k)');
    $lock->executeStatement(['k' => $key]);
    if ($lock->rowCount() === 0) {
        return $this->json(['status' => 'duplicate'], 200);
    }

    // Upsert result
    $result = $em->getRepository(LabResult::class)->findOneBy(['externalId' => $key]) ?? new LabResult();
    $result->setExternalId($key);
    $result->hydrateFromArray($data); // map LOINC, values, unit, ref range
    $em->persist($result);
    $em->flush();

    return $this->json(['status' => 'ok'], 201);
}
```

---

## 4) **Pseudonymisation** (SHA-256 + salt + FPE-like masking)

```php
// src/Security/Pseudonymizer.php
namespace App\Security;

final class Pseudonymizer
{
    public function __construct(private string $salt) {}

    /** Privacy-preserving hash for PHI fields */
    public function hash(string $value): string
    {
        return hash('sha256', $this->salt.'|'.$value);
    }

    /** Mask display: keep format but hide identity (e.g., "Du***-19**") */
    public function maskName(string $family, string $birthYear): string
    {
        return mb_substr($family, 0, 2) . str_repeat('*', 3) . '-' . substr($birthYear, 0, 2) . '**';
    }
}
```

---

## 5) **Consentement** + « Purpose of Use » (Voter RBAC)

```php
// src/Security/Voter/MedicalRecordVoter.php
namespace App\Security\Voter;

use App\Entity\MedicalRecord;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;

final class MedicalRecordVoter extends Voter
{
    public const READ = 'MR_READ';

    protected function supports(string $attr, mixed $subject): bool
    { return $attr === self::READ && $subject instanceof MedicalRecord; }

    protected function voteOnAttribute(string $attr, mixed $mr, TokenInterface $token): bool
    {
        $user = $token->getUser();
        // Purpose of Use (e.g., TREATMENT, EMERGENCY, BILLING)
        $purpose = $token->getAttribute('purpose_of_use') ?? 'TREATMENT';

        if ($mr->isConsentRevoked()) return false;
        if ($purpose === 'EMERGENCY') return true; // break-glass policy could audit

        // RBAC + patient relationship
        return $user?->hasRole('ROLE_MD') && $mr->isAssignedTo($user);
    }
}
```

---

## 6) **Audit trail** (GDPR/HIPAA-style) via middleware

```php
// src/Audit/AuditSubscriber.php
namespace App\Audit;

use Symfony\Component\HttpKernel\KernelEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Psr\Log\LoggerInterface;

final class AuditSubscriber implements EventSubscriberInterface
{
    public function __construct(private LoggerInterface $auditLogger) {}

    public static function getSubscribedEvents(): array
    { return [KernelEvents::TERMINATE => 'onTerminate']; }

    public function onTerminate(TerminateEvent $e): void
    {
        $req  = $e->getRequest();
        $resp = $e->getResponse();

        // Minimal PHI-safe audit
        $this->auditLogger->info('access_log', [
            'actor'    => $req->getUser() ?? 'anon',
            'purpose'  => $req->attributes->get('purpose_of_use', 'TREATMENT'),
            'endpoint' => $req->getPathInfo(),
            'method'   => $req->getMethod(),
            'status'   => $resp?->getStatusCode(),
            'subject'  => $req->attributes->get('patient_id') ? 'patient#'.$req->attributes->get('patient_id') : null,
            'ts'       => (new \DateTimeImmutable())->format(DATE_ATOM),
        ]);
    }
}
```

---

## 7) **Dosage** mg/kg avec plafonds et arrondi pharmaceutique

```php
final class DosageCalculator
{
    /**
     * @param float $weightKg Patient weight
     * @param float $doseMgPerKg Dose per kg
     * @param float $maxSingleDoseMg Max cap
     * @param int   $roundToMg Round to nearest N mg (e.g., 5mg)
     */
    public static function mgPerDose(float $weightKg, float $doseMgPerKg, float $maxSingleDoseMg, int $roundToMg = 5): int
    {
        if ($weightKg <= 0) throw new \InvalidArgumentException('weightKg > 0 required');
        $dose = $weightKg * $doseMgPerKg;
        $dose = min($dose, $maxSingleDoseMg);
        // Round to nearest multiple suitable for tablets
        return (int) (round($dose / $roundToMg) * $roundToMg);
    }
}

// Example: 70kg, 10mg/kg, max 600mg, round 25mg → returns 600
```

---

## 8) **Unité & conversion** (µmol/L ⇄ mg/dL) avec typage VO

```php
final readonly class Glucose
{
    private function __construct(private float $mmolL) {}

    public static function fromMgDl(float $mgdl): self
    {
        // 1 mmol/L glucose ≈ 18.0182 mg/dL
        return new self($mgdl / 18.0182);
    }

    public static function fromMmolL(float $mmolL): self { return new self($mmolL); }

    public function asMmolL(): float { return round($this->mmolL, 2); }
    public function asMgDl(): float  { return round($this->mmolL * 18.0182, 1); }
}
```

---

## 9) Planification de **RDV timezone-safe** (DST, ISO 8601)

```php
final class AppointmentScheduler
{
    public static function schedule(
        \DateTimeImmutable $localStart,
        string $tzLocal = 'Europe/Paris',
        string $tzClinic = 'Europe/Paris'
    ): array {
        $startLocal = $localStart->setTimezone(new \DateTimeZone($tzLocal));
        $clinicStart = $startLocal->setTimezone(new \DateTimeZone($tzClinic));
        return [
            'patient_iso' => $startLocal->format(DATE_ATOM),
            'clinic_iso'  => $clinicStart->format(DATE_ATOM),
            'duration_min'=> 30,
        ];
    }
}
```

---

## 10) **Code médical** (ICD-10 / LOINC) — validation & mapping léger

```php
final class CodeValidator
{
    /** ICD-10 like: A00–Z99 with optional dot (A09.1) */
    public static function isValidIcd10(string $code): bool
    { return (bool) preg_match('/^[A-TV-Z][0-9]{2}(\.[0-9A-Z]{1,4})?$/', strtoupper($code)); }

    /** LOINC like: 1234-5 */
    public static function isValidLoinc(string $code): bool
    { return (bool) preg_match('/^[0-9]{1,7}-[0-9]$/', $code); }
}

final class LoincMapping
{
    /** @var array<string,string> simple mapping LOINC -> human label */
    private const MAP = [
        '718-7'   => 'Hemoglobin [Mass/volume] in Blood',
        '2345-7'  => 'Glucose [Mass/volume] in Blood',
    ];

    public static function label(string $loinc): ?string
    { return self::MAP[$loinc] ?? null; }
}
```

---

## 11) **Circuit breaker** pour EHR/FHIR externe

```php
final class CircuitBreaker
{
    private int $failures = 0;
    private ?int $openedAt = null;

    public function call(callable $fn, int $threshold = 3, int $cooldownSec = 30): mixed
    {
        if ($this->openedAt && (time() - $this->openedAt) < $cooldownSec) {
            throw new \RuntimeException('Circuit open');
        }

        try {
            $result = $fn();
            $this->failures = 0; $this->openedAt = null;
            return $result;
        } catch (\Throwable $e) {
            $this->failures++;
            if ($this->failures >= $threshold) $this->openedAt = time();
            throw $e;
        }
    }
}
```

---

## 12) **HL7 v2** parsing minimal (PID segment)

```php
final class Hl7v2
{
    /** Very small parser for PID|; returns ['patientId' => ..., 'family' => ..., 'given' => ...] */
    public static function parsePid(string $message): array
    {
        foreach (preg_split("/[\r\n]+/", $message) as $line) {
            $fields = explode('|', $line);
            if (($fields[0] ?? '') === 'PID') {
                // PID-3 Patient Identifier List, PID-5 Patient Name (family^given)
                $pid3 = $fields[3] ?? '';
                $pid5 = $fields[5] ?? '';
                [$family, $given] = array_pad(explode('^', $pid5), 2, null);
                return [
                    'patientId' => trim($pid3),
                    'family'    => $family ? trim($family) : null,
                    'given'     => $given ? trim($given) : null,
                ];
            }
        }
        throw new \InvalidArgumentException('No PID segment');
    }
}
```

---

### Besoin d’un **mini-squelette DDD + Hexagonal** prêt pour un micro-service médical (Patient, Encounter, LabResult) avec contrôleur, use-cases, ports/adapters (Doctrine + FHIR) et tests ?

