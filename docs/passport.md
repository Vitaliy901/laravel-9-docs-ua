# Laravel Passport

- [Вступ](#introduction)
  - [Passport чи Sanctum](#passport-or-sanctum)
- [Встановлення](#installation)
  - [Розгортання Passport](#deploying-passport)
  - [Налаштування міграції](#migration-customization)
  - [Оновлення Passport](#upgrading-passport)
- [Налаштування](#configuration)
  - [Хешування секретного ключа клієнта](#client-secret-hashing)
  - [Термін життя токена](#token-lifetimes)
  - [Перевизначення моделей за умовчанням](#overriding-default-models)
- [Видача токенів доступу](#issuing-access-tokens)
  - [Керування клієнтами](#managing-clients)
  - [Запит на отримання токенів](#requesting-tokens)
  - [Оновлення токенів](#refreshing-tokens)
  - [Відкликання токенів](#revoking-tokens)
  - [Очищення токенів](#purging-tokens)
- [Надання коду авторизації за допомогою PKCE](#code-grant-pkce)
  - [Створення клієнта](#creating-a-auth-pkce-grant-client)
  - [Запит на отримання токенів](#requesting-auth-pkce-grant-tokens)
- [Парольні токени доступу](#password-grant-tokens)
  - [Створення парольних токенів доступу](#creating-a-password-grant-client)
  - [Запит на отримання токенів](#requesting-password-grant-tokens)
  - [Запит всіх областей](#requesting-all-scopes)
  - [Налаштування постачальника служб користувача](#customizing-the-user-provider)
  - [Налаштування поля імені користувача](#customizing-the-username-field)
  - [Налаштування перевірки пароля](#customizing-the-password-validation)
- [Неявні токени](#implicit-grant-tokens)
- [Облікові дані клієнта - надання Токенів](#client-credentials-grant-tokens)
- [Особисті токени доступу](#personal-access-tokens)
  - [Створення клієнта персонального доступу](#creating-a-personal-access-client)
  - [Керування особистими токенами доступу](#managing-personal-access-tokens)
- [Захист маршрутів](#protecting-routes)
  - [Через посередник](#via-middleware)
  - [Передача токена доступу](#passing-the-access-token)
- [Області дії токена](#token-scopes)
  - [Визначення областей](#defining-scopes)
  - [Область за замовчуванням](#default-scope)
  - [Призначення областей токенам](#assigning-scopes-to-tokens)
  - [Перевірка областей](#checking-scopes)
- [Використання вашого API за допомогою JavaScript](#consuming-your-api-with-javascript)
- [Події](#events)
- [Тестування](#testing)

<a name="introduction"></a>

## Вступ

[Passport Laravel](https://github.com/laravel/passport) забезпечує повну реалізацію сервера OAuth2 для вашого додатка Laravel за лічені хвилини. Passport створено на основі сервера [League OAuth2](https://github.com/thephpleague/oauth2-server), який обслуговують Andy Millington і Simon Hamp.

> **Warning**  
> Ця документація передбачає, що ви вже знайомі з OAuth2. Якщо ви нічого не знаєте про OAuth2, спробуйте ознайомитися із загальною [термінологією](https://oauth2.thephpleague.com/terminology/) та особливостями OAuth2, перш ніж продовжити.

<a name="passport-or-sanctum"></a>

### Passport чи Sanctum?

Перш ніж почати, ви можете визначитися, чи буде ваш додаток краще обслуговуватися через Laravel Passport або [Laravel Sanctum](sanctum.md). Якщо ваш додаток потребує підтримки OAuth2, вам слід використовувати Laravel Passport.

Однак, якщо ви намагаєтеся автентифікувати односторінковий додаток, мобільний додаток або видаєте токени API, вам слід використовувати [Laravel Sanctum](sanctum.md). Laravel Sanctum не підтримує OAuth2; однак він забезпечує набагато простіший досвід розробки автентифікації API.

<a name="installation"></a>

## Встановлення

Для початку встановіть Passport через менеджер пакетів Composer:

```shell
composer require laravel/passport
```

[Сервіс-провайдер](providers.md) Passport реєструє власний каталог з міграціями бази даних, тому вам слід виконати міграцію бази даних після встановлення пакета. Міграції Passport створять таблиці, необхідні вашому додатку для зберігання клієнтів OAuth2 і токенів доступу:

```shell
php artisan migrate
```

Далі потрібно виконати команду Artisan `passport:install`. Ця команда створить ключі шифрування, необхідні для створення токенів безпечного доступу. Крім того, команда створить клієнти «personal access» та «password grant», які використовуватимуться для створення токенів доступу:

```shell
php artisan passport:install
```

> **Note**  
> Якщо ви бажаєте використовувати UUID як значення первинного ключа моделі `Client` Passport замість цілих чисел з автоінкрементом, встановіть Passport за допомогою [параметра `uuids`](#client-uuids).

Після виконання команди `passport:install` додайте трейт `Laravel\Passport\HasApiTokens` до моделі App\Models\User. Цей трейт надасть декілька допоміжних методів для вашої моделі, які дозволять вам перевіряти токен автентифікованого користувача і області. Якщо ваша модель вже використовує трейт `Laravel\Sanctum\HasApiTokens`, ви можете видалити цей трейт:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

Нарешті, у конфігураційному файлі додатка `config/auth.php` ви повинні визначити захист автентифікації для `api` та встановити параметру `driver` значення `passport`. Це вказує вашому додатку використовувати `TokenGuard` Passport під час автентифікації вхідних запитів API:

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

<a name="client-uuids"></a>

#### Клієнт UUIDs

Ви також можете виконати команду `passport:install` із параметром `--uuids`. Цей параметр вкаже Passport, що ви хочете використовувати UUID замість цілих чисел з автоінкрементом як значення первинного ключа моделі `Client` Passport.
Після виконання команди `passport:install` із параметром `--uuids` ви отримаєте додаткові інструкції щодо вимкнення стандартних міграцій Passport:

```shell
php artisan passport:install --uuids
```

<a name="deploying-passport"></a>

### Розгортання Passport

Під час першого розгортання Passport на серверах вашого додатка вам, ймовірно, потрібно буде запустити команду `passport:keys`. Ця команда генерує ключі шифрування, необхідні Passport для створення токенів доступа. Згенеровані ключі зазвичай не зберігаються в системі контролю версій:

```shell
php artisan passport:keys
```

При необхідності ви можете визначити шлях, звідки повинні завантажуватися ключі Passport. Для цього можна використати метод `Passport::loadKeysFrom`. Як правило, цей метод слід викликати з метода `boot` класа `App\Providers\AuthServiceProvider` вашого додатка:

```php
/**
 * Зареєструйте будь-які служби аутентифікації / авторизації.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
}
```

<a name="loading-keys-from-the-environment"></a>

#### Завантаження ключів із середовища

Крім того, ви можете опублікувати файл конфігурації Passport за допомогою команди `vendor:publish` Artisan:

```shell
php artisan vendor:publish --tag=passport-config
```

Після публікації файла конфігурації ви можете завантажити ключі шифрування вашого додатка, визначивши їх як змінні середовища:

```ini
PASSPORT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
<private key here>
-----END RSA PRIVATE KEY-----"

PASSPORT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
<public key here>
-----END PUBLIC KEY-----"
```

<a name="migration-customization"></a>

### Налаштування міграції

Якщо ви не збираєтеся використовувати стандартні міграції Passport, вам слід викликати метод `Passport::ignoreMigrations` в методі `register` вашого класа `App\Providers\AppServiceProvider`. Ви можете експортувати стандартні міграції за допомогою команди `vendor:publish` Artisan:

```shell
php artisan vendor:publish --tag=passport-migrations
```

<a name="upgrading-passport"></a>

### Оновлення Passport

Під час оновлення до нової основної версії Passport важливо уважно переглянути [посібник з оновлення](https://github.com/laravel/passport/blob/master/UPGRADE.md).

<a name="configuration"></a>

## Налаштування

<a name="client-secret-hashing"></a>

### Хешування секретного ключа клієнта

Якщо ви бажаєте, щоб секретні ключі клієнта хешувалися під час зберігання в базі даних, вам слід викликати метод `Passport::hashClientSecrets` в методі `boot` вашого класа `App\Providers\AuthServiceProvider`:

```php
use Laravel\Passport\Passport;

Passport::hashClientSecrets();
```

Після додавання цього метода всі секретні ключі, відображатимуться користувачеві лише один раз, одразу після їх створення. Оскільки значення секретного ключа у вигляді звичайного текста ніколи не зберігається в базі даних, неможливо відновити значення ключа, якщо воно втрачено.

<a name="token-lifetimes"></a>

### Термін життя токена

За замовчуванням Passport видає довгострокові токени доступу, термін дії яких закінчується через один рік. Якщо ви хочете налаштувати довший або коротший термін дії токена, ви можете використовувати методи `tokensExpireIn`, `refreshTokensExpireIn` і `personalAccessTokensExpireIn`. Ці методи слід викликати з метода `boot` класа `App\Providers\AuthServiceProvider` вашого додатка

```php
/**
 * Зареєструйте будь-які служби аутентифікації / авторизації.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Passport::tokensExpireIn(now()->addDays(15));
    Passport::refreshTokensExpireIn(now()->addDays(30));
    Passport::personalAccessTokensExpireIn(now()->addMonths(6));
}
```

> **Warning**  
> Стовпці `expires_at` в таблицях бази даних Passport доступні лише для читання та лише для відображення. Під час випуску токенів Passport зберігає інформацію про закінчення терміну дії в підписаних і зашифрованих токенах. Якщо вам потрібно зробити токен недійсним, його слід [відкликати](#revoking-tokens).

<a name="overriding-default-models"></a>

### Перевизначення моделей за умовчанням

Ви можете розширити моделі, які використовуються всередині Passport, визначивши власну модель і розширивши відповідну модель Passport:

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

Після визначення вашої моделі ви можете вказати Passport використовувати вашу модель через клас `Laravel\Passport\Passport`. Як правило, ви повинні повідомити Passport про власні моделі в методі `boot` класа `App\Providers\AuthServiceProvider` вашого додатка:

```php
use App\Models\Passport\AuthCode;
use App\Models\Passport\Client;
use App\Models\Passport\PersonalAccessClient;
use App\Models\Passport\Token;

/**
 * Register any authentication / authorization services.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Passport::useTokenModel(Token::class);
    Passport::useClientModel(Client::class);
    Passport::useAuthCodeModel(AuthCode::class);
    Passport::usePersonalAccessClientModel(PersonalAccessClient::class);
}
```

<a name="issuing-access-tokens"></a>

## Видача токенів доступу

Використання OAuth2 через коди авторизації – це те, через що більшість розробників знайомляться з OAuth2. Під час використання кодів авторизації клієнтський додаток перенаправлятиме користувача на ваш сервер, де він схвалить або відхилить запит на видачу токена доступа клієнту.

<a name="managing-clients"></a>

### Керування клієнтами

По-перше, розробники, яким необхідно взаємодіяти з API вашого додатка, повинні будуть зареєструвати свій додаток у вашому, створивши «клієнта» (client). Як правило, це складається з надання назви свого додатка та URL-адреси, на яку ваш додаток може перенаправити після того, як користувачі схвалять свій запит на авторизацію.

<a name="the-passportclient-command"></a>

#### Команда `passport:client`

Найпростішим способом створення клієнта є використання команди `passport:client` Artisan. Цю команду можна використовувати для створення власних клієнтів для тестування функціональності OAuth2. Коли ви запускаєте команду `client`, Passport запросить у вас додаткову інформацію про вашого клієнта і надасть вам ID та секретний ключ:

```shell
php artisan passport:client
```

**URL-адреси перенаправлення**

Якщо ви хочете дозволити декілька URL-адрес перенаправлення для вашого клієнта, ви можете вказати їх за допомогою списку, розділеного комами, коли команда `passport:client` запитує URL-адресу. Всі URL-адреси, які містять коми, мають бути закодовані:

```shell
http://example.com/callback,http://examplefoo.com/callback
```

<a name="clients-json-api"></a>

#### JSON API

Оскільки користувачі вашого додатка не зможуть використовувати команду `client`, Passport надає JSON API, який ви можете використовувати для створення клієнтів. Це позбавить вас від необхідності вручну кодувати контролери для створення, оновлення та видалення клієнтів.

Однак вам потрібно буде об’єднати JSON API Passport із вашим власним зовнішнім інтерфейсом, щоб надати користувачам інформаційну панель для керування своїми клієнтами. Нижче ми розглянемо всі кінцеві точки API для керування клієнтами. Для зручності ми використаємо [Axios](https://github.com/axios/axios), щоб продемонструвати виконання HTTP-запитів до кінцевих точок.

JSON API захищається `web` та `auth` посередником; отже, його можна викликати лише з вашого власного додатка. Його не можна викликати із зовнішнього джерела.

<a name="get-oauthclients"></a>

#### `GET /oauth/clients`

Цей маршрут повертає всіх клієнтів для автентифікованого користувача. Цей маршрут насамперед корисний для перегляду всіх клієнтів користувача, щоб вони могли редагувати або видаляти їх:

```js
axios.get("/oauth/clients").then((response) => {
  console.log(response.data);
});
```

<a name="post-oauthclients"></a>

#### `POST /oauth/clients`

Цей маршрут використовується для створення нових клієнтів. Для цього потрібні два поля: `name` та `redirect` URL. URL-адреса перенаправлення – це місце, куди буде перенаправлено користувача після схвалення або відхилення запита на авторизацію.

Коли клієнт буде створено, йому буде видано ID та секретний ключ клієнта. Ці значення будуть використовуватися під час запита токенів доступу від вашого додатка. Маршрут створення клієнта поверне новий екземпляр клієнта:

```js
const data = {
  name: "Client Name",
  redirect: "http://example.com/callback",
};

axios
  .post("/oauth/clients", data)
  .then((response) => {
    console.log(response.data);
  })
  .catch((response) => {
    // List errors on response...
  });
```

<a name="put-oauthclientsclient-id"></a>

#### `PUT /oauth/clients/{client-id}`

Цей маршрут використовується для оновлення клієнтів. Для цього потрібні два поля: `name` та `redirect` URL. URL-адреса перенаправлення – це місце, куди буде перенаправлено користувача після схвалення або відхилення запита на авторизацію. Маршрут поверне оновлений екземпляр клієнта:

```js
const data = {
  name: "New Client Name",
  redirect: "http://example.com/callback",
};

axios
  .put("/oauth/clients/" + clientId, data)
  .then((response) => {
    console.log(response.data);
  })
  .catch((response) => {
    // Список помилок у відповіді...
  });
```

<a name="delete-oauthclientsclient-id"></a>

#### `DELETE /oauth/clients/{client-id}`

Цей маршрут використовується для видалення клієнтів:

```js
axios.delete("/oauth/clients/" + clientId).then((response) => {
  //
});
```

<a name="requesting-tokens"></a>

### Запит на отримання токенів

<a name="requesting-tokens-redirecting-for-authorization"></a>

#### Перенаправлення для авторизації

Після створення клієнта розробники можуть використовувати свій ідентифікатор клієнта та секретний ключ для того, щоб запитати код авторизації та токен доступу у вашому додатку. По-перше, додаток-споживача має зробити запит на перенаправлення до маршрута `/oauth/authorize` вашого додатка таким чином:

```php
use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => '',
        'state' => $state,
    ]);

    return redirect('http://passport-app.test/oauth/authorize?'.$query);
});
```

> **Note**  
> Пам’ятайте, що маршрут `/oauth/authorize` вже визначено в Passport. Вам не потрібно вручну визначати цей маршрут.

<a name="approving-the-request"></a>

#### Підтвердження запита

Після отримання запитів на авторизацію, Passport автоматично відображає шаблон для користувача, який дозволяє підтвердити або відхилити запит авторизації. Якщо вони схвалять запит, вони будуть перенаправлені назад до `redirect_uri`, який було вказано в додатку-споживачем. Адреса `redirect_uri` має відповідати URL-адресі `redirect`, яка була вказана під час створення клієнта.

Якщо ви бажаєте налаштувати екран підтвердження авторизації, ви можете опублікувати шаблони Passport за допомогою команди `vendor:publish` Artisan. Опубліковані шаблони будуть розміщені в каталозі `resources/views/vendor/passport`:

```shell
php artisan vendor:publish --tag=passport-views
```

Іноді вам може знадобитись пропустити запит на авторизацію, наприклад, під час авторизації основного клієнта. Ви можете досягти цього, [розширивши модель `Client`](#overriding-default-models) та визначивши метод `skipsAuthorization`. Якщо `skipsAuthorization` повертає `true`, клієнт буде схвалено, а користувача буде негайно перенаправлено назад до `redirect_uri`:

```php
<?php

namespace App\Models\Passport;

use Laravel\Passport\Client as BaseClient;

class Client extends BaseClient
{
    /**
     * Визначте, чи повинен клієнт пропускати запит авторизації.
     *
     * @return bool
     */
    public function skipsAuthorization()
    {
        return $this->firstParty();
    }
}
```

<a name="requesting-tokens-converting-authorization-codes-to-access-tokens"></a>

#### Перетворення кодів авторизації на токени доступу

Якщо користувач схвалить запит на авторизацію, він буде перенаправлений назад до додатка-споживача. Споживач повинен спочатку порівняти параметр `state` зі значенням, яке було збережено до перенаправлення. Якщо параметр `state` збігається, споживач повинен надіслати вашому додатку запит `POST`, щоб отримати токен доступу. Запит має містити код авторизації, виданий вашим додатком, коли користувач схвалив запит авторизації:

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class
    );

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'code' => $request->code,
    ]);

    return $response->json();
});
```

Цей маршрут `/oauth/token` поверне відповідь JSON, який містить атрибути `access_token`, `refresh_token` і `expires_in`. Атрибут `expires_in` містить кількість секунд до закінчення терміну дії токена доступу.

> **Note**  
> Як і маршрут `/oauth/authorize`, маршрут `/oauth/token` визначається для вас за допомогою Passport. Немає необхідності вручну визначати цей маршрут.

<a name="tokens-json-api"></a>

#### JSON API

Passport також містить JSON API для керування токенами авторизованого доступу. Ви можете поєднати це з вашим власним інтерфейсом, щоб запропонувати своїм користувачам інформаційну панель для керування токенами доступу. Для зручності ми використаємо [Axios](https://github.com/mzabriskie/axios), щоб продемонструвати виконання HTTP-запитів до кінцевих точок. JSON API захищається `web` та `auth` посередником; отже, його можна викликати лише з вашого власного додатка.

<a name="get-oauthtokens"></a>

#### `GET /oauth/tokens`

Цей маршрут повертає всі авторизовані токени доступу, створені користувачем, який автентифікований. Цей маршрут насамперед корисний для перегляду всіх токенів користувача, щоб вони могли їх відкликати:

```js
axios.get("/oauth/tokens").then((response) => {
  console.log(response.data);
});
```

<a name="delete-oauthtokenstoken-id"></a>

#### `DELETE /oauth/tokens/{token-id}`

Цей маршрут можна використовувати для відкликання авторизованих токенів доступу та пов’язаних із ними токенів оновлення:

```js
axios.delete("/oauth/tokens/" + tokenId);
```

<a name="refreshing-tokens"></a>

### Оновлення токенів

Якщо ваш додаток видає короткочасні токени доступу, користувачам потрібно буде оновити свої токени доступу за допомогою токена оновлення, наданого їм під час видачі токена доступу:

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'refresh_token',
    'refresh_token' => 'the-refresh-token',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'scope' => '',
]);

return $response->json();
```

Цей маршрут `/oauth/token` поверне відповідь JSON, яка містить атрибути `access_token`, `refresh_token` і `expires_in`. Атрибут `expires_in` містить кількість секунд до закінчення терміну дії токена доступу.

<a name="revoking-tokens"></a>

### Відкликання токенів

Ви можете відкликати токен за допомогою метода `revokeAccessToken` в `Laravel\Passport\TokenRepository`. Ви можете відкликати токен оновлення, який оновлює токен доступу, за допомогою метода `revokeRefreshTokensByAccessTokenId` у `Laravel\Passport\RefreshTokenRepository`. Ці класи можна вирішити за допомогою [службового контейнера](container.md): Laravel:

```php
use Laravel\Passport\TokenRepository;
use Laravel\Passport\RefreshTokenRepository;

$tokenRepository = app(TokenRepository::class);
$refreshTokenRepository = app(RefreshTokenRepository::class);

// Відкликати токен доступу...
$tokenRepository->revokeAccessToken($tokenId);

// Відкликати всі токени оновлення, які оновлюють токен доступу...
$refreshTokenRepository->revokeRefreshTokensByAccessTokenId($tokenId);
```

<a name="purging-tokens"></a>

### Очищення токенів

Якщо токени відкликано або термін їх дії закінчився, ви можете видалити їх з бази даних. Команда `passport:purge` Artisan може зробити це за вас:

```shell
# Очистити відкликані токени та токени, у яких закінчився термін дії, а також їх коди авторизації...
php artisan passport:purge

# Очистити лише відкликані токени та їх коди авторизації...
php artisan passport:purge --revoked

# Очистити лише токени термін діх яких закінчився, та їх коди авторизації...
php artisan passport:purge --expired
```

Ви також можете налаштувати [заплановане завдання](scheduling.md) в класі `App\Console\Kernel` вашого додатка для автоматичного скорочення токенів за розкладом:

```php
/**
 * Визначте розклад команд додатка.
 *
 * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
 * @return void
 */
protected function schedule(Schedule $schedule)
{
    $schedule->command('passport:purge')->hourly();
}
```

<a name="code-grant-pkce"></a>

## Надання коду авторизації за допомогою PKCE

Надання коду авторизації за допомогою `Proof Key for Code Exchange` (PKCE), — це безпечний спосіб автентифікації односторінкових додатків або власних додатків для доступу до вашого API. Цей дозвіл слід використовувати, коли ви не можете гарантувати, що секретний ключ клієнта зберігатиметься конфіденційно, або щоб зменшити загрозу перехоплення коду авторизації зловмисником. Поєднання `code verifier` та `code challenge` замінює секретний ключ клієнта під час обміну кодом авторизації на токен доступу.

<a name="creating-a-auth-pkce-grant-client"></a>

### Створення клієнта

Перш ніж ваш додаток зможе видавати токени через надання коду авторизації за допомогою PKCE, вам потрібно буде створити клієнт із підтримкою PKCE. Ви можете зробити це за допомогою команди `passport:client` Artisan з параметром --public:

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>

### Запит на отримання токенів

<a name="code-verifier-code-challenge"></a>

#### Code Verifier & Code Challenge

Оскільки цей дозвіл на авторизацію не надає секретний ключ клієнта, розробникам потрібно буде згенерувати комбінацію `сode Verifier` та `code challenge`, щоб здійснити запит на отримання токена.

`Code Verifier` має бути випадковим рядком від 43 до 128 символів, що містить літери, цифри та символи `"-"`, `"."`, `"_"`, `"~"` як визначено в [специфікації RFC 7636](https://tools.ietf.org/html/rfc7636).

`Сode challenge` має бути рядком у кодуванні Base64 із безпечними символами URL-адресою та безпечними для імені файла символами. Кінцеві символи `'='` мають бути видалені, і не повинно бути розривів рядків, пробілів чи інших додаткових символів.

```php
$encoded = base64_encode(hash('sha256', $code_verifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```

<a name="code-grant-pkce-redirecting-for-authorization"></a>

#### Перенаправлення для авторизації

Після створення клієнта ви можете використати ID клієнта та згенеровані `сode Verifier` та `code challenge`, щоб запитати код авторизації та токен доступу у вашому додатку. По-перше, додаток-споживач має зробити запит перенаправлення на маршрут вашого додатка `/oauth/authorize`:

```php
use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $request->session()->put(
        'code_verifier', $code_verifier = Str::random(128)
    );

    $codeChallenge = strtr(rtrim(
        base64_encode(hash('sha256', $code_verifier, true))
    , '='), '+/', '-_');

    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => '',
        'state' => $state,
        'code_challenge' => $codeChallenge,
        'code_challenge_method' => 'S256',
    ]);

    return redirect('http://passport-app.test/oauth/authorize?' . $query);
});
```

<a name="code-grant-pkce-converting-authorization-codes-to-access-tokens"></a>

#### Перетворення кодів авторизації на токени доступу

Якщо користувач схвалить запит на авторизацію, він буде перенаправлений назад до додатка-споживача. Споживач повинен перевірити параметр `state` на значення, яке було збережено до перенаправлення, як при стандартному наданні кода авторизації.

Якщо параметр стану збігається, споживач повинен надіслати вашій програмі запит `POST`, щоб отримати маркер доступу. Запит має містити код авторизації, який було видано вашою програмою, коли користувач схвалив запит на авторизацію разом із початково згенерованим верифікатором коду:

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');

    $codeVerifier = $request->session()->pull('code_verifier');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class
    );

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'code_verifier' => $codeVerifier,
        'code' => $request->code,
    ]);

    return $response->json();
});
```

<a name="password-grant-tokens"></a>

## Парольні токени доступу

> **Warning**  
> Ми більше не рекомендуємо використовувати парольні токени доступу. Натомість вам слід обрати [тип дозволу, який в даний час рекомендовано сервером OAuth2.](https://oauth2.thephpleague.com/authorization-server/which-grant/).

Надання пароля OAuth2 дозволяє вашим іншим основним клієнтам, таким як мобільний додаток, отримати токен доступу за допомогою адреси електронної пошти / імені користувача та пароля. Це дає змогу безпечно видавати токени доступу вашим основним клієнтам, не вимагаючи від користувачів проходити весь процес перенаправлень коду авторизації OAuth2.

<a name="creating-a-password-grant-client"></a>

### Створення парольних токенів доступу

Перш ніж ваш додаток зможе видавати токени через парольні токени, вам потрібно буде створити клієнт надання пароля. Ви можете зробити це за допомогою команди `passport:client` Artisan із параметром `--password`. **Якщо ви вже запустили команду `passport:install`, вам не потрібно виконувати цю команду:**

```shell
php artisan passport:client --password
```

<a name="requesting-password-grant-tokens"></a>

### Запит на отримання парольних токенів

Створивши клієнт для надання пароля, ви можете запитати токен доступу, надіславши запит `POST` до маршрута `/oauth/token` з адресою електронної пошти та паролем користувача. Пам'ятайте, що цей маршрут вже зареєстрований Passport, тому немає необхідності визначати його вручну. Якщо запит виконано успішно, ви отримаєте `access_token`, `expires_in` і `refresh_token` у відповіді JSON від сервера:

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'username' => 'taylor@laravel.com',
    'password' => 'my-password',
    'scope' => '',
]);

return $response->json();
```

Пам’ятайте, що токени доступу доступу за замовчуванням є довгостроковими. Однак ви можете налаштувати максимальний термін служби маркера доступу, якщо це необхідно.

> **Note**  
> Пам’ятайте, що токени доступу доступу за замовчуванням є довгостроковими. Однак ви можете налаштувати максимальний термін служби маркера доступу, якщо це необхідно.

<a name="requesting-all-scopes"></a>

### Запит всіх областей

При використанні доступу за паролем або доступу з обліковими даними клієнта ви можете авторизувати токен для всіх областей, які підтримуються вашим додатком. Ви можете зробити це, запросивши область `*`. Якщо ви запитуєте область `*`, метод `can` екземпляра токена завжди повертатиме значення `true`. Цю область можна призначити лише токену, виданому за допомогою `password` або дозволу `client_credentials`:

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'username' => 'taylor@laravel.com',
    'password' => 'my-password',
    'scope' => '*',
]);
```

<a name="customizing-the-user-provider"></a>

### Налаштування провайдера користувача

Якщо ваш додаток використовує більше ніж один [провайдер автентифікації користувача](authentication.md#introduction), ви можете вказати, який провайдер використовує клієнт надання пароля, надавши параметр `--provider` під час створення клієнта за допомогою команди `artisan passport:client --password`. Вказана назва провайдера має збігатися з дійсним постачальником, визначеним у конфігураційному файлі додатка `config/auth.php`. Потім ви можете [захистити свій маршрут за допомогою посередника](#via-middleware), щоб гарантувати авторизацію лише користувачів із зазначеного постачальника `guard`.

<a name="customizing-the-username-field"></a>

### Налаштування поля імені користувача

Під час автентифікації за допомогою надання пароля, Passport використовуватиме атрибут `email` вашої автентифікованої моделі як «ім’я користувача». Однак ви можете налаштувати цю поведінку, визначивши метод `findForPassport` у своїй моделі:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;

    /**
     * Повертає екземпляр користувача для надісланого імені.
     *
     * @param  string  $username
     * @return \App\Models\User
     */
    public function findForPassport($username)
    {
        return $this->where('username', $username)->first();
    }
}
```

<a name="customizing-the-password-validation"></a>

### Налаштування перевірки пароля користувача

Під час автентифікації за допомогою надання пароля, Passport використовуватиме атрибут `password` вашої моделі для перевірки наданого пароля. Якщо ваша модель не має атрибут `password` або ви бажаєте налаштувати логіку перевірки пароля, ви можете визначити метод `validateForPassportPasswordGrant` у своїй моделі:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Support\Facades\Hash;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;

    /**
     * Перевірте пароль користувача для надання дозволу.
     *
     * @param  string  $password
     * @return bool
     */
    public function validateForPassportPasswordGrant($password)
    {
        return Hash::check($password, $this->password);
    }
}
```

<a name="implicit-grant-tokens"></a>

## Неявні токени

> **Warning**  
> Ми більше не рекомендуємо використовувати неявні токени доступу. Натомість вам слід обрати [тип дозволу, який зараз рекомендовано сервером OAuth2.](https://oauth2.thephpleague.com/authorization-server/which-grant/).

Неявний дозвіл подібний до надання коду авторизації; однак токен повертається клієнту без обміну кодом авторизації.
Цей дозвіл найчастіше використовується для JavaScript або мобільних додатків, де облікові дані клієнта неможливо безпечно зберегти. Щоб увімкнути дозвіл, викличте метод `enableImplicitGrant` в методі `boot` класа `App\Providers\AuthServiceProvider` вашого додатка:

```php
/**
 * Реєстрація сервісів автентифікації та авторизації.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Passport::enableImplicitGrant();
}
```

Після ввімкнення дозволу розробники можуть використовувати свій ID клієнта для запиту токена доступу від вашого додатка. Додаток-споживач має зробити запит на перенаправлення до маршруту `/oauth/authorize` вашого додатка таким чином:

```php
use Illuminate\Http\Request;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'response_type' => 'token',
        'scope' => '',
        'state' => $state,
    ]);

    return redirect('http://passport-app.test/oauth/authorize?' . $query);
});
```

> **Note**  
> Пам’ятайте, що маршрут /oauth/authorize вже визначено в Passport. Вам не потрібно вручну визначати цей маршрут.

<a name="client-credentials-grant-tokens"></a>

## Токени облікових даних

Надання облікових даних клієнта підходить для міжмашинної автентифікації (machine-to-machine). Наприклад, ви можете використовувати цей дозвіл в запланованому завданні, яке виконує завдання обслуговування через API.

Перш ніж ваш додаток зможе видавати токени через надання облікових даних клієнта, вам потрібно буде створити клієнт надання облікових даних. Ви можете зробити це за допомогою параметра `--client` команди `passport:client` Artisan:

```shell
php artisan passport:client --client
```

Далі, щоб використовувати цей тип дозволу, вам потрібно додати посередник `CheckClientCredentials` до властивості `$routeMiddleware` вашого файла `app/Http/Kernel.php`:

```php
use Laravel\Passport\Http\Middleware\CheckClientCredentials;

protected $routeMiddleware = [
    'client' => CheckClientCredentials::class,
];
```

Потім приєднайте посередник до маршрута:

```php
Route::get('/orders', function (Request $request) {
    ...
})->middleware('client');
```

Щоб обмежити доступ до маршруту певними областями, ви можете надати список необхідних областей, розділених комами, під час підключення `client` посердника до маршрута:

```php
Route::get('/orders', function (Request $request) {
    ...
})->middleware('client:check-status,your-scope');
```

<a name="retrieving-tokens"></a>

### Отримання токенів

Щоб отримати токен за допомогою цього типу дозволу, зробіть запит до кінцевої точки `oauth/token`:

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'client_credentials',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'scope' => 'your-scope',
]);

return $response->json()['access_token'];
```

<a name="personal-access-tokens"></a>

## Особисті токени доступу

Іноді ваші користувачі можуть захотіти видати собі токен доступу, не проходячи типової течії перенаправленнь кода авторизації. Дозвіл користувачам видавати собі токени через UI вашого додатка може бути корисним, для надання користувачам можливості експериментувати з вашим API або може бути більш простим підходом до видачі токенів доступу в цілому.

> **Note**
> Якщо ваш додаток в основному використовує Passport для видачі особистих токенів доступу, подумайте про використання [Laravel Sanctum](sanctum.md), легкої основної бібліотеки Laravel для видачі токенів доступу API.

<a name="creating-a-personal-access-client"></a>

### Створення токенів персонального доступу

Перш ніж ваш додаток зможе видавати особисті токени доступу, вам потрібно буде створити клієнт особистого доступу. Ви можете зробити це, виконавши команду `passport:client` Artisan із параметром `--personal`. Якщо ви вже запустили команду `passport:install`, вам не потрібно виконувати цю команду:

```shell
php artisan passport:client --personal
```

Після створення клієнта персонального доступу розмістіть ID клієнта та секретний ключ у вигляді звичайного тексту у файлі `.env` додатка:

```ini
PASSPORT_PERSONAL_ACCESS_CLIENT_ID="client-id-value"
PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET="unhashed-client-secret-value"
```

<a name="managing-personal-access-tokens"></a>

### Керування токенами персонального доступу

Після створення клієнта персонального доступу ви, можете видати токени для певного користувача за допомогою метода `createToken` в екземплярі моделі `App\Models\User`. Метод `createToken` приймає ім’я токена першим аргументом і додатковий масив [областей](#token-scopes) другим аргументом:

```php
use App\Models\User;

$user = User::find(1);

// Створення токена без областей...
$token = $user->createToken('Token Name')->accessToken;

// Створення токена з областями...
$token = $user->createToken('My Token', ['place-orders'])->accessToken;
```

<a name="personal-access-tokens-json-api"></a>

#### JSON API

Passport також містить JSON API для керування токенами особистого доступу. Ви можете поєднати це з вашим власним інтерфейсом, щоб запропонувати своїм користувачам інформаційну панель для керування особистими токенами доступу. Нижче ми розглянемо всі кінцеві точки API для керування особистими токенами доступу. Для зручності ми використаємо [Axios](https://github.com/mzabriskie/axios), щоб продемонструвати виконання HTTP-запитів до кінцевих точок.

JSON API захищається посередником для `web` and `auth`; отже, його можна викликати лише з вашго власного додатка. Його не можна викликати із зовнішнього джерела.

<a name="get-oauthscopes"></a>

#### `GET /oauth/scopes`

Цей маршрут повертає всі [області](#token-scopes), визначені для вашого додатка. Ви можете використовувати цей маршрут, щоб отримати список областей, які користувач може призначити особистому токену доступа:

```js
axios.get("/oauth/scopes").then((response) => {
  console.log(response.data);
});
```

<a name="get-oauthpersonal-access-tokens"></a>

#### `GET /oauth/personal-access-tokens`

Цей маршрут повертає всі особисті токени доступа, створені автентифікованим користувачем. Це в першу чергу корисно для отримання списку всіх токенів користувача, щоб вони могли редагувати або відкликати їх:

```js
axios.get("/oauth/personal-access-tokens").then((response) => {
  console.log(response.data);
});
```

<a name="post-oauthpersonal-access-tokens"></a>

#### `POST /oauth/personal-access-tokens`

Цей маршрут створює нові персональні токени доступа. Для цього потрібні дві частини даних: `name` токена та `scopes`, які слід призначити токенові:

```js
const data = {
  name: "Token Name",
  scopes: [],
};

axios
  .post("/oauth/personal-access-tokens", data)
  .then((response) => {
    console.log(response.data.accessToken);
  })
  .catch((response) => {
    // Список помилок у відповіді...
  });
```

<a name="delete-oauthpersonal-access-tokenstoken-id"></a>

#### `DELETE /oauth/personal-access-tokens/{token-id}`

Цей маршрут можна використовувати для відкликання особистих токенів доступа:

This route may be used to revoke personal access tokens:

```js
axios.delete("/oauth/personal-access-tokens/" + tokenId);
```

<a name="protecting-routes"></a>

## Захист маршрутів

<a name="via-middleware"></a>

### Через Middleware

Passport містить [захист автентифікації](authentication.md#adding-custom-guards), який перевіряє токени доступа під час вхідних запитів. Після того, як ви налаштували захист `api` для використання драйвера `passport`, вам потрібно лише вказати посередник `auth:api` на будь-яких маршрутах, для яких потрібен дійсний токен доступа:

```php
Route::get('/user', function () {
    //
})->middleware('auth:api');
```

> **Warning**  
> Якщо ви використовуєте [токени облікових даних клієнта](#client-credentials-grant-tokens), вам слід використовувати [посередник `client`](#client-credentials-grant-tokens) для захисту ваших маршрутів замість посередника `auth:api`.

<a name="multiple-authentication-guards"></a>

#### Множинний захист аутентифікації

Якщо ваш додаток автентифікує різні типи користувачів, які, можливо, використовують абсолютно різні моделі Eloquent, вам, імовірно, потрібно буде визначити конфігурацію захисту для кожного типу провайдера користувачів у вашому додатку. Це дозволяє захистити запити, призначені для конкретних постачальників служб. Наприклад, враховуючи наступну конфігурацію захисту, файл конфігурації `config/auth.php`:

```php
'api' => [
    'driver' => 'passport',
    'provider' => 'users',
],

'api-customers' => [
    'driver' => 'passport',
    'provider' => 'customers',
],
```

У наступному маршруті для автентифікації вхідних запитів буде використовуватися захист `api-customers`, який використовує провайдер користувача `customers`:

```php
Route::get('/customer', function () {
    //
})->middleware('auth:api-customers');
```

> **Note**  
> Щоб отримати докладнішу інформацію про використання декількох провайдерів користувачів з Passport, зверніться до [документації щодо надання пароля](#customizing-the-user-provider).

<a name="passing-the-access-token"></a>

### Передача токена доступа

Під час виклику маршрутів, які захищені Passport, користувачі API вашого додатка повинні вказати свій токен доступа як токен `Bearer` в заголовку `Authorization` свого запита. Наприклад, під час використання HTTP-бібліотеки Guzzle:

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => 'Bearer ' . $accessToken,
])->get('https://passport-app.test/api/user');

return $response->json();
```

<a name="token-scopes"></a>

## Області токенів

Області дозволяють вашим клієнтам API запитувати певний набір дозволів під час запита авторизації для доступу до облікового запису. Наприклад, якщо ви створюєте додаток електронної комерції, не всім користувачам API знадобиться можливість розміщувати замовлення. Натомість ви можете дозволити споживачам запитувати авторизацію лише для доступу до статусів доставки замовлень. Іншими словами, області дозволяють користувачам вашого додатка обмежувати дії, які інший додаток може виконувати від їх імені.

<a name="defining-scopes"></a>

### Визначення областей

Ви можете визначити область свого API за допомогою метода `Passport::tokensCan` у методі `boot` класа `App\Providers\AuthServiceProvider` вашого додатка. Метод `tokensCan` приймає масив імен і описів областей. Опис області може бути будь-яким за вашим бажанням і відображатиметься користувачам на екрані затвердження авторизації:

```php
/**
 * Реєстрація сервісів автентифікації та авторизації.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Passport::tokensCan([
        'place-orders' => 'Place orders',
        'check-status' => 'Check order status',
    ]);
}
```

<a name="default-scope"></a>

### Області за замовчуванням

Якщо клієнт не запитує жодних конкретних областей, ви можете налаштувати свій сервер Passport, щоб приєднати область(і) за замовчуванням до токена за допомогою метода `setDefaultScope`. Як правило, ви повинні викликати цей метод із метода `boot` класа `App\Providers\AuthServiceProvider` вашого додатка:

```php
use Laravel\Passport\Passport;

Passport::tokensCan([
    'place-orders' => 'Place orders',
    'check-status' => 'Check order status',
]);

Passport::setDefaultScope([
    'check-status',
    'place-orders',
]);
```

<a name="assigning-scopes-to-tokens"></a>

### Призначення областей токенам

<a name="when-requesting-authorization-codes"></a>

#### Під час запита кодів авторизації

Запитуючи токен доступу за допомогою надання коду авторизації, споживачі повинні вказати бажані `scope` у вигляді параметра рядка запита. Параметр `scope` має бути списком областей, розділених пробілами:

```php
Route::get('/redirect', function () {
    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://example.com/callback',
        'response_type' => 'code',
        'scope' => 'place-orders check-status',
    ]);

    return redirect('http://passport-app.test/oauth/authorize?' . $query);
});
```

<a name="when-issuing-personal-access-tokens"></a>

#### При видачі токенів особистого доступу

Якщо ви видаєте особисті токени доступу за допомогою метода `createToken` моделі `App\Models\User`, ви можете передати масив бажаних областей як другий аргумент метода:

```php
$token = $user->createToken('My Token', ['place-orders'])->accessToken;
```

<a name="checking-scopes"></a>

### Перевірка областей

Passport містить два посередника, які можна використовувати для перевірки автентифікації вхідного запита за допомогою токена, якому надано задану область. Щоб розпочати, додайте такий посередник до властивості `$routeMiddleware` вашого файла `app/Http/Kernel.php`:

```php
'scopes' => \Laravel\Passport\Http\Middleware\CheckScopes::class,
'scope' => \Laravel\Passport\Http\Middleware\CheckForAnyScope::class,
```

<a name="check-for-all-scopes"></a>

#### Перевірка всіх областей

Посередник `scopes` може бути призначено маршруту, щоб перевірити, чи токен доступа вхідного запита має всі перелічені області:

```php
Route::get('/orders', function () {
    // Токен доступа має всі області дії "перевірити статус" та "розмістити замовлення"...
})->middleware(['auth:api', 'scopes:check-status,place-orders']);
```

<a name="check-for-any-scopes"></a>

#### Перевірка будь-яких областей

Посередник `scope` може бути призначено маршруту, щоб перевірити, чи токен доступа вхідного запита має принаймні одну з перелічених областей:

```php
Route::get('/orders', function () {
    // Токен доступа має принаймні одну область дії "перевірити статус" або "розмістити замовлення"...
})->middleware(['auth:api', 'scope:check-status,place-orders']);
```

<a name="checking-scopes-on-a-token-instance"></a>

#### Перевірка областей на екземплярі токена

Після того як запит з автентифікацією токена доступу надійшов у ваш додаток, ви все ще можете перевірити, чи токен має задану область, використовуючи метод `tokenCan` на автентифікованому екземплярі `App\Models\User`:

```php
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    if ($request->user()->tokenCan('place-orders')) {
        //
    }
});
```

<a name="additional-scope-methods"></a>

#### Додаткові методи області

Метод `scopeIds` поверне масив всіх визначених IDs /імен:

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

Метод `scopes` поверне масив всіх визначених областей як екземпляри `Laravel\Passport\Scope`:

```php
Passport::scopes();
```

Метод `scopesFor` поверне масив екземплярів `Laravel\Passport\Scope`, які відповідають заданим IDs / іменам:

```php
Passport::scopesFor(['place-orders', 'check-status']);
```

Ви можете визначити, чи була визначена дана область за допомогою метода `hasScope`:

```php
Passport::hasScope('place-orders');
```

<a name="consuming-your-api-with-javascript"></a>

## Використання вашого API за допомогою JavaScript

Під час створення API може бути надзвичайно корисним мати можливість використовувати власний API із додатка JavaScript. Такий підхід до розробки API дозволяє вашому власному додатку використовувати той самий API, яким ви ділитеся зі світом. Той самий API може використовуватися вашим веб-додатком, мобільними додатками, сторонніми додатками та будь-якими SDK, які ви можете опублікувати в різних менеджерах пакетів.

Як правило, якщо ви хочете використовувати свій API із додатка JavaScript, вам потрібно буде вручну надіслати токен доступа до додатка та передавати його з кожним запитом до вашого додатка. Однак Passport містить посередника, який може впоратися з цим за вас. Все, що вам потрібно зробити, це додати посередник `CreateFreshApiToken` до вашої групи `web` посередників у вашому файлі `app/Http/Kernel.php`:

```php
'web' => [
    // Інші посередники...
    \Laravel\Passport\Http\Middleware\CreateFreshApiToken::class,
],
```

> **Warning**  
> Ви повинні переконатися, що посередник `CreateFreshApiToken` є останнім посередником, зазначеним у вашому списку посередників.

Цей посередник додаватиме файл cookie `laravel_token` до ваших вихідних відповідей. Цей файл cookie містить зашифрований JWT, який Passport використовуватиме для автентифікації запитів API від вашого додатка JavaScript. Час життя JWT дорівнює вашому значенню конфігурації `session.lifetime`. Тепер, оскільки браузер автоматично надсилатиме файли cookie з усіма наступними запитами, ви можете надсилати запити до API свого додатка без явної передачі токена доступу:

```php
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```

<a name="customizing-the-cookie-name"></a>

#### Налаштування імені Cookie

За потреби ви можете налаштувати назву файла cookie `laravel_token` за допомогою методу `Passport::cookie`. Як правило, цей метод слід викликати з метода `boot` класу `App\Providers\AuthServiceProvider` вашого додатка:

```php
/**
 * Зареєструйте будь-які служби аутентифікації / авторизації.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Passport::cookie('custom_name');
}
```

<a name="csrf-protection"></a>

#### CSRF - захист

Використовуючи цей метод автентифікації, вам потрібно буде переконатися, що до ваших запитів включено дійсний заголовок токена CSRF. До стандартних шаблонів JavaScript Laravel включає екземпляр Axios, який автоматично використовуватиме зашифроване значення файла cookie `XSRF-TOKEN` для надсилання заголовка `X-XSRF-TOKEN` на запити того самого походження.

> **Note**  
> Якщо ви вирішите надіслати заголовок `X-CSRF-TOKEN` замість `X-XSRF-TOKEN`, вам потрібно буде використовувати незашифрований токен, наданий `csrf_token()`.

<a name="events"></a>

## Події

Passport викликає події під час видачі токенів доступа та токенів оновлення. Ви можете використовувати ці події для видалення або відкликання інших токенів доступа в вашій базі даних. Якщо ви бажаєте, ви можете приєднати слухачів до цих подій в класі `App\Providers\EventServiceProvider` вашого додатка:

```php
/**
 * Передплатники подій програми.
 *
 * @var array
 */
protected $listen = [
    'Laravel\Passport\Events\AccessTokenCreated' => [
        'App\Listeners\RevokeOldTokens',
    ],

    'Laravel\Passport\Events\RefreshTokenCreated' => [
        'App\Listeners\PruneOldTokens',
    ],
];
```

<a name="testing"></a>

## Тестування

Метод `actingAs` Passport може використовуватися для визначення поточного автентифікованого користувача, а також його областей. Перший аргумент, наданий методу `actingAs`, — це екземпляр користувача, а другий — це масив областей, які слід надати токену користувача:

```php
use App\Models\User;
use Laravel\Passport\Passport;

public function test_servers_can_be_created()
{
    Passport::actingAs(
        User::factory()->create(),
        ['create-servers']
    );

    $response = $this->post('/api/create-server');

    $response->assertStatus(201);
}
```

Метод `actingAsClient` Passport може бути використаний для визначення поточного автентифікованого клієнта, а також його областей. Перший аргумент, наданий методу `actingAsClient`, — це екземпляр клієнта, а другий — це масив областей, які слід надати токену клієнта:

```php
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

public function test_orders_can_be_retrieved()
{
    Passport::actingAsClient(
        Client::factory()->create(),
        ['check-status']
    );

    $response = $this->get('/api/orders');

    $response->assertStatus(200);
}
```
