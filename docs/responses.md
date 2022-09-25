# HTTP-відповіді

- [Створення відповідей](#creating-responses)
  - [Додавання заголовків до відповідей](#attaching-headers-to-responses)
  - [Додавання файлів cookie до відповідей](#attaching-cookies-to-responses)
  - [Файли Cookies та шифрування](#cookies-and-encryption)
- [Перенаправлення](#redirects)
  - [Перенаправлення на іменовані маршрути](#redirecting-named-routes)
  - [Перенаправлення по діям контроллера](#redirecting-controller-actions)
  - [Перенаправлення на зовнішні домени](#redirecting-external-domains)
  - [Перенаправлення з одноразовим збереженням данних в сессії](#redirecting-with-flashed-session-data)
- [Інші типи відповідей](#other-response-types)
  - [Візуалізовані відповіді з HTML-шаблонами](#view-responses)
  - [Відповіді JSON](#json-responses)
  - [Відповіді з завантаженням файлів](#file-downloads)
  - [Відповіді, які відображають вміст файлів](#file-responses)
- [Макрос відповіді](#response-macros)

<a name="creating-responses"></a>

## Створення відповідей

<a name="strings-arrays"></a>

#### Рядки & Масиви

Всі маршрути та контроллери повинні повертати відповідь, яка відправлятиметься назад до браузера користувачів. Laravel надає декілька варіантів для повернення відповідей. Основновна відповідь - це повернення рядка з маршрута або контроллера. Фреймвор атоматично конвертує рядок у повну HTTP-відповідь:

```php
Route::get('/', function () {
    return 'Hello World';
});
```

Додатково до повернення рядків з ваших маршрутів та контроллерів, ви можете повертати масиви. Фреймворк автоматично автоматично конвертує масив у JSON відповідь:

```php
Route::get('/', function () {
    return [1, 2, 3];
});
```

> **Note**  
> Чи ви знали, що ви також можете повернути [Eloquent колекції](eloquent-collections.md) з ваших маршрутів та контроллерів? Вони автоматично будуть конвертовані у JSON. Спробуйте!

<a name="response-objects"></a>

#### Об'єкти відповіді

Як правило, ви не просто повертатимете прості рядки чи масиви з ваших маршруту. Натомість, ви повертатимите цілі екземпляри `Illuminate\Http\Response` або [візуалізації](views.md).

Повернення цілого екземпляра `Response` дозволяє вам налаштувати код статуса HTTP і заголовки у вашій відповіді. Екземпляр `Response` успадковує з класа `Symfony\Component\HttpFoundation\Response`, який надає різноманіття методів для створення HTTP-відповіді:

```php
Route::get('/home', function () {
    return response('Hello World', 200)
        ->header('Content-Type', 'text/plain');
});
```

<a name="eloquent-models-and-collections"></a>

#### Моделі Eloquent & Колекції

Ви також можите повернути моделі та колекції [Eloquent ORM](eloquent.md) безпосередньо з ваших маршрутів або контроллерів. Коли ви це зробите, Laravel автоматично конвертуватиме моделі та колекції у JSON відповідь дотримуючись [прихованих атрибутів](eloquent-serialization.md#hiding-attributes-from-json) моделі:

```php
use App\Models\User;

Route::get('/user/{user}', function (User $user) {
    return $user;
});
```

<a name="attaching-headers-to-responses"></a>

### Додавання заголовків до відповідей

Пам'ятайте, що більшість методів відповідей є ланцюжковими, що дозволяє зручно формувати екземпляри відповідей. Наприклад, ви можете використовувати метод `header`, щоб додати серію заголовків до відповіді перед тим, як відправити її користувачеві:

```php
return response($content)
    ->header('Content-Type', $type)
    ->header('X-Header-One', 'Header Value')
    ->header('X-Header-Two', 'Header Value');
```

Або, ви можете використати метод `withHeaders`, щоб визначити масив заголовків, які будуть додані до відповіді:

```php
return response($content)
    ->withHeaders([
        'Content-Type' => $type,
        'X-Header-One' => 'Header Value',
        'X-Header-Two' => 'Header Value',
    ]);
```

<a name="cache-control-middleware"></a>

#### Посередник керування кешем

Laravel включає посередник `cache.headers`, який може бути використаний для швидкого встановлення заголовка `Cache-Control` для групи маршрутів. Інструкції по керуванню кешем мають бути надані з використанням «зміїного регістру», які повинні відокремлюватись крапкою з комою. Якщо у списку інструкцій зазначено `etag`, MD5-хеш вмісту відповіді буде автоматично встановлено як ідентифікатор ETag:

```php
Route::middleware('cache.headers:public;max_age=2628000;etag')->group(function () {
    Route::get('/privacy', function () {
        // ...
    });

    Route::get('/terms', function () {
        // ...
    });
});
```

<a name="attaching-cookies-to-responses"></a>

### Додавання файлів Cookies до відповіді

Ви можете додати Cookies у вихідний екземпляр `Illuminate\Http\Response` використовуючи метод `cookie`. Вам потрібно передати ключ - значення, та кількість хвилин протягом яких Сookie буде вважатись дійсним:

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes
);
```

Метод `cookie` також приймає декілька аргументів, які використовуються рідше. Загалом ці аргументи мають ту саму мету та значення, що й аргументи, надані рідному методу [setcookie](https://secure.php.net/manual/en/function.setcookie.php) PHP:

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes, $path, $domain, $secure, $httpOnly
);
```

Якщо ви хочете, щоб Cookie відправлявся разом з вихідною відповіддю, але у вас ще немає екземпляра цієї відповіді, ви можете використати фасад `Cookie`, щоб "поставити в чергу" файли Cookies для додавання їх до відповіді під час її відправлення. Метод `queue` приймає аргументи необхідні для створення екземпляра Cookie. Ці Cookies будуть додані до вихідної відповіді перед тим, як її буде надіслано в браузер:

```php
use Illuminate\Support\Facades\Cookie;

Cookie::queue('name', 'value', $minutes);
```

<a name="generating-cookie-instances"></a>

#### Створення екземплярів Cookie

Якщо ви бажаєте створити екземпляр `Symfony\Component\HttpFoundation\Cookie`, який можна додати до екземпляра відповіді пізніше, ви можете використати глобальний помічник `cookie`. Цей файл Cookie не буде відправлено назад до клієнта, поки він ще не прикріплений до екземпляра відповіді:

```php
$cookie = cookie('name', 'value', $minutes);

return response('Hello World')->cookie($cookie);
```

<a name="expiring-cookies-early"></a>

#### Дострокове закінчення терміну дії файлів Cookies

Ви можете видалити файл Cookie скинувши його термін дії до нуля за допомогою метода `withoutCookie` у вихідній відповіді:

```php
return response('Hello World')->withoutCookie('name');
```

Якщо ви ще не маєте екземпляр вихідної відповіді, ви можете використати метод `expire` фасада `Cookie`, щоб закінчити термін дії Сookie:

```php
Cookie::expire('name');
```

<a name="cookies-and-encryption"></a>

### Файли Cookies та шифрування

За замовчування, всі Cookies згенеровани Laravel шифровані та підписані, тому вони не можуть бути змінені або прочитані клієнтом.
Якщо ви бажаєте вимкнути шифрування підможини Cookies згенерованих вашим додатком, ви можете використати властивість `$except` посередника `App\Http\Middleware\EncryptCookies`, який розташовано в директорії `app/Http/Middleware`:

```php
    /**
    * Імена cookies, які не потрібно шифрувати.
    *
    * @var array
    */
protected $except = [
    'cookie_name',
];
```

<a name="redirects"></a>

## Перенаправлення

Відповіді перенаправлення - це екземпляри класа `Illuminate\Http\RedirectResponse`, які містять відповідні заголовки для перенаправлення користувача на інший URL. Є декілька варіантів для створення екземпляра `RedirectResponse`. Простим варіантом буде використання глобального помічника `redirect`:

```php
Route::get('/dashboard', function () {
    return redirect('home/dashboard');
});
```

Можливо іноді вам потрібно буде перенаправити користувача до його попередньої локації, наприклад, під час відправлення невалідної форми. Ви можете зробити це, за допомогою глобального помічника `back`. Оскільки цей функціонал використовує [сеанси](session.md),
переконайтеся, що маршрут, який викликає функцію `back`, використовує групу посередників `web`:

```php
Route::post('/user/profile', function () {
    // Підтвердити запит...

    return back()->withInput();
});
```

<a name="redirecting-named-routes"></a>

### Перенаправлення на іменовіні маршрути

Коли ви викликаєте помічника `redirect` без параметрів, буде повернуто екземпляр `Illuminate\Routing\Redirector`, який дозволяє вам викликати будь-які методи екземпляра `Redirector`. Наприклад, щоб згенерувати `RedirectResponse` на іменований маршрут, вам потрібно використати метод `route`:

```php
return redirect()->route('login');
```

Якщо ваш маршрут має параметри, ви можете передати їх другим аргументом до метода `route`:

```php
// Для маршруту з таким URL: /profile/{id}

return redirect()->route('profile', ['id' => 1]);
```

<a name="populating-parameters-via-eloquent-models"></a>

#### Заповнення параметрів за допомогою Eloquent моделі

Якщо ви перенаправляєте на маршрут, який має параметр "ID", ви можете просто передати модель Eloquent. ID моделі буде вилучено автоматично:

```php
// Для маршруту з таким URL: /profile/{id}

return redirect()->route('profile', [$user]);
```

Якщо ви бажаєте налаштувати значення, яке розташоване в параметрі маршрута, ви можете вказати стовпець у визначенні параметра маршрута (`/profile/{id:column}`) або ви можете переписати метод `getRouteKey` у вашій моделі Eloquent:

```php
/**
 * Отримати значення ключа маршрута моделі.
 *
 * @return mixed
 */
public function getRouteKey()
{
    return $this->column;
}
```

<a name="redirecting-controller-actions"></a>

### Перенаправлення по діям контроллера

Ви також можете генерувати перенапрвалення по [діям контроллера](controllers.md), щоб це зробити, передайте відповідний контроллер та ім'я дії методу `action`:

```php
use App\Http\Controllers\UserController;

return redirect()->action([UserController::class, 'index']);
```

Якщо ваш маршрут контроллера вимагає параметри, ви можете передати їх другим аргументом методу `action`:

```php
return redirect()->action(
    [UserController::class, 'profile'], ['id' => 1]
);
```

<a name="redirecting-external-domains"></a>

### Перенаправлення на зовнішні домени

Іноді вам може знадобитись перенаправити на зовнішній домен з вашого додатка. Ви можете зробити це, викликавши метод `away`, який створить `RedirectResponse` без будь-якого додаткового кодування URL-адреси, валідації, або перевірки:

```php
return redirect()->away('https://www.google.com');
```

<a name="redirecting-with-flashed-session-data"></a>

### Перенаправлення з одноразовим збереженням данних в сессії

Перенаправлення на нову URL-адресу [з одноразовим додавання даних до сесії](session.md#flash-data) зазвичай виконуються одночасно.
Як правило, це буде зроблено після успішного виконання дії, коли ви відправляєте повідомлення про успішне завершення в сессію. Для зручності, ви можете створити екземпляр `RedirectResponse` і додати дані до сесії в єдиному ланцюжку методів:

```php
Route::post('/user/profile', function () {
    // ...

    return redirect('dashboard')->with('status', 'Profile updated!');
});
```

Після того, як користувачь буде перенаправленний, ви можете відобразити збережене повідомлення з [сесії](session.md). Наприклад, використовуючи [синтаксис Blade](blade.md):

```blade
@if (session('status'))
    <div class="alert alert-success">
        {{ session('status') }}
    </div>
@endif
```

<a name="redirecting-with-input"></a>

#### Перенаправлення разом з одноразовим збереженням вхідними даними полів

Ви можете використати метод `withInput` наданий екземпляром `RedirectResponse`, щоб зберегти дані поточного запита до сесії перед тим, як перенаправити користувача на нову локацію. Це зазвичай робиться коли користувач натрапив на помилку валідації. Як тільки поля будуть додані до сесії, ви можете дуже просто [їх отримати](requests.md#retrieving-old-input), під час наступного запита для повторного заповнення форми:

```php
return back()->withInput();
```

<a name="other-response-types"></a>

## Інші типи відповідей

Помічник `response` може використовуватись для створення інших типів екземплярів відповіді. Коли помічник `response` викликається без аргументів, буде повернено реалізацію [контракта](contracts.md) `Illuminate\Contracts\Routing\ResponseFactory`. Цей контракт надає декілька корисних методів для генерації відповідей.

<a name="view-responses"></a>

### View Responses

Якщо вам потрібен контроль над статусом відповіді і заголовками, але також потрібно повернути [візуалізацію](views.md), як вміст відповіді, вам варто використати метод `view`:

```php
return response()
    ->view('hello', $data, 200)
    ->header('Content-Type', $type);
```

Звичайно, якщо вам не потрібно передавати власний код HTTP-статуса або власні заголовки, ви можете використати глобальну функцію `view`:

<a name="json-responses"></a>

### Відповіді JSON

Метод `json` автоматично встановить заголовок `Content-Type` зі значенням `application/json`, а також перетворить отриманий масив в JSON використовуючи PHP функцію `json_encode`:

```php
return response()->json([
    'name' => 'Abigail',
    'state' => 'CA',
]);
```

Якщо ви бажаєте створити відповідь JSONP, ви можете використати метод `json` у поєднанні з методом `withCallback`:

```php
return response()
    ->json(['name' => 'Abigail', 'state' => 'CA'])
    ->withCallback($request->input('callback'));
```

<a name="file-downloads"></a>

### Відповіді з завантаженням файлів

Метод `download` може бути викорстаний для створення відповіді, яка змусить браузер користувача завантажити файл за вказаним шляхом.
Метод `download` приймає ім'я файла другим аргументом, який визначить назву файла, яку побачить користувач, завантаженого файла. Нарешті, ви можете передати масив заголовків HTTP, третім аргументов:

```php
return response()->download($pathToFile);

return response()->download($pathToFile, $name, $headers);
```

> **Warning**  
> Symfony HttpFoundation, який керує завантаженням файлів, вимагає, щоб ім'я завантажуваного файла було в кодуванні ASCII.

<a name="streamed-downloads"></a>

#### Потокове завантаження

За бажанням можна перетворити рядкову відповідь переданої функції на відповідь, яка завантажується, без необхідності записувати результуючий вміст на диск. В цьому сценарії ви можете використовувати метод `streamDownload`. Цей метод приймає замикання, ім'я файла, та необов'язковий масив заголовків третім аргументом:

```php
use App\Services\GitHub;

return response()->streamDownload(function () {
    echo GitHub::api('repo')
        ->contents()
        ->readme('laravel', 'laravel')['contents'];
}, 'laravel-readme.md');
```

<a name="file-responses"></a>

### Відповіді, які відображають вміст файлів

Метод `file` може використовуватись, щоб відобразити файл, як зображення або PDF, безпосередньо в браузері користувача замість ініціювання завантаження. Цей метод приймеє шлях до файла першим аргументом, та масив заголовків другим аргументом:

```php
return response()->file($pathToFile);

return response()->file($pathToFile, $headers);
```

<a name="response-macros"></a>

## Макрос відповіді

Якщо ви бажаєте визначити власну відповідь, яку пожна використовувати повторно в різних маршрутах та контроллерах, ви можете використати метод `macro` фасада `Response`. Як правило, вам варто викликати цей метод з метода `boot`, одного з [постачальників послуг](providers.md) вашого додатка, таких як `App\Providers\AppServiceProvider`:

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Response;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Завантажте будь-які служби додатків.
     *
     * @return void
     */
    public function boot()
    {
        Response::macro('caps', function ($value) {
            return Response::make(strtoupper($value));
        });
    }
}
```

Метод `macro` приймає ім'я першим аргументом, другим аргументом замикання. Замикання макроса буде виконано під час виклику імені макроса в реалізації `ResponseFactory` або помічника `response`:

```php
return response()->caps('foo');
```
