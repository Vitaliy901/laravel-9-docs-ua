# Eloquent: Ресурси API

- [Вступ](#introduction)
- [Створення ресурсів](#generating-resources)
- [Огляд концепції](#concept-overview)
  - [Колекції ресурсів](#resource-collections)
- [Написання ресурсів](#writing-resources)
  - [Пакування даних](#data-wrapping)
  - [Пагінація](#pagination)
  - [Умовні атрибути](#conditional-attributes)
  - [Умовні відношення](#conditional-relationships)
  - [Додавання метаданих](#adding-meta-data)
- [Відповіді ресурса](#resource-responses)

<a name="introduction"></a>

## Вступ

Під час створення API вам може знадобитися прошарок трансформації, який знаходиться між вашими моделями Eloquent і відповідями JSON, які фактично повертаються користувачам вашого додатка. Наприклад, за бажанням ви можете відображати певні атрибути для підмножини користувачів, а не для всіх, або ви можете завжди включати певні відношення в JSON-представлення своїх моделей. Класи ресурсів Eloquent дозволяють вам експресивно та легко перетворювати ваші моделі та колекції моделей у JSON.

Звичайно, ви завжди можете конвертувати моделі або колекції Eloquent у JSON за допомогою їхніх методів `toJson`; однак ресурси Eloquent забезпечують більш детальний і надійний контроль над JSON-серіалізацією ваших моделей та їхніх відношень.

<a name="generating-resources"></a>

## Створення ресурсів

Щоб створити клас ресурса, ви можете використати команду make:resource Artisan. За замовчуванням ресурси будуть розміщені в каталозі `app/Http/Resources` вашого додатка. Ресурси розширюють клас `Illuminate\Http\Resources\Json\JsonResource`:

```shell
php artisan make:resource UserResource
```

<a name="generating-resource-collections"></a>

#### Колекції ресурсів

На додаток до створення ресурсів, які трансформують окремі моделі, ви можете створити ресурси, які відповідають за трансформацію колекцій моделей. Це дозволяє вашим JSON-відповідям містити посилання та іншу метаінформацію, яка стосується всієї колекції даного ресурса.

Щоб створити колекцію ресурсів, ви повинні використовувати прапорець `--collection` під час створення ресурса. Або включення слова `Collection` в назву ресурса - це вкаже Laravel, що він повинен створити ресурс колекції. Ресурси колекції розширюють клас `Illuminate\Http\Resources\Json\ResourceCollection`:

To create a resource collection, you should use the `--collection` flag when creating the resource. Or, including the word `Collection` in the resource name will indicate to Laravel that it should create a collection resource. Collection resources extend the `Illuminate\Http\Resources\Json\ResourceCollection` class:

```shell
php artisan make:resource User --collection

php artisan make:resource UserCollection
```

<a name="concept-overview"></a>

## Огляд концепції

> **Note**  
> Це загальний огляд ресурсів і колекцій ресурсів. Радимо прочитати інші розділи цієї документації, щоб отримати глибше розуміння налаштувань і можливостей, які пропонують вам ресурси.

Перш ніж заглибитися у всі доступні для вас варіанти під час написання ресурсів, давайте спочатку поглянемо на те, як ресурси використовуються в Laravel. Клас ресурса представляє єдину модель, яку потрібно перетворити на структуру JSON. Наприклад, ось простий клас ресурсів `UserResource`:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Перетворити ресурс на масив.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

Кожен клас ресурсів визначає метод `toArray`, який повертає масив атрибутів, які повинні бути перетворені на JSON,коли ресурс повертається як відповідь з метода маршрута або контролера.

Зверніть увагу, що ми можемо отримати доступ до властивостей моделі безпосередньо із змінної `$this`. Це пов'язано з тим, що клас ресурсів автоматично проксіює властивості та методи до базової моделі для зручності доступу. Як тільки ресурс визначено, його можна повернути з маршрута чи контролера. Ресурс приймає основний екземпляр моделі через свій конструктор:

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function ($id) {
    return new UserResource(User::findOrFail($id));
});
```

<a name="resource-collections"></a>

### Колекції ресурсів

Якщо ви повертаєте колекцію ресурсів або розбиту на сторінки відповідь, вам слід використовувати метод `collection`,наданий вашим класом ресурса, під час створення екземпляра ресурса у вашому маршруті чи контролері:

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all());
});
```

Зауважте, що це не дозволяє додавати спеціальні метадані, які, можливо, потрібно буде повернути разом із вашою колекцією. Якщо ви бажаєте отримати більший контроль над відповіддю колекції ресурсів, ви можете створити спеціальний ресурс для представлення колекції:

```shell
php artisan make:resource UserCollection
```

Після створення класу колекції ресурсів ви можете легко визначити будь-які метадані, які слід включити у відповідь:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * Перетворення колекції ресурсів на масив.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        return [
            'data' => $this->collection,
            'links' => [
                'self' => 'link-value',
            ],
        ];
    }
}
```

Після визначення колекції ресурсів її можна повернути з маршруту або контролера:

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

<a name="preserving-collection-keys"></a>

#### Збереження ключів колекції

Під час повернення колекції ресурсів з маршрута Laravel скидає ключі колекції таким чином, щоб вони були в порядку нумерації. Однак ви можете додати властивість `preserveKeys` до свого класу ресурсів, яка вказує, чи слід зберігати оригінальні ключі колекції:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Вказує, чи потрібно зберегти ключі колекції ресурса.
     *
     * @var bool
     */
    public $preserveKeys = true;
}
```

Якщо для властивості `preserveKeys` встановлено значення `true`, ключі колекції будуть збережені, коли колекція повертається з маршрута або контролера:

```php
    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/users', function () {
        return UserResource::collection(User::all()->keyBy->id);
    });
```

<a name="customizing-the-underlying-resource-class"></a>

#### Налаштування базового класу ресурсів

Як правило, властивість `$this->collection` колекції ресурсів автоматично заповнюється результатом зіставлення кожного елемента колекції з його окремим класом ресурса. Припускається, що єдиний клас ресурса є назвою класу колекції без кінцевої частини імені класа `Collection`. Крім того, залежно від ваших особистих уподобань, клас ресурса в однині може мати суфікса `Resource` або може і не мати його.

Наприклад, `UserCollection` спробує зіставити задані екземпляри користувача з ресурсом `UserResource`. Щоб налаштувати цю поведінку, ви можете змінити властивість `$collects` вашої колекції ресурсів:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * Ресурс, используемый при формировании коллекции.
     *
     * @var string
     */
    public $collects = Member::class;
}
```

<a name="writing-resources"></a>

## Написання ресурсів

> **Note**  
> Якщо ви не прочитали [огляд концепції](#concept-overview), радимо зробити це, перш ніж продовжити роботу з цією документацією.

По суті, ресурси прості. Їм потрібно лише перетворити задану модель у масив. Отже, кожен ресурс містить метод `toArray`, який перетворює атрибути вашої моделі в масив, зручний для API, який можна повернути з маршрутів або контролерів вашого додатка:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Перетворення ресурсу в масив.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

Після визначення ресурса його можна повернути безпосередньо з маршрута або контролера:

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function ($id) {
    return new UserResource(User::findOrFail($id));
});
```

<a name="relationships"></a>

#### Відношення

Якщо ви хочете включити пов’язані ресурси у свою відповідь, ви можете додати їх до масива, який повертає метод `toArray` вашого ресурса. У цьому прикладі ми будемо використовувати метод `collection` ресурса `PostResource`, щоб додати пости користувача з блогу у відповідь ресурса:

```php
use App\Http\Resources\PostResource;

/**
 * Перетворення ресурсу в масив.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function toArray($request)
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'posts' => PostResource::collection($this->posts),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

> **Note**  
> Якщо ви хочете включити відношення лише тоді, коли вони вже завантажені, перегляньте документацію щодо умовних [умовних відношень](#conditional-relationships).

<a name="writing-resource-collections"></a>

#### Колекції ресурсів

Тоді як ресурси перетворюють одну модель на масив, колекції ресурсів перетворюють колекцію моделей на масив. Однак немає абсолютної необхідності визначати клас колекції ресурса для кожної з ваших моделей, оскільки всі ресурси надають метод `collection` для генерування «спеціальної» колекції ресурсів на льоту:

```php
    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/users', function () {
        return UserResource::collection(User::all());
    });
```

Однак, якщо вам потрібно налаштувати метадані, які повертаються з колекцією, необхідно визначити власну колекцію ресурса:

```php
    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\ResourceCollection;

    class UserCollection extends ResourceCollection
    {
        /**
         * Преобразовать коллекцию ресурсу в массив.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return array
         */
        public function toArray($request)
        {
            return [
                'data' => $this->collection,
                'links' => [
                    'self' => 'link-value',
                ],
            ];
        }
    }
```

Подібно до окремих ресурсів, колекції ресурсів можуть повертатися безпосередньо з маршрутів або контролерів:

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

<a name="data-wrapping"></a>

### Пакування даних

За замовчуванням ваш верхній ресурс загортається в ключ `data`, коли відповідь ресурса перетворюється на JSON. Так, наприклад, типова відповідь колекції ресурса виглядає так:

```json
{
  "data": [
    {
      "id": 1,
      "name": "Eladio Schroeder Sr.",
      "email": "therese28@example.com"
    },
    {
      "id": 2,
      "name": "Liliana Mayert",
      "email": "evandervort@example.com"
    }
  ]
}
```

Якщо ви хочете використовувати спеціальний ключ замість `data`, ви можете визначити атрибут `$wrap` у класі ресурса:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Обгортка "data", яку слід застосувати.
     *
     * @var string|null
     */
    public static $wrap = 'user';
}
```

Якщо ви хочете вимкнути огортання крайнього ресурса, вам слід викликати метод `withoutWrapping` у базовому класі `Illuminate\Http\Resources\Json\JsonResource`. Як правило, ви повинні викликати цей метод від вашого `AppServiceProvider` або іншого [постачальника послуг](providers.md), який завантажується під час кожного запита до вашого додатка:

```php
<?php

namespace App\Providers;

use Illuminate\Http\Resources\Json\JsonResource;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Зареєструйте будь-які сервіси додатків.
     *
     * @return void
     */
    public function register()
    {
        //
    }

    /**
     * Завантажте будь-які служби додатків.
     *
     * @return void
     */
    public function boot()
    {
        JsonResource::withoutWrapping();
    }
}
```

> **Warning**  
> Метод `withoutWrapping` впливає лише на верхню відповідь і не видаляє ключі `data`, які ви вручну додаєте до власних колекцій ресурсів.

<a name="wrapping-nested-resources"></a>

#### Огортання вкладених ресурсів

Ви маєте повну свободу визначати, як огортаються відношення вашого ресурса. Якщо ви хочете, щоб всі колекції ресурсів були загорнуті в ключ `data`, незалежно від їх вкладеності, вам слід визначити клас колекції ресурсів для кожного ресурсу та повернути колекцію в ключі `data`.

Можливо, вам цікаво, чи призведе це до того, що ваш зовнішній ресурс буде загорнутий у два ключі `data`. Не хвилюйтеся, Laravel ніколи не дозволить вашим ресурсам випадково подвійно загорнутися, тому вам не потрібно турбуватися про рівень вкладеності колекції ресурсів, яку ви трансформуєте:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class CommentsCollection extends ResourceCollection
{
    /**
     * Перетворити колекцію ресурса на масив.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        return ['data' => $this->collection];
    }
}
```

<a name="data-wrapping-and-pagination"></a>

#### Огортання даних і пагінація

Під час повернення поділених на сторінки колекцій через відповідь ресурса, Laravel загорне дані вашого ресурса в ключ `data`, навіть якщо було викликано метод `withoutWrapping`. Це пояснюється тим, що відповіді з розбиттям на сторінки завжди містять `meta` та `links` ключі з інформацією про стан пагінатора:

```json
{
  "data": [
    {
      "id": 1,
      "name": "Eladio Schroeder Sr.",
      "email": "therese28@example.com"
    },
    {
      "id": 2,
      "name": "Liliana Mayert",
      "email": "evandervort@example.com"
    }
  ],
  "links": {
    "first": "http://example.com/pagination?page=1",
    "last": "http://example.com/pagination?page=1",
    "prev": null,
    "next": null
  },
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 1,
    "path": "http://example.com/pagination",
    "per_page": 15,
    "to": 10,
    "total": 10
  }
}
```

<a name="pagination"></a>

### Пагінація

Ви можете передати екземпляр пагінатора Laravel до метода `collection` ресурса або спеціальної колекції ресурсів:

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::paginate());
});
```

Поділені на сторінки відповіді завжди містять `meta` та `links` ключі з інформацією про стан пагінатора:

```json
{
  "data": [
    {
      "id": 1,
      "name": "Eladio Schroeder Sr.",
      "email": "therese28@example.com"
    },
    {
      "id": 2,
      "name": "Liliana Mayert",
      "email": "evandervort@example.com"
    }
  ],
  "links": {
    "first": "http://example.com/pagination?page=1",
    "last": "http://example.com/pagination?page=1",
    "prev": null,
    "next": null
  },
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 1,
    "path": "http://example.com/pagination",
    "per_page": 15,
    "to": 10,
    "total": 10
  }
}
```

<a name="conditional-attributes"></a>

### Умови атрибутів

Іноді вам може знадобитися включити атрибут у відповідь ресурса, лише якщо задана умова виконується. Наприклад, ви можете включити значення, лише якщо поточний користувач є «адміністратором». Laravel надає різноманітні допоміжні методи, які допоможуть вам у цій ситуації. Метод `when` можна використовувати для умовного додавання атрибута до відповіді ресурса:

```php
/**
 * Перетворення ресурсу в масив.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function toArray($request)
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'secret' => $this->when($request->user()->isAdmin(), 'secret-value'),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

У цьому прикладі `secret` ключ буде повернено лише в остаточній відповіді ресурса, якщо метод `isAdmin` автентифікованого користувача повертає значення `true`. Якщо метод повертає `false`, секретний ключ буде видалено з відповіді ресурса, перш ніж він буде надісланий клієнту. Метод `when` дозволяє чітко визначати ваші ресурси, не вдаючись до умовних операторів під час створення масиву.

Метод `when` також приймає замикання як свій другий аргумент, дозволяючи вам обчислити результуюче значення, лише якщо дана умова `true`:

```php
'secret' => $this->when($request->user()->isAdmin(), function () {
    return 'secret-value';
}),
```

Крім того, метод `whenNotNull` можна використовувати для включення атрибута у відповідь ресурса, якщо атрибут не є null:

```php
'name' => $this->whenNotNull($this->name),
```

<a name="merging-conditional-attributes"></a>

#### Об’єднання атрибутів через умову

Іноді у вас може бути кілька атрибутів, які слід включити у відповідь ресурса лише на основі однієї умови. У цьому випадку ви можете використовувати метод `mergeWhen`, щоб включити атрибути у відповідь лише тоді, коли дана умова `true`:

```php
/**
 * Перетворення ресурсу в масив.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function toArray($request)
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        $this->mergeWhen($request->user()->isAdmin(), [
            'first-secret' => 'value',
            'second-secret' => 'value',
        ]),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

Знову ж таки, якщо дана умова є `false`, ці атрибути будуть видалені з відповіді ресурса, перш ніж вона буде надіслана клієнту.

> **Warning**  
> Метод `mergeWhen` не слід використовувати з масивами, які поєднують рядкові та цифрові ключі. Крім того, його не слід використовувати в масивах із числовими ключами, які не впорядковані послідовно.

<a name="conditional-relationships"></a>

### Умови відношення

Окрім умов завантаження атрибутів, ви можете через умову включити відношення у свої відповіді ресурса на основі того, чи було відношення вже завантажено в модель. Це дозволяє вашому контролеру вирішувати, які відношення слід завантажити в модель, і ваш ресурс може легко включити їх лише тоді, коли вони фактично завантажені. Зрештою, це полегшує уникнення проблем із запитом "N+1" у ваших ресурсах.

Метод `whenLoaded` можна використовувати для умови завантаження відношення. Щоб уникнути непотрібного завантаження відношень, цей метод приймає назву відношення замість самого відношення:

```php
use App\Http\Resources\PostResource;

/**
 * Перетворення ресурсу в масив.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function toArray($request)
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'posts' => PostResource::collection($this->whenLoaded('posts')),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

У цьому прикладі, якщо відношення не було завантажено, ключ дописів буде видалено з відповіді ресурса, перш ніж його буде надіслано клієнту.

<a name="conditional-relationship-counts"></a>

#### Умова підрахунку відношення

На додаток до умови включення відношень, ви можете через умову включити «кількість» відношень у свої відповіді ресурсів на основі того, чи був підрахунок відношень завантажений в модель:

```php
new UserResource($user->loadCount('posts'));
```

Метод `whenCounted` можна використовувати для умови включення підрахунку відншень у відповідь вашого ресурса. Цей метод дозволяє уникнути непотрібного включення атрибута, якщо кількість відношень відсутня:

```php
    /**
     * Transform the resource into an array.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'posts_count' => $this->whenCounted('posts'),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
```

У цьому прикладі, якщо кількість відношень `posts` не завантажено, ключ `posts_count` буде видалено з відповіді ресурса, перш ніж її буде надіслано клієнту.

<a name="conditional-pivot-information"></a>

#### Умовна проміжної інформації

На додаток до умови включення інформації про зв’язок у ваші відповіді ресурса, ви можете через умову включити дані з проміжних таблиць відношень «багато-до-багатьох» за допомогою метода `whenPivotLoaded`. Метод `whenPivotLoaded` приймає назву проміжної таблиці як свій перший аргумент. Другим аргументом має бути замикання, яке повертає значення, яке буде повернуто, якщо в моделі доступна зведена інформація:

```php
/**
 * Перетворення ресурсу в масив.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function toArray($request)
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'expires_at' => $this->whenPivotLoaded('role_user', function () {
            return $this->pivot->expires_at;
        }),
    ];
}
```

Якщо ваше відношення використовує [власну модель проміжної таблиці](eloquent-relationships.md#defining-custom-intermediate-table-models), ви можете передати екземпляр моделі проміжної таблиці як перший аргумент методу `whenPivotLoaded`:

```php
'expires_at' => $this->whenPivotLoaded(new Membership, function () {
    return $this->pivot->expires_at;
}),
```

Якщо ваша проміжна таблиця використовує інструмент доступу, який відрізняється від `pivot`, ви можете використовувати метод `whenPivotLoadedAs`:

```php
/**
 * Перетворення ресурсу в масив.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function toArray($request)
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'expires_at' => $this->whenPivotLoadedAs('subscription', 'role_user', function () {
            return $this->subscription->expires_at;
        }),
    ];
}
```

<a name="adding-meta-data"></a>

### Додавання метаданних

Деякі стандарти JSON API вимагають додавання метаданих до вашого ресурсу та відповідей колекцій ресурсів. Це часто включає такі речі, як "посилання" на ресурс або пов’язані ресурси, або метадані про сам ресурс. Якщо вам потрібно повернути додаткові метадані про ресурс, додайте їх у свій метод `toArray`. Наприклад, ви можете включити інформацію про `link` під час трансформації колекції ресурсів:

```php
/**
 * Перетворення ресурсу в масив.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function toArray($request)
{
    return [
        'data' => $this->collection,
        'links' => [
            'self' => 'link-value',
        ],
    ];
}
```

Повертаючи додаткові метадані з ваших ресурсів, вам ніколи не доведеться турбуватися про випадкове перевизначення посилань або мета-ключів, які автоматично додає Laravel під час повернення поділених на сторінки відповідей. Будь-які додаткові `links`, які ви визначаєте, будуть об’єднані з посиланнями, наданими пагінатором.

<a name="top-level-meta-data"></a>

#### Метадані верхнього рівня

Метадані верхнього рівня

Іноді вам може знадобитися лише включити певні метадані у відповідь ресурсу, якщо ресурс є найверхнім ресурсом, який повертається. Як правило, це містить метаінформацію про відповідь у цілому. Щоб визначити ці метадані, додайте метод `with` до свого класу ресурсів. Цей метод має повертати масив метаданих, які будуть включені до відповіді ресурсу, лише якщо ресурс є найверхнім ресурсом, який перетворюється:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * Перетворити колекцію ресурсу на масив.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        return parent::toArray($request);
    }

    /**
     * Отримайте додаткові дані, які повинні бути повернуті з масивом ресурсів.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function with($request)
    {
        return [
            'meta' => [
                'key' => 'value',
            ],
        ];
    }
}
```

<a name="adding-meta-data-when-constructing-resources"></a>

#### Додавання метаданих під час створення ресурсів

Ви також можете додати дані верхнього рівня під час створення екземплярів ресурсів у вашому маршруті чи контролері. метод `additional`, доступний на всіх ресурсах, приймає масив даних, які слід додати до відповіді ресурсу:

```php
return (new UserCollection(User::all()->load('roles')))
        ->additional(['meta' => [
            'key' => 'value',
        ]]);
```

<a name="resource-responses"></a>

## Відповіді ресурса

Як ви вже зрозуміли, ресурси можуть повертатися безпосередньо з маршрутів і контролерів:

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function ($id) {
    return new UserResource(User::findOrFail($id));
});
```

Однак іноді вам може знадобитися налаштувати вихідну відповідь HTTP, перш ніж її буде надіслано клієнту. Для цього є два способи. По-перше, ви можете прив’язати метод `response` до ресурса. Цей метод поверне екземпляр `Illuminate\Http\JsonResponse`, даючи вам повний контроль над заголовками відповіді:

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user', function () {
    return (new UserResource(User::find(1)))
            ->response()
            ->header('X-Value', 'True');
});
```

Крім того, ви можете визначити метод `withResponse` в самому ресурсі. Цей метод буде викликано, коли ресурс повертається як найверхній ресурс у відповіді:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Перетворення ресурсу в масив.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        return [
            'id' => $this->id,
        ];
    }

    /**
     * Налаштуйте вихідну відповідь для ресурса.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Illuminate\Http\Response  $response
     * @return void
     */
    public function withResponse($request, $response)
    {
        $response->header('X-Value', 'True');
    }
}
```
