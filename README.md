# AliceBundle

A [Symfony](http://symfony.com) bundle for populating development and test databases from fixture files. It discovers shared and environment-specific fixtures, loads them through Doctrine ORM, and provides PHPUnit helpers for resetting database state.

[nelmio/alice](https://github.com/nelmio/alice) creates objects with generated data from [Faker](https://github.com/fzaninotto/Faker); [FidryAliceDataFixtures](https://github.com/theofidry/AliceDataFixtures) handles persistence and purging. AliceBundle adds Symfony integration, fixture discovery, service factories, application parameters, and database testing helpers. If you only need to load explicit fixture files, FidryAliceDataFixtures may be sufficient. Consult that project's documentation for its database support; this bundle's command uses Doctrine ORM.

**This documentation covers HautelookAliceBundle 2.x. For 1.x, see the [1.x documentation](https://github.com/hautelook/AliceBundle/tree/1.x).**

[![Package version](https://img.shields.io/packagist/v/hautelook/alice-bundle.svg?style=flat-square)](https://packagist.org/packages/hautelook/alice-bundle)
[![Build Status](https://img.shields.io/travis/hautelook/AliceBundle/master.svg?style=flat-square)](https://travis-ci.org/hautelook/AliceBundle?branch=master)
[![SensioLabsInsight](https://img.shields.io/sensiolabs/i/d93a3fc4-3fe8-4be3-aa62-307f53898199.svg?style=flat-square)](https://insight.sensiolabs.com/projects/d93a3fc4-3fe8-4be3-aa62-307f53898199)
[![Dependency Status](https://www.versioneye.com/user/projects/55d26478265ff6001a000084/badge.svg?style=flat)](https://www.versioneye.com/user/projects/55d26478265ff6001a000084)
[![Scrutinizer Code Quality](https://img.shields.io/scrutinizer/g/hautelook/AliceBundle.svg?style=flat-square)](https://scrutinizer-ci.com/g/hautelook/AliceBundle/?branch=master)
[![Code Coverage](https://img.shields.io/scrutinizer/coverage/g/hautelook/AliceBundle.svg?b=master&style=flat-square)](https://scrutinizer-ci.com/g/hautelook/AliceBundle/?branch=master)
[![Slack](https://img.shields.io/badge/slack-%23alice--fixtures-red.svg?style=flat-square)](https://symfony-devs.slack.com/shared_invite/MTYxMjcxMjc0MTc5LTE0OTA3ODE4OTQtYzc4NWVmMzRmZQ)

## Installation

Use Composer and a Symfony application compatible with [composer.json](composer.json): PHP `^7.3 || ^8.0`, Symfony FrameworkBundle `^3.4.24 || ^4.0 || ^5.0`, and Doctrine ORM `^2.5.11`. DoctrineBundle and Doctrine Data Fixtures are required dependencies.

From your application's root directory, with [Symfony Flex](https://symfony.com/doc/current/setup/flex.html):

```sh
# If Doctrine is not installed yet:
composer require doctrine-orm

composer require --dev hautelook/alice-bundle
```

These commands modify the application's Composer dependencies.

Without Flex, install the bundle package and register the bundles in your kernel, such as `app/AppKernel.php`:

```php
public function registerBundles()
{
    $bundles = [
        new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
        // Other application bundles...
        new Doctrine\Bundle\DoctrineBundle\DoctrineBundle(),
    ];

    if (in_array($this->getEnvironment(), ['dev', 'test'])) {
        $bundles[] = new Nelmio\Alice\Bridge\Symfony\NelmioAliceBundle();
        $bundles[] = new Fidry\AliceDataFixtures\Bridge\Symfony\FidryAliceDataFixturesBundle();
        $bundles[] = new Hautelook\AliceBundle\HautelookAliceBundle();
    }

    return $bundles;
}
```

Configure fixture discovery in each environment where you use the bundle. For a Flex application, use the following in `config/packages/dev/hautelook_alice.yaml`, and configure the test environment similarly when needed:

```yaml
hautelook_alice:
    fixtures_path: 'fixtures'
    root_dirs:
        - '%kernel.project_dir%'
```

`fixtures_path` accepts a string or an array of paths relative to each root directory. Its code default is `Resources/fixtures`, so configure it explicitly for a project-level `fixtures` directory. Older applications can use `Resources/fixtures` under the application root or configured bundle directories. Default roots include `%kernel.root_dir%` on Symfony versions before 5 and `%kernel.project_dir%` on all supported versions.

## Basic usage

First configure your application's [Doctrine](http://www.doctrine-project.org/projects/orm.html) connection and create the schema for the entities you intend to populate. The required packages include [DoctrineBundle](https://github.com/doctrine/DoctrineBundle) and [Doctrine Data Fixtures](https://github.com/doctrine/data-fixtures).

For application entities with the corresponding writable properties, create:

```yaml
# fixtures/dummy.yaml
App\Entity\Dummy:
    dummy_{1..10}:
        name: <name()>
        related_dummy: '@related_dummy*'
```

```yaml
# fixtures/related_dummy.yaml
App\Entity\RelatedDummy:
    related_dummy_{1..10}:
        name: <name()>
```

From the application root, load the development fixtures:

```sh
php bin/console hautelook:fixtures:load --env=dev
```

**This purges the selected database before inserting fixtures.** Use a disposable development or test database. Interactive execution asks for confirmation; non-interactive execution skips that prompt. `--append` retains existing data; it cannot be combined with `--purge-with-truncate`.

Discovery includes YAML and PHP files directly inside each configured fixture directory and its selected environment subdirectory, such as `fixtures/dev`. It does not recursively scan deeper directories. Older application layouts commonly use `app/Resources/fixtures` or `src/AppBundle/Resources/fixtures`, with matching discovery configuration.

Prefer explicit `root_dirs` for selecting fixture locations. The legacy `--bundle`/`-b` selection is deprecated; the command defaults to skipping automatic bundle discovery. Database selection options include `--manager` and `--shard`.

## Database testing

The helpers in `Hautelook\AliceBundle\PhpUnit`, [inspired by Laravel](https://laravel.com/docs/5.6/database-testing#resetting-the-database-after-each-test), work with tests extending Symfony's `KernelTestCase`, including `WebTestCase` and [Panther](https://github.com/symfony/panther) tests. They are useful for [functional testing](https://symfony.com/doc/current/testing.html#functional-tests):

- `RefreshDatabaseTrait` populates fixtures on the first kernel boot, then starts a transaction on each boot and rolls it back during kernel shutdown.
- `ReloadDatabaseTrait` purges and reloads fixtures on every kernel boot. Use this when changes happen in another process, as with Panther, and cannot be isolated by the test process's transaction.
- `RecreateDatabaseTrait` drops and recreates the mapped schema, then loads fixtures on every kernel boot. This supports in-memory SQLite testing and destroys existing mapped data.

For example:

```php
namespace App\Tests;

use Hautelook\AliceBundle\PhpUnit\RefreshDatabaseTrait;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class NewsTest extends WebTestCase
{
    use RefreshDatabaseTrait;

    public function testNewsPage()
    {
        $client = static::createClient();
        $client->request('GET', '/my-news');
        self::assertSame(200, $client->getResponse()->getStatusCode());
    }
}
```

Replace the imported and used trait with `ReloadDatabaseTrait` for a `PantherTestCase`, or `RecreateDatabaseTrait` for schema recreation.

Configure the protected static properties in `setUpBeforeClass()` when needed: `$manager`, `$bundles`, `$append`, `$purgeWithTruncate`, `$shard`, and `$connection`. The shared [BaseDatabaseTrait](src/PhpUnit/BaseDatabaseTrait.php) also exposes loaded objects through `$fixtures`. Purging with TRUNCATE defaults to enabled for these helpers. To append, set both `$append = true` and `$purgeWithTruncate = false`.

## Documentation

- [Advanced usage](doc/advanced-usage.md): [database support](doc/advanced-usage.md#enabling-databases), [environment-specific fixtures](doc/advanced-usage.md#environment-specific-fixtures), [fixture parameters](doc/advanced-usage.md#fixtures-parameters), [Alice parameters](doc/advanced-usage.md#alice-parameters), [application parameters](doc/advanced-usage.md#application-parameters), [service factories](doc/advanced-usage.md#use-service-factories), and [loading order](doc/advanced-usage.md#load-fixtures-in-a-specific-order), including [file ordering](doc/advanced-usage.md#ordering-the-files-found) and [persistence ordering](doc/advanced-usage.md#persisting-the-classes-in-a-specific-order).
- [Custom Faker providers](doc/faker-providers.md).
- [Custom Alice processors](doc/alice-processors.md).
- [Upgrade guide](UPGRADE.md), including [1.x to 2.x](UPGRADE.md#from-1x-to-2x) and [0.2 to 1.x](UPGRADE.md#from-02-to-1x).
- [Changelog](CHANGELOG.md).

## Repository development

Implementation lives in `src/`, service wiring in `resources/config/`, and unit/integration tests in `tests/`, supported by application and fixture examples in `fixtures/`.

The [CI workflow](.github/workflows/ci.yml) defines dependency preparation, MySQL setup, and PHP version checks. After preparing that test environment and installing development dependencies, its test command runs from the repository root:

```sh
vendor/bin/phpunit --verbose --stop-on-failure
```

[phpunit.xml.dist](phpunit.xml.dist) selects the suite and bootstrap. Database tests reset their configured database; use a dedicated test database. CI also runs `php-cs-fixer fix --dry-run --diff --ansi` from the repository root with PHP-CS-Fixer installed separately.

## Resources

- Behat extension: [AliceBundleExtension](https://github.com/theofidry/AliceBundleExtension).
- Generate compatible fixtures from Doctrine entities: [AliceGeneratorBundle](https://github.com/trappar/AliceGeneratorBundle).

## Credits

This bundle was originally developed by [Baldur RENSCH](https://github.com/baldurrensch) and [HauteLook](https://github.com/hautelook). The existing project documentation identifies [Théo FIDRY](https://github.com/theofidry) as its maintainer.

[Other contributors](https://github.com/hautelook/AliceBundle/graphs/contributors).

## License

[![license](https://img.shields.io/badge/license-MIT-red.svg?style=flat-square)](LICENSE)
