# База даних: Початок

- [Вступ](#introduction)
  - [Конфігурація](#configuration)
  - [Підключення для читання та запису](#read-and-write-connections)
- [Виконання запитів SQL](#running-queries)
  - [Використання кількох підключень до бази даних](#using-multiple-database-connections)
  - [Прослуховування подій запита](#listening-for-query-events)
  - [Моніторинг загального часу запита](#monitoring-cumulative-query-time)
- [Транзакції бази даних](#database-transactions)
- [Підключення до бази даних за допомогою інтерфейсу командного рядка CLI](#connecting-to-the-database-cli)

<a name="introduction"></a>

## Вступ

Майже кожен сучасний веб-додаток взаємодіє з базою даних. Laravel надзвичайно спрощує взаємодію з базами даних за допомогою різноманітної підтримки баз даних, використовуючи чистий SQL [конструктора запитів](queries) і [Eloquent ORM](eloquent). Наразі Laravel надає першу підтримку для п’яти баз даних:

<div class="content-list" markdown="1">

- MariaDB 10.2+ ([Version Policy](https://mariadb.org/about/#maintenance-policy))
- MySQL 5.7+ ([Version Policy](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL 10.0+ ([Version Policy](https://www.postgresql.org/support/versioning/))
- SQLite 3.8.8+
- SQL Server 2017+ ([Version Policy](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))

</div>

<a name="configuration"></a>

### Конфігурація

Конфігурація служб бази даних Laravel міститься у конфігураційному файлі додатка `config/database.php`. У цьому файлі ви можете визначити всі підключення до бази даних, а також вказати, яке підключення має використовуватися за замовчуванням. Більшість параметрів конфігурації в цьому файлі керуються значеннями змінних середовища вашого додатка. У цьому файлі наведено приклади для більшості систем баз даних, які підтримує Laravel.

За замовчуванням типова [конфігурація середовища](configuration.md#environment-configuration) Laravel готова до використання з [Laravel Sail](sail), яка є конфігурацією Docker для розробки додатків Laravel на вашій локальній машині. Однак ви можете змінювати конфігурацію вашої бази даних за потреби для вашої локальної бази даних.

<a name="sqlite-configuration"></a>

#### Конфігурація SQLite

Бази даних SQLite містяться в одному файлі вашої файлової системи. Ви можете створити нову базу даних SQLite за допомогою команди `touch` у вашому терміналі: `touch database/database.sqlite`. Після створення бази даних ви можете легко налаштувати змінні середовища, щоб вони вказували на цю базу даних, розмістивши абсолютний шлях до бази даних у змінній середовища `DB_DATABASE`:

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

Щоб увімкнути обмеження зовнішнього ключа для з’єднань SQLite, вам слід встановити для змінної середовища `DB_FOREIGN_KEYS` значення `true`:

```ini
DB_FOREIGN_KEYS=true
```

<a name="mssql-configuration"></a>

#### Конфігурація Microsoft SQL Server

Щоб використовувати базу даних Microsoft SQL Server, ви повинні переконатися, що у вас встановлено розширення PHP `sqlsrv` і `pdo_sqlsrv`, а також будь-які залежності, які можуть знадобитися для них, наприклад драйвер Microsoft SQL ODBC.

<a name="configuration-using-urls"></a>

#### Конфігурація за допомогою URL-адрес

Як правило, підключення до бази даних налаштовуються за допомогою кількох значень конфігурації, таких як `host`, `database`, `username`, `password`, тощо. Кожне з цих значень конфігурації має власну відповідну змінну середовища. Це означає, що під час налаштування інформації про підключення до бази даних на робочому сервері вам потрібно керувати декількома змінними середовища.

Деякі постачальники керованих баз даних, такі як AWS і Heroku, надають єдину «URL-адресу» бази даних, яка містить усю інформацію про підключення до бази даних в одному рядку. Приклад URL-адреси бази даних може виглядати приблизно так:

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

Ці URL-адреси зазвичай відповідають стандартній схемі:

```html
driver://username:password@host:port/database?options
```

Для зручності Laravel підтримує ці URL-адреси як альтернативу налаштуванню вашої бази даних із кількома параметрами конфігурації. Якщо параметр конфігурації URL-адреси (або відповідної змінної середовища `DATABASE_URL`) присутній, він використовуватиметься для отримання з’єднання з базою даних і облікових даних.

<a name="read-and-write-connections"></a>

### Підключення для читання та запису

Іноді ви можете використовувати одне підключення до бази даних для операторів SELECT, а інше для операторів INSERT, UPDATE і DELETE. Laravel спрощує це завдання, і завжди використовуватимуться належні з’єднання, незалежно від того, використовуєте ви сирі запити конструктора запитів, чи Eloquent ORM.

Щоб побачити, як мають бути налаштовані з’єднання для читання/запису, розглянемо цей приклад:

```php
'mysql' => [
	'read' => [
		'host' => [
			'192.168.1.1',
			'196.168.1.2',
		],
	],
	'write' => [
		'host' => [
			'196.168.1.3',
		],
	],
	'sticky' => true,
	'driver' => 'mysql',
	'database' => 'database',
	'username' => 'root',
	'password' => '',
	'charset' => 'utf8mb4',
	'collation' => 'utf8mb4_unicode_ci',
	'prefix' => '',
],
```

Зауважте, що до масиву конфігурації додано три ключі: `read`, `write` і `sticky`. Ключі `read` та `write` мають значення масива, що містить один ключ: `host`. Решта параметрів бази даних для підключень `read` та `write` буде об’єднано з основного масиву конфігурації `mysql`.

Вам потрібно лише розмістити елементи в масивах для `read` та `write`, якщо ви бажаєте замінити значення з основного масиву `mysql`. Отже, у цьому випадку `192.168.1.1` буде використано як хост для з’єднання «читання», а `192.168.1.3` — для з’єднання «запису». Облікові дані бази даних, префікс, набір символів та всі інші параметри в основному масиві `mysql` будуть спільні для обох з’єднань. Якщо в масиві конфігурації `host` існує кілька значень, для кожного запиту буде випадково обрано хост бази даних.

<a name="the-sticky-option"></a>

#### Варіант `sticky`

Параметр `sticky` — це _необов’язкове_ значення, яке можна використовувати, щоб дозволити негайне читання записів, які були записані до бази даних під час поточного циклу запиту. Якщо параметр `sticky` увімкнено і під час поточного циклу запиту щодо бази даних було виконано операцію «запису», будь-які подальші операції «читання» використовуватимуть з’єднання «запис». Це гарантує, що будь-які дані, записані під час циклу запита, можуть бути негайно прочитані з бази даних під час того самого запиту. Ви вирішуєте, чи це бажана поведінка для вашого додатка.

<a name="running-queries"></a>

## Виконання запитів SQL

Після того як ви налаштували підключення до бази даних, ви можете виконувати запити за допомогою фасаду `DB`. Фасад `DB` надає методи для кожного типу запита: `select`, `update`, `insert`, `delete`, і `statement`.

<a name="running-a-select-query"></a>

#### Виконання Select-запита

Щоб виконати базовий запит SELECT, ви можете використати метод `select` фасаду `DB`:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Support\Facades\DB;

class UserController extends Controller
{
	/**
	 * Показати список усіх користувачів додатка.
	 *
	 * @return \Illuminate\Http\Response
	 */
	public function index()
	{
		$users = DB::select('select * from users where active = ?', [1]);

		return view('user.index', ['users' => $users]);
	}
}
```

Перший аргумент, який передається в метод `select`, — це SQL-запит, тоді як другий аргумент — це будь-які прив’язки параметрів, які потрібно прив’язати до запита. Як правило, це значення умовних виразів `where`. Зв'язування параметрів забезпечує захист від ін`єкцій SQL.

Метод `select` завжди повертатиме "масив" результатів. Кожен результат у масиві буде об’єктом PHP `stdClass`, який представлятиме запис із бази даних:

```php
use Illuminate\Support\Facades\DB;

$users = DB::select('select * from users');

foreach ($users as $user) {
	echo $user->name;
}
```

<a name="selecting-scalar-values"></a>

#### Відбір скалярних значень

Іноді ваш запит до бази даних може повернути одне скалярне значення. Замість того, щоб вилучати результат з об’єкта запису, Laravel дозволяє безпосередньо отримати це значення за допомогою `scalar` методу:

```php
$burgers = DB::scalar(
	"select count(case when food = 'burger' then 1 end) as burgers from menu"
);
```

<a name="using-named-bindings"></a>

#### Використання іменованих прив’язок

Замість використання символа `?` для зв'язування параметрів, ви можете виконати запит, використовуючи іменовані зв’язки:

```php
$results = DB::select('select * from users where id = :id', ['id' => 1]);
```

<a name="running-an-insert-statement"></a>

#### Виконання Insert-запита

Щоб виконати запит `insert`, ви можете застосувати `insert` метод фасаду `DB`. Як і `select`, цей метод приймає SQL-запит як перший аргумент, а параметри прив’язки – як другий аргумент:

```php
use Illuminate\Support\Facades\DB;

DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);
```

<a name="running-an-update-statement"></a>

#### Виконання Update-запита

Метод `update` слід використовувати для оновлення існуючих записів у базі даних. Кількість рядків, на які впливає оператор, повертається методом:

```php
use Illuminate\Support\Facades\DB;

$affected = DB::update(
	'update users set votes = 100 where name = ?',
	['Anita']
);
```

<a name="running-a-delete-statement"></a>

#### Виконання Delete-запита

Для видалення записів із бази даних слід використовувати метод `delete`. Подібно до оновлення, кількість видалених рядків буде повернено методом:

```php
use Illuminate\Support\Facades\DB;

$deleted = DB::delete('delete from users');
```

<a name="running-a-general-statement"></a>

#### Виконання загального запита

Деякі оператори бази даних не повертають жодного значення. Для цих типів операцій ви можете використовувати метод `statement` фасаду `DB`:

```php
DB::statement('drop table users');
```

<a name="running-an-unprepared-statement"></a>

#### Виконання непідготовленого запита

Іноді вам може знадобитися виконати інструкцію SQL без зв’язування будь-яких значень. Для цього можна використати `unprepared` метод фасаду `DB`:

```php
DB::unprepared('update users set votes = 100 where name = "Dries"');
```

> **Warning**  
> Оскільки непідготовлені запити не зв’язують параметри, вони можуть бути вразливими до ін'єкцій SQL. Ви ніколи не повинні пропускати у непідготовлений вираз значення, керовані користувачем.

<a name="implicit-commits-in-transactions"></a>

#### Неявні коміти

Використовуючи методи `statement` і `unprepared` фасаду `DB` в транзакціях, ви повинні бути обережними, щоб уникнути операторів, які викликають неявні [неявні коміти](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html). Ці оператори змусять механізм бази даних опосередковано зафіксувати всю транзакцію, залишаючи Laravel без відому про рівень транзакції бази даних. Прикладом такого оператора є створення таблиці бази даних:

```php
DB::unprepared('create table a (col varchar(1) null)');
```

Будь ласка, зверніться до посібника з MySQL, щоб отримати [список усіх операторів](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html), які запускають неявні коміти.

<a name="using-multiple-database-connections"></a>

### Використання кількох підключень до бази даних

Якщо ваш додаток визначає кілька з’єднань у вашому файлі конфігурації `config/database.php`, ви можете отримати доступ до кожного з’єднання за допомогою методу `connection`, наданого фасадом `DB`. Ім’я підключення, передане методу `connection`, має відповідати одному з підключень, перелічених у вашому файлі конфігурації `config/database.php` або налаштованих під час виконання за допомогою помічника `config`:

```php
use Illuminate\Support\Facades\DB;

$users = DB::connection('sqlite')->select(/* ... */);
```

Ви можете отримати доступ до необробленого базового екземпляра PDO поточного з’єднання за допомогою методу `getPdo` екземпляра з’єднання:

```php
$pdo = DB::connection()->getPdo();
```

<a name="listening-for-query-events"></a>

### Прослуховування подій запита

Якщо ви бажаєте вказати замикання, яке викликається для кожного SQL-запита, що виконується вашим додатком, ви можете використовувати метод `listen` фасаду `DB`. Цей метод може бути корисним для логування запитів або налагодження. Ви можете зареєструвати замикання слухача запита в методі `boot` [постачальника служб](providers.md):

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
	/**
	 * Зареєструйте будь-які сервіси додатка.
	 *
	 * @return void
	 */
	public function register()
	{
		//
	}

	/**
	 * Завантажте будь-які служби додатка.
	 *
	 * @return void
	 */
	public function boot()
	{
		DB::listen(function ($query) {
			// $query->sql;
			// $query->bindings;
			// $query->time;
		});
	}
}
```

<a name="monitoring-cumulative-query-time"></a>

### Моніторинг загального часу запита

Загальною продуктивністю сучасних веб-додатків є кількість часу, який вони витрачають на запити до баз даних. На щастя, Laravel може викликати функцію зворотного виклику за вашим вибором, якщо він витрачає надто багато часу на запит до бази даних під час одного запиту. Щоб почати, вкажіть порогове значення часу запиту (у мілісекундах) і замикання для методу `whenQueryingForLongerThan`. Ви можете викликати цей метод у методі `boot` [постачальника служб](providers.md):

```php
<?php

namespace App\Providers;

use Illuminate\Database\Connection;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
	/**
	 * Зареєструйте будь-які сервіси додатка.
	 *
	 * @return void
	 */
	public function register()
	{
		//
	}

	/**
	 * Завантажте будь-які служби додатка.
	 *
	 * @return void
	 */
	public function boot()
	{
		DB::whenQueryingForLongerThan(500, function (Connection $connection) {
			// Повідомити команду розробників...
		});
	}
}
```

<a name="database-transactions"></a>

## Транзакції бази даних

Ви можете використовувати метод `transaction`, наданий фасадом `DB`, щоб виконати набір операцій у транзакції бази даних. Якщо під час замикання транзакції виникає виняток, транзакцію буде автоматично відкликано, а виняток буде створено повторно. Якщо замикання виконано успішно, транзакцію буде автоматично зафіксовано. Вам не потрібно турбуватися про ручне відкликання або фіксацію під час використання методу `transaction`:

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
	DB::update('update users set votes = 1');

	DB::delete('delete from posts');
});
```

<a name="handling-deadlocks"></a>

#### Обробка взаємоблокувань

Метод `transaction` приймає необов’язковий другий аргумент, який визначає кількість повторних спроб транзакції, коли виникає взаємоблокування. Після того, як ці спроби буде вичерпано, буде створено виняток:

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
	DB::update('update users set votes = 1');

	DB::delete('delete from posts');
}, 5);
```

<a name="manually-using-transactions"></a>

#### Використання транзакцій вручну

Якщо ви бажаєте розпочати транзакцію вручну та мати повний контроль над відхиленням та комітами, ви можете використати метод `beginTransaction`, наданий фасадом `DB`:

```php
use Illuminate\Support\Facades\DB;

DB::beginTransaction();
```

Ви можете відхилити транзакцію за допомогою методу `rollBack`:

```php
DB::rollBack();
```

Нарешті, ви можете зафіксувати транзакцію за допомогою методу `commit`:

```php
DB::commit();
```

> **Note**  
> Методи транзакцій фасаду `DB` контролюють транзакції як для [конструктора запитів](queries.md), так і для [Eloquent ORM](eloquent.md).

<a name="connecting-to-the-database-cli"></a>

## Підключення до бази даних за допомогою CLI

Якщо ви хочете підключитися до CLI вашої бази даних, ви можете скористатися командою `db` Artisan:

```shell
php artisan db
```

За потреби ви можете вказати ім’я з’єднання з базою даних, щоб підключитися до з’єднання з базою даних, яке не є з’єднанням за замовчуванням:

```shell
php artisan db mysql
```
