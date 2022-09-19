# Генерація URL-адрес

- [Вступ](#introduction)
- [Основи](#the-basics)
  - [Генерація URL-адрес](#generating-urls)
  - [Доступ до поточного URL](#accessing-the-current-url)
- [URL-адреси іменованих маршрутів](#urls-for-named-routes)
  - [Підписання URL-адрес](#signed-urls)
- [URL-адреси за діями контролера](#urls-for-controller-actions)
- [Значення за замовчуванням](#default-values)

<a name="introduction"></a>

## Вступ

Laravel надає декілька помічників, щоб допомогти вам у створенні URL-адрес для вашого додатка. В першу чергу ці помічники корисні під час створення посилань в ваших шаблонах та в API відповідях, або потрібно створити відповідь перенаправлення до іншої частини вашого додатка.

<a name="the-basics"></a>

## Основи

<a name="generating-urls"></a>

### Генерація URL-адрес

Пімічник `url` можна використовувати, щоб створити довільні URL-адреси для вашого додатка. Сгенерована URL-адреса автоматично буде використовувати схему (HTTP or HTTPS) та хост з поточного запита, який опрацьовується додатком.

```php
$post = App\Models\Post::find(1);

echo url("/posts/{$post->id}");

// http://example.com/posts/1
```

<a name="accessing-the-current-url"></a>

### Доступ до поточного URL

Якщо помічникові `url` не надано жодного шляху, буде повернуто екземпляр `Illuminate\Routing\UrlGenerator`, що дозволяє вам отримати більше інформації про поточну URL-адресу:

```php
// Отримати поточну URL-адресу без параметрів запита...
echo url()->current();

// Отримати поточну URL-адресу з параметрами запита...
echo url()->full();

// Отримати повну URL-адресу попереднього запита...
echo url()->previous();
```

Кожний з цих методів також доступний через [фасад](facades.md) `URL`:

```php
use Illuminate\Support\Facades\URL;

echo URL::current();
```

<a name="urls-for-named-routes"></a>

## URL-адреси іменованих маршрутів

Помічник `route` може використовуватись для генерації URL-адрес [іменованих маршрутів](routing.md#named-routes). Іменовані маршрути дозволяють створювати URL-адреси без прив'язки до фактичної URL-адреси. Відповідно, якщо URL-адреса маршрута змінюється, вам не потрібно вносити зміни у ваші виклики функції `route`. Наприклад, уявимо, що ваш додаток містить наступний визначений маршрут:

```php
Route::get('/post/{post}', function (Post $post) {
    //
})->name('post.show');
```

Щоб згенерувати URL-адресу цього маршрута, ви можете використати помічника `route` на зразок:
To generate a URL to this route, you may use the `route` helper like so:

```php
echo route('post.show', ['post' => 1]);

// http://example.com/post/1
```

Звичайно, помічник `route` також може використовуватись для генерації URL-адрес маршрутів, які мають декілька параметрів:

```php
Route::get('/post/{post}/comment/{comment}', function (Post $post, Comment $comment) {
    //
})->name('comment.show');

echo route('comment.show', ['post' => 1, 'comment' => 3]);

// http://example.com/post/1/comment/3
```

Будь-які додаткові елементи в масиві, які не відповідають визначеним параметрам маршрутів, будуть додані до URL-адреси як рядок параметрів записта:

```php
echo route('post.show', ['post' => 1, 'search' => 'rocket']);

// http://example.com/post/1?search=rocket
```

<a name="eloquent-models"></a>

#### Моделі Eloquent

Ви будете часто генерувати URL-адреси використовуючи ключ маршрута (зазвичай первинний ключ) [моделі Eloquent](eloquent.md). По цій причині, ви можете передавати модель Eloquent, як значення параметра. Помічник `route` автоматично вилучить первинний ключ моделі для маршрута:

```php
echo route('post.show', ['post' => $post]);
```

<a name="signed-urls"></a>

### Підписання URL-адрес

Laravel дозволяє вам легко створювати "підписані" URL-адреси для іменованих маршрутів. Ці URL-адреси мають хешований "підпис",доданий до рядка запита, який дозволяє Laravel перевірити, що URL-адреса не була змінена з моменту його створення. Підписані URL-адреси особливо корисні для маршрутів, які загальнодоступні, але потребують рівень захисту від маніпуляцій з URL-адресою.

Наприклад, ви можете використовувати підписані URL-адреси для реалізації публічного посилання "відписатися", яке відправляється вашим клієнтам на електронну пошту. Щоб створити підписану URL-адресу для іменованого маршрута, використовуйте метод `signedRoute` фасада `URL`, або помічника `url`:

```php
use Illuminate\Support\Facades\URL;

return URL::signedRoute('unsubscribe', ['user' => 1]);
```

Якщо ви бажаєте сгенерувати для маршрута тимчасову підписану URL-адресу, термін дії якої завершується через визначений проміжок часу, ви можите використовувати метод `temporarySignedRoute`. Коли Laravel перевіряє тимчасовий підпис URL-адреси маршрута, він гарантує, що часова мітка з терміном дії, яка закодована в підписаній URL-адресі, ще не вичерпана.

```php
use Illuminate\Support\Facades\URL;

return URL::temporarySignedRoute(
    'unsubscribe', now()->addMinutes(30), ['user' => 1]
);
```

<a name="validating-signed-route-requests"></a>

#### Валідація запита з підписаним маршрутом

Щоб переконатися, що вхідний запит має дійсний підпис, вам потрібно буде викликати метод `hasValidSignature` вхідного екземпляра `Illuminate\Http\Request`:

```php
use Illuminate\Http\Request;

Route::get('/unsubscribe/{user}', function (Request $request) {
    if (! $request->hasValidSignature()) {
        abort(401);
    }

    // ...
})->name('unsubscribe');
```

Іноді, потрібно дозволити зовнішньому інтерфейсу вашого додатка, додавати дані до підписаної URL-адреси, наприклад, під час пагінації на боці клієнта. Таким чином, ви можете визначити параметри запита, які слід ігнорувати під час перевірки підписаної URL-адреси, використовуючи метод `hasValidSignatureWhileIgnoring`. Пам'ятайте, ігнорування параметрів дозволяє будь-кому змінювати ці параметри:

```php
if (! $request->hasValidSignatureWhileIgnoring(['page', 'order'])) {
    abort(401);
}
```

Замість того, щоб перевіряти підписані URL-адреси за допомогою екземпляра вхідного запита, ви можете призначити [посередика](middleware.md) `Illuminate\Routing\Middleware\ValidateSignature` для маршрута. Якщо його ще не представленно, вам потрібно призначити йому ключ в масиві `routeMiddleware` HTTP-kernel:

```php
/**
 * Посередники маршрутів програми.
 *
 * Ці посередники можуть бути груповими або використовуватися окремо.
 *
 * @var array
 */
protected $routeMiddleware = [
    'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,
];
```

Як тільки ви зареєструєте посередника у вашому файлі `Kernel.php`, ви можете його приеднати до маршрута. Якщо вхідний запит не матиме дійсного підписа, посередник автоматично поверне HTTP відповідь `403`.

```php
Route::post('/unsubscribe/{user}', function (Request $request) {
    // ...
})->name('unsubscribe')->middleware('signed');
```

<a name="responding-to-invalid-signed-routes"></a>

#### Відповідь на недійсні підписи маршрутів

Коли хтось відвідує підписану URL-адресою термін дії якої минув, вони отримають сгенеровану строінку з кодом помилки `403` статуса HTTP. Однак, ви можете змінити цю поведінку, визначивши власне замикання в методі `renderable` для винятка `InvalidSignatureException` в вашому виконавці винятків:

```php
use Illuminate\Routing\Exceptions\InvalidSignatureException;

/**
 * Зареєструвати замикання, які обробляють винятки додатка.
 *
 * @return void
 */
public function register()
{
    $this->renderable(function (InvalidSignatureException $e) {
        return response()->view('error.link-expired', [], 403);
    });
}
```

<a name="urls-for-controller-actions"></a>

## URL-адреси за діями контролера

Функція `action` генерує URL-адресу за наданою дією контроллера:

```php
use App\Http\Controllers\HomeController;

$url = action([HomeController::class, 'index']);
```

Якщометод контролерра приймає параметри маршрута, ви можете передати асоціативний масив параметрів маршрута, другим аргументом:

```php
$url = action([UserController::class, 'profile'], ['id' => 1]);
```

<a name="default-values"></a>

## Значення за замовчуванням

Можливо для деяких додатків, ви захочите визначити значення за замовчуванням для певних параметрів URL-адреси. Наприклад, уявіть, що багато з ваших маршрутів визначені як `{locale}` параметр:

```php
Route::get('/{locale}/posts', function () {
    //
})->name('post.index');
```

Це може втомлювати завжди передавати `locale` кожного разу, коли ви викликаєте пімічника `route`. Отже, ви можете використати метод `URL::defaults`, щоб визначити значення за замовчуванням для цього параметра, який буде завжди застосовуватись протягом поточного запита. Ви можите викликати цей метод з [посередника маршрутів](middleware.md#assigning-middleware-to-routes), щоб мати доступ для поточного запита:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\URL;

class SetDefaultLocaleForUrls
{
    /**
     * Опрацювати вхідний запит.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return \Illuminate\Http\Response
     */
    public function handle($request, Closure $next)
    {
        URL::defaults(['locale' => $request->user()->locale]);

        return $next($request);
    }
}
```

Як тільки значення за замовчуванням для параметра `locale` буде встановлено, вам більше не доведеться передавати це значення при генерації URL-адрес за допомогою помічника `route`.

<a name="url-defaults-middleware-priority"></a>

#### Параметри URL-адреси за замовчуванням & Приоритет посередника

Встановлені значення для URL-адреси за замовчуванням може заважати Laravel опрацьовувати неявні при'язки моделі. Тому, вам варто [визначити пріорітет посереднику](middleware.md#sorting-middleware), в якому встановлено URL-адресу за замовчуванням, щоб він був виконаний перед посередником `SubstituteBindings`. Ви можете досягти цього, переконавшись, що ваш посередник визначений перед посередником `SubstituteBindings` у властивості `$middlewarePriority` HTTP `Kernel.php` вашого додатка.

Властивість `$middlewarePriority` визначено у базовому класі `Illuminate\Foundation\Http\Kernel`. Ви можете скопіювати це визначення з цього класу і перевизначити його у HTTP `Kernel.php` вашого додатка, щоб змінити пріоритет:

```php
/**
 * Список посередників, відсортовані за пріоритетністю.
 *
 * Змусить неглобальні посередники завжди виконуватись у заданому порядку.
 *
 * @var array
 */
protected $middlewarePriority = [
    // ...
        \App\Http\Middleware\SetDefaultLocaleForUrls::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        // ...
];
```
