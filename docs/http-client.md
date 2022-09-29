# HTTP-клієнт

- [Вступ](#introduction)
- [Виконання запитів](#making-requests)
  - [Дані запита](#request-data)
  - [Заголовки](#headers)
  - [Автентифікація](#authentication)
  - [Час очікування](#timeout)
  - [Повторні спроби](#retries)
  - [Опрацювання помилок](#error-handling)
  - [Посередник Guzzle](#guzzle-middleware)
  - [Параметри Guzzle](#guzzle-options)
- [Одночасні запити](#concurrent-requests)
- [Макроси](#macros)
- [Тестування](#testing)
  - [Хибні відповіді](#faking-responses)
  - [Перевірка запитів](#inspecting-requests)
  - [Попередження випадкових запитів](#preventing-stray-requests)
- [Події](#events)

<a name="introduction"></a>

## Вступ

Laravel надає виразний, мінімальний API навколо [HTTP-клієнта Guzzle](http://docs.guzzlephp.org/en/stable/), що дозволяє вам з легкісттю відправляти вихідні HTTP-запити, для взаємодії з іншими web-додатками. Guzzle загорнутий в Laravel, який орієнтований на найпоширеніших випадках використання та чудовому досвіді розробника.

Перед тим як розпочати, ви повінні переконатись в тому, що ви встановили пакунок Guzzle, як залежність вашого додатка. За замовчування, Laravel автоматично включає цю залежність. Однак, якищо ви раніше видаляли цей пакунок, вам знову потрібно його встановити за допомогою Composer:

```shell
composer require guzzlehttp/guzzle
```

<a name="making-requests"></a>

## Виконання запитів

Щоб відправляти запити, вам потрібно використовувати методи `head`, `get`, `post`, `put`, `patch`, та `delete`, які надані фасадом `Http`. Спершу, давайте перевіремо як відправляти базові `GET` запити на інший URL:

```php
use Illuminate\Support\Facades\Http;

$response = Http::get('http://example.com');
```

Метод `get` віддає екземпляр `Illuminate\Http\Client\Response`, який надає різноманітні методи, які можна використовувати для отримання інформації про відповідь:

```php
$response->body() : string;
$response->json($key = null) : array|mixed;
$response->object() : object;
$response->collect($key = null) : Illuminate\Support\Collection;
$response->status() : int;
$response->ok() : bool;
$response->successful() : bool;
$response->redirect(): bool;
$response->failed() : bool;
$response->serverError() : bool;
$response->clientError() : bool;
$response->header($header) : string;
$response->headers() : array;
```

Об'єкт `Illuminate\Http\Client\Response` також реалізує інтерфейс PHP `ArrayAccess`, який дозволяє вам отримати доступ до даних відповіді JSON безпосередньо у відповіді:

```php
return Http::get('http://example.com/users/1')['name'];
```

<a name="dumping-requests"></a>

#### Dump запитів

Якщо ви бажаєте отримати інформацію про сформований екземпляра вихідного запита перед його відправленням і припинити виконання сценарія, ви можете додати метод `dd` на початку визначення запита:

```php
return Http::dd()->get('http://example.com');
```

<a name="request-data"></a>

### Данні запита

Звичайно, під час виконання запитів `POST`, `PUT`, та `PATCH` зазвичай надсилаються додаткові дані разом із запитом, тому ці методи приймають масив даних другим аргументом. За замовчуванням, дані відправлятимуться за допомогою типу контента `application/json`.

```php
use Illuminate\Support\Facades\Http;

$response = Http::post('http://example.com/users', [
    'name' => 'Steve',
    'role' => 'Network Administrator',
]);
```

<a name="get-request-query-parameters"></a>

#### Параметри GET-запита

Виконуючи запити `GET`, ви можете додати рядок параметрів до URL-адреси безпосередньо, або передати масив пар ключ / значення другим аргументом методу `get`:

```php
$response = Http::get('http://example.com/users', [
    'name' => 'Taylor',
    'page' => 1,
]);
```

<a name="sending-form-url-encoded-requests"></a>

#### Надсилання запитів з передачею даних в URL-кодованому рядку

Якщо ви бажаєте відправити дані використовуючи тип контента `application/x-www-form-urlencoded`, то вам потрібно викликати метод `asForm` перед виконанням запита:

```php
$response = Http::asForm()->post('http://example.com/users', [
    'name' => 'Sara',
    'role' => 'Privacy Consultant',
]);
```

<a name="sending-a-raw-request-body"></a>

#### Надсилання запита з непідготовленим тілом

Ви можете використовувати метод `withBody`, якщо ви бажаєте надати тіло непідготовленого зпита під час виконання запита. Тип контента може бути наданий другим аргументом:

```php
$response = Http::withBody(
    base64_encode($photo), 'image/jpeg'
)->post('http://example.com/photo');
```

<a name="multi-part-requests"></a>

#### Багатокомпонентні запити

Якщо ви хочете відправляти файли в запитах, вам слід викликати метод `attach` перед тим, як зробити запит. Цей метод приймає назву файла та його вміст. Якщо потрібно, ви можете надати третій аргумент, який вважатиметься назвою файла:

```php
$response = Http::attach(
    'attachment', file_get_contents('photo.jpg'), 'photo.jpg'
)->post('http://example.com/attachments');
```

Замість передачі непідготовленного контенту файла, ви можете передати потоковий ресурс:

```php
$photo = fopen('photo.jpg', 'r');

$response = Http::attach(
    'attachment', $photo, 'photo.jpg'
)->post('http://example.com/attachments');
```

<a name="headers"></a>

### Заголовки

Заголовки можна додавати до запитів за допомогою метода `withHeaders`. Цей метод `withHeaders` приймає масив пар ключ / значення:

```php
$response = Http::withHeaders([
    'X-First' => 'foo',
    'X-Second' => 'bar'
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

Ви можете використовувати метод `accept`, щоб вказати тип контента, який ваш додаток очікує у відповідь на ваш запит:

```php
$response = Http::accept('application/json')->get('http://example.com/users');
```

Для зручності ви можете використовувати метод `acceptJson`, щоб швидко вказати, що ваш додаток очікує тип контента `application/json` у відповідь на ваш запит:

```php
$response = Http::acceptJson()->get('http://example.com/users');
```

<a name="authentication"></a>

### Автентифікація

Ви можете вказати облікові дані **basic** and **digest** за допомогою методів `withBasicAuth` і `withDigestAuth` відповідно:

```php
// Basic HTTP-автентифікація...
$response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

// Digest HTTP-автентифікація...
$response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);
```

<a name="bearer-tokens"></a>

#### Токени Bearer

Якщо ви бажаєте швидко додати токен Bearer до заголовка `Authorization` запита, ви можете використати метод `withToken`:

```php
$response = Http::withToken('token')->post(/* ... */);
```

<a name="timeout"></a>

### Час очікування

Метод `timeout` можна використовувати для визначення максимальної кількості секунд для очікування відповіді:

```php
$response = Http::timeout(3)->get(/* ... */);
```

Якщо заданий час очікування перевищено, буде створено екземпляр `Illuminate\Http\Client\ConnectionException`.

Ви можете вказати максимальну кількість секунд очікування під час спроби підключитися до сервера за допомогою метода `connectTimeout`:

```php
$response = Http::connectTimeout(3)->get(/* ... */);
```

<a name="retries"></a>

### Повторні спроби

Якщо ви бажаєте, щоб HTTP-клієнт автоматично повторював запит, якщо сталася помилка клієнта або сервера, ви можете скористатися методом `retry`. Метод `retry` приймає максимальну кількість спроб запита та кількість мілісекунд, які Laravel має чекати між спробами:

```php
$response = Http::retry(3, 100)->post(/* ... */);
```

Якщо необхідно, ви можете передати третій аргумент методу `retry`. Третій аргумент має бути замиканням, та має визначати, чи дійсно варто виконати повторну спрому. Наприклад, ви можете повторити запит, лише якщо початковий запит натрапляє `ConnectionException`:

```php
$response = Http::retry(3, 100, function ($exception, $request) {
    return $exception instanceof ConnectionException;
})->post(/* ... */);
```

Якщо спроба запита не вдається, ви можете внести зміни до запита перед новою спробою. Ви можете досягти цього, змінивши аргумент запита, наданий замиканню, який ви надали методу `retry`. Наприклад, ви можете повторити запит із новим токеном авторизації, якщо перша спроба повернула помилку автентифікації:

```php
$response = Http::withToken($this->getToken())->retry(2, 0, function ($exception, $request) {
    if (! $exception instanceof RequestException || $exception->response->status() !== 401) {
        return false;
    }

    $request->withToken($this->getNewToken());

    return true;
})->post(/* ... */);
```

Якщо всі запити не виконуються, буде створено екземпляр `Illuminate\Http\Client\RequestException`. Якщо ви хочете вимкнути цю поведінку, ви можете надати `throw` аргумент зі значенням `false`. Якщо вимкнено, остання відповідь, отримана клієнтом, буде повернена після всіх повторних спроб:

```php
$response = Http::retry(3, 100, throw: false)->post(/* ... */);
```

> **Warning**  
> Якщо всі запити завершуються невдачею через проблему підключення, все одно буде створено виняток `Illuminate\Http\Client\ConnectionException`, навіть якщо аргумент `throw` має значення `false`.

<a name="error-handling"></a>

### Опрацювання помилок

На відміну від поведінки Guzzle за замовчуванням, HTTP-клієнт загорнутий в Laravel не створює винятків для помилок клієнта чи сервера (відповіді `400` і `500` відповідно). Ви можете визначити, чи було повернуто одну з цих помилок, використовуючи методи `successful`, `clientError` або `serverError`:

```php
// Визначити, чи є код статуса >= 200 і < 300...
$response->successful();

// Визначаємо, чи код статуса >= 400...
$response->failed();

// Визначити, чи відповідь має код статуса рівня 400...
$response->clientError();

// Визначити, чи відповідь має код статуса рівня 500...
$response->serverError();

// Негайно виконати вказаний замикання, якщо сталася помилка клієнта чи сервера...
$response->onError(callable $callback);
```

<a name="throwing-exceptions"></a>

#### Створення винятків

Якщо у вас є екземпляр відповіді, і ви хочете створити екземпляр `Illuminate\Http\Client\RequestException`, якщо код статуса відповіді вказує на помилку клієнта або сервера, ви можете використати методи `throw` або `throwIf`:

```php
$response = Http::post(/* ... */);

// Створення винятку, якщо сталася помилка клієнта або сервера...
$response->throw();

// Створення винятку, якщо сталася помилка і дана умова віддає true...
$response->throwIf($condition);

// Створення винятку, якщо сталася помилка і дана умова є false...
$response->throwUnless($condition);

return $response['user']['id'];
```

Екземпляр `Illuminate\Http\Client\RequestException` має публічну властивість `$response`, яка дозволить вам перевірити отриману відповідь.

Метод `throw` віддає екземпляр відповіді `Illuminate\Http\Client\Response`, якщо не сталася помилка, дозволяючи вам зв’язати інші операції з методом `throw`:

```php
return Http::post(/* ... */)->throw()->json();
```

Якщо ви бажаєте виконати додаткову логіку перед тим, як буде створено виняток, ви можете передати Замикання методу `throw`.Виняток буде створено автоматично після виклику замикання, тому вам не потрібно повторно створювати виняток з середини замикання:

```php
return Http::post(/* ... */)->throw(function ($response, $e) {
    //
})->json();
```

<a name="guzzle-middleware"></a>

### Посередник Guzzle

Оскільки HTTP-клієнт Laravel працює на основі Guzzle, ви можете скористатися [Guzzle Middleware](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html), щоб маніпулювати вихідним запитом або перевіряти вхідну відповідь. Щоб маніпулювати вихідним запитом, зареєструйте посередник Guzzle за допомогою метода `withMiddleware` у поєднанні з фабрикою посередника `mapRequest` від Guzzle:

```php
use GuzzleHttp\Middleware;
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\RequestInterface;

$response = Http::withMiddleware(
    Middleware::mapRequest(function (RequestInterface $request) {
        $request->withHeader('X-Example', 'Value');

        return $request;
    })
->get('http://example.com');
```

Так само ви можете перевірити вхідну HTTP-відповідь, зареєструвавши посередник за допомогою метода `withMiddleware` у поєднанні з фабрикою посередника `mapRequest` від Guzzle:

```php
use GuzzleHttp\Middleware;
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\ResponseInterface;

$response = Http::withMiddleware(
    Middleware::mapResponse(function (ResponseInterface $response) {
        $header = $response->getHeader('X-Example');

        // ...

        return $response;
    })
)->get('http://example.com');
```

<a name="guzzle-options"></a>

### Параметри Guzzle

Ви можете вказати додаткові [параметри запита Guzzle](http://docs.guzzlephp.org/en/stable/request-options.html) за допомогою метода `withOptions`. Метод `withOptions` приймає масив пар ключ / значення:

```php
$response = Http::withOptions([
    'debug' => true,
])->get('http://example.com/users');
```

<a name="concurrent-requests"></a>

## Одночасні запити

Іноді вам може знадобитися зробити декілька HTTP-запитів одночасно. Іншими словами, ви хочете, щоб декілька запитів надсилалися одночасно, а не надсилали запити послідовно. Це може призвести до значного підвищення продуктивності під час взаємодії з повільними HTTP API.

На щастя, ви можете досягти цього за допомогою метода `pool`. Метод `pool` приймає замикання, яке отримує екземпляр `Illuminate\Http\Client\Pool`, що дозволяє вам легко додавати запити до групи запитів для відправки:

```php
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Http;

$responses = Http::pool(fn (Pool $pool) => [
    $pool->get('http://localhost/first'),
    $pool->get('http://localhost/second'),
    $pool->get('http://localhost/third'),
]);

return $responses[0]->ok() &&
    $responses[1]->ok() &&
    $responses[2]->ok();
```

Як ви бачите, до кожного екземпляра відповіді можна отримати доступ відповідно до порядку їх додавання до групи. Якщо ви бажаєте, ви можете визначити імена запитам за допомогою метода `as`, який дозволяє отримати доступ до відповідних відповідей за їх назвою:

```php
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Http;

$responses = Http::pool(fn (Pool $pool) => [
    $pool->as('first')->get('http://localhost/first'),
    $pool->as('second')->get('http://localhost/second'),
    $pool->as('third')->get('http://localhost/third'),
]);

return $responses['first']->ok();
```

<a name="macros"></a>

## Макроси

HTTP-клієнт Laravel дозволяє визначати «макроси», які можуть служити гнучким, виразним механізмом для налаштування загальних шляхів запита та заголовків під час взаємодії зі службами у вашому додатку. Щоб почати, ви можете визначити макрос у методі `boot` класа `App\Providers\AppServiceProvider` вашого додатка:

The Laravel HTTP client allows you to define "macros", which can serve as a fluent, expressive mechanism to configure common request paths and headers when interacting with services throughout your application. To get started, you may define the macro within the `boot` method of your application's `App\Providers\AppServiceProvider` class:

```php
use Illuminate\Support\Facades\Http;

/**
 * Завантажте будь-які служби додатків.
 *
 * @return void
 */
public function boot()
{
    Http::macro('github', function () {
        return Http::withHeaders([
            'X-Example' => 'example',
        ])->baseUrl('https://github.com');
    });
}
```

Після того, як ваш макрос буде налаштовано, ви можете викликати його з будь-якого місця у вашому додатку, щоб створити очікуваний запит із зазначеною конфігурацією:

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>

## Тестування

Багато служб Laravel надають функціональні можливості, які допомагають вам легко та виразно писати тести, і HTTP-клієнт Laravel не є винятком. Метод `fake` фасада `Http` дозволяє вам вказати клієнту `Http` повертати заглушені / хибні відповіді, під час виконання запитів.

<a name="faking-responses"></a>

### Хибні відповіді

Наприклад, щоб вказати HTTP-клієнту віддавати порожні відповіді з кодом статуса `200` на кожен запит, ви можете викликати `fake` метод без аргументів:

```php
use Illuminate\Support\Facades\Http;

Http::fake();

$response = Http::post(/* ... */);
```

<a name="faking-specific-urls"></a>

#### Підробка конкретних URL-адрес

Крім того, ви можете передати масив методу `fake`. Ключі масива мають представляти шаблони URL-адрес, які ви хочете підробити, і пов’язані з ними відповіді. Символ `*` можна використовувати як символ підстановки. Будь-які запити до URL-адрес, які не були фактично підроблені, виконуватимуться. Ви можете використовувати метод `response` фасада `Http` для створення заглушки / хибних відповідей для цих кінцевих точок:

```php
Http::fake([
    // Залиште відповідь JSON для кінцевих точок GitHub...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

    // Залиште рядок відповіді для кінцевих точок Google...
    'google.com/*' => Http::response('Hello World', 200, $headers),
]);
```

Якщо ви бажаєте вказати шаблон резервної URL-адреси, який заглушить всі невідповідні URL-адреси, ви можете використати один символ `*`:

```php
Http::fake([
    // Залиште відповідь JSON для кінцевих точок GitHub...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

    // Залиште рядок відповіді для кінцевих точок Google...
    '*' => Http::response('Hello World', 200, ['Headers']),
]);
```

<a name="faking-response-sequences"></a>

#### Хибні послідовності відповідей

Іноді може знадобитися вказати, що одна URL-адреса має повертати серію фальшивих відповідей у ​​певному порядку. Ви можете зробити це за допомогою метода `Http::sequence` для створення відповідей:

```php
Http::fake([
    // Залиште серію відповідей для кінцевих точок GitHub...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->pushStatus(404),
]);
```

Коли всі відповіді в послідовності відповідей використано, будь-які наступні запити спричинять виняток. Якщо ви бажаєте вказати відповідь за замовчуванням, яка має повертатися, коли послідовність порожня, ви можете використати метод `whenEmpty`:

```php
Http::fake([
    // Залиште серію відповідей для кінцевих точок GitHub...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->whenEmpty(Http::response()),
]);
```

Якщо ви хочете підробити послідовність відповідей, але не потрібно вказувати конкретний шаблон URL-адреси, який потрібно підробити, ви можете використати метод `Http::fakeSequence`:

```php
Http::fakeSequence()
    ->push('Hello World', 200)
    ->whenEmpty(Http::response());
```

<a name="fake-callback"></a>

#### Химбний Callback

Якщо вам потрібна більш складна логіка, щоб визначити, які відповіді віддавати для певних кінцевих точок, ви можете передати замикання методу `fake`. Це замикання отримає екземпляр `Illuminate\Http\Client\Request` і має віддати цей екземпляр відповіді. У своєму замиканні ви можете виконати будь-яку необхідну логіку, щоб визначити, який тип відповіді віддавати:

```php
use Illuminate\Http\Client\Request;

Http::fake(function (Request $request) {
    return Http::response('Hello World', 200);
});
```

<a name="inspecting-requests"></a>

### Перевірка запитів

Підробляючи відповіді, ви можете час від часу захотіти перевірити запити, які отримує клієнт, щоб переконатися, що ваш додаток відправляє правильні дані або заголовки. Це можна зробити, викликавши метод `Http::assertSent` після виклику `Http::fake`.

Метод `assertSent` приймає замикання, яке отримає екземпляр `Illuminate\Http\Client\Request` і має віддати логічне значення, яке вказує, чи відповідає запит вашим очікуванням. Для успішного проходження тесту повинен бути відправлений хоча б один запит, який відповідає зазначеним очікуванням:

```php
use Illuminate\Http\Client\Request;
use Illuminate\Support\Facades\Http;

Http::fake();

Http::withHeaders([
    'X-First' => 'foo',
])->post('http://example.com/users', [
    'name' => 'Taylor',
    'role' => 'Developer',
]);

Http::assertSent(function (Request $request) {
    return $request->hasHeader('X-First', 'foo') &&
            $request->url() == 'http://example.com/users' &&
            $request['name'] == 'Taylor' &&
            $request['role'] == 'Developer';
});
```

Якщо необхідно, ви можете підтвердити, що певний запит не було відправлено за допомогою метода `assertNotSent`:

```php
use Illuminate\Http\Client\Request;
use Illuminate\Support\Facades\Http;

Http::fake();

Http::post('http://example.com/users', [
    'name' => 'Taylor',
    'role' => 'Developer',
]);

Http::assertNotSent(function (Request $request) {
    return $request->url() === 'http://example.com/posts';
});
```

Ви можете використовувати метод`assertSentCount`, щоб підтвердити, скільки запитів було «відправлено» під час тесту:

```php
Http::fake();

Http::assertSentCount(5);
```

Або ви можете використати метод `assertNothingSent`, щоб підтвердити, що жодних запитів не було відправлено під час тесту:

```php
Http::fake();

Http::assertNothingSent();
```

<a name="recording-requests-and-responses"></a>

#### Записування Запитів / Відповідей

Ви можете використовувати метод `recorded` для збору всіх запитів і їх відповідних відповідей. Метод `recorded` віддає колекцію масивів, яка містить екземпляри `Illuminate\Http\Client\Request` і `Illuminate\Http\Client\Response`:

```php
Http::fake([
    'https://laravel.com' => Http::response(status: 500),
    'https://nova.laravel.com/' => Http::response(),
]);

Http::get('https://laravel.com');
Http::get('https://nova.laravel.com/');

$recorded = Http::recorded();

[$request, $response] = $recorded[0];
```

Крім того, метод `recorded` приймає замикання, яке отримає екземпляри `Illuminate\Http\Client\Request` і `Illuminate\Http\Client\Response`, та може використовуватися для фільтрації пар запит / відповідь на основі ваших очікувань:

```php
use Illuminate\Http\Client\Request;
use Illuminate\Http\Client\Response;

Http::fake([
    'https://laravel.com' => Http::response(status: 500),
    'https://nova.laravel.com/' => Http::response(),
]);

Http::get('https://laravel.com');
Http::get('https://nova.laravel.com/');

$recorded = Http::recorded(function (Request $request, Response $response) {
    return $request->url() !== 'https://laravel.com' &&
           $response->successful();
});
```

<a name="preventing-stray-requests"></a>

### Попередження випадкових запитів

Якщо ви хочете бути впевненими, що всі запити, надіслані через HTTP-клієнт, були підроблені під час вашого окремого тесту або повного набору тестів, ви можете викликати метод `preventStrayRequests`. Після виклику цього метода будь-які запити, які не мають відповідної хибної відповіді, створять виняток, на заміну фактичного HTTP-запита:

```php
use Illuminate\Support\Facades\Http;

Http::preventStrayRequests();

Http::fake([
    'github.com/*' => Http::response('ok'),
]);

// Віддається відповідь "ок"...
Http::get('https://github.com/laravel/framework');

// Виникає виняток...
Http::get('https://laravel.com');
```

<a name="events"></a>

## Події

Laravel запускає три події під час процесу відправлення HTTP-запитів. Подія `RequestSending` запускається перед надсиланням запита, тоді як подія `ResponseReceived` запускається після отримання відповіді на даний запит. Подія `ConnectionFailed` запускається, якщо на даний запит не отримано відповіді.

Події `RequestSending` і `ConnectionFailed` містять публічну властивість `$request`, яку можна використовувати для перевірки екземпляра `Illuminate\Http\Client\Request`. Так само подія `ResponseReceived` містить властивість `$request`, та властивість `$response`, які можна використовувати для перевірки екземпляра `Illuminate\Http\Client\Response`. Ви можете зареєструвати слухачів подій для цієї події у вашому постачальнику служб `App\Providers\EventServiceProvider`:

```php
/**
 * Мапа слухачів подій додатка.
 *
 * @var array
 */
protected $listen = [
    'Illuminate\Http\Client\Events\RequestSending' => [
        'App\Listeners\LogRequestSending',
    ],
    'Illuminate\Http\Client\Events\ResponseReceived' => [
        'App\Listeners\LogResponseReceived',
    ],
    'Illuminate\Http\Client\Events\ConnectionFailed' => [
        'App\Listeners\LogConnectionFailed',
    ],
];
```
