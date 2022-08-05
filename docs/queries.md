# База даних: конструктор запитів

- [Вступ](#introduction)
- [Виконання запитів до бази даних](#running-database-queries)
    - [Фрагментація результатів](#chunking-results)
    - [Відкладена потокова передача результатів](#streaming-results-lazily)
    - [Агрегати](#aggregates)
- [Вирази Select](#select-statements)
- [Необроблені вирази](#raw-expressions)
- [З'єднання Joins](#joins)
- [Об'єднання результатів Unions](#unions)
- [Основні вирази Where](#basic-where-clauses)
    - [Вирази Where](#where-clauses)
    - [Вирази Or Where](#or-where-clauses)
    - [Вирази Where Not](#where-not-clauses)
    - [Вирази Where та JSON](#json-where-clauses)
    - [Додаткові вирази Where](#additional-where-clauses)
    - [Логічне групування](#logical-grouping)
- [Розширені вирази Where](#advanced-where-clauses)
    - [Вирази Where Exists](#where-exists-clauses)
    - [Підзапити виразів Where](#subquery-where-clauses)
    - [Повнотекстові вирази Where](#full-text-where-clauses)
- [Упорядкування, групування, обмеження та зміщення](#ordering-grouping-limit-and-offset)
    - [Замовлення](#ordering)
    - [Групування](#grouping)
    - [Обмеження і зміщення](#limit-and-offset)
- [Умовні вирази When](#conditional-clauses)
- [Вставка Insert](#insert-statements)
    - [Оновлення-вставки](#upserts)
- [Вирази оновлення](#update-statements)
    - [Оновлення стовпців JSON](#updating-json-columns)
    - [Інкремент та Декримент](#increment-and-decrement)
- [Вирази видалення](#delete-statements)
- [Песимістичне блокування](#pessimistic-locking)
- [Налагодження](#debugging)

<a name="introduction"></a>
## Вступ

Конструктор запитів до бази даних Laravel забезпечує зручний, гнучкий інтерфейс для створення та виконання запитів до бази даних. Його можна використовувати для виконання більшості операцій з базою даних у вашому додатку він ідеально працює з усіма підтримуваними системами баз даних Laravel.

Конструктор запитів Laravel використовує зв’язування параметрів PDO, щоб захистити ваш додаток від атак SQL-ін’єкцій. Немає необхідності очищувати рядки, передані конструктору запитів як при'язки запитів.

> **Warning**  
> PDO не підтримує зв’язування імен стовпців. Тому, ви ніколи не повинні використовувати будь-які дані, що надходять від користувача, як імена стовпців, які використовуються вашими запитами, включаючи стовпці в запитах "order by".

<a name="running-database-queries"></a>
## Виконання запитів до бази даних

<a name="retrieving-all-rows-from-a-table"></a>
#### Отримання всіх рядків із таблиці

Ви можете використовувати метод `table`, наданий фасадом `DB`, щоб почати запит. Метод `table` повертає гнучкий екземпляр конструктора запитів для даної таблиці, дозволяючи вам зв’язати більше обмежень із запитом, а потім, нарешті, отримати результати запиту за допомогою методу `get`:

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
		$users = DB::table('users')->get();

		return view('user.index', ['users' => $users]);
	}
}
```
Метод `get` повертає екземпляр `Illuminate\Support\Collection`, який містить результати запита, де кожен результат є екземпляром об’єкта PHP `stdClass`. Ви можете отримати доступ до значення кожного стовпця, звернувшись до стовпця як до властивості об’єкта:

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->get();

foreach ($users as $user) {
	echo $user->name;
}
```
> **Note**  
> Колекції Laravel надають різноманітні надзвичайно потужні методи для відображення та скорочення даних. Щоб дізнатися більше про колекції Laravel, перегляньте [документацію колекції](collections.md).

<a name="retrieving-a-single-row-column-from-a-table"></a>
#### Отримання одного рядка/стовпця з таблиці

Якщо вам просто потрібно отримати один рядок із таблиці бази даних, ви можете скористатися методом `first` фасаду `DB`. Цей метод поверне один об’єкт `stdClass`:

```php
$user = DB::table('users')->where('name', 'John')->first();

return $user->email;
```
Якщо вам не потрібен цілий рядок, ви можете отримати одне значення із запису за допомогою методу `value`. Цей метод поверне значення стовпця безпосередньо:

```php
$email = DB::table('users')->where('name', 'John')->value('email');
```
Щоб отримати один рядок за значенням стовпця `id`, використовуйте метод `find`:

```php
    $user = DB::table('users')->find(3);
```
<a name="Getting-a-few-columns-from-a-table"></a>
#### Отримання окремо взятих стовпців з таблиці

Якщо вам потрібен рядок лише з кількох окремо взятих стовпців, ви можете вказати назви цих стовпців у масиві:
```php
$user = DB::table('users')->where('name', 'John')->first(['name','email']);
```
```php
$user = DB::table('users')->where('name', 'John')->get(['name','email']);
```
<a name="retrieving-a-list-of-column-values"></a>
#### Отримання списку значень стовпців

Якщо ви бажаєте отримати екземпляр `Illuminate\Support\Collection`, що містить значення одного стовпця, ви можете використати метод `pluck`. У цьому прикладі ми отримаємо колекцію назв користувачів:

```php
use Illuminate\Support\Facades\DB;

$titles = DB::table('users')->pluck('title');

foreach ($titles as $title) {
	echo $title;
}
```
Ви можете вказати стовпець, який результуюча колекція повинна використовувати як свої ключі, надавши другий аргумент методу `pluck`:

```php
$titles = DB::table('users')->pluck('title', 'name');

foreach ($titles as $name => $title) {
	echo $title;
}
```

<a name="chunking-results"></a>
### Фрагментація результатів

Якщо вам потрібно працювати з тисячами записів бази даних, розгляньте можливість використання методу `chunk`, який надає фасад `DB`. Цей метод одночасно отримує невелику порцію результатів і передає кожну порцію в замикання для обробки. Наприклад, давайте отримаємо всю таблицю `users` порціями по 100 записів одночасно. Даний метод працює разом з методом `orderBy`:

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->chunk(100, function ($users) {
	foreach ($users as $user) {
		//
	}
});
```
Ви можете зупинити подальшу обробку фрагментів, повернувши `false` із замикання:

```php
DB::table('users')->orderBy('id')->chunk(100, function ($users) {
	// Обробка записів...

	return false;
});
```
Якщо ви оновлюєте записи бази даних під час фрагментування результатів, ваші результати фрагментів можуть змінитися неочікуваним чином. Якщо ви плануєте оновлювати отримані записи під час фрагментації, завжди краще використовувати метод `chunkById`. Цей метод автоматично розбиває результати на фрагменти на основі первинного ключа запису:

```php
DB::table('users')->where('active', false)
	->chunkById(100, function ($users) {
		foreach ($users as $user) {
			DB::table('users')
			->where('id', $user->id)
			->update(['active' => true]);
		}
	});
```
> **Warning**  
> Під час оновлення або видалення записів у зворотному виклику послідовності будь-які зміни первинного ключа або зовнішніх ключів можуть вплинути на запит послідовності. Це потенційно може призвести до того, що записи не будуть включені до фрагментів результатів.

<a name="streaming-results-lazily"></a>
### Відкладена потокова передача результатів

Метод `lazy` працює подібно до [методу `chunk`](#chunking-results) у тому сенсі, що він виконує запити порційно. Однак замість того, щоб передавати кожну порцію у зворотний виклик, метод `lazy()` повертає [`LazyCollection`](collections.md#lazy-collections), що дозволяє вам взаємодіяти з результатами як єдиним потоком:

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function ($user) {
    //
});
```
Знову ж таки, якщо ви плануєте оновлювати отримані записи під час їх ітерації, найкраще замість цього використовувати методи `lazyById` або `lazyByIdDesc`. Ці методи автоматично розбиватимуть результати на сторінки на основі первинного ключа запису:

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function ($user) {
        DB::table('users')
		->where('id', $user->id)
		->update(['active' => true]);
    });
```
> **Warning**  
> При оновленні або видаленні записів під час їхньої ітерації будь-які зміни первинного ключа або зовнішніх ключів можуть вплинути на запит фрагмента. Це може призвести до того, що записи не будуть включені в результуючий набір.

<a name="aggregates"></a>
### Вилучення Агрегатів

Конструктор запитів також надає різноманітні методи для отримання агрегатних значень, таких як `count`, `max`, `min`, `avg`, і `sum`. Після створення запиту ви можете викликати будь-який із цих методів:

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->count();

$price = DB::table('orders')->max('price');
```
Звичайно, ви можете комбінувати ці методи з іншими виразами, щоб точно налаштувати спосіб обчислення вашого сукупного значення:

```php
$price = DB::table('orders')
	->where('finalized', 1)
	->avg('price');
```

<a name="determining-if-records-exist"></a>
#### Визначення наявності записів

Замість того, щоб використовувати метод `count`, для визначення існування будь-яких записів, які відповідають обмеженням вашого запита, ви можете використовувати методи `exists` and `doesntExist`:

```php
if (DB::table('orders')->where('finalized', 1)->exists()) {
	// ...
}

if (DB::table('orders')->where('finalized', 1)->doesntExist()) {
	// ...
}
```

<a name="select-statements"></a>
## Вирази Select

<a name="specifying-a-select-clause"></a>
#### Уточнення виразу Select

Можливо, вам не завжди потрібно вибирати усі стовпці з таблиці бази даних. Використовуючи метод `select`, ви можете вказати власний вираз SELECT для запиту:

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
		->select('name', 'email as user_email')
		->get();
```
Метод distinct дозволяє вам змусити запит повертати унікальні результати:

```php
$users = DB::table('users')->distinct()->get();
```

Якщо у вас вже є екземпляр конструктора запитів і ви бажаєте додати стовпець до його існуючого виразу `select`, ви можете використати метод `addSelect`:

```php
$query = DB::table('users')->select('name');

$users = $query->addSelect('age')->get();
```

<a name="raw-expressions"></a>
## Необроблені вирази

Іноді може знадобитися вставити довільний рядок у запит. Щоб створити необроблений рядковий вираз, ви можете використати `raw` метод, наданий фасадом `DB`:

```php
$users = DB::table('users')
		->select(DB::raw('count(*) as user_count, status'))
		->where('status', '<>', 1)
		->groupBy('status')
		->get();
```
> **Warning**  
> Необроблені оператори будуть вставлені в запит як рядки, тому ви повинні бути надзвичайно обережними, щоб уникнути створення вразливостей SQL-ін’єкцій.

<a name="raw-methods"></a>
### Необроблені методи

Замість використання методу `DB::raw` ви також можете використовувати наведені нижче методи, щоб вставити необроблений вираз у різні частини вашого запиту. **Пам’ятайте, що Laravel не може гарантувати, що будь-який запит, який використовує необроблені вирази, захищений від вразливості SQL-ін’єкції.**


<a name="selectraw"></a>
#### `selectRaw`

Метод `selectRaw` можна використовувати замість `addSelect(DB::raw(/* ... */))`. Цей метод приймає необов’язковий масив прив’язок як другий аргумент:

```php
$orders = DB::table('orders')
		->selectRaw('price * ? as price_with_tax', [1.0825])
		->get();
```

<a name="whereraw-orwhereraw"></a>
#### `whereRaw / orWhereRaw`

Методи `whereRaw` і `orWhereRaw` можна використовувати для вставки необробленого виразу «where» у ваш запит. Ці методи приймають необов’язковий масив прив’язок другим аргументом:

```php
$orders = DB::table('orders')
		->whereRaw('price > IF(state = "TX", ?, 100)', [200])
		->get();
```
<a name="havingraw-orhavingraw"></a>
#### `havingRaw / orHavingRaw`

Методи `havingRaw` і `orHavingRaw` можна використовувати для вставки необробленого рядка як значення виразу `HAVING`. Ці методи приймають необов’язковий масив прив’язок другим аргументом:

```php
$orders = DB::table('orders')
		->select('department', DB::raw('SUM(price) as total_sales'))
		->groupBy('department')
		->havingRaw('SUM(price) > ?', [2500])
		->get();
```

<a name="orderbyraw"></a>
#### `orderByRaw`

Метод `orderByRaw` використовується для вставки необробленого рядка як значення виразу `ORDER BY`:

```php
$orders = DB::table('orders')
		->orderByRaw('updated_at - created_at DESC')
		->get();
```

<a name="groupbyraw"></a>
### `groupByRaw`

Метод `groupByRaw` використовується для вставки необробленого рядка як значення виразу `GROUP BY`:

```php
$orders = DB::table('orders')
		->select('city', 'state')
		->groupByRaw('city, state')
		->get();
```

<a name="joins"></a>
## З'єднання Joins

<a name="inner-join-clause"></a>
#### Inner Join

Конструктор запитів також можна використовувати для додавання вираів об’єднання до ваших запитів. Щоб виконати базовий «inner join», ви можете використати метод `join` в екземплярі конструктора запитів. Перший аргумент, який передається методу `join`, — це ім’я таблиці, до якої потрібно приєднатися, тоді як решта аргументів визначають обмеження стовпців для об’єднання. Ви навіть можете об’єднати кілька таблиць в одному запиті:

```php
    use Illuminate\Support\Facades\DB;

$users = DB::table('users')
		->join('contacts', 'users.id', '=', 'contacts.user_id')
		->join('orders', 'users.id', '=', 'orders.user_id')
		->select('users.*', 'contacts.phone', 'orders.price')
		->get();
```
<a name="left-join-right-join-clause"></a>
#### Left Join / Right Join

Якщо ви хочете виконати «left join» або «right join» замість «inner join», використовуйте методи `leftJoin` або `leftJoin`. Ці методи мають ту саму сигнатуру, що й метод `join`:

```php
$users = DB::table('users')
		->leftJoin('posts', 'users.id', '=', 'posts.user_id')
		->get();

$users = DB::table('users')
		->rightJoin('posts', 'users.id', '=', 'posts.user_id')
		->get();
```

<a name="cross-join-clause"></a>
#### Cross Join

Ви можете використовувати метод `crossJoin` для виконання "cross join". Перехресні об’єднання створюють декартовий добуток між першою таблицею та об’єднаною таблицею:

```php
$sizes = DB::table('sizes')
		->crossJoin('colors')
		->get();
```

<a name="advanced-join-clauses"></a>
#### Розширені вирази з'єднування

Ви також можете вказати більш розширені вирази об’єднання. Для початку передайте замикання другим аргументом методу `join`. Замикання отримає екземпляр `Illuminate\Database\Query\JoinClause`, який дозволить вам вказати обмеження на вираз «join»:

```php
DB::table('users')
	->join('contacts', function ($join) {
		$join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
	})
	->get();
```
Якщо ви бажаєте використовувати вираз «where» у своїх об’єднаннях, ви можете використовувати методи `where` і `orWhere`, надані екземпляром `JoinClause`. Замість порівняння двох стовпців ці методи порівнюють стовпець із значенням:

```php
DB::table('users')
	->join('contacts', function ($join) {
		$join->on('users.id', '=', 'contacts.user_id')
		->where('contacts.user_id', '>', 5);
	})
	->get();
```

<a name="subquery-joins"></a>
#### Subquery разом з Joins

Щоб приєднати запит до підзапиту, можна використовувати методи `joinSub`, `leftJoinSub`, і `rightJoinSub`. Кожен із цих методів отримує три аргументи: підзапит, його псевдонім таблиці та замикання, яке визначає відповідні стовпці. У цьому прикладі ми отримаємо колекцію користувачів, де кожен запис користувача також містить позначку часу `created_at` останньої опублікованої публікації користувача в блозі:

```php
$latestPosts = DB::table('posts')
		->select('user_id', DB::raw('MAX(created_at) as last_post_created_at'))
		->where('is_published', true)
		->groupBy('user_id');

$users = DB::table('users')
		->joinSub($latestPosts, 'latest_posts', function ($join) {
			$join->on('users.id', '=', 'latest_posts.user_id');
		})->get();
```

<a name="unions"></a>
## Об'єднання результатів Unions

Конструктор запитів також надає зручний метод «об'єднання» двох або більше запитів. Наприклад, ви можете створити початковий запит і використати метод `union`, щоб об’єднати його з іншими запитами:

```php
use Illuminate\Support\Facades\DB;

$first = DB::table('users')
		->whereNull('first_name');

$users = DB::table('users')
		->whereNull('last_name')
		->union($first)
		->get();
```
На додаток до методу `union` конструктор запитів надає метод `unionAll`. Запити, об'єднані з використанням методу `unionAll`, не будуть видаляти результати, які повторюються. Метод `unionAll` має ту саму сигнатуру методу, що й метод union.

<a name="basic-where-clauses"></a>
## Основні вирази Where

<a name="where-clauses"></a>
### Вирази Where

Ви можете використовувати метод `where` конструктора запитів, щоб додати вирази «WHERE» до запиту. Найпростіший виклик метода `where` вимагає трьох аргументів. Перший аргумент - ім'я стовпця. Другим аргументом - оператор, який може бути будь-яким із підтримуваних операторів бази даних. Третій аргумент — це значення для порівняння зі значенням стовпця.

Наприклад, наступний запит отримує користувачів, у яких значення стовпця `votes` дорівнює `100`, а значення стовпця `age` більше, ніж `35`:

```php
$users = DB::table('users')
		->where('votes', '=', 100)
		->where('age', '>', 35)
		->get();
```
Для зручності, якщо ви хочете перевірити, що стовпець відповідає `=` заданому значенню, ви можете передати значення як другий аргумент методу where. Laravel припустить, що ви хочете використовувати оператор `=`:

```php
$users = DB::table('users')->where('votes', 100)->get();
```

Як зазначалося раніше, ви можете використовувати будь-який оператор, який підтримується вашою системою баз даних:

```php
$users = DB::table('users')
		->where('votes', '>=', 100)
		->get();

$users = DB::table('users')
		->where('votes', '<>', 100)
		->get();

$users = DB::table('users')
		->where('name', 'like', 'T%')
		->get();
```
Ви також можете передати масив умов до функції `where`. Кожен елемент масиву має бути масивом, що містить три аргументи, які зазвичай передаються методу `where`:

```php

$users = DB::table('users')->where([
	['status', '=', '1'],
	['subscribed', '<>', '1'],
])->get();
```
> **Warning**  
> PDO не підтримує зв’язування імен стовпців. Тому, ви ніколи не повинні використовувати будь-які дані, що надходять від користувача, як імена стовпців, які використовуються вашими запитами, включаючи стовпці в запитах "order by".

<a name="or-where-clauses"></a>
### Вирази Or Where

При об'єднанні ланцюжком викликів метода `where` конструктора запитів вирази "WHERE" будуть об'єднані разом за допомогою оператора "AND". Однак, можна використовувати метод `orWhere` для додавання виразу до запита за допомогою оператора `or`. Метод `orWhere` приймає ті ж аргументи, що й метод `where`:

```php
$users = DB::table('users')
		->where('votes', '>', 100)
		->orWhere('name', 'John')
		->get();
```
Якщо вам потрібно згрупувати умову "OR" у круглих дужках, ви можете передати замикання першим аргументом методу `orWhere`:

```php
$users = DB::table('users')
		->where('votes', '>', 100)
		->orWhere(function($query) {
			$query->where('name', 'Abigail')
			->where('votes', '>', 50);
		})
		->get();
```

Наведений вище приклад створить наступний SQL:

```sql
select * from users where votes > 100 or (name = 'Abigail' and votes > 50)
```
> **Warning**  
> Завжди слід групувати виклики `orWhere`, щоб уникнути неочікуваної поведінки під час застосування [глобальних областей запита](eloquent.md#query-scopes).

<a name="where-not-clauses"></a>
### Вирази Where Not

Методи `whereNot` і `orWhereNot` можна використовувати для заперечення певної групи обмежень запита. Наприклад, наступний запит ігнорує продукти, які розмитнюються або мають ціну нижче десяти:

```php
$products = DB::table('products')
		->whereNot(function ($query) {
			$query->where('clearance', true)
			->orWhere('price', '<', 10);
		})
		->get();
```

<a name="json-where-clauses"></a>
### Вирази Where та JSON

Laravel також підтримує запити в базах даних, які забезпечують підтримку JSON типів стовпців. Наразі це MySQL 5.7+, PostgreSQL, SQL Server 2016 і SQLite 3.9.0 (з розширенням [JSON1 extension](https://www.sqlite.org/json1.html)). Для запиту стовпця JSON використовуйте оператор `->`:

```php
$users = DB::table('users')
		->where('preferences->dining->meal', 'salad')
		->get();
```
Ви можете використовувати `whereJsonContains` для запиту масивів JSON. Ця функція не підтримується базою даних SQLite:

```php
$users = DB::table('users')
		->whereJsonContains('options->languages', 'en')
		->get();
```
Якщо ваш додаток використовує бази даних MySQL або PostgreSQL, ви можете передати масив значень методу `whereJsonContains`:

```php
$users = DB::table('users')
		->whereJsonContains('options->languages', ['en', 'de'])
		->get();
```
Ви можете використовувати метод `whereJsonLength` для запиту масивів JSON за їх довжиною:

```php
$users = DB::table('users')
		->whereJsonLength('options->languages', 0)
		->get();

$users = DB::table('users')
		->whereJsonLength('options->languages', '>', 1)
		->get();
```
<a name="additional-where-clauses"></a>
### Додаткові вирази Where

**whereBetween / orWhereBetween**

Метод `whereBetween` перевіряє, чи значення стовпця знаходиться між двома значеннями:

```php
$users = DB::table('users')
		->whereBetween('votes', [1, 100])
		->get();
```

**whereNotBetween / orWhereNotBetween**

Метод `whereNotBetween` перевіряє, що значення стовпця знаходиться за межами двох значень:

```php
$users = DB::table('users')
		->whereNotBetween('votes', [1, 100])
		->get();
```
**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

Метод `whereIn` перевіряє, чи значення даного стовпця міститься в заданому масиві:

```php
$users = DB::table('users')
		->whereIn('id', [1, 2, 3])
		->get();
```
Метод `whereNotIn` перевіряє, чи значення даного стовпця не міститься в заданому масиві:

```php
$users = DB::table('users')
		->whereNotIn('id', [1, 2, 3])
		->get();
```
> **Warning**  
> Якщо ви додаєте великий масив цілочисельних прив’язок до свого запита, можна використати методи `whereIntegerInRaw` або `whereIntegerNotInRaw`, щоб значно зменшити використання пам’яті.

**whereNull / whereNotNull / orWhereNull / orWhereNotNull**

Метод `whereNull` перевіряє, що значення даного стовпця дорівнює `NULL`:

```php
$users = DB::table('users')
		->whereNull('updated_at')
		->get();
```
Метод `whereNotNull` перевіряє, що значення стовпця не дорівнює `NULL`:

```php
$users = DB::table('users')
		->whereNotNull('updated_at')
		->get();
```
**whereDate / whereMonth / whereDay / whereYear / whereTime**

Метод `whereDate` можна використовувати для порівняння значення стовпця з датою:

```php
$users = DB::table('users')
		->whereDate('created_at', '2016-12-31')
		->get();
```
Метод `whereMonth` можна використовувати для порівняння значення стовпця з певним місяцем:

```php
$users = DB::table('users')
		->whereMonth('created_at', '12')
		->get();
```
Метод `whereDay` можна використовувати для порівняння значення стовпця з певним днем ​​місяця:

```php
$users = DB::table('users')
		->whereDay('created_at', '31')
		->get();
```
Метод `whereYear` можна використовувати для порівняння значення стовпця з певним роком:

```php
$users = DB::table('users')
		->whereYear('created_at', '2016')
		->get();
```
Метод `whereTime` можна використовувати для порівняння значення стовпця з певним часом:

```php
$users = DB::table('users')
		->whereTime('created_at', '=', '11:20:45')
		->get();
```
**whereColumn / orWhereColumn**

Метод `whereColumn` можна використовувати для перевірки рівності двох стовпців:

```php
$users = DB::table('users')
		->whereColumn('first_name', 'last_name')
		->get();
```
Ви також можете передати оператор порівняння методу `whereColumn`:

```php
$users = DB::table('users')
		->whereColumn('updated_at', '>', 'created_at')
		->get();
```
Ви також можете передати масив порівнянь стовпців у метод `whereColumn`. Ці умови будуть об’єднані за допомогою оператора `and`:

```php
$users = DB::table('users')
		->whereColumn([
			['first_name', '=', 'last_name'],
			['updated_at', '>', 'created_at'],
		])->get();
```


<a name="logical-grouping"></a>
### Логічне групування

Іноді вам може знадобитися згрупувати кілька виразів «WHERE» в дужках, щоб досягти бажаного логічного групування вашого запита. Насправді, ви завжди повинні групувати виклики метода `orWhere` в дужках, щоб уникнути неочікуваної поведінки запита. Щоб досягти цього, ви можете передати замикання методу `where`:


```php
$users = DB::table('users')
		->where('name', '=', 'John')
		->where(function ($query) {
			$query->where('votes', '>', 100)
			->orWhere('title', '=', 'Admin');
		})
		->get();
```
Як бачите, передача замикання в метод `where` вказує конструктору запитів розпочати групу обмежень. Замикання отримає екземпляр конструктора запитів, який можна використовувати для встановлення обмежень, які мають міститися в групі дужок. Наведений вище приклад створить такий SQL:

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```
> **Warning**  
> Завжди слід групувати виклики `orWhere`, щоб уникнути неочікуваної поведінки під час застосування [глобальних областей](eloquent.md#query-scopes).

<a name="advanced-where-clauses"></a>
### Розширені вирази Where

<a name="where-exists-clauses"></a>
### Вирази Where Exists

Метод `whereExists` дозволяє писати вирази SQL "where exists". Метод `whereExists` приймає замикання, яке отримає екземпляр конструктора запитів, дозволяючи вам визначити запит, який слід розмістити в реченні «exists»:

```php
$users = DB::table('users')
		->whereExists(function ($query) {
			$query->select(DB::raw(1))
			->from('orders')
			->whereColumn('orders.user_id', 'users.id');
		})
		->get();
```
Наведений вище запит створить наступний SQL:

```sql
select * from users
where exists (
    select 1
    from orders
    where orders.user_id = users.id
)
```

<a name="subquery-where-clauses"></a>
### Підзапити виразів Where

Іноді вам може знадобитися створити вираз «WHERE», який порівнює результати підзапита з вказаним значенням у другому аргументі. Ви можете досягти цього, передавши замикання та значення для порівняння методу `where`. Наприклад, наступний запит отримає всіх користувачів, які нещодавно були «членами» певного типу;

```php
use App\Models\User;

$users = User::where(function ($query) {
	$query->select('type')
		->from('membership')
		->whereColumn('membership.user_id', 'users.id')
		->orderByDesc('membership.start_date')
		->limit(1);
}, 'Pro')->get();
```
Або вам може знадобитися створити вираз «WHERE», який порівнює стовпець із результатами підзапита. Ви можете досягти цього, передавши стовпець, оператор і замикання методу `where`. Наприклад, наступний запит отримає всі записи про доходи, сума яких менша за середню;

```php
use App\Models\Income;

$incomes = Income::where('amount', '<', function ($query) {
	$query->selectRaw('avg(i.amount)')->from('incomes as i');
})->get();
```

<a name="full-text-where-clauses"></a>
### Повнотекстові вирази Where

> **Warning**  
> Повнотекстові вирази де наразі підтримуються MySQL і PostgreSQL.

Методи `whereFullText` і `orWhereFullText` можна використовувати для додавання повнотекстових виразів "WHERE" у запит для стовпців, які мають [повнотекстові індекси](migrations.md#available-index-types). Laravel перетворює ці методи у відповідний SQL для базової системи бази даних. Наприклад, вираз `MATCH AGAINST` буде згенеровано для додатків, які використовують MySQL:

```php
$users = DB::table('users')
		->whereFullText('bio', 'web developer')
		->get();
```
<a name="ordering-grouping-limit-and-offset"></a>
## Упорядкування, групування, обмеження та зміщення

<a name="ordering"></a>
### Замовлення

<a name="orderby"></a>
#### Метод `orderBy`

Метод `orderBy` дозволяє сортувати результати запита за заданим стовпцем. Перший аргумент, прийнятий методом orderBy, має бути стовпцем, за яким ви бажаєте відсортувати, тоді як другий аргумент визначає напрямок сортування та може бути як за зростанням `asc`, так і за спадом `desc`:

```php
$users = DB::table('users')
		->orderBy('name', 'desc')
		->get();
```
Щоб відсортувати за кількома стовпцями, ви можете просто викликати `orderBy` стільки разів, скільки потрібно:

```php
$users = DB::table('users')
		->orderBy('name', 'desc')
		->orderBy('email', 'asc')
		->get();
```
<a name="latest-oldest"></a>
#### Методи `latest` і `oldest`

Методи `latest` та `oldest` дозволяють легко впорядкувати результати за датою. За замовчуванням результат буде впорядковано за стовпцем `created_at` таблиці. Або ви можете передати ім’я стовпця, за яким потрібно відсортувати:

```php
$user = DB::table('users')
		->latest()
		->first();
```
<a name="random-ordering"></a>
#### Випадкове сортування

Метод `inRandomOrder` можна використовувати для сортування результатів запита випадковим чином. Наприклад, ви можете використовувати цей метод для отримання випадкового користувача:

```php
$randomUser = DB::table('users')
		->inRandomOrder()
		->first();
```
<a name="removing-existing-orderings"></a>
#### Видалення існуючого сортування

Метод `reorder` видаляє всі вирази "order by", які раніше були застосовані до запита:

```php
$query = DB::table('users')->orderBy('name');

$unorderedUsers = $query->reorder()->get();
```
Ви можете передати стовпець і напрямок під час виклику метода `reorder`, щоб видалити всі існуючі запити «order by» і застосувати абсолютно новий порядок до запиту:

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorder('email', 'desc')->get();
```
<a name="grouping"></a>
### Групування

<a name="groupby-having"></a>
#### Методи `groupBy` і `having`

Як і слід було очікувати, для групування результатів запиту можна використовувати методи `groupBy` та `having`. Сигнатура методу `having` подібна до сигнатури метода `where`:

```php
$users = DB::table('users')
		->groupBy('account_id')
		->having('account_id', '>', 100)
		->get();
```
Ви можете використовувати метод `havingBetween` для фільтрації результатів у заданому діапазоні:

```php
$report = DB::table('orders')
		->selectRaw('count(id) as number_of_orders, customer_id')
		->groupBy('customer_id')
		->havingBetween('number_of_orders', [5, 15])
		->get();
```
Ви можете передати кілька аргументів методу `groupBy` для групування за кількома стовпцями:

```php
$users = DB::table('users')
		->groupBy('first_name', 'status')
		->having('account_id', '>', 100)
		->get();
```
Щоб побудувати розширені вирази `having`, перегляньте метод [`havingRaw`](#raw-methods).

<a name="limit-and-offset"></a>
### Ліміт і зміщення

<a name="skip-take"></a>
#### Методи `skip` і `take`

Ви можете використовувати методи `skip` і `take`, щоб обмежити кількість результатів, які повертає запит, або пропустити певну кількість результатів у запиті:

```php
$users = DB::table('users')->skip(10)->take(5)->get();
```
Крім того, ви можете використовувати методи `limit` and `offset`. Ці методи функціонально еквівалентні методам `take` і `skip` відповідно:

```php
$users = DB::table('users')
		->offset(10)
		->limit(5)
		->get();
```
<a name="conditional-clauses"></a>
## Умовні вирази

Іноді вам може знадобитися, щоб певні положення запиту застосовувалися до запиту на основі іншої умови. Наприклад, ви можете застосувати оператор `where`, лише якщо надане вхідне значення присутнє у вхідному HTTP-запиті. Ви можете зробити це за допомогою методу `when`:

```php
$role = $request->input('role');

$users = DB::table('users')
		->when($role, function ($query, $role) {
			$query->where('role_id', $role);
		})
		->get();
```
Метод `when` виконує вказане замикання лише тоді, коли перший аргумент є істинним. Якщо перший аргумент false, замикання не буде виконано. Отже, у наведеному вище прикладі замикання, надане методу `when`, буде викликано, лише якщо поле `role` присутнє у вхідному запиті та оцінюється як `true`.

Ви можете передати інше замикання як третій аргумент методу `when`. Це замикання буде виконано, лише якщо перший аргумент оцінюється як `false`. Щоб проілюструвати, як можна використовувати цей функціонал, ми використаємо його для налаштування порядку сортування запита за замовчуванням:

```php
$sortByVotes = $request->input('sort_by_votes');

$users = DB::table('users')
		->when($sortByVotes, function ($query, $sortByVotes) {
			$query->orderBy('votes');
		}, function ($query) {
			$query->orderBy('name');
		})
		->get();
```
<a name="insert-statements"></a>
## Вставка

Конструктор запитів також надає метод `insert`, який можна використовувати для вставлення записів у таблицю бази даних. Метод `insert` приймає масив імен і значень стовпців:

```php
DB::table('users')->insert([
	'email' => 'kayla@example.com',
	'votes' => 0
]);
```
Ви можете вставити кілька записів одночасно, передавши двовимірний масив. Кожен масив представляє запис, який потрібно вставити в таблицю:

```php
DB::table('users')->insert([
	['email' => 'picard@example.com', 'votes' => 0],
	['email' => 'janeway@example.com', 'votes' => 0],
]);
```
Метод `insertOrIgnore` ігноруватиме помилки під час вставлення записів у базу даних. Використовуючи цей метод, слід пам’ятати, що помилки записів, які повторюються, ігноруватимуться, а інші типи помилок також можуть ігноруватися залежно від механізму бази даних. Наприклад, `insertOrIgnore` [обійде суворий режим MySQL](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution):

```php
DB::table('users')->insertOrIgnore([
	['id' => 1, 'email' => 'sisko@example.com'],
	['id' => 2, 'email' => 'archer@example.com'],
]);
```
Метод `insertUsing` вставлятиме нові записи в таблицю, використовуючи результат підзапита для визначення даних, які слід вставити:

```php
DB::table('pruned_users')->insertUsing([
	'id', 'name', 'email', 'email_verified_at'
], DB::table('users')->select(
	'id', 'name', 'email', 'email_verified_at'
)->where('updated_at', '<=', now()->subMonth()));
```

<a name="auto-incrementing-ids"></a>
#### Автоматичне збільшення ідентифікаторів

Якщо таблиця має ідентифікатор, який автоматично збільшується, використовуйте метод `insertGetId`, щоб вставити запис, а потім отримати ID:

```php
$id = DB::table('users')->insertGetId(
	['email' => 'john@example.com', 'votes' => 0]
);
```
> **Warning**  
> Під час використання PostgreSQL метод `insertGetId` очікує, що стовпець із автоматичним збільшенням матиме назву `id`. Якщо ви хочете отримати ID з іншої «послідовності», ви можете передати назву стовпця як другий параметр метода `insertGetId`.

<a name="upserts"></a>
### Оновлення-вставки

Метод `upsert` вставлятиме записи, які не існують, і оновлюватиме записи, які вже існують, новими значеннями, які ви можете вказати. Перший аргумент методу складається зі значень, які потрібно вставити або оновити, тоді як другий аргумент містить список стовпців, які однозначно ідентифікують записи в пов’язаній таблиці. Третій і останній аргумент методу — це масив стовпців, які слід оновити, якщо відповідний запис уже існує в базі даних:

```php
DB::table('flights')->upsert(
	[
		['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
		['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
	],
	['departure', 'destination'],
	['price']
);
```
У наведеному вище прикладі Laravel спробує вставити два записи. Якщо вже існує запис із однаковими значеннями стовпців `departure` та `destination`, Laravel оновить стовпець `price` цього запису.

> **Warning**  
> Усі бази даних, крім SQL Server, вимагають, щоб стовпці у другому аргументі метода `upsert` мали «primary» або «unique» індекс. Крім того, драйвер бази даних MySQL ігнорує другий аргумент метода `upsert` і завжди використовує «primary» і «unique» індекси таблиці для виявлення існуючих записів.

<a name="update-statements"></a>
## Вирази оновлення

Окрім вставки записів у базу даних, конструктор запитів також може оновлювати наявні записи за допомогою методу `update`. Метод `update`, як і метод `insert`, приймає масив стовпців і пар значень, що вказують на стовпці, які потрібно оновити. Метод `update` повертає кількість оновлених рядків. Ви можете обмежити запит `update` за допомогою виразу `where`:

```php
$affected = DB::table('users')
		->where('id', 1)
		->update(['votes' => 1]);
```
<a name="update-or-insert"></a>
#### Оновлення або вставка

Іноді вам може знадобитися оновити існуючий запис у базі даних або створити його, якщо відповідного запису немає. У цьому випадку можна використовувати метод `updateOrInsert`. Метод `updateOrInsert` приймає два аргументи: масив умов, за якими потрібно знайти запис, і масив пар стовпців і значень, що вказують на стовпці, які потрібно оновити.


Метод `updateOrInsert` намагатиметься знайти відповідний запис бази даних, використовуючи пари стовпця та значення першого аргументу. Якщо запис існує, його буде оновлено значеннями другого аргументу. Якщо запис не знайдено, буде вставлено новий запис із об’єднаними атрибутами обох аргументів:


```php
DB::table('users')
	->updateOrInsert(
		['email' => 'john@example.com', 'name' => 'John'],
		['votes' => '2']
	);
```

> {More details} У першому умовному масиві `['email' => 'john@example.com', 'name' => 'John']` ми перевіряємо, чи існує в таблиці запис *john@example.com*. Якщо запис не існує, він створюється, але якщо він існує, ми оновлюємо стовпець *votes* таблиці користувачів до 2, що містить другий масив `['votes' => '2']`.

<a name="updating-json-columns"></a>
### Оновлення стовпців JSON

При оновленні стовпця JSON необхідно використовувати синтаксис `->` для оновлення відповідного ключа в об'єкті JSON. Ця операція підтримується в MySQL 5.7+ та PostgreSQL 9.5+:

```php
$affected = DB::table('users')
		->where('id', 1)
		->update(['options->enabled' => true]);
```

<a name="increment-and-decrement"></a>
### Інкремент та Декримент

Конструктор запитів також надає зручні методи для збільшення або зменшення значення даного стовпця. Обидва ці методи приймають принаймні один аргумент: стовпець, який потрібно змінити. Може бути наданий другий аргумент, щоб визначити суму, на яку слід збільшити або зменшити стовпець:

```php
DB::table('users')->increment('votes');

DB::table('users')->increment('votes', 5);

DB::table('users')->decrement('votes');

DB::table('users')->decrement('votes', 5);
```
Ви також можете вказати додаткові стовпці для оновлення під час операції:

```php
    DB::table('users')->increment('votes', 1, ['name' => 'John']);
```
<a name="delete-statements"></a>
## Видалення

Для видалення записів із таблиці можна використовувати метод `delete` конструктора запитів. Метод `delete` повертає кількість видалених рядків. Ви можете обмежити оператори `delete`, додавши метод `where` перед викликом методу `delete`:

```php
$deleted = DB::table('users')->delete();

$deleted = DB::table('users')->where('votes', '>', 100)->delete();
```
Якщо ви бажаєте лдразу очистити всю таблицю, що призведе до видалення всіх записів із таблиці та скидання auto-incrementing ID, ви можете скористатися методом `truncate`:

```php
DB::table('users')->truncate();
```
> **Warning**  
> Ці методи видалятимуть дані без функціонала `SoftDeletes` навіть, якщо він визначений в моделі.

<a name="table-truncation-and-postgresql"></a>
#### Очищення таблиці і PostgreSQL

Під час загального очищення бази даних PostgreSQL буде застосовано поведінку CASCADE`. Це означає, що всі записи, пов’язані із зовнішнім ключем, в інших таблицях також будуть видалені.

<a name="pessimistic-locking"></a>
## Песимістичне блокування

Конструктор запитів також містить кілька функцій, які допоможуть вам досягти "песимістичного блокування" під час виконання ваших операторів `select`. Щоб виконати оператор із «спільним блокуванням», ви можете викликати метод `sharedLock`. Спільне блокування запобігає зміні обраних рядків до завершення транзакції:

```php
DB::table('users')
	->where('votes', '>', 100)
	->sharedLock()
	->get();
```
Крім того, ви можете використовувати метод `lockForUpdate`. Блокування «для оновлення» запобігає зміні обраних записів або їх вибору за допомогою іншого спільного блокування:

```php
DB::table('users')
	->where('votes', '>', 100)
	->lockForUpdate()
	->get();
```

<a name="debugging"></a>
## Налагодження

Ви можете використовувати методи `dd` і `dump` під час створення запита, щоб створити дамп поточних прив’язок запита та SQL. Метод `dd` відобразить інформацію про налагодження, а потім припинить виконання запиту. Метод `dump` відобразить інформацію про налагодження, але дозволить запиту продовжити виконання:

```php
DB::table('users')->where('votes', '>', 100)->dd();

DB::table('users')->where('votes', '>', 100)->dump();
```