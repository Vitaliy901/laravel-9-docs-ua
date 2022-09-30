# Постачальник Служб

- [Вступ](#introduction)
- [Написання постачальників служб](#writing-service-providers)
  - [Метод `Register`](#the-register-method)
  - [Метод `Boot`](#the-boot-method)
- [Реєстрування постачальників](#registering-providers)
- [Відкладені постачальники](#deferred-providers)

<a name="introduction"></a>

## Вступ

Постачальники служб - це місце зосередження початкового завантаження всіх додатків Laravel. Ваш власний додаток, та також всі основні служби Laravel, завантажуються через постачальників служб.

Але, що ми маємо на увазі під «початковим завантаженням»? Загалом ми маємо на увазі **реєстрацію** речей, включаючи реєстрацію зв'язування з контейнером служб, слухачів подій, посередників і навіть маршрутів. Постачальники служб - це місце зосередження, в якому налаштовується ваш додатка.

Якщо ви відкриєте файл `config/app.php`, який вже присутній в Laravel, ви побачите масив `providers`. Всі ці класи постачальників служб, будуть завантажені для вашого додатка. За замовчуванням, набір основних постачальників послуг Laravel, перераховні в цьому масиві. Ці постачальники завантажують основні компоненти Laravel, такі як поштомат, черги, кеш, та інші. Багато з цих постачальників є «відкладеними», це означає, що вони не будуть завантажуватись за кожним запитом, а лише тоді, коли служби, які вони надають, дійсно потрібній.

В цьому огляді, ви будете вивчати як написати ваші власні постачальники служб, так як зареєструвати їх у вашому додатку Laravel.

> **Note**  
> Якщо ви бажаєте дізнатися більше про те, як Laravel працює в середині і опрацьовує запити, перегляньте нашу документацію по [життевому циклу запитів](lifecycle.md) Laravel.

<a name="writing-service-providers"></a>

## Написання постачальників служб

Всі постачальники служб розширюють клас `Illuminate\Support\ServiceProvider`. Більшість постачальників служб містять методи `register` і `boot`. **Зв'язування сутностей з [контейнером служб](container.md) здійснюється лише** в методі `register`. Ви ніколи не повинні реєструвати слухачів подій, маршрути, або будь-які інші частини функціонала в методі `register`.

Artisan CLI може створити нового постачальника за допомогою команди `make:provider`:

```shell
php artisan make:provider RiakServiceProvider
```

<a name="the-register-method"></a>

### Метод `Register`

Як згадувалося раніше, зв'язування сутностей з [контейнером служб](container.md) здійснюється лише в методі `register`. Ви ніколи не повинні реєструвати слухачів подій, маршрути, або будь-які інші частини функціонала в методі `register`. Інакше, ви можете випадково використати службу, надану постачальником служб, яка ще не завантажена.

Давайте поглянемо на базовий постачальник служб. У будь-якому методі вашого постачальника служб, ви завжди маєте доступ до властивості `$app`, яка в свою чергу надає доступ до контейнера служби:

```php
<?php

namespace App\Providers;

use App\Services\Riak\Connection;
use Illuminate\Support\ServiceProvider;

class RiakServiceProvider extends ServiceProvider
{
    /**
     * Зареєструвати будь-які служби додатка.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton(Connection::class, function ($app) {
            return new Connection(config('riak'));
        });
    }
}
```

Цей постачальник служб визначає лише метод `register` та використовує цей метод для визначення реалізації `App\Services\Riak\Connection` в контейнері служби. Якщо ви ще не знайомі з контейнером служб Laravel, перегляньте [його документацію](container.md).

<a name="the-bindings-and-singletons-properties"></a>

#### Властивості `bindings` і `singletons`

Якщо ваш постачальник служб реєструє багато простих зв'язувань, ви можете використовувати властивості `bindings`, та `singletons` замість ручної реєстрації кожного зв’язування з контейнером. Коли фреймворк завантажує постачальника служб, він автоматично перевіряє ці властивості та реєструє їх зв'язування:

```php
<?php

namespace App\Providers;

use App\Contracts\DowntimeNotifier;
use App\Contracts\ServerProvider;
use App\Services\DigitalOceanServerProvider;
use App\Services\PingdomDowntimeNotifier;
use App\Services\ServerToolsProvider;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Всі зв'язування з контейнером, які потрібно зареєструвати.
     *
     * @var array
     */
    public $bindings = [
        ServerProvider::class => DigitalOceanServerProvider::class,
    ];

    /**
     * Всі singletons (одинаки) контейнерів, які потрібно зареєструвати.
     *
     * @var array
     */
    public $singletons = [
        DowntimeNotifier::class => PingdomDowntimeNotifier::class,
        ServerProvider::class => ServerToolsProvider::class,
    ];
}
```

<a name="the-boot-method"></a>

### Метод `Boot`

Отже, а що, якщо нам потрібно зареєструвати [view composer](views.md#view-composers) в нашому постачальнику служб? Це слід зробити в методі `boot`. **Цей метод викликається після реєстрації всіх інших постачальників служб**, тобто ви маєте доступ до всіх інших служб, зареєстрованих фреймворком:

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\View;
use Illuminate\Support\ServiceProvider;

class ComposerServiceProvider extends ServiceProvider
{
    /**
     * Завантаження будь-яких служб додатка.
     *
     * @return void
     */
    public function boot()
    {
        View::composer('view', function () {
            //
        });
    }
}
```

<a name="boot-method-dependency-injection"></a>

#### Впровадження залежності в методі `boot`

Ви можете оголосити тип залежностей для метода `boot` вашого постачальника служб. [Контейнер служб](container.md) автоматично додасть будь-які потрібні вам залежності:

```php
use Illuminate\Contracts\Routing\ResponseFactory;

/**
 * Завантаження будь-яких служб додатка.
 *
 * @param  \Illuminate\Contracts\Routing\ResponseFactory  $response
 * @return void
 */
public function boot(ResponseFactory $response)
{
    $response->macro('serialized', function ($value) {
        //
    });
}
```

<a name="registering-providers"></a>

## Реєстрація постачальників

Всі постачальники служб реєструються в конфігураційному файлі `config/app.php`. Цей файл містить масив `providers`, де ви можете перерахувати імена класів ваших постачальників служб. За замовчуванням, в цьому масиві перераховано набір основних постачальників служб Laravel. Ці постачальники завантажують основні компоненти Laravel, такі як поштомат, черги, кеш та інші.

Щоб зареєструвати свій постачальник служб, просто додайте його до масива:

```php
'providers' => [
    // Інші постачальники служб.

    App\Providers\ComposerServiceProvider::class,
],
```

<a name="deferred-providers"></a>

## Відкладені постачальники

Якщо ваш постачальник реєструє **лише** зв'язування з [контейнером служб](container.md), ви можете відкласти його реєстрацію, доки одне з зареєстрованих зв'язувань дійсно не знадобиться. Відкладення завантаження такого постачальника покращить продуктивність вашого додатка, оскільки він не завантажується з файлової системи за кожним запитом.

Laravel компілює та зберігає список всіх служб, які надають відкладені постачальники служб, разом із назвою класа постачальників служб. Далі, Laravel завантажить постачальника служб, лише за необхідності в одній із цих служб.

Щоб відкласти завантаження постачальника служб, реалізуйте інтерфейс `\Illuminate\Contracts\Support\DeferrableProvider` і визначте метод `provides`. Метод `provides` має віддати зв'язування контейнера служб, які зареєстровані постачальником служб:

```php
<?php

namespace App\Providers;

use App\Services\Riak\Connection;
use Illuminate\Contracts\Support\DeferrableProvider;
use Illuminate\Support\ServiceProvider;

class RiakServiceProvider extends ServiceProvider implements DeferrableProvider
{
    /**
     * Зареєструвати будь-які служби додатка.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton(Connection::class, function ($app) {
            return new Connection($app['config']['riak']);
        });
    }

    /**
     * Отримати служби, надані постачальником служб.
     *
     * @return array
     */
    public function provides()
    {
        return [Connection::class];
    }
}
```
