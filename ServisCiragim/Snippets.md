```markdown
# Backend Developer — Portfolio Code Snippets

> **Note:** The following snippets represent actual structures developed for a real-world SaaS platform in production. 
> Sensitive information such as company names, domains, API keys, and exact database schemas have been abstracted or removed.

---

## 1. JWT Auth Middleware — Token Blacklist + Version Check

When a user logs out, simply deleting the token on the client side is not enough; active tokens must be securely invalidated on the server. I solved this by pairing an `AccessTokenVersion` (an INT column in the database) with a Redis blacklist.

```php
<?php
declare(strict_types=1);

namespace App\Middlewares;

use App\Exception\HttpException;
use App\Http\Request;
use App\Models\UserModel;
use App\Utils\JwtHelper;
use App\Utils\CacheManager;

/**
 * Validates Bearer JWT; also checks Redis blacklist and DB token-version
 * so logout immediately invalidates all existing tokens without a DB token table scan.
 */
final class AuthMiddleware
{
    public static function handle(Request $request, bool $needsAuth): Request
    {
        if (!$needsAuth) return $request;

        $token = $request->getBearerToken();
        if (!$token) throw new HttpException(401, 'Unauthorized', 'UNAUTHORIZED');

        // Redis blacklist check (O(1)) — avoids DB hit for revoked tokens
        if (CacheManager::get('blacklist:' . md5($token))) {
            throw new HttpException(401, 'Token is blacklisted', 'TOKEN_REVOKED');
        }

        try {
            $payload = JwtHelper::decode($token);
        } catch (\Throwable) {
            throw new HttpException(401, 'Invalid or expired token', 'UNAUTHORIZED');
        }

        if (!isset($payload['userId'], $payload['companyId'], $payload['enc_perms'])) {
            throw new HttpException(401, 'Invalid token claims', 'UNAUTHORIZED');
        }

        // Version check: logout increments DB counter → old tokens instantly rejected
        $userId = (int) $payload['userId'];
        $row    = (new UserModel())->findById($userId);
        if (!$row || $row['Status'] !== 'Active') {
            throw new HttpException(401, 'Unauthorized', 'UNAUTHORIZED');
        }
        if ((int)($payload['atv'] ?? 0) !== (int)($row['AccessTokenVersion'] ?? 0)) {
            throw new HttpException(401, 'Session terminated', 'TOKEN_REVOKED');
        }

        return $request->withJwtPayload($payload);
    }
}

```

**Demonstrated Competencies:** Stateless JWT, Redis integration, secure logout mechanisms, middleware pattern.

---

## 2. Permission Middleware — Encrypted Permissions in JWT Payload

Instead of querying the database for user permissions on every request, I embedded them securely within the JWT using encryption. The `PermissionMiddleware` simply decrypts the payload and checks the required code — resulting in zero DB queries for authorization.

```php
<?php
declare(strict_types=1);

namespace App\Middlewares;

use App\Exception\HttpException;
use App\Http\Request;
use App\Services\PlatformFeatureService;
use App\Utils\CryptoHelper;

/**
 * Decrypts `enc_perms` claim from JWT and checks required permission code.
 * Zero DB queries — permissions are embedded (encrypted) inside the token.
 * Supports OR-separated codes: "invoice.view|invoice.create"
 */
final class PermissionMiddleware
{
    public static function handle(Request $request, ?string $requiredPermission): void
    {
        if (!$requiredPermission) return;

        $jwt = $request->getJwtPayload()
            ?? throw new HttpException(403, 'Forbidden', 'FORBIDDEN');

        try {
            $plain = CryptoHelper::decrypt((string)($jwt['enc_perms'] ?? ''));
        } catch (\Throwable) {
            throw new HttpException(403, 'Invalid permissions payload', 'FORBIDDEN');
        }

        $codes   = $plain['permissionCodes'] ?? [];
        $blocked = (new PlatformFeatureService())->getGloballyDisabledPermissionCodes();

        // OR logic: any one matching and non-blocked code grants access
        foreach (explode('|', $requiredPermission) as $rp) {
            $rp = trim($rp);
            if (in_array($rp, $codes, true) && !in_array($rp, $blocked, true)) {
                return; // ✓ granted
            }
        }

        throw new HttpException(403, "Missing permission: $requiredPermission", 'FORBIDDEN');
    }
}

```

**Demonstrated Competencies:** Encryption-in-JWT, platform feature flag architecture, OR permission logic.

---

## 3. Multi-Tenant BaseService — Centralized Tenant Ownership Guard

All core services extend from this base class. Tenant ownership (`CompanyId`) is strictly verified before every `UPDATE` or `DELETE` operation. SQL injection is aggressively prevented using a strict whitelist for table and column names.

```php
<?php
declare(strict_types=1);

namespace App\Services;

use App\Exception\HttpException;
use PDO;

/**
 * Provides tenant-scoped read, update, and soft-delete helpers.
 * All UPDATE paths go through updateForTenant() so no service can
 * accidentally modify another tenant's data.
 */
class BaseService
{
    /** Whitelist — prevents arbitrary table names in dynamic SQL */
    private const UPDATE_ALLOWED_TABLES = [
        'officerprofile', 'vehicleinformation', 'vehicleowner',
        'product', 'invoices', 'payment', 'supportrequest',
    ];

    protected PDO $pdo;

    public function __construct(?PDO $pdo = null)
    {
        $this->pdo = $pdo ?? \App\Config\Database::getConnection();
    }

    /**
     * Loads row only if it belongs to the tenant; throws 404 otherwise.
     * @return array<string, mixed>
     */
    protected function assertTenantRow(
        string $table, string $idCol, int $id, int $companyId,
        string $tenantCol = 'CompanyId'
    ): array {
        $sql = sprintf(
            'SELECT * FROM `%s` WHERE `%s` = :id AND `%s` = :cid LIMIT 1',
            str_replace('`', '', $table),
            str_replace('`', '', $idCol),
            str_replace('`', '', $tenantCol)
        );
        $st = $this->pdo->prepare($sql);
        $st->execute(['id' => $id, 'cid' => $companyId]);
        $row = $st->fetch(PDO::FETCH_ASSOC);
        if ($row === false) throw new HttpException(404, 'Resource not found', 'NOT_FOUND');
        return $row;
    }

    /**
     * Central UPDATE: validates table whitelist, ownership, and column whitelist.
     * @param array<string, mixed> $fields
     * @param list<string>         $allowedColumns
     */
    protected function updateForTenant(
        string $table, string $idCol, int $id, int $companyId,
        array $fields, array $allowedColumns, string $tenantCol = 'CompanyId'
    ): void {
        if (!in_array($table, self::UPDATE_ALLOWED_TABLES, true)) {
            throw new \InvalidArgumentException('Table not in update whitelist: ' . $table);
        }
        $this->assertTenantRow($table, $idCol, $id, $companyId, $tenantCol);

        $sets = []; $params = ['id' => $id, 'cid' => $companyId];
        foreach ($fields as $k => $v) {
            if (!in_array($k, $allowedColumns, true)) continue;
            $col = str_replace('`', '', (string)$k);
            $sets[] = "`$col` = :f_$col";
            $params["f_$col"] = $v;
        }
        if (!$sets) throw new HttpException(400, 'No valid fields to update', 'BAD_REQUEST');

        $sql = sprintf(
            'UPDATE `%s` SET %s WHERE `%s` = :id AND `%s` = :cid',
            str_replace('`', '', $table), implode(', ', $sets),
            str_replace('`', '', $idCol), str_replace('`', '', $tenantCol)
        );
        $this->pdo->prepare($sql)->execute($params);
    }

    /** Soft delete: sets DeletedAt instead of hard removal. */
    protected function softDeleteForTenant(
        string $table, string $idCol, int $id, int $companyId,
        string $tenantCol = 'CompanyId', string $deletedAtCol = 'DeletedAt'
    ): void {
        $this->assertTenantRow($table, $idCol, $id, $companyId, $tenantCol);
        $sql = sprintf(
            'UPDATE `%s` SET `%s` = NOW() WHERE `%s` = :id AND `%s` = :cid',
            str_replace('`', '', $table), str_replace('`', '', $deletedAtCol),
            str_replace('`', '', $idCol),  str_replace('`', '', $tenantCol)
        );
        $this->pdo->prepare($sql)->execute(['id' => $id, 'cid' => $companyId]);
    }
}

```

**Demonstrated Competencies:** Multi-tenancy architecture, strict SQL injection prevention (whitelisting), soft delete implementation, OOP inheritance.

---

## 4. AuthService — Permission Intersection Logic (Package ∩ User)

The effective permissions embedded in a user's JWT are dynamically calculated using strict set intersection: `User Permissions ∩ Active Package Permissions ∩ Platform Feature Flags`.

```php
<?php
// AuthService::getEffectivePermissionCodes (simplified)

/**
 * Returns the effective permission code list embedded in the JWT.
 * Formula: (user_codes ∩ package_codes) filtered by platform feature flags.
 *
 * SYSTEM_ADMIN bypasses intersection and gets the full platform code set.
 *
 * @return list<string>
 */
public function getEffectivePermissionCodes(int $userId, int $companyId): array
{
    // System admins always get full access regardless of package
    if ($this->getRoleCodeForUserId($userId) === 'SYSTEM_ADMIN') {
        return (new PlatformFeatureService($this->pdo))
            ->filterPermissionCodes(self::SYSTEM_ADMIN_PERMISSION_CODES);
    }

    $userCodes    = $this->decodeUserPermissionCodes($this->perms->findActiveByUserId($userId));
    $packageCodes = $this->mergePackagePermissionCodes($companyId);
    $pf           = new PlatformFeatureService($this->pdo);

    // Graceful fallback when one side is empty
    if ($userCodes === [])    return $pf->filterPermissionCodes($packageCodes);
    if ($packageCodes === []) return $pf->filterPermissionCodes($userCodes);

    // Strict intersection: user only gets what the subscribed package allows
    return $pf->filterPermissionCodes(
        array_values(array_intersect($userCodes, $packageCodes))
    );
}

/**
 * Merges permission codes from all active package subscriptions of the company,
 * then applies per-company "PlatformPermissionDeny" overrides.
 *
 * @return list<string>
 */
private function mergePackagePermissionCodes(int $companyId): array
{
    $codes = [];
    foreach (CompanyPackageHelper::listActiveSubscriptionsWithPackages($this->pdo, $companyId) as $row) {
        $j = is_string($row['PackagePermissions'])
            ? json_decode($row['PackagePermissions'], true)
            : $row['PackagePermissions'];
        if (is_array($j) && isset($j['permissionCodes'])) {
            $codes = array_merge($codes, $j['permissionCodes']);
        }
    }
    return $this->applyPlatformPermissionDeny($companyId, array_values(array_unique($codes)));
}

```

**Demonstrated Competencies:** Role-Based Access Control (RBAC), package-based entitlement constraints, feature flag systems, array intersection algorithms.

---

## 5. Rate Limit Middleware — Redis Sliding Window

```php
<?php
declare(strict_types=1);

namespace App\Middlewares;

use App\Exception\HttpException;
use App\Http\Request;
use App\Utils\CacheManager;

/**
 * Per-IP rate limiting via Redis (sliding 1-minute window).
 * Key format: rate_limit:{md5(ip:Y-m-d-H-i)} → TTL 60 s
 */
final class RateLimitMiddleware
{
    private const MAX_RPM = 150;

    public static function handle(Request $request): Request
    {
        $ip  = $_SERVER['REMOTE_ADDR'] ?? '127.0.0.1';
        $key = 'rate_limit:' . md5($ip . ':' . date('Y-m-d-H-i'));

        $current = (int) CacheManager::get($key);
        if ($current >= self::MAX_RPM) {
            throw new HttpException(429, 'Too Many Requests', 'RATE_LIMIT_EXCEEDED');
        }
        CacheManager::set($key, $current + 1, 60);

        return $request;
    }
}

```

**Demonstrated Competencies:** Redis caching, API rate limiting, middleware request lifecycle.

---

## 6. E-Invoice Integration — Series Resolution & UBL Payload Builder

```php
<?php
// NilveraDraftBuilder — simplified excerpt

/**
 * Builds a UBL-compatible e-Invoice payload from internal invoice + company rows.
 * Series resolution order: EInvoiceSeriesJson (JSON col) → VARCHAR series col → fallback prefix.
 */
private function resolveSeriesPrefix(array $company, string $kind): string
{
    $parsed = $this->parseEInvoiceSeriesJson($company['EInvoiceSeriesJson'] ?? null);
    if ($kind === 'efatura' && $parsed['efatura'] !== null) return $parsed['efatura'];
    if ($kind === 'earsiv'  && $parsed['earsiv']  !== null) return $parsed['earsiv'];
    return $kind === 'efatura'
        ? $this->seriesPrefix($company['NilveraEInvoiceSeries'] ?? null, 'SRV')
        : $this->seriesPrefix($company['NilveraEArchiveSeries'] ?? null, 'ARS');
}

/**
 * Maps internal invoice lines to Nilvera InvoiceLines structure.
 * Falls back to TotalAmount/TotalTaxedAmount when no line-level detail exists.
 *
 * @return list<array<string, mixed>>
 */
private function mapInvoiceLines(array $invoice): array
{
    $raw = is_string($invoice['Lines'] ?? null)
        ? json_decode($invoice['Lines'], true) : ($invoice['Lines'] ?? []);

    $out = [];
    foreach ((array)$raw as $idx => $row) {
        $qty     = max(1.0, (float)($row['qty'] ?? 1));
        $price   = (float)($row['price'] ?? 0);
        $taxPct  = (float)($row['tax'] ?? 20); // Extracted dynamically in production
        $lineExt = round($qty * $price, 2);
        $out[]   = [
            'Index'               => $idx + 1,
            'Name'                => (string)($row['desc'] ?? 'Service'),
            'Quantity'            => $qty,
            'UnitType'            => 'NIU',
            'Price'               => $price,
            'KDVPercent'          => $taxPct,
            'KDVTotal'            => round($lineExt * $taxPct / 100, 2),
            'LineExtensionAmount' => $lineExt,
        ];
    }

    // Fallback: single aggregate line from invoice totals
    if ($out === []) {
        $ta  = (float)($invoice['TotalAmount'] ?? 0);
        $tta = (float)($invoice['TotalTaxedAmount'] ?? 0);
        if ($ta <= 0 && $tta > 0) $ta = round($tta / 1.2, 2);
        $out[] = [
            'Index' => 1, 'Name' => 'Total', 'Quantity' => 1,
            'UnitType' => 'NIU', 'Price' => $ta, 'KDVPercent' => $tta > $ta ? 20 : 0,
            'KDVTotal' => round(max(0, $tta - $ta), 2), 'LineExtensionAmount' => $ta,
        ];
    }
    return $out;
}

```

**Demonstrated Competencies:** External API integration (Turkish e-Invoice/Nilvera), UBL data formatting, fallback logic, JSON column parsing.

---

## 7. Async Job — Queue-Based Invoice Dispatch

```php
<?php
declare(strict_types=1);

namespace App\Jobs;

use App\Services\InvoiceIntegrationService;

/**
 * Queued job: submits an invoice to the e-invoice gateway asynchronously.
 * Decouples HTTP response from the (potentially slow) external API call.
 *
 * Payload keys: companyId, invoiceId, payload, tenantApiKey, type (e_fatura|e_arsiv)
 */
class SendInvoiceJob
{
    public function handle(array $payload): void
    {
        $companyId = $payload['companyId'] ?? 0;
        $invoiceId = $payload['invoiceId'] ?? 0;
        $type      = $payload['type']      ?? 'e_fatura';

        if ($companyId === 0 || $invoiceId === 0) {
            throw new \Exception('Invalid payload: companyId and invoiceId are required.');
        }

        $svc = new InvoiceIntegrationService();

        match ($type) {
            'e_fatura' => $svc->sendEInvoice(
                $companyId, $invoiceId,
                $payload['payload'], $payload['tenantApiKey'] ?? null
            ),
            'e_arsiv'  => $svc->sendEArchive(
                $companyId, $invoiceId,
                $payload['payload'], $payload['tenantApiKey'] ?? null
            ),
            default    => throw new \Exception("Unknown invoice type: $type"),
        };
    }
}

```

**Demonstrated Competencies:** Asynchronous job processing, queue architecture, decoupling HTTP request lifecycles.

---

## 8. VehicleService — Automated Stock Update (Diff Algorithm)

When a vehicle service record is updated and the list of used spare parts is modified, the system calculates the exact difference (delta) and updates the inventory atomically to prevent stock anomalies.

```php
<?php
// VehicleService::update — stock diff logic (simplified)

/**
 * When a vehicle's ChangedPartList is updated, compute the quantity diff
 * between old and new parts and adjust product stock atomically.
 * Uses GREATEST(0, ...) to prevent negative stock.
 */
private function applyPartListStockDiff(array $oldParts, array $newParts): void
{
    $count = fn(array $parts): array => array_reduce($parts, function (array $acc, array $p): array {
        $pid = (int)($p['productId'] ?? 0);
        if ($pid > 0) $acc[$pid] = ($acc[$pid] ?? 0) + 1;
        return $acc;
    }, []);

    $oldCounts = $count($oldParts);
    $newCounts = $count($newParts);

    $changes = [];
    foreach ($newCounts as $pid => $qty) {
        $diff = $qty - ($oldCounts[$pid] ?? 0);
        if ($diff !== 0) $changes[$pid] = -$diff;          // used more → decrease stock
    }
    foreach ($oldCounts as $pid => $qty) {
        if (!isset($newCounts[$pid])) $changes[$pid] = $qty; // removed → restore stock
    }

    foreach ($changes as $pid => $delta) {
        $sign = $delta > 0 ? '+' : '-';
        $abs  = abs($delta);
        // GREATEST prevents stock going below 0
        $this->pdo
            ->prepare("UPDATE product SET Quantity = GREATEST(0, CAST(Quantity AS SIGNED) {$sign} {$abs}) WHERE id = :id")
            ->execute([':id' => $pid]);
    }
}

```

**Demonstrated Competencies:** Delta/diff algorithms, atomic database operations, robust inventory business logic encapsulation.

---

## 9. HierarchyService — Parent / Sub-Dealer Architecture

```php
<?php
// HierarchyService — multi-level dealer network

/**
 * Resolves the set of CompanyIds the caller is allowed to read from.
 * NETWORK_ADMIN role with Parent status → self + all active sub-dealers.
 * All other roles → own company only.
 *
 * @return list<int>
 */
public function accessibleCompanyIds(int $jwtCompanyId, bool $isParent): array
{
    $ids = [$jwtCompanyId];
    if ($isParent) {
        foreach ($this->companies->listChildCompanies($jwtCompanyId) as $child) {
            $ids[] = (int)$child['Id'];
        }
    }
    return array_values(array_unique($ids));
}

/**
 * Bulk-queries officers across all accessible companies in a single SQL IN().
 * @return list<array<string, mixed>>
 */
public function listOfficersForParent(int $parentId, int $limit, int $offset): array
{
    $ids = $this->accessibleCompanyIds($parentId, true);
    $placeholders = implode(',', array_fill(0, count($ids), '?'));
    $lim = max(1, min(200, $limit));
    $sql = "SELECT * FROM `officerprofile`
            WHERE CompanyId IN ($placeholders)
              AND DeletedAt IS NULL
            ORDER BY id DESC LIMIT $lim OFFSET " . max(0, $offset);
    $st = $this->pdo->prepare($sql);
    $st->execute($ids);
    return $st->fetchAll(\PDO::FETCH_ASSOC);
}

```

**Demonstrated Competencies:** Hierarchical multi-tenant data access, secure `IN()` placeholder generation, franchise network architecture.

---

## 10. OfficerInviteService — Transactional User Invitation

```php
<?php
// OfficerInviteService::inviteOfficer (simplified)

/**
 * Creates a technician (officer) user with a temporary password,
 * links to tenant, assigns permissions, and sends credentials via email.
 * All DB operations run inside a single transaction; mail failure rolls back.
 *
 * Handles re-invite: if same email exists as Inactive in this company → reactivate instead of conflict.
 */
public function inviteOfficer(int $companyId, array $input): array
{
    $name  = trim((string)($input['NameSurname'] ?? ''));
    $email = strtolower(trim((string)($input['Email'] ?? '')));

    if (!$name || !$email)                        throw new HttpException(400, 'Name and email required', 'VALIDATION');
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) throw new HttpException(400, 'Invalid email', 'VALIDATION');
    if ($this->users->findByEmail($email))         throw new HttpException(409, 'Email already active', 'CONFLICT');

    $password = $this->generateSecureNumericPassword(); // cryptographically random
    $hash     = password_hash($password, PASSWORD_DEFAULT);
    $roleId   = $this->resolveStaffRoleId();

    $this->pdo->beginTransaction();
    try {
        $userId = $this->users->insert([
            'Name' => $name, 'Email' => $email,
            'HashedPassword' => $hash, 'RoleId' => $roleId,
            'DefaultCompanyId' => $companyId, 'Status' => 'Active',
        ]);
        $this->userCompanies->insert($userId, $companyId, 'Active');
        $this->perms->insert($userId, self::STAFF_PERMISSION_CODES);

        $officerId = $this->officers->insert($companyId, [
            'NameSurname' => $name, 'UserId' => $userId, 'Status' => 'Active',
        ]);

        // Mail failure must roll back the entire record — no orphaned users
        if ($input['sendEmail'] ?? true) {
            $sent = $this->mail->sendCredentials($email, $name, $password, $companyId);
            if (!$sent) throw new HttpException(502, 'Mail delivery failed', 'MAIL_SEND_FAILED');
        }
        $this->pdo->commit();
    } catch (\Throwable $e) {
        if ($this->pdo->inTransaction()) $this->pdo->rollBack();
        throw $e;
    }

    return ['officerId' => $officerId, 'userId' => $userId, 'temporaryPassword' => $password];
}

private function generateSecureNumericPassword(): string
{
    return implode('', array_map(fn() => (string)random_int(0, 9), range(1, 8)));
}

```

**Demonstrated Competencies:** Atomic database transactions, mail-failure rollback patterns, secure account provisioning, `random_int` cryptography.

---

## Technology Summary

| Layer | Technology |
| --- | --- |
| **Backend** | PHP 8.1+ (strict_types, named arguments, match expressions) |
| **Authentication** | JWT (HS256) + AES-256 Encrypted Permissions |
| **Cache & Rate Limit** | Redis (`CacheManager`) |
| **Database** | MySQL / MariaDB via PDO (Multi-tenant, Soft-delete) |
| **External Integrations** | REST APIs (UBL format for E-Invoice, SMTP via PHPMailer) |
| **Architecture** | Service Layer, Repository/Model pattern, Asynchronous Job Queue |
| **Security** | Strict SQL Whitelisting, RBAC, Token Versioning, bcrypt Hashing |

```

```
