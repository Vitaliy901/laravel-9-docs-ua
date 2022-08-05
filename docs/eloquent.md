# Eloquent: Початок

- [Вступ](#introduction)
- [Створення Модельних Класів](#generating-model-classes)
- [Eloquent Домовленості по Іменуванню Моделей](#eloquent-model-conventions)
    - [Іменування Таблиць](#table-names)
    - [Первинні Ключі](#primary-keys)
    - [Мітки Часу](#timestamps)
    - [Підключення до Бази Даних](#database-connections)
    - [Значення Атрибутів за Замовчуванням](#default-attribute-values)
- [Отримання Моделей](#retrieving-models)
    - [Колекції](#collections)
    - [Фрагментовані Результати](#chunking-results)
    - [Фрагментація Результатів із Відкладеними Колекціями](#chunking-using-lazy-collections)
    - [Курсори](#cursors)
    - [Розширені підзапити](#advanced-subqueries)
- [Отримання Окремих Моделей/Агрегатів](#retrieving-single-models)
    - [Отримання або Створення Моделей](#retrieving-or-creating-models)
    - [Отримання агрегатів](#retrieving-aggregates)
- [Вставка та Оновлення Моделей](#inserting-and-updating-models)
    - [Вставка](#inserts)
    - [Оновлення](#updates)
    - [Масова Вставка](#mass-assignment)
    - [Оновлення Вставки](#upserts)
- [Видалення моделей](#deleting-models)
    - [Програмне Видалення](#soft-deleting)
    - [Запит Програмно Видалених Моделей](#querying-soft-deleted-models)
- [Очищення Застарілих Моделей](#pruning-models)
- [Відтворення (Replicating) моделей](#replicating-models)
- [Області Запитів](#query-scopes)
    - [Глобальні Області](#global-scopes)
    - [Локальні Області](#local-scopes)
- [Порівняння моделей](#comparing-models)
- [Події](#events)
    - [Використання замикань](#events-using-closures)
    - [Спостерігачі](#observers)
    - [Вимкнення подій](#muting-events)

<a name="introduction"></a>
## Вступ

Laravel містить Eloquent, об’єктно-реляційне відображення (ORM), який робить взаємодію з вашою базою даних приємною. Під час використання Eloquent кожна таблиця бази даних має відповідну «Модель», яка використовується для взаємодії з цією таблицею. Окрім отримання записів із таблиці бази даних, моделі Eloquent також дозволяють вставляти, оновлювати та видаляти записи з таблиці.

Перш ніж почати, обов’язково налаштуйте з’єднання з базою даних у конфігураційному файлі програми `config/database.php`. Для отримання додаткової інформації про налаштування бази даних перегляньте документацію щодо [налаштування бази даних](database#configuration).

> **Note**  
> Перш ніж почати, обов’язково налаштуйте з’єднання з базою даних у конфігураційному файлі програми `config/database.php`. Для отримання додаткової інформації про налаштування бази даних перегляньте документацію щодо [налаштування бази даних](database#configuration).

<a name="generating-model-classes"></a>
## Створення Модельних Класів

Для початку давайте створимо модель Eloquent. Моделі зазвичай знаходяться в каталозі `app\Models` і розширюють клас `Illuminate\Database\Eloquent\Model`. Ви можете використати команду `make:model` [Artisan](artisan) для створення нової моделі:

```shell
php artisan make:model Flight
```
Якщо ви хочете генерувати [міграцію бази даних](migrations) під час генерації моделі, ви можете використати параметр `--migration` або `-m`:

```shell
php artisan make:model Flight --migration
```
Ви можете генерувати інші типи класів під час генерації моделі, наприклад `factories`, `seeders`, `policies`, `controllers` та `form requests`. Крім того, ці параметри можна комбінувати для створення кількох класів одночасно:


```shell
# Згенеруйте модель разом з FlightFactory класом...
php artisan make:model Flight --factory
php artisan make:model Flight -f

# Згенеруйте модель разом з FlightSeeder класом...
php artisan make:model Flight --seed
php artisan make:model Flight -s

# Згенеруйте модель разом з FlightController класом...
php artisan make:model Flight --controller
php artisan make:model Flight -c

# Згенеруйте модель разом з: FlightController, resource, та form request класами...
php artisan make:model Flight --controller --resource --requests
php artisan make:model Flight -crR

# Згенеруйте модель разом з FlightPolicy класом...
php artisan make:model Flight --policy

# Згенеруйте модель разом з: migration, factory, seeder, та controller...
php artisan make:model Flight -mfsc

# Коротка команда для генерації model, migration, factory, seeder, policy, controller, та form requests...
php artisan make:model Flight --all

# Створити зведену модель...
php artisan make:model Member --pivot
```

<a name="inspecting-models"></a>
#### Огляд моделей

Іноді буває складно визначити всі доступні атрибути та зв’язки моделі, просто проглянувши її код. Натомість спробуйте команду `model:show` Artisan, яка забезпечує зручний огляд усіх атрибутів і зв’язків моделі:

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Домовленості по Іменуванню Моделей

Моделі, згенеровані командою `make:model`, будуть розміщені в каталозі `app/Models`. Давайте розглянемо базовий клас моделі та обговоримо деякі ключові умовності Eloquent:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	//
}
```

<a name="table-names"></a>
### Імена таблиць

Переглянувши наведений вище приклад, ви могли помітити, що ми не повідомили Eloquent, яка таблиця бази даних відповідає нашій моделі `Flight`. Згідно з угодою, ім’я класу у «зміїному_реєстрі», та множині буде використано як ім’я таблиці, якщо інше ім’я не вказано явно. Отже, у цьому випадку Eloquent припускатиме, що модель `Flight` зберігає записи в таблиці `flights`, тоді як модель `AirTrafficController` зберігатиме записи в таблиці `air_traffic_controllers`.

Якщо відповідна таблиця бази даних вашої моделі не відповідає цій угоді, ви можете вручну вказати назву таблиці моделі, визначивши властивість `table` в моделі:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	/**
	 * Таблиця, асоціюється з моделлю.
	 *
	 * @var string
	 */
	protected $table = 'my_flights';
}
```

<a name="primary-keys"></a>
### Первинні ключі

Eloquent також припускатиме, що відповідна таблиця бази даних кожної моделі має стовпець первинного ключа під назвою `id`. За потреби ви можете визначити захищену властивість `$primaryKey` у вашій моделі, щоб вказати інше ім'я стовпця, який буде первинним ключем вашої моделі:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	/**
	 * Первинний ключ, пов'язаний з таблицею.
	 *
	 * @var string
	 */
	protected $primaryKey = 'flight_id';
}
```
Крім того, Eloquent передбачає, що первинний ключ є похитивним цілим значенням, це означатиме, що Eloquent автоматично перетворює первинний ключ на ціле число. Якщо ви бажаєте використовувати неінкрементний або нечисловий первинний ключ, ви повинні визначити загальнодоступну властивість `$incrementing` у вашій моделі, яка повина мати значення `false`:

```php
<?php

class Flight extends Model
{
	/**
	 * Вказує, чи ID моделі автоматично збільшується.
	 *
	 * @var bool
	 */
	public $incrementing = false;
}
```
Якщо первинний ключ вашої моделі не є цілим числом, вам слід визначити захищену властивість `$keyType` у вашій моделі. Ця властивість повинна мати значення `string`:


```php
<?php

class Flight extends Model
{
	/**
	 * Тип даних ID, що автоматично збільшується.
	 *
	 * @var string
	 */
	protected $keyType = 'string';
}
```

<a name="composite-primary-keys"></a>
#### «Складові» первинні ключі

Eloquent вимагає, щоб кожна модель мала принаймні один унікальний "ID", який може служити її первинним ключем. «Складові» первинні ключі не підтримуються моделями Eloquent. Однак ви можете додавати додаткові унікальні індекси з кількома стовпцями до таблиць бази даних на додаток до унікального первинного ключа таблиці.

<a name="timestamps"></a>
### Мітки часу

За замовчуванням Eloquent очікує наявності стовпців `created_at` і `updated_at` у відповідній таблиці бази даних вашої моделі. Eloquent автоматично встановлюватиме значення цих стовпців під час створення або оновлення моделей. Якщо ви не хочете, щоб ці стовпці автоматично керувалися Eloquent, вам слід визначити властивість `$timestamps` у вашій моделі зі значенням `false`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	/**
	 * Вказує, чи повинна модель мати мітку часу.
	 *
	 * @var bool
	 */
	public $timestamps = false;
}
```
Якщо вам потрібно налаштувати формат позначок часу вашої моделі, встановіть у своїй моделі властивість `$dateFormat`. Ця властивість визначає, як атрибути дати зберігаються в базі даних, а також їх формат, коли модель серіалізовано в масив або JSON:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	/**
	 * Формат зберігання стовпців дати моделі.
	 *
	 * @var string
	 */
	protected $dateFormat = 'U';
}
```
Якщо вам потрібно налаштувати назви стовпців, які використовуються для зберігання позначок часу, ви можете вказати інше значення для констант `CREATED_AT` і `UPDATED_AT` у своїй моделі:

```php
<?php

class Flight extends Model
{
	const CREATED_AT = 'creation_date';
	const UPDATED_AT = 'updated_date';
}
```

<a name="database-connections"></a>
### Підключення до бази даних

За замовчуванням усі моделі Eloquent використовуватимуть стандартне підключення до бази даних, налаштоване для вашої програми. Якщо ви хочете вказати інше з’єднання, яке слід використовувати під час взаємодії з певною моделлю, вам слід визначити властивість `$connection` у моделі:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	/**
	 * Підключення до бази даних, яке має використовувати модель.
	 *
	 * @var string
	 */
	protected $connection = 'sqlite';
}
```

<a name="default-attribute-values"></a>
### Значення атрибутів за замовчуванням

За замовчуванням щойно створений екземпляр моделі не міститиме жодних значень атрибутів. Якщо ви хочете визначити значення за замовчуванням для деяких атрибутів вашої моделі, ви можете визначити властивість `$attributes` у своїй моделі:

By default, a newly instantiated model instance will not contain any attribute values. If you would like to define the default values for some of your model's attributes, you may define an `$attributes` property on your model:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	/**
	 * Стандартні значення для атрибутів моделі.
	 *
	 * @var array
	 */
	protected $attributes = [
		'delayed' => false,
	];
}
```

<a name="retrieving-models"></a>
## Отримання моделей

Після того, як ви створили модель і [пов’язану з нею таблицю бази даних](migrations#writing-migrations), ви готові почати отримувати дані з вашої бази даних. Ви можете розглядати кожну модель Eloquent як потужний [конструктор запитів](queries), який дозволяє вільно надсилати запити до таблиці бази даних, пов’язаної з моделлю. Метод `all` моделі отримає всі записи з пов’язаної з моделлю таблиці бази даних:

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
	echo $flight->name;
}
```

<a name="building-queries"></a>
#### Побудова запитів

Метод `all` Eloquent поверне всі результати в таблиці моделі. Однак, оскільки кожна модель Eloquent служить як [конструктор запитів](queries), ви можете додати додаткові обмеження до запитів, а потім викликати метод `get` для отримання результатів:

```php
$flights = Flight::where('active', 1)
				->orderBy('name')
				->take(10)
				->get();
```
> **Note**  
> Оскільки моделі Eloquent є конструкторами запитів, вам слід переглянути всі методи, надані [конструктором запитів](queries) Laravel. Ви можете використовувати будь-який із цих методів під час написання запитів Eloquent.

<a name="refreshing-models"></a>
#### Оновлення моделей

Якщо у вас уже є екземпляр моделі Eloquent, який було отримано з бази даних, ви можете «оновити» модель за допомогою методів `fresh` і `refresh`. Метод `fresh` повторно отримає модель із бази даних. На існуючий екземпляр моделі це не вплине:

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh(): object;
```
Метод `refresh` повторно оновить існуючу модель, використовуючи нові дані з бази даних. Крім того, усі його завантажені зв’язки також будуть оновлені:

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```

<a name="collections"></a>
### Колекції

Як ми бачили, методи Eloquent такі як `all` і `get` дозволяють отримати кілька записів із бази даних. Однак ці методи не повертають звичайний масив PHP. Натомість повертається екземпляр `Illuminate\Database\Eloquent\Collection`.

Клас Eloquent `Collection` розширює базовий клас Laravel `Illuminate\Support\Collection`, який містить [безліч корисних методів](collections#available-methods) для взаємодії з даними колекцій. Наприклад, метод `reject` використовується для видалення моделей із колекції на основі результатів функції зворотного виклику:

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function ($flight) {
    return $flight->cancelled;
});
```
На додаток до методів, наданих базовим класом колекції Laravel, клас колекції Eloquent надає [кілька додаткових методів](eloquent-collections#available-methods), які спеціально призначені для взаємодії з колекціями моделей Eloquent.

Оскільки всі колекції Laravel реалізують ітераційні інтерфейси PHP, ви можете перебирати колекції так, ніби вони були масивом:

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```

<a name="chunking-results"></a>
### Фрагментовані Результати

У вашому додатку може вичерпатися пам’ять, якщо ви спробуєте завантажити десятки тисяч записів Eloquent за допомогою методів `all` або `get`. Замість використання цих методів можна використати метод `chunk` для більш ефективної обробки великої кількості моделей.

Метод `chunk` буде отримувати підмножину моделей Eloquent, передаючи їх у замикання для обробки. Оскільки за один раз витягується лише поточна частина моделей Eloquent, метод `chunk` значно зменшить використання пам’яті під час роботи з великою кількістю моделей:

```php
use App\Models\Flight;

Flight::chunk(200, function ($flights) {
    foreach ($flights as $flight) {
        //
    }
});
```
Перший аргумент, який передається до методу `chunk`, — це кількість записів, які ви бажаєте отримати на «фрагмент». Замикання, передане другим аргументом, буде викликано для кожного блоку, отриманого з бази даних. Буде виконано запит до бази даних, щоб отримати кожну частину записів, переданих до замикання.

Якщо ви фільтруєте результати методу `chunk` на основі стовпця, який ви також будете оновлювати під час ітерації результатів, вам слід використовувати метод `chunkById`. Використання методу фрагментів у цих сценаріях може призвести до несподіваних і суперечливих результатів. Внутрішньо метод `chunkById` завжди отримуватиме моделі зі стовпцем `id`, більшим за останню модель у попередньому фрагменті:

```php
Flight::where('departed', true)
    ->chunkById(200, function ($flights) {
        $flights->each->update(['departed' => false]);
    }, $column = 'id');
```

<a name="chunking-using-lazy-collections"></a>
### Фрагментація результатів із відкладеними колекціями

Метод `lazy` працює подібно до [методу `chunk` ](#chunking-results) у тому сенсі, що він виконує запит частинами. Однак замість того, щоб передавати кожний фрагмент прямо в зворотний виклик як є, метод `lazy` повертає екземпляр [`LazyCollection`](collections#lazy-collections) моделей Eloquent, що дозволяє вам взаємодіяти з результатами як єдиним потоком:

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    //
}
```
Якщо ви фільтруєте результати методу `lazy` на основі стовпця, який також оновлюватимете під час ітерації результатів, вам слід використовувати метод `lazyById`. Внутрішньо метод `lazyById` завжди отримуватиме моделі зі стовпцем `id`, більшим за останню модель у попередньому фрагменті:

```php
Flight::where('departed', true)
    ->lazyById(200, $column = 'id')
    ->each->update(['departed' => false]);
```
Ви можете фільтрувати результати за спаданням `id` за допомогою методу `lazyByIdDesc`.

<a name="cursors"></a>
### Курсори

Подібно до `lazy` методу, метод `cursor` можна використовувати для значного зменшення споживання пам’яті вашим додатком під час ітерації десятків тисяч записів моделі Eloquent.

Метод `cursor` виконає лише один запит бази даних; однак окремі моделі Eloquent не будуть додані до результативного набору, доки їх фактично не перевірять. Таким чином, лише одна модель Eloquent зберігається в пам’яті в будь-який момент часу під час ітерації з використанням `cursor`.

> **Warning**  
> Оскільки метод `cursor` завжди зберігає у пам'яті лише одну модель Eloquent, то нетерпляче завантаження стосунків неприпустиме. Якщо вам потрібно нетерпляче завантажити відносини, розгляньте можливість використання методу [методу `lazy`](#chunking-using-lazy-collections).

Всередині метод `cursor` використовує [генератори](https://www.php.net/manual/en/language.generators.overview.php) PHP для реалізації цієї функції:


```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    //
}
```
`cursor` повертає екземпляр `Illuminate\Support\LazyCollection`. [Відкладені колекції](collections#lazy-collections)дозволяють використовувати багато методів колекцій, доступних у типових колекціях Laravel, одночасно завантажуючи в пам’ять лише одну модель:

```php
use App\Models\User;

$users = User::cursor()->filter(function ($user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```
Незважаючи на те, що метод `cursor` використовує набагато менше пам’яті, ніж звичайний запит (утримуючи в пам’яті лише одну модель Eloquent за один раз), все одно в кінцевому реультаті може вичерпати пам’ять. Це відбувається через те, що [драйвер PHP PDO внутрішньо кешує всі необроблені результати запиту у своєму буфері](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php). Якщо ви маєте справу з дуже великою кількістю записів Eloquent, подумайте про використання [методу `lazy`](#chunking-using-lazy-collections).

<a name="advanced-subqueries"></a>
### Розширені підзапити

<a name="subquery-selects"></a>
#### Відбір

Eloquent також пропонує розширену підтримку підзапитів, яка дозволяє отримувати інформацію з пов’язаних таблиць в одному запиті. Наприклад, уявімо, що у нас є таблиця `destinations` (пункти призначення) і таблиця `flights` (рейсів). У таблиці `flights` міститься стовпець `arrived_at`, який вказує, коли рейс прибув у пункт призначення.

Використовуючи функціональність підзапиту, доступного для методів `select` і `addSelect` конструктора запитів, ми можемо вибрати всі `destinations` і додатково назву рейсу, який останнім прибув до цього пункту призначення, використовуючи один запит:

```php
use App\Models\Destination;
use App\Models\Flight;

return Destination::addSelect(['last_flight' => Flight::select('name')
	->whereColumn('destination_id', 'destinations.id')
	->orderByDesc('arrived_at')
	->limit(1)
])->get();
```

У наведеному вище прикладі ключ `last_flight` у запиті SQL буде як еліас колонки підзапита, який буде додано до моделі `destinations`.

<a name="subquery-ordering"></a>
#### Сортування

Крім того, функція `orderBy` конструктора запитів підтримує підзапити. Продовжуючи використовувати наш приклад рейсу, ми можемо використовувати цю функцію для сортування всіх пунктів призначення на основі того, коли останній рейс прибув у цей пункт призначення. Знову ж таки, це можна зробити під час виконання одного запиту до бази даних:

```php
return Destination::orderByDesc(
	Flight::select('arrived_at')
		->whereColumn('destination_id', 'destinations.id')
		->orderByDesc('arrived_at')
		->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## Отримання Окремих Моделей/Агрегатів

На додаток до отримання всіх записів, які відповідають зазначеному запиту, ви також можете отримати окремі записи, використовуючи методи `find`, `first`, або `firstWhere`. Замість того, щоб повертати колекцію моделей, ці методи повертають єдиний екземпляр моделі:

```php
use App\Models\Flight;

// Отримати модель за її первинним ключем.
$flight = Flight::find(1);

// Отримати першу модель, що відповідає умовам запиту.
$flight = Flight::where('active', 1)->first();

// Альтернатива отриманню першої моделі, що відповідає обмеженням запиту...
$flight = Flight::firstWhere('active', 1);
```
Іноді вам може знадобитися виконати якусь іншу дію, якщо результатів не знайдено. Методи `findOr` і `firstOr` повертають один екземпляр моделі або, якщо результатів не знайдено, виконають задане замикання. Значення, яке повертає замикання, вважатиметься результатом методу:

```php
$flight = Flight::findOr(1, function () {
	// ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
	// ...
});
```

<a name="not-found-exceptions"></a>
#### Винятки за відсутності результатів запиту

Іноді ви можете створити виняток, якщо модель не знайдено. Це особливо корисно для маршрутів або контролерів. Методи `findOrFail` і `firstOrFail` отримають перший результат запиту; однак, якщо результат не знайдено, буде створено виняток `Illuminate\Database\Eloquent\ModelNotFoundException`:

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```
Якщо `ModelNotFoundException` не перехоплено, клієнту автоматично надсилається відповідь HTTP 404:

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function ($id) {
	return Flight::findOrFail($id);
});
```

<a name="retrieving-or-creating-models"></a>
### Отримання або створення моделей

Метод `firstOrCreate` намагатиметься знайти запис бази даних за допомогою заданих пар стовпець/значення. Якщо модель неможливо знайти в базі даних, буде вставлено запис із атрибутами, отриманими в результаті об’єднання першого аргументу масиву з необов’язковим аргументом другого масиву:

Метод `firstOrNew`, як і `firstOrCreate`, намагатиметься знайти запис у базі даних, що відповідає заданим атрибутам. Однак якщо модель не знайдено, буде повернено новий екземпляр моделі. Зауважте, що модель, яку повертає `firstOrNew`, ще не збережено в базі даних. Вам потрібно буде вручну викликати метод `save`, щоб зберегти його:

```php
use App\Models\Flight;

// Отримати рейс за назвою або створити його, якщо він не існує...
$flight = Flight::firstOrCreate([
	'name' => 'London to Paris'
]);

// Отримати рейс за назвою або створити його з атрибутами name, delayed і arrival_time...
$flight = Flight::firstOrCreate(
	['name' => 'London to Paris'],
	['delayed' => 1, 'arrival_time' => '11:30']
);

// Отримати рейс за назвою або створити новий екземпляр Flight...
$flight = Flight::firstOrNew([
	'name' => 'London to Paris'
]);

// Отримання рейсу за назвою або створення екземпляра з атрибутами name, delayed і arrival_time...
$flight = Flight::firstOrNew(
	['name' => 'Tokyo to Sydney'],
	['delayed' => 1, 'arrival_time' => '11:30']
);
```

<a name="retrieving-aggregates"></a>
### Отримання агрегатів

Під час взаємодії з моделями Eloquent ви також можете використовувати `count`, `sum`, `max` та інші [агрегатні методи](queries#aggregates), надані [конструктором запитів](queries) Laravel. Як і слід було очікувати, ці методи повертають скалярне значення замість екземпляра моделі Eloquent:

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## Вставка та Оновлення Моделей

<a name="inserts"></a>
### Вставка

Звичайно, використовуючи Eloquent, нам потрібно не тільки отримувати моделі з бази даних. Нам також потрібно вставити нові записи. На щастя, Eloquent робить це просто. Щоб вставити новий запис у базу даних, вам слід створити новий екземпляр моделі та встановити атрибути для моделі. Потім викличте метод `save` в екземплярі моделі:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Flight;
use Illuminate\Http\Request;

class FlightController extends Controller
{
	/**
	 * Збережіть новий рейс у базі даних.
	 *
	 * @param  \Illuminate\Http\Request  $request
	 * @return \Illuminate\Http\Response
	 */
	public function store(Request $request)
	{
		// Валідація запиту ...

		$flight = new Flight;

		$flight->name = $request->name;

		$flight->save();
	}
}
```
У цьому прикладі ми призначаємо поле `name` з вхідного HTTP-запиту атрибуту `name` екземпляра моделі `App\Models\Flight`. Коли ми викликаємо метод `save`, запис буде вставлено в базу даних. Часові мітки моделі `created_at` і `updated_at` будуть автоматично встановлені під час виклику методу `save`, тому немає необхідності встановлювати їх вручну.

Крім того, ви можете використати метод `create`, щоб «зберегти» нову модель за допомогою одного оператора PHP. Вставлений екземпляр моделі буде повернено вам методом `create`:

```php
use App\Models\Flight;

$flight = Flight::create([
	'name' => 'London to Paris',
]);
```
Однак перед використанням методу `create` вам потрібно буде вказати властивість `fillable` або `guarded` в класі моделі. Ці властивості є обов’язковими, оскільки всі моделі Eloquent за замовчуванням захищені від уразливостей масового призначення. Щоб дізнатися більше про масове призначення, зверніться до [документації щодо масового призначення](#mass-assignment).

<a name="updates"></a>
### Оновлення

Метод `save` також можна використовувати для оновлення моделей, які вже існують у базі даних. Щоб оновити модель, ви повинні отримати її та встановити всі атрибути, які ви хочете оновити. Потім слід викликати метод `save` моделі, цей метод оновить модель разом з даними в базі даних. Знову ж таки, мітка часу `updated_at` буде автоматично оновлена, тому немає необхідності вручну встановлювати її значення:

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

<a name="mass-updates"></a>
#### Масові оновлення

Оновлення також можна виконувати для моделей, які відповідають певному запиту. У цьому прикладі всі рейси, які `active` і мають пункт `destination` в `San Diego`, будуть позначені як затримані:

```php
Flight::where('active', 1)
		->where('destination', 'San Diego')
		->update(['delayed' => 1]);
```
Метод `update` очікує масив пар стовпців і значень, що представляють стовпці, які потрібно оновити. Метод `update` повертає кількість оновлених рядків.

> **Warning**  
> Під час масового оновлення через Eloquent події `saving`, `saved`, `updating`, та `updated` моделі не запускатимуться для оновлених моделей. Це пов’язано з тим, що моделі ніколи не завантажуються під час масового оновлення.

<a name="examining-attribute-changes"></a>
#### Вивчення змін атрибутів

Eloquent містить методи `isDirty`, `isClean`, та `wasChanged` для перевірки внутрішнього стану моделі та визначення того, як змінилися її атрибути з моменту початкового вилучення моделі.

Метод `isDirty` визначає, чи були змінені будь-які атрибути моделі з моменту отримання моделі. Ви можете передати конкретне ім'я атрибута або масив атрибутів методу `isDirty`, щоб визначити, чи є атрибут «брудним». Метод `isClean` визначає, чи залишився атрибут незмінним з моменту отримання моделі. Цей метод також приймає необов'язковий аргумент атрибуту:

```php
use App\Models\User;

$user = User::create([
	'first_name' => 'Taylor',
	'last_name' => 'Otwell',
	'title' => 'Developer',
]);

$user->title = 'Painter';

$user->isDirty(); // true
$user->isDirty('title'); // true
$user->isDirty('first_name'); // false
$user->isDirty(['first_name', 'title']); // true

$user->isClean(); // false
$user->isClean('title'); // false
$user->isClean('first_name'); // true
$user->isClean(['first_name', 'title']); // false

$user->save();

$user->isDirty(); // false
$user->isClean(); // true
```
Метод `wasChanged` визначає, чи були змінені якісь атрибути під час останнього збереження моделі в поточному циклі запиту. За потреби ви можете передати назву атрибута, щоб побачити, чи був змінений певний атрибут:

```php
$user = User::create([
	'first_name' => 'Taylor',
	'last_name' => 'Otwell',
	'title' => 'Developer',
]);

$user->title = 'Painter';

$user->save();

$user->wasChanged(); // true
$user->wasChanged('title'); // true
$user->wasChanged(['title', 'slug']); // true
$user->wasChanged('first_name'); // false
$user->wasChanged(['first_name', 'title']); // true
```
Метод `getOriginal` повертає масив, що містить вихідні атрибути моделі незалежно від будь-яких змін у моделі з моменту її отримання. Якщо необхідно, ви можете передати конкретне ім’я атрибута, щоб отримати вихідне значення певного атрибута:

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = "Jack";
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Масив оригінальних атрибутів...
```

<a name="mass-assignment"></a>
### Масова вставка

Ви можете використовувати метод `create`, щоб «зберегти» нову модель за допомогою одного оператора PHP. Вставлений екземпляр моделі буде повернено вам методом:

```php
use App\Models\Flight;

$flight = Flight::create([
	'name' => 'London to Paris',
]);
```
Однак перед використанням методу `create` вам потрібно буде вказати властивість `fillable` або `guarded` в класі моделі. Ці властивості є обов’язковими, оскільки всі моделі Eloquent за замовчуванням захищені від вразливостей масового призначення.

Вразливість масового призначення виникає, коли користувач передає неочікуване поле запиту HTTP, і це поле змінює стовпець у вашій базі даних, якого ви не очікували. Наприклад, зловмисник може надіслати параметр is_admin через HTTP-запит, який потім передається в метод `create` вашої моделі, дозволяючи користувачеві передати себе адміністратору.

Отже, щоб розпочати, ви повинні визначити, які атрибути моделі ви хочете зробити масовими для призначення. Ви можете зробити це за допомогою властивості `$fillable` у моделі. Наприклад, давайте зробимо атрибут `name` нашої моделі `Flight` доступним для призначення:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
	/**
	 * Атрибути масового призначення.
	 *
	 * @var array
	 */
	protected $fillable = ['name'];
}
```
Після того, як ви вказали, які атрибути можна призначати масово, ви можете використовувати метод `create`, щоб вставити новий запис у базу даних. Метод `create` повертає щойно створений екземпляр моделі:

```php
$flight = Flight::create(['name' => 'London to Paris']);
```
Якщо у вас вже є екземпляр моделі, ви можете використовувати метод `fill`, щоб заповнити його масивом атрибутів:

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```
<a name="mass-assignment-json-columns"></a>
#### Масове присвоєння та JSON-стовпці

При призначенні JSON-стовпців необхідно вказати масово призначений ключ для кожного стовпця в масиві `$fillable` моделі.
Для безпеки Laravel не підтримує оновлення вкладених атрибутів JSON при використанні властивості `guarded`:

```php
/**
 * Атрибути масового призначення.
 *
 * @var array
 */
protected $fillable = [
	'options->enabled',
];
```

<a name="allowing-mass-assignment"></a>
#### Захист масового присвоєння

Якщо ви хочете, щоб всі ваші атрибути мали масове признаачення, ви можете визначити властивість моделі `$guarded` як порожній масив. Якщо ви вирішите не захищати свою модель, вам слід подбати про те, щоб завжди вручну оброблялися масиви, передані в методи Eloquent `fill`, `create`, і `update`:

```php
/**
 * Атрибути, яким заборонено масове присвоєння значень.
 *
 * @var array
 */
protected $guarded = [];
```

<a name="upserts"></a>
### Оновлення вставки

Іноді вам може знадобитися оновити існуючу модель або створити нову модель, якщо відповідної моделі не існує. Як і метод `firstOrCreate`, метод `updateOrCreate` зберігає модель, тому немає необхідності вручну викликати метод `save`.


У наведеному нижче прикладі, якщо існує рейс із місцем відправлення в `Oakland` та місцем призначення в `San Diego`, його стовпці з `price` та `discounted` буде оновлено. Якщо такий рейс не існує, буде створено новий рейс з атрибутами, отриманими в результаті об’єднання першого масиву аргументів із другим масивом аргументів:

```php
$flight = Flight::updateOrCreate(
	['departure' => 'Oakland', 'destination' => 'San Diego'], // if exist
	['price' => 99, 'discounted' => 1] // updating
);
```
Якщо ви бажаєте виконати декілька «оновлених вставок» в одному запиті, тоді вам слід використовувати метод `upsert`. Перший аргумент методу складається зі значень, які потрібно вставити або оновити, тоді як другий аргумент містить список стовпців, які однозначно ідентифікують записи у пов’язаній таблиці. Третій і останній аргумент методу — це масив стовпців, які слід оновити, якщо відповідний запис уже існує в базі даних. Метод `upsert` автоматично встановить часові позначки `created_at` і `updated_at`, якщо часові позначки ввімкнено в моделі:

```php
Flight::upsert([
	['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
	['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], ['departure', 'destination'], ['price']);
```
> **Warning**  
> Усі бази даних, крім SQL Server, вимагають, щоб стовпці у другому аргументі методу `upsert` мали "primary" або "unique" індекс. Крім того, драйвер бази даних MySQL ігнорує другий аргумент методу `upsert` і завжди використовує «первинний» і "primary" індекси таблиці для виявлення існуючих записів.

<a name="deleting-models"></a>
## Видалення моделей

Щоб видалити модель, ви можете викликати метод `delete` екземпляра моделі:

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```
Ви можете викликати метод `truncate`, щоб видалити всі записи з таблиці до якої відноситься модель бази даних. Операція `truncate` також скине всі автоінкриментні ідентифікатори у пов’язаній з моделлю таблиці:

```php
Flight::truncate();
```

<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### Видалення існуючої моделі за її первинним ключем

У наведеному вище прикладі ми отримуємо модель із бази даних перед викликом методу `delete`. Однак, якщо ви знаєте первинний ключ моделі, ви можете видалити модель без її явного отримання, викликавши метод `destroy`. На додаток до прийняття єдиного первинного ключа, метод `destroy` прийматиме кілька первинних ключів, масив первинних ключів або [колекцію](collections) первинних ключів:

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```
> **Warning**  
> Метод `destroy` завантажує кожну модель окремо і викликає для них метод `delete`, щоб спрацювали події `deleting` та `deleted` належним чином для кожної моделі.

<a name="deleting-models-using-queries"></a>
#### Видалення моделей за допомогою запитів

Звичайно, ви можете створити запит Eloquent, щоб видалити всі моделі, які відповідають критеріям вашого запиту. У цьому прикладі ми видалимо всі рейси, позначені як неактивні. Подібно до масових оновлень, масові видалення не надсилатимуть події моделі для моделей, які видаляються:

```php
$deleted = Flight::where('active', 0)->delete();
```
> **Warning**  
> Під час виконання оператора масового видалення через Eloquent події `deleting` та `deleted` моделі не надсилатимуться для видалених моделей. Це пояснюється тим, що моделі ніколи фактично не витягуються під час виконання оператора delete.

<a name="soft-deleting"></a>
### Програмне видалення

Окрім фактичного видалення записів із вашої бази даних, Eloquent також може «програмно видаляти» моделі. Коли моделі програмно видаляються, вони фактично не видаляються з вашої бази даних. Замість цього для моделі встановлюється атрибут `deleted_at`, який вказує дату й час, коли модель було "видалено". Щоб увімкнути програмне видалення для моделі, додайте до моделі трейт `Illuminate\Database\Eloquent\SoftDeletes`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Flight extends Model
{
	use SoftDeletes;
}
```

> **Note**  
> Трейт `SoftDeletes` автоматично типізує атрибут `deleted_at` до екземпляра `DateTime` / `Carbon`.

Ви також повинні обов'язково додати стовпець `deleted_at` до таблиці бази даних. [Конструктор схем](migrations) Laravel містить допоміжний метод для створення цього стовпця:

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('flights', function (Blueprint $table) {
	$table->softDeletes();
});

Schema::table('flights', function (Blueprint $table) {
	$table->dropSoftDeletes();
});
```
Тепер, коли ви викликаєте метод `delete` у моделі, стовпець `deleted_at` буде встановлено на поточну дату й час. Однак запис бази даних моделі залишиться в таблиці. Під час запиту моделі, яка використовує програмне видалення, програмно видалені моделі будуть автоматично виключені з усіх результатів запиту.

Щоб визначити, чи був даний екземпляр моделі програмно видалений, ви можете використати метод `trashed`:

```php
if ($flight->trashed()) {
	//
}
```

<a name="restoring-soft-deleted-models"></a>
#### Відновлення програмно видалених моделей

Іноді вам може знадобитися «скасувати видалення» програмно видаленої моделі. Щоб відновити програмно видалену модель, ви можете викликати метод `restore` в екземплярі моделі. Метод `restore` встановить стовпець `deleted_at` моделі на `null`:

```php
$flight->restore();
```
Ви також можете використовувати метод `restore` в запиті для відновлення кількох моделей. Знову ж таки, як і інші «масові» операції, це не буде відправляти жодних подій моделі для моделей, які відновлюються:

```php
Flight::withTrashed()
		->where('airline_id', 1)
		->restore();
```

Метод `restore` також використовується при побудові запитів, що використовують [відношення](eloquent-relationships):

```php
    $flight->history()->restore();
```

<a name="permanently-deleting-models"></a>
#### Видалення моделей назавжди

Іноді вам може знадобитися справді видалити модель із вашої бази даних. Ви можете використовувати метод `forceDelete`, щоб назавжди видалити програмно видалену модель із таблиці бази даних:

```php
    $flight->forceDelete();
```

Ви також можете використовувати метод `forceDelete` під час створення запитів, які використовують відношення Eloquent:

```php
    $flight->history()->forceDelete();
```

<a name="querying-soft-deleted-models"></a>
### Запит програмно видалених моделей

<a name="including-soft-deleted-models"></a>
#### Включення програмно видалених моделей

Як зазначено вище, програмно видалені моделі будуть автоматично виключені з результатів запиту. Однак ви можете змусити програмно видалені моделі включити в результати запиту, викликавши метод `withTrashed` у запиті:

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
			->where('account_id', 1)
			->get();
```
Метод `withTrashed` також може бути викликаний під час створення запиту [відношень](eloquent-relationships):

```php
    $flight->history()->withTrashed()->get();
```

<a name="retrieving-only-soft-deleted-models"></a>
#### Вилучення лише програмно видалених моделей

Метод `onlyTrashed` буде вилучати **тільки** програмно видалені моделі:

```php
$flights = Flight::onlyTrashed()
			->where('airline_id', 1)
			->get();
```

<a name="pruning-models"></a>
## Очищення Застарілих Моделей

Іноді вам може знадобитися періодично видаляти моделі, які більше не потрібні. Щоб досягти цього, ви можете додати трейт `Illuminate\Database\Eloquent\Prunable` або `Illuminate\Database\Eloquent\MassPrunable` до моделей, які ви хочете періодично очищувати. Після додавання однієї з властивостей до моделі реалізуйте метод `prunable`, який повертає конструктор запитів Eloquent, який розпізнає моделі, які більше не потрібні:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Prunable;

class Flight extends Model
{
	use Prunable;

	/**
	 * Отримати запит моделі, яка очищується.
	 *
	 * @return \Illuminate\Database\Eloquent\Builder
	 */
	public function prunable()
	{
		return static::where('created_at', '<=', now()->subMonth());
	}
}
```
Позначаючи моделі як такі, що підлягають `Prunable`, ви також можете визначити метод `pruning` на моделі. Цей метод буде викликано перед видаленням моделі. Цей метод може бути корисним для видалення будь-яких додаткових ресурсів, пов’язаних із моделлю, таких як збережені файли, до того, як модель буде остаточно видалено з бази даних:

```php
/**
 * Підготувати модель для очищення
 *
 * @return void
 */
protected function pruning()
{
	//
}
```
Після налаштування моделі, яку можна очистити, вам слід запланувати команду `model:prune` Artisan у класі `App\Console\Kernel` вашого додатку. Ви можете обрати відповідний інтервал, через який слід виконувати цю команду:


```php
/**
 * Визначити розклад виконання команд додатку.
 *
 * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
 * @return void
 */
protected function schedule(Schedule $schedule)
{
	$schedule->command('model:prune')->daily();
}
```
За лаштунками команда `model:prune` автоматично визначить моделі, які можна "скоротити", у каталозі `app/Models` вашого додатку. Якщо ваші моделі знаходяться в іншому місці, ви можете використовувати параметр `--model`, щоб вказати імена класів моделей:

```php
$schedule->command('model:prune', [
	'--model' => [Address::class, Flight::class],
])->daily();
```
Якщо ви бажаєте виключити певні моделі з очищення під час очищення всіх інших виявлених моделей, ви можете використати опцію `--except`:

```php
$schedule->command('model:prune', [
	'--except' => [Address::class, Flight::class],
])->daily();
```
Ви можете імітувати свій запит `prunable`, виконавши команду `model:prune` із параметром `--pretend`. Під час імітації команда `model:prune` просто повідомить, скільки записів буде вирізано, якби команда справді була виконана:

```shell
php artisan model:prune --pretend
```
> **Warning**  
> Програмно видалені моделі будуть видалені (`forceDelete`) без можливості відновлення, якщо вони відповідають запиту очищення.

<a name="mass-pruning"></a>
#### Масове очищення застарілих моделей

Коли моделі позначені отрейтом `Illuminate\Database\Eloquent\MassPrunable`, моделі видаляються з бази даних за допомогою запитів масового видалення. Таким чином, метод `pruning` не буде викликано, а також не будуть відправлені події моделі `deleting` та `deleted`. Це пов’язано з тим, що моделі ніколи не відновлюються перед видаленням, що робить процес скорочення набагато ефективнішим:

```php
    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\MassPrunable;

    class Flight extends Model
    {
        use MassPrunable;

        /** 
         * Отримати оброблений запит моделі
         *
         * @return \Illuminate\Database\Eloquent\Builder
         */
        public function prunable()
        {
            return static::where('created_at', '<=', now()->subMonth());
        }
    }
```

<a name="replicating-models"></a>
## Відтворення (Replicating) моделей

Ви можете створити незбережену копію існуючого екземпляра моделі за допомогою методу `replicate`. Цей метод особливо корисний, коли у вас є екземпляри моделі, які мають багато однакових атрибутів:

```php
use App\Models\Address;

$shipping = Address::create([
	'type' => 'shipping',
	'line_1' => '123 Example Street',
	'city' => 'Victorville',
	'state' => 'CA',
	'postcode' => '90001',
]);

$billing = $shipping->replicate()->fill([
	'type' => 'billing'
]);

$billing->save();
```
Щоб виключити реплікацію одного або кількох атрибутів у нову модель, ви можете передати масив методу `replicate` :

```php
$flight = Flight::create([
	'destination' => 'LAX',
	'origin' => 'LHR',
	'last_flown' => '2020-03-04 11:00:00',
	'last_pilot_id' => 747,
]);

$flight = $flight->replicate([
	'last_flown',
	'last_pilot_id'
]);
```

<a name="query-scopes"></a>
## Області Запитів

<a name="global-scopes"></a>
### Глобальне області

Глобальні області дозволяють додавати обмеження до всіх запитів для певної моделі. Власна функція [програмного видалення](#soft-deleting) Laravel використовує глобальні області видимості для отримання лише «не видалених» моделей із бази даних. Написання власних глобальних областей може забезпечити зручний і простий спосіб переконатися, що кожен запит для даної моделі отримує певні обмеження.

<a name="writing-global-scopes"></a>
#### Написання глобальних областей

Написати глобальну область видимості просто. Спочатку визначте клас, який реалізує інтерфейс `Illuminate\Database\Eloquent\Scope`. Laravel не має звичайного розташування, де ви повинні розміщувати класи області видимості, тому ви можете розмістити цей клас у будь-якому каталозі, який забажаєте.

Інтерфейс `Scope` вимагає від вас реалізації одного методу: `apply`. Метод `apply` може додавати обмеження `where` або інші типи умов до запиту за потреби:

```php
<?php

namespace App\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class AncientScope implements Scope
{
	/**
	 * Застосуйте область до заданого конструктора запитів Eloquent.
	 *
	 * @param  \Illuminate\Database\Eloquent\Builder  $builder
	 * @param  \Illuminate\Database\Eloquent\Model  $model
	 * @return void
	 */
	public function apply(Builder $builder, Model $model)
	{
		$builder->where('created_at', '<', now()->subYears(2000));
	}
}
```
> **Note**  
> Якщо ваша глобальна область запиту додає стовпці у вираз `SELECT` запиту, ви повинні використовувати метод `addSelect` замість `select`. Це попередить ненавмисну заміну існуючого виразу `SELECT` запиту.

<a name="applying-global-scopes"></a>
#### Застосування глобальних областей запиту

Щоб призначити глобальну область запиту моделі, необхідно перевизначити метод моделі `booted` і викликати метод моделі `addGlobalScope`. Метод `addGlobalScope` приймає екземпляр вашої області запиту як єдиний аргумент:

```php
<?php

namespace App\Models;

use App\Scopes\AncientScope;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
	/**
	 * Виконайте всі необхідні дії після завантаження моделі.
	 *
	 * @return void
	 */
	protected static function booted()
	{
		static::addGlobalScope(new AncientScope);
	}
}
```
Після додавання області до моделі `App\Models\User` у наведеному вище прикладі, виклик методу `User::all()` виконає наступний SQL-запит:

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```

<a name="anonymous-global-scopes"></a>
#### Анонімні глобальні області запиту

Eloquent також дозволяє визначати глобальні області за допомогою замикань, що особливо корисно для простих областей, які не вимагають окремого класу. Визначаючи глобальну область за допомогою замикання, ви повинні надати ім'я області за власним вибором як перший аргумент методу `addGlobalScope`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
	/**
	 * Виконайте всі необхідні дії після завантаження моделі.
	 *
	 * @return void
	 */
	protected static function booted()
	{
		static::addGlobalScope('ancient', function (Builder $builder) {
			$builder->where('created_at', '<', now()->subYears(2000));
		});
	}
}
```

<a name="removing-global-scopes"></a>
#### Видалення глобальних областей

Якщо ви хочете видалити глобальну область для певного запиту, ви можете використати метод `withoutGlobalScope`. Цей метод приймає назву класу глобальної області як єдиний аргумент:

```php
    User::withoutGlobalScope(AncientScope::class)->get();
```

Або, якщо ви визначили глобальну область за допомогою замикання, вам слід передати назву рядка, яку ви призначили глобальній області:

```php
    User::withoutGlobalScope('ancient')->get();
```
Якщо ви хочете видалити кілька або навіть усі глобальні області запиту, ви можете використати метод `withoutGlobalScopes`:

```php
// Видалити всі глобальні області...
User::withoutGlobalScopes()->get();

// Видалення деяких глобальних областей..
User::withoutGlobalScopes([
	FirstScope::class, SecondScope::class
])->get();
```

<a name="local-scopes"></a>
### Локальні області

Локальні області дозволяють визначати загальні набори обмежень запитів, які можна легко повторно використовувати у своему додатку. Наприклад, часте отримання «популярних» користувачів. Щоб визначити область запиту, додайте префікс `scope` до моделі Eloquent.


Області запиту повинні завжди повертати екземпляр конструктора запитів або `void`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
	/**
	 * Область запиту містить лише популярних користувачів.
	 *
	 * @param  \Illuminate\Database\Eloquent\Builder  $query
	 * @return \Illuminate\Database\Eloquent\Builder
	 */
	public function scopePopular($query)
	{
		return $query->where('votes', '>', 100);
	}

	/**
	 * Область запиту, що містить лише активних користувачів.
	 *
	 * @param  \Illuminate\Database\Eloquent\Builder  $query
	 * @return void
	 */
	public function scopeActive($query)
	{
		$query->where('active', 1);
	}
}
```

<a name="utilizing-a-local-scope"></a>
#### Використання локальної області

Після того, як область визначена, ви можете викликати методи області під час запиту моделі. Однак ви не повинні включати префікс `scope` під час виклику методу. Ви навіть можете зв'язувати виклики з різними областями:

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

Об'єднання кількох областей моделі Eloquent за допомогою оператора або запиту може вимагати використання замикань для досягнення правильного [логічного угруповання](queries#logical-grouping):

```php
$users = User::popular()->orWhere(function (Builder $query) {
	$query->active();
})->get();
```
Однак, оскільки це може бути громіздким, Laravel надає метод «вищого порядку» `orWhere`, який дозволяє вільно об’єднувати області без використання замикань:

```php
    $users = App\Models\User::popular()->orWhere->active()->get();
```

<a name="dynamic-scopes"></a>
#### Динамічні області

Іноді вам може знадобитися визначити область, яка приймає параметри. Щоб почати, просто додайте додаткові параметри до сигнатури методу області видимості. Параметри області слід визначати після параметра `$query`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
	/**
	 * Область запиту, що містить користувачів лише певного типу.
	 *
	 * @param  \Illuminate\Database\Eloquent\Builder  $query
	 * @param  mixed  $type
	 * @return \Illuminate\Database\Eloquent\Builder
	 */
	public function scopeOfType($query, $type)
	{
		return $query->where('type', $type);
	}
}
```
Після додавання очікуваних аргументів до сигнатури методу області видимості, ви можете передати аргументи під час виклику області:

```php
    $users = User::ofType('admin')->get();
```

<a name="comparing-models"></a>
## Порівняння моделей

Іноді вам може знадобитися визначити, чи є дві моделі «однаковими» чи ні. Методи `is` і `isNot` можна використовувати для швидкої перевірки наявності у двох моделей однакового первинного ключа, таблиці, та підключення до бази даних чи ні:

```php
if ($post->is($anotherPost)) {
	//
}

if ($post->isNot($anotherPost)) {
	//
}
```

Методи `is` і `isNot` також доступні при використанні [відношень](eloquent-relationships) `belongsTo`, `hasOne`, `morphTo`, і `morphOne`. Ці методи є особливо корисними, якщо ви хочете порівняти пов'язану модель без запиту на отримання цієї моделі:

```php
if ($post->author()->is($user)) {
	//
}
```

<a name="events"></a>
## Події

> **Note**  
> Бажаєте транслювати свої події Eloquent безпосередньо у клієнтський додаток? Перегляньте [трансляцію подій моделей ](broadcasting#model-broadcasting).

Моделі Eloquent ініціюють деякі події, що дозволяє використовувати такі гачки життєвого циклу моделі: `retrieved`, `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `trashed`, `forceDeleted`, `restoring`, `restored`, і `replicating`.

Подія `retrieved` буде відправлена, коли існуючу модель буде отримано з бази даних. Коли нова модель зберігається вперше, `creating` та `created` події будуть відправлені. Події оновлення/оновлення надсилатимуться, коли буде змінено існуючу модель і буде викликано метод збереження. Події `updating` / `updated` відправляються, коли модель створюється або оновлюється, навіть якщо атрибути моделі не змінено. Імена подій, що закінчуються на `-ing`, відправляються до збереження будь-яких змін у моделі, тоді як події, що закінчуються на `-ed`, відправляються після збереження змін у моделі.

Щоб розпочати прослуховування подій моделі, визначте властивість `$dispatchesEvents` у своїй моделі Eloquent. Ця властивість відображає різні точки життєвого циклу моделі Eloquent на ваші власні [класи подій](events). Кожен клас подій моделі повинен очікувати отримання екземпляра порушеної моделі через свій конструктор:

```php
<?php

namespace App\Models;

use App\Events\UserDeleted;
use App\Events\UserSaved;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
	use Notifiable;

	/**
	 * Мапа подій для моделей.
	 *
	 * @var array
	 */
	protected $dispatchesEvents = [
		'saved' => UserSaved::class,
		'deleted' => UserDeleted::class,
	];
}
```
Після визначення та порівняння подій можна використовувати [слухачів подій](events#defining-listeners) для їх обробки.

> **Warning**  
> Події моделі Eloquent `saved`, `updated`, `deleting`, та `deleted` під час масового оновлення або видалення не будуть ініційовані для порушених моделей. Це пов'язано з тим, що моделі фактично не вилучаються під час масового оновлення або видалення.

<a name="events-using-closures"></a>
### Використання замикань

Замість використання користувацьких класів подій ви можете зареєструвати замикання, які виконуються, коли надсилаються різні події моделі. Як правило, ви повинні зареєструвати ці замикання в методі `booted` вашої моделі:

```php
    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Виконайте всі необхідні дії після завантаження моделі.
         *
         * @return void
         */
        protected static function booted()
        {
            static::created(function ($user) {
                //
            });
        }
    }
```
За потреби ви можете використовувати [анонімні слухачі подій](events#queuable-anonymous-event-listeners), що стоять у черзі, під час реєстрації подій моделі. Це вкаже Laravel виконати модель прослуховування подій у фоновому режимі за допомогою [черги](queues) вашого додатку:

```php
use function Illuminate\Events\queueable;

static::created(queueable(function ($user) {
	//
}));
```

<a name="observers"></a>
### Спостерігачі

<a name="defining-observers"></a>
#### Визначення спостерігачів

Якщо ви прослуховуєте багато подій на даній моделі, ви можете використовувати спостерігачів, щоб об’єднати всіх слухачів в один клас. Класи спостерігачів мають назви методів, які відображають події Eloquent, які ви хочете прослухати. Кожен із цих методів отримує порушену модель як єдиний аргумент. Команда `make:observer` Artisan — це найпростіший спосіб створити новий клас спостерігача:

```shell
php artisan make:observer UserObserver --model=User
```
Ця команда помістить нового спостерігача у ваш каталог `App/Observers`. Якщо цей каталог не існує, Artisan створить його для вас. Ваш новий спостерігач виглядатиме так:

```php
    <?php

    namespace App\Observers;

    use App\Models\User;

    class UserObserver
    {
        /**
         * Обробити подію "created" моделі User.
         *
         * @param  \App\Models\User  $user
         * @return void
         */
        public function created(User $user)
        {
            //
        }

        /**
         * Обробити подію "updated" моделі User.
         *
         * @param  \App\Models\User  $user
         * @return void
         */
        public function updated(User $user)
        {
            //
        }

        /**
         * Обробити подію "deleted" моделі User.
         *
         * @param  \App\Models\User  $user
         * @return void
         */
        public function deleted(User $user)
        {
            //
        }
        
        /**
         * Handle the User "restored" event.
         *
         * @param  \App\Models\User  $user
         * @return void
         */
        public function restored(User $user)
        {
            //
        }

        /**
         * Обробити подію "forceDeleted" моделі User.
         *
         * @param  \App\Models\User  $user
         * @return void
         */
        public function forceDeleted(User $user)
        {
            //
        }
    }
```
Щоб зареєструвати спостерігача, вам потрібно викликати метод `observe` у моделі, яку ви хочете спостерігати. Ви можете зареєструвати спостерігачів у методі `boot` постачальника послуг `App\Providers\EventServiceProvider` вашого додатку:

```php
use App\Models\User;
use App\Observers\UserObserver;

/**
 * Зареєструйте будь-які події для вашого додатку.
 *
 * @return void
 */
public function boot()
{
	User::observe(UserObserver::class);
}
```
Крім того, ви можете перерахувати своїх спостерігачів у властивості `$observers` класу `App\Providers\EventServiceProvider` ваших програм:

```php
use App\Models\User;
use App\Observers\UserObserver;

/**
 * Модель спостерігачів для вашого додатку.
 *
 * @var array
 */
protected $observers = [
	User::class => [UserObserver::class],
];
```
> **Note**  
> Існують додаткові події, які спостерігач може прослухати, наприклад `saving` та `retrieved`. Ці події описані в документації [подій](#events).

<a name="observers-and-database-transactions"></a>
#### Спостерігачі та операції з базами даних

Коли моделі створюються в рамках транзакції бази даних, ви можете вказати спостерігачу виконувати свої обробники подій лише після того, як транзакція бази даних буде зафіксована. Ви можете досягти цього, визначивши властивість `$afterCommit` у спостерігачі. Якщо транзакція бази даних не в процесі, обробники подій виконаються негайно:

```php
<?php

namespace App\Observers;

use App\Models\User;

class UserObserver
{
	/**
	 * Обробляти події після завершення всіх транзакцій.
	 *
	 * @var bool
	 */
	public $afterCommit = true;

	/**
	 * Обробити подію «created» моделі User.
	 *
	 * @param  \App\Models\User  $user
	 * @return void
	 */
	public function created(User $user)
	{
		//
	}
}
```

<a name="muting-events"></a>
### Вимкнення подій

Час від часу вам може знадобитися тимчасово "вимкнути" всі події, які запускає модель. Ви можете досягти цього за допомогою методу `withoutEvents` . Метод `withoutEvents` приймає замикання як єдиний аргумент. Будь-який код, який виконується в цьому замиканні, не відправлятиме події моделі, а будь-яке значення, що повертається замиканням, повертатиметься методом `withoutEvents`:

```php
use App\Models\User;

$user = User::withoutEvents(function () use () {
	User::findOrFail(1)->delete();

	return User::find(2);
});
```

<a name="saving-a-single-model-without-events"></a>
#### Збереження однієї моделі без подій

Іноді вам може знадобитися "зберегти" надану модель без відправки жодних подій. Ви можете зробити це за допомогою методу `saveQuietly`:

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```