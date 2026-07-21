# Code Review: codraw/tester-bundle

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **M3 (composer.json)**: added `"php": ">=8.5"` to `require`; moved `phpunit/phpunit` (`^11.3 || ^12.0`) from `require-dev` to `require` (shipped traits, constraints and runner extensions hard-import PHPUnit classes); added a `suggest` section covering the optional integrations: `dama/doctrine-test-bundle` (DoctrineTransactionExtension), `doctrine/dbal` (SQL profiling middleware, DBAL 4 required), `doctrine/orm` (SqlProfiler, entity autowiring, temporary entity cleaner), `codraw/profiling`, `codraw/security` (JWTLoginTrait), `symfony/console` (EventDispatcherTesterTrait), `symfony/mailer` / `symfony/mime` (TemplatedMailerAssertionsTrait), `symfony/messenger` (TransportTester); filled in the empty `description`. `composer validate --no-check-publish` passes. Open items from M3 not addressed: no `conflict` entry against DBAL 3 was added, and no dependency was removed.
- **H1**: `WebTestCase::createJsonClient()` now uses the server-parameter keys `CONTENT_TYPE` and `HTTP_ACCEPT` so the JSON headers actually reach the application (previously the `Content-Type`/`Accept` keys were silently ignored).
- **H2**: `JWTLoginTrait::logout()` now unsets `HTTP_AUTHORIZATION` (the key `loginUser()` actually sets) instead of the never-set `Authorization` key, so the Bearer token no longer survives logout. The secondary point (also clearing the session cookie created by `parent::loginUser()`) was not changed — see skipped items.
- **L2**: `EventDispatcherTesterTrait` — `tempnam()` and `file_put_contents()` failures now throw clear `RuntimeException`s; the shutdown `unlink` is registered before cleaning so a cleaner exception no longer leaks the temp file; `\DOMDocument::load()` failure now throws instead of silently writing back an empty file.
- **L3**: `KernelShutdownExtension` — non-`TestMethod` events are now skipped with an `instanceof` guard instead of `\assert()` (no more fatal with `zend.assertions=0` and `.phpt` tests), and the static `kernel`/`class` properties are only reset when they exist (`hasProperty` guards).
- **L4**: `AutowireServiceMock` — a non-intersection property type now falls through to the intended descriptive `RuntimeException` instead of failing an `assert()` or fatalling on `getTypes()`.
- **L6 (partial)**: `AutowireEntity` — uses `getManagerForClass($class)` instead of the default manager (matching `AutowireReloadedEntity`), and a `null` `findOneBy()` result on a non-nullable property now throws a `RuntimeException` naming the class and criteria instead of producing an opaque `TypeError`. Nullable properties still accept `null`. The union-type error-message wording was not changed.

### Validation pass (2026-07-20)

- `composer install --optimize-autoloader --no-interaction --prefer-dist --no-scripts` resolves and installs cleanly with the new `composer.json` constraints (no constraint adjustment was needed).
- `vendor/bin/phpunit` (PHPUnit 12.5.31, PHP 8.5.8, MySQL `codraw_tester_bundle`): 38 tests, 55 assertions, 0 failures. The 3 risky warnings in `Tests/Messenger/TransportTesterTest.php` ("did not remove its own exception handlers") are pre-existing — verified identical with the fixes stashed. No test-expectation updates were required by the fixes above.
- PHPStan (level from `phpstan.dist.neon`): 65 errors, byte-identical with and without the fixes — all pre-existing in this environment and caused by optional dev dependencies not installed locally (`doctrine/orm`, `doctrine/dbal`, `dama/doctrine-test-bundle`, `symfony/mailer`/`mime`, `codraw/security`), plus unused-trait and two minor `Tests/` findings. None reference behavior changed by the fixes; the empty `phpstan-baseline.neon` was not touched.
- `markdownlint-cli2 --fix "**/*.md"`: only auto-formatting of this file itself; final run reports 0 errors across 6 files.
- No further code changes were made in this pass; none of the remaining findings (M1, M2, M4–M6, L1, L5) are exercised by existing unit tests, so they were left as documented.

## Overall Assessment

This is a compact, modern (PHP 8, Symfony 6.4) test-support bundle providing web-test helpers, messenger/mailer/event-dispatcher assertion traits, SQL profiling via DBAL middleware, and a set of PHPUnit runner extensions and property-autowiring attributes. The code is generally small, focused, and idiomatic, with a clean DI setup and a sensible snapshot-testing pattern (auto-creating expected fixture files on first run). However, two user-facing helpers are functionally broken (`createJsonClient` headers are never applied; `JWTLoginTrait::logout()` removes the wrong server key), several assertion helpers are internally inconsistent, the composer manifest omits a large set of optional dependencies (Doctrine, DAMA test bundle, Mailer, Console, PHP version itself), and test coverage is thin precisely in the areas where the bugs live. Because this is a test bundle, none of the findings are exploitable security issues, but the broken helpers can silently weaken the tests of consuming applications.

---

## Findings

### High

#### **[FIXED]** H1. `createJsonClient()` headers are silently ignored

`WebTestCase.php:55-66`

```php
return static::createClient(
    $options,
    array_merge(
        [
            'Content-Type' => 'application/json',
            'Accept' => 'application/json',
        ],
        $server
    )
);
```

These values are passed to `KernelBrowser::setServerParameters()`, i.e. they become entries of the PHP *server* array used to build the `Request`. HttpFoundation's `ServerBag::getHeaders()` only turns keys prefixed with `HTTP_` (plus `CONTENT_TYPE` / `CONTENT_LENGTH` / `CONTENT_MD5`) into HTTP headers. The literal keys `Content-Type` and `Accept` are therefore never seen as headers by the application; requests made through this "JSON client" are sent without `Content-Type: application/json` or `Accept: application/json`. The correct keys are `CONTENT_TYPE` and `HTTP_ACCEPT`. The bug is masked when tests use `$client->jsonRequest()` (which sets those keys itself), which is probably why it went unnoticed — but any test relying on `createJsonClient()` + `request()` is not testing what it claims to.

#### **[FIXED]** H2. `JWTLoginTrait::logout()` unsets the wrong key — Bearer token survives logout

`HttpKernel/JWTLoginTrait.php:24` vs `:32`

```php
$this->server['HTTP_AUTHORIZATION'] = 'Bearer '.$this->jwtAuthenticator->generaToken($user); // line 24
...
public function logout(): void
{
    unset($this->server['Authorization']); // line 32 — key never set
}
```

`loginUser()` stores the JWT under `HTTP_AUTHORIZATION`, but `logout()` unsets `Authorization`, which is never set. After calling `logout()` the client keeps sending the valid `Authorization: Bearer ...` header on every subsequent request, so a test asserting "after logout the user gets 401" will spuriously pass authentication (or, more insidiously, the test author believes the anonymous path is exercised when it is not). `logout()` also does not clear the session cookie that `parent::loginUser()` created, so even non-JWT authentication survives.

### Medium

#### M1. `TemplatedEmailCount` counts queued emails; the companion trait excludes them

`Mailer/Constraint/TemplatedEmailCount.php:44-71` vs `Mailer/TemplatedMailerAssertionsTrait.php:21-23`

`getTemplatedMailerMailerEvents()` skips events where `$event->isQueued()`, but the `TemplatedEmailCount` constraint used by `assertHtmlTemplatedEmailCount()` / `assertTextTemplatedEmailCount()` iterates `$events->getEvents($this->transport)` without that filter. With a messenger-backed mailer transport each email produces both a queued and a sent event, so `assertHtmlTemplatedEmailCount(1, ...)` can fail (count 2) while `count(getHtmlTemplatedMailerMailerEvents(...))` returns 1 — two APIs in the same feature disagree on what an "email" is.

#### M2. `DoctrineTransactionExtension`: transaction starts on the root (non-class) suite, defeating `#[NoTransaction]` for the first test class

`PHPUnit/Extension/DoctrineTransaction/DoctrineTransactionExtension.php:44-58, 62-82`

`TestSuiteStarted` fires for every suite, including the root suite (whose name is the suite label from `phpunit.xml`, not a class). `asNoTransaction()` returns `false` for non-classes, so `startTransactionIfNeeded()` calls `StaticDriver::setKeepStaticConnections(true)` and `begin()` as soon as the root suite starts. Consequently: (a) if the *first* test class in the run is annotated `#[NoTransaction]`, it still executes inside the transaction begun at root-suite start — the attribute only works for classes that run after another suite's `Finished` event rolled the transaction back; (b) `setKeepStaticConnections(false)` at `:55` runs on *every* suite-finished event (including nested ones), churning the flag mid-run. The begin/rollback logic should ignore non-class suites.

#### **[FIXED]** M3. Extensive undeclared optional dependencies, and no `php` requirement

`composer.json:13-28`

The package code references, with no `require`, `require-dev`, or even `suggest` entry:

- `doctrine/orm` / `doctrine/dbal` (`Profiling/SqlProfiler.php`, `Profiling/Sql/*`, `PHPUnit/Extension/DeleteTemporaryEntity/BaseTemporaryEntityCleaner.php`, `PHPUnit/Extension/SetUpAutowire/AutowireEntity.php`, `AutowireReloadedEntity.php`)
- `dama/doctrine-test-bundle` (`DoctrineTransactionExtension.php` — `StaticDriver` is a hard dependency of that class)
- `symfony/mailer` / `symfony/mime` (`Mailer/*`)
- `symfony/console` (`EventDispatcher/EventDispatcherTesterTrait.php` uses `CommandTester`; framework-bundle does not require console)
- `codraw/security` (`HttpKernel/JWTLoginTrait.php` uses `JwtAuthenticator`)
- `codraw/profiling` is only in `require-dev`, yet `DrawTesterBundle::boot()` and `Configuration` reference it at runtime (guarded by `class_exists`, which is fine, but a `suggest` entry is warranted).

There is also no `"php": "^8.x"` constraint, and `"description"` is empty. Consumers get late, confusing failures ("class not found" inside a trait) instead of an install-time signal. Note the DBAL middleware classes use the DBAL 4 signatures (`ParameterType` enum, `bindValue(): void`), so an explicit `doctrine/dbal: ^4.0` conflict/suggest matters — under DBAL 3 these classes are fatally incompatible.

#### M4. Profiling defaults to enabled when `codraw/profiling` is installed, but `SqlProfiler` hard-requires Doctrine ORM

`DependencyInjection/Configuration.php:19`, `Resources/config/profiling.xml:14-19`, `Profiling/SqlProfiler.php:19-24`

`Configuration` flips `profiling` to `canBeDisabled()` (enabled by default) merely because `ProfilerCoordinator` exists. `profiling.xml` then unconditionally registers `SqlProfiler` (autowired constructor requiring `EntityManagerInterface`) and `ProfilingMiddleware` tagged `doctrine.middleware`. In an application that has `codraw/profiling` but no Doctrine ORM, `SqlProfiler` is kept alive through the `ProfilerCoordinator` tagged-iterator argument and container compilation fails. The profiling config should also be conditioned on Doctrine's presence (or the SQL services split out and guarded).

#### M5. `PublicPass` makes every container definition public

`DependencyInjection/Compiler/PublicPass.php:11-14`

Running at `TYPE_AFTER_REMOVING`, this pass flips *all* definitions of the application's test container to public. This bloats the compiled container (every service gets a public factory method / entry), hides real service-visibility mistakes from tests (a service that would be removed or unreachable in prod is happily fetchable in tests), and duplicates what `framework.test: true` already provides via `test.service_container`. At minimum this deserves an opt-out; ideally the bundle should rely on the standard `TestContainer` instead.

#### M6. `DeleteTemporaryEntityExtension` boots a kernel for every finished `KernelTestCase` suite

`PHPUnit/Extension/DeleteTemporaryEntity/DeleteTemporaryEntityExtension.php:40-46`

`ReflectionAccessor::callMethod($class, 'getContainer')` will *boot* a fresh kernel if none is booted (which is the common case at suite teardown, especially combined with `KernelShutdownExtension`, which shuts the kernel down after every test). So every `KernelTestCase` subclass in the run pays a kernel boot + container compile-load just to check whether a cleaner service exists — even when `BaseTemporaryEntityCleaner::$temporaryEntities` is empty. An early exit when there is nothing to clean (or lazily checking a static registry before touching the container) would avoid a significant slowdown on large suites.

### Low

#### L1. `'_invoke'` instead of `'__invoke'`

`Messenger/HandleMessagesMappingProvider.php:22`, `Messenger/HandlerConfigurationDumper.php:45`

The default messenger handler method is `__invoke` (two underscores). Both the provider and the XML dumper label it `_invoke`. Snapshot comparisons remain internally consistent, but the dumped `method="_invoke"` attribute misdocuments the configuration, and users of the (deprecated) `assertHandlerMessageConfiguration()` must write the misspelled `'_invoke'` in their expectations.

#### **[FIXED]** L2. Unchecked filesystem operations in `EventDispatcherTesterTrait`

`EventDispatcher/EventDispatcherTesterTrait.php:37-38, 69-75`

`tempnam()` may return `false` (then `file_put_contents(false, ...)` fails obscurely); `\DOMDocument::load($filePath)` returns `false` on malformed XML and the code continues on an empty DOM, silently writing back an empty file; the shutdown `unlink` is registered only after `cleanEventDispatcherFile()`, so a cleaner exception leaks the temp file.

#### **[FIXED]** L3. `KernelShutdownExtension` fragile assumptions

`PHPUnit/Extension/KernelShutdown/KernelShutdownExtension.php:25-27, 48-53`

`\assert($test instanceof TestMethod)` — with `zend.assertions=0` (or a `.phpt` test in the suite) a non-`TestMethod` reaches `->className()` and fatals. Similarly, any class exposing `ensureKernelShutdown()` is assumed to also have static `kernel` and `class` properties; a `ReflectionException` escapes otherwise. Guards (`instanceof` check, `hasProperty`) would make the extension robust in mixed suites.

#### **[FIXED]** L4. `AutowireServiceMock` crashes unhelpfully on non-intersection property types

`PHPUnit/Extension/SetUpAutowire/AutowireServiceMock.php:40`

`\assert($type instanceof \ReflectionIntersectionType)` — when the attribute is used without an explicit service id on a property typed as a plain class (no `MockObject&Foo` intersection), the code either fails an assert or calls `getTypes()` on a `ReflectionNamedType` and fatals, instead of throwing the intended descriptive `RuntimeException`.

#### L5. `BaseTemporaryEntityCleaner` details

`PHPUnit/Extension/DeleteTemporaryEntity/BaseTemporaryEntityCleaner.php:16-33`

Uses the default entity manager for every entity class (`getClassMetadata` on the wrong EM throws for entities managed elsewhere — `getManagerForClass()` would be correct, as `AutowireReloadedEntity` already does); entries are `unset` from the static list before `flush()`, so a failed flush permanently drops the cleanup of those entities; and the public static array is shared mutable global state across the whole run (acceptable for a test tool, but worth documenting).

#### **[FIXED]** (partially) L6. `AutowireEntity` minor issues

`PHPUnit/Extension/SetUpAutowire/AutowireEntity.php:37-59`

Uses `->getManager()` (default EM) rather than `getManagerForClass($class)`; when `findOneBy` returns `null` the property assignment produces a `TypeError` on a non-nullable typed property with no hint about the failed criteria; and the "must have a type hint" error message is also emitted for union types, where the real problem is ambiguity, not absence.

---

## Strengths

- **Modern, clean codebase**: PHP 8 attributes, constructor promotion, enums-aware DBAL 4 middleware signatures, `#[\SensitiveParameter]` on connection params (`ProfilingDriver.php:22`), final/internal markers on internals.
- **Good snapshot-testing UX**: `assertEventDispatcherConfiguration()` and `assertMessageHandlerConfiguration()` auto-create the expected fixture on first run and fail with a clear "review and commit it" message — a pragmatic workflow.
- **Sensible DBAL middleware profiling design**: `ProfilingMiddleware` → `ProfilingDriver` → `ProfilingConnection` / `ProfilingStatement` mirrors the official Doctrine middleware architecture, with cheap `isEnabled()` guards so the overhead is negligible when profiling is off.
- **Careful deprecation hygiene**: deprecated traits use `trigger_deprecation()` with version and replacement (`Messenger/MessageHandlerAssertionTrait.php:8`, `EventDispatcher/EventListenerTestTrait.php:7`).
- **Conditional configuration**: `Configuration` adapts defaults via `class_exists(ProfilerCoordinator::class)` and `DrawTesterBundle::boot()` guards on container presence, so the bundle degrades gracefully (modulo M4).
- **Static analysis discipline**: phpstan level 5 across the whole package with an *empty* baseline (`phpstan-baseline.neon`), plus a custom phpstan property-extension (`extension.neon`) so autowired test properties aren't flagged as never-written.
- **Thoughtful test-container workaround**: `KernelTestCaseAutowireDependentTrait::getPublicContainer()` documents and solves the real `TestContainer::set()`-routes-to-privates pitfall when replacing services with mocks.

---

## Test Coverage

Coverage is concentrated on the DI layer and one or two leaf classes; the helpers that consuming projects actually call are mostly untested.

**Covered:**

- `QueryCollector` — thorough unit tests (`Tests/Profiling/Sql/QueryCollectorTest.php`): start/stop, disabled paths, multiple queries, reset.
- `MessengerPass` — argument copying verified (`Tests/DependencyInjection/Compiler/MessengerPassTest.php`).
- `DrawTesterExtension` / `Configuration` — service registration with and without profiling, invalid config, autoconfiguration tag (`Tests/DependencyInjection/*`).
- `TransportTester` + `AutowireTransportTester` — real integration through `Tests/AppKernel.php` with an in-memory transport, including the failure path.

**Not covered (notably including both high-severity findings):**

- `WebTestCase` / `createJsonClient()` (H1) and `JsonResponseAssertionsTrait`
- `JWTLoginTrait` (H2)
- `TemplatedMailerAssertionsTrait` / `TemplatedEmailCount` (M1)
- `EventDispatcherTesterTrait` (including the DOM-cleaning regexes) and `EventListenerTestTrait`
- All PHPUnit runner extensions: `KernelShutdownExtension`, `DoctrineTransactionExtension`, `DeleteTemporaryEntityExtension`
- All `SetUpAutowire` attributes except `AutowireTransportTester` (i.e. `AutowireClient`, `AutowireEntity`, `AutowireReloadedEntity`, `AutowireServiceMock`, `AutowireParameter`, `AutowireAdminUserEntity`, `AutowireLoggerTester`)
- `SqlProfiler` (param sanitization / substitution logic), `ProfilingConnection` / `ProfilingStatement` / `ProfilingDriver` / `ProfilingMiddleware`
- `HandlerConfigurationDumper::xmlDump()`, `HandleMessagesMappingProvider`, `CompilerPass`, `PublicPass`

The correlation is telling: every functional bug found in this review sits in untested code. Adding even shallow integration tests for the web-test and mailer helpers (the package's headline features) would have caught H1, H2, and M1.
