# Посередник

- [Вступ](#introduction)
- [Визначення посередника](#defining-middleware)
- [Реєстрація посередника](#registering-middleware)
  - [Глобальний стек HTTP-посередників](#global-middleware)
  - [Призначення посередників маршрутам](#assigning-middleware-to-routes)
  - [Групи посередників](#middleware-groups)
  - [Сортування посередників](#sorting-middleware)
- [Параметри посередника](#middleware-parameters)
- [Завершальний посередник](#terminable-middleware)

<a name="introduction"></a>

## Вступ

Посередник забезпечує зручний механізм для перевірки та фільтрації HTTP-запитів, які надходять до вашого додатка. Наприклад, Laravel має посередник, який перевіряє, чи автентифікований користувача вашого додатка. Якщо користувач не автентифікований, посередник перенаправить користувача на сторінку входу вашого додатка. Однак, якщо користувач автентифікований, посередник дозволить запиту перейти далі в додаток.

Додатковий посередник може бути написаний для виконання різноманітних завдань, крім автентифікації. Наприклад, посередник логування може записувати в _log_ всі вхідні запити до вашого додатка. До складу Laravel входить декілька посередників, зокрема посередник для автентифікації та захисту CSRF. Всі ці посередники розташовані в каталозі `app/Http/Middleware`.

<a name="defining-middleware"></a>

## Визначення посередника

Щоб створити новий посередник, скористайтеся командою Artisan `make:middleware`:

```shell
php artisan make:middleware EnsureTokenIsValid
```

Ця команда розмістить новий клас `EnsureTokenIsValid` у вашому каталозі `app/Http/Middleware`. В цьому посереднику ми дозволимо доступ до маршрута, лише якщо наданий вхідний `token` відповідає вказаному значенню. В іншому випадку ми перенаправимо користувачів назад до `home` URI:

```php
<?php

namespace App\Http\Middleware;

use Closure;

class EnsureTokenIsValid
{
    /**
     * Опрацювати вхідний запит.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if ($request->input('token') !== 'my-secret-token') {
            return redirect('home');
        }

        return $next($request);
    }
}
```

Як ви бачите, якщо даний `token` не відповідає нашому секретниму токену, посередник поверне HTTP-перенаправлення клієнту; інакше запит буде передано далі в додаток. Щоб передати запит глибше в додаток (дозволяючи «пройти» посередник), ви повинні викликати зворотний виклик `$next` з `$request`.

Щоб краще уявити собі посередника, дивіться на них як на серію «шарів», через які мають пройти HTTP-запити, перш ніж вони потраплять до вашого додатка. Кожен шар може перевірити запит і навіть повністю його відхилити.

> **Note**  
> Всі посередники проходять через [контейнер служб](container.md), тому ви можете вказати будь-які потрібні вам типи залежностей в конструкторі посередника.

<a name="before-after-middleware"></a>
<a name="middleware-and-responses"></a>

#### Посередники та Відповіді

Звичайно, посередник може виконувати завдання до або після передачі запита в додаток. Наприклад, наступний посередник виконає певне завдання **до** того, як запит буде опрацьовано додатком:

```php
<?php

namespace App\Http\Middleware;

use Closure;

class BeforeMiddleware
{
    public function handle($request, Closure $next)
    {
        // Виконати дію

        return $next($request);
    }
}
```

А цей посередник, виконає своє завдання **після** того, як додаток опрацює вхідний запит:

```php
<?php

namespace App\Http\Middleware;

use Closure;

class AfterMiddleware
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

         // Виконати дію

        return $response;
    }
}
```

<a name="registering-middleware"></a>

## Реєстрація посередника

<a name="global-middleware"></a>

### Глобальний стек HTTP-посередників

Якщо ви хочете, щоб посередник запускався під час кожного запита HTTP до вашого додатка, вкажіть клас посередника у властивості `$middleware` вашого класа `app/Http/Kernel.php`.

<a name="assigning-middleware-to-routes"></a>

### Призначення посередників маршрутам

Якщо ви бажаєте призначити посередник для певних маршрутів, вам слід спочатку визначити йому ключ у файлі `app/Http/Kernel.php` вашого додатка. За замовчуванням властивість `$routeMiddleware` цього класа вже містить записи про посередники, присутні з Laravel. Ви можете додати власний посередник до цього списку та визначити йому ключ на свій вибір:

```php
// Within App\Http\Kernel class...

protected $routeMiddleware = [
    'auth' => \App\Http\Middleware\Authenticate::class,
    'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
    'bindings' => \Illuminate\Routing\Middleware\SubstituteBindings::class,
    'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class,
    'can' => \Illuminate\Auth\Middleware\Authorize::class,
    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
    'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
];
```

Після визначення посередника в HTTP-kernel ви можете використовувати метод `middleware`, щоб призначити посередник маршруту:

```php
Route::get('/profile', function () {
    //
})->middleware('auth');
```

Ви можете призначити декілька посередників, передавши масив імен посередників методу `middleware`:

```php
Route::get('/', function () {
    //
})->middleware(['first', 'second']);
```

При призначенні посередника ви також можете передати повне ім’я класа:

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', function () {
    //
})->middleware(EnsureTokenIsValid::class);
```

<a name="excluding-middleware"></a>

#### Виключення посередника

Призначаючи посередника групі маршрутів, іноді може знадобитися заборонити застосуванню посередника окремому маршруту в цій групі. Ви можете зробити це за допомогою метода `withoutMiddleware`:

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::middleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/', function () {
        //
    });

    Route::get('/profile', function () {
        //
    })->withoutMiddleware([EnsureTokenIsValid::class]);
});
```

Ви також можете виключити певний набір посередників для [групи](routing.md#route-groups) визначених маршрутів:

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/profile', function () {
        //
    });
});
```

Метод `withoutMiddleware` може видалити лише посередники маршрутів і не застосовується до [глобальних посередників](#global-middleware).

<a name="middleware-groups"></a>

### Група посередників

Іноді вам може знадобитися згрупувати декілька посередників під одним ключем, щоб полегшити їх призначення маршрутам. Ви можете зробити це за допомогою властивості `$middlewareGroups` у вашому HTTP-kernel.

З коробки Laravel постачається з групами посередників для `web` та `api`, які містять загальні посередники, які ви можете застосувати до своїх маршрутів `web.php` та `api.php`. Пам’ятайте, що ці групи посередників автоматично застосовуються постачальником служб `App\Providers\RouteServiceProvider` вашого додатка до маршрутів у ваших відповідних файлах маршрутів `web.php` та `api.php`:

```php
/**
 * Групи посередників маршрутів додатка.
 *
 * @var array
 */
protected $middlewareGroups = [
    'web' => [
        \App\Http\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \App\Http\Middleware\VerifyCsrfToken::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],

    'api' => [
        'throttle:api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],
];
```

Групи посередників можуть бути призначені для маршрутів і дій контролера, використовуючи той самий синтаксис, що й окремий посередник. Знову ж таки, групи посередників дозволяють призначити одночасно багато посередників на маршрут або групу маршрутів, що дуже зручно:

```php
Route::get('/', function () {
    //
})->middleware('web');

Route::middleware(['web'])->group(function () {
    //
});
```

> **Note**  
> З коробки, групи посередники для `web` and `api` автоматично застосовуються до відповідних файлів `route/web.php` і `routes/api.php` вашого додатка `App\Providers\RouteServiceProvider`.

<a name="sorting-middleware"></a>

### Сортування посередників

Іноді вам може знадобитися, щоб ваші посередники виконувалися в певному порядку, але ви не можете контролювати їх порядок, коли вони призначені маршруту. У цьому випадку ви можете вказати пріоритет ваших посередників за допомогою властивості `$middlewarePriority` у вашому файлі `app/Http/Kernel.php`. Ця властивість може не існувати у вашому HTTP-kernel за замовчуванням. Якщо вона не існує, ви можете скопіювати її приклад за замовчуванням нижче:

```php
/**
* Список посередників, відсортований за пріоритетністю.
*
* Змусить неглобальні посередники завжди бути у заданому порядку.
*
* @var string[]
*/
protected $middlewarePriority = [
    \Illuminate\Cookie\Middleware\EncryptCookies::class,
    \Illuminate\Session\Middleware\StartSession::class,
    \Illuminate\View\Middleware\ShareErrorsFromSession::class,
    \Illuminate\Contracts\Auth\Middleware\AuthenticatesRequests::class,
    \Illuminate\Routing\Middleware\ThrottleRequests::class,
    \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
    \Illuminate\Contracts\Session\Middleware\AuthenticatesSessions::class,
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
    \Illuminate\Auth\Middleware\Authorize::class,
];
```

<a name="middleware-parameters"></a>

## Параметри посередника

Посередник також може отримувати додаткові параметри. Наприклад, якщо вашому додатку потрібно перевірити, чи має автентифікований користувачь відповідну "роль" перед виконанням заданої дії, ви можете створити посередник `EnsureUserHasRole`, який отримуватиме назву ролі як додатковий аргумент.

Додаткові параметри посередника будуть передані йому після аргумента `$next`:

```php
<?php

namespace App\Http\Middleware;

use Closure;

class EnsureUserHasRole
{
    /**
     * Опрацювати вхідний запит.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @param  string  $role
     * @return mixed
     */
    public function handle($request, Closure $next, $role)
    {
        if (! $request->user()->hasRole($role)) {
            // Перенаправити...
        }

        return $next($request);
    }

}
```

Параметри посредника можна вказати під час визначення маршрута, розділивши ім’я посередника та параметри символом `:`. Декілька параметрів повинні бути розділені комами:

```php
Route::put('/post/{id}', function ($id) {
    //
})->middleware('role:editor');
```

<a name="terminable-middleware"></a>

## Завершальний посередник

Іноді посереднику може знадобитися виконати певну роботу після того, як відповідь HTTP вже буде надіслано браузеру. Якщо ви визначаєте метод `terminate` у своєму посереднику, а ваш веб-сервер використовує FastCGI, буде автоматично викликано метод `terminate`, після надсилання відповіді браузеру:

```php
<?php

namespace Illuminate\Session\Middleware;

use Closure;

class TerminatingMiddleware
{
    /**
     * Опрацювати вхідний запит.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        return $next($request);
    }

    /**
     * Виконуйте завдання після того, як відповідь буде надіслано браузеру.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Illuminate\Http\Response  $response
     * @return void
     */
    public function terminate($request, $response)
    {
        // ...
    }
}
```

Метод `terminate` повинен отримати як запит, так і відповідь. Після того, як ви визначили посередник з можливістю завершення, ви повинні додати його до списку маршрутів або глобальних посередників у файлі `app/Http/Kernel.php`.

Під час виклику методу `terminate` у вашому посереднику, Laravel впровадить новий екземпляр посередника з [контейнера служби](container.md). Якщо ви бажаєте використовувати той самий екземпляр посередника під час виклику методів `handle` та `terminate`, зареєструйте посередник в контейнері за допомогою метода `singleton`. Зазвичай це потрібно зробити в методі `register` вашого `AppServiceProvider`:

```php
use App\Http\Middleware\TerminatingMiddleware;

/**
 * Зареєструвати будь-які служби додатка.
 *
 * @return void
 */
public function register()
{
    $this->app->singleton(TerminatingMiddleware::class);
}
```
