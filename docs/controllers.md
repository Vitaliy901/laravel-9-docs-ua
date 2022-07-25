# Контроллери

- [Вступ](#introduction)
- [Написання контроллерів](#writing-controllers)
    - [Базові контроллери](#basic-controllers)
    - [Контроллери одного методу](#single-action-controllers)
- [Middleware контроллера](#controller-middleware)
- [Контроллери ресурсів](#resource-controllers)
    - [Часткові маршрути ресурсів](#restful-partial-resource-routes)
    - [Вкладені ресурси](#restful-nested-resources)
    - [Перейменування ресурсних маршрутів](#restful-naming-resource-routes)
    - [Перейменування параметрів ресурсних маршрутів](#restful-naming-resource-route-parameters)
    - [Обмеження ресурсних маршрутів](#restful-scoping-resource-routes)
    - [Локалізація URI ресурсів](#restful-localizing-resource-uris)
    - [Додаткові маршрути ресурсних контролерів](#restful-supplementing-resource-controllers)
- [Впровадження залежностей і контроллери](#dependency-injection-and-controllers)

<a name="introduction"></a>
## Вступ

Замість того, щоб визначати всю логіку обробки запитів як замикання у файлах маршрутів, ви можете організувати цю поведінку за допомогою класів «контроллера». Контроллери можуть групувати пов’язану логіку обробки запитів в один клас. Наприклад, клас `UserController` може обробляти всі вхідні запити, пов’язані з користувачами, включаючи показ, створення, оновлення та видалення користувачів. За замовчуванням контроллери зберігаються в каталозі `app/Http/Controllers`

<a name="writing-controllers"></a>
## Написання контроллерів

<a name="basic-controllers"></a>
### Базові контроллери

Давайте розглянемо приклад базового контроллера. Зверніть увагу, що контроллер розширює базовий клас контроллера, включений у Laravel: `App\Http\Controllers\Controller`:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\User;

class UserController extends Controller
{
	/**
	 * Показати профіль певного користувача.
	 *
	 * @param  int  $id
	 * @return \Illuminate\View\View
	 */
	public function show($id)
	{
		return view('user.profile', [
			'user' => User::findOrFail($id)
		]);
	}
}
```
Ви можете визначити маршрут до цього методу контроллера так:

```php
use App\Http\Controllers\UserController;

Route::get('/user/{id}', [UserController::class, 'show']);
```
Коли вхідний запит відповідає вказаному URI маршруту, буде викликано метод `show` у класі `App\Http\Controllers\UserController`, і параметри маршруту будуть передані цьому методу.

> {tip} Контроллери не **зобов’язані** розширювати базовий клас. Однак ви не матимете доступу до зручних особливостей, таких як `middleware` та `authorize`.

<a name="single-action-controllers"></a>
### Контроллери одного методу

Якщо дія контроллера особливо складна, вам може бути зручніше присвятити цілий клас контроллера цій одній дії. Щоб досягти цього, ви можете визначити один метод `__invoke` у контроллері:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\User;

class ProvisionServer extends Controller
{
	/**
	 * Підготувати новий веб-сервер.
	 *
	 * @return \Illuminate\Http\Response
	 */
	public function __invoke()
	{
		// ...
	}
}
```
Під час реєстрації маршрутів для контроллерів одиночної дії вам не потрібно вказувати метод контроллера. Натомість вам потрібно лише передати маршрутизатору ім'я контроллера:

```php
use App\Http\Controllers\ProvisionServer;

Route::post('/server', ProvisionServer::class);
```
Ви можете створити готовий контроллер одиночної дії за допомогою параметра `--invokable` команди `make:controller` Artisan:

```shell
php artisan make:controller ProvisionServer --invokable
```

> {tip} Заготовки контроллера можна налаштувати за допомогою [публікації заготовок](artisan.md#stub-customization).

<a name="controller-middleware"></a>
## Middleware контроллера

[Посередник](middleware.md) може бути призначено маршрутам контроллера у ваших файлах маршрутів:

```php
Route::get('profile', [UserController::class, 'show'])->middleware('auth');
```
Або вам може бути зручно вказати посередника в конструкторі вашого контроллера. Використовуючи метод `middleware` в конструкторі вашого контроллера, ви можете призначити посередника для дій контроллерара:


```php
class UserController extends Controller
{
	/**
	 * Створити новий екземпляр контроллерара.
	 *
	 * @return void
	 */
	public function __construct()
	{
		$this->middleware('auth');
		$this->middleware('log')->only('index');
		$this->middleware('subscribed')->except('store');
	}
}
```
Контроллери також дозволяють реєструвати посередника за допомогою функції зворотного виклику. Це забезпечує зручний спосіб визначення вбудованого посередника для одного контроллера без визначення одного окремого класу посередника:

```php
    $this->middleware(function ($request, $next) {
        return $next($request);
    });
```

<a name="resource-controllers"></a>
## Контроллери ресурсів

Якщо ви розглядаєте кожну модель Eloquent у своєму додатку як «ресурс», то для кожного ресурсу у вашому додатку зазвичай виконуються одні й ті самі набори дій. Наприклад, уявіть, що ваш додаток містить `Photo` і `Movie`. Імовірно, користувачі можуть створювати, читати, оновлювати або видаляти ці ресурси.

Через цей поширений варіант використання маршрутизація ресурсів Laravel призначає типові маршрути створення, читання, оновлення та видалення «CRUD» контроллеру за допомогою одного рядка коду. Щоб почати, ми можемо використати опцію `--resource` команди `make:controller` Artisan, щоб швидко створити контроллер для обробки цих дій:

```shell
php artisan make:controller PhotoController --resource
```
Ця команда створить контроллер у `app/Http/Controllers/PhotoController.php`. Контроллер вже міститиме методи для кожної з доступних ресурсних операцій.

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class);
```
Ця єдина декларація маршруту створює кілька маршрутів для виконання різноманітних дій на ресурсі. Згенерований контроллер уже матиме закріплені методи для кожної з цих дій. Пам’ятайте, що ви завжди можете отримати швидкий огляд маршрутів вашого додатку, виконавши команду `route:list` Artisan.

Ви навіть можете зареєструвати багато ресурсних контроллерів одночасно, передавши масив методу `resources`:

```php
Route::resources([
	'photos' => PhotoController::class,
	'posts' => PostController::class,
]);
```

<a name="actions-handled-by-resource-controller"></a>
#### Дії, які обробляє ресурсний контроллер

Verb      | URI                    | Action       | Route Name
----------|------------------------|--------------|---------------------
GET       | `/photos`              | index        | photos.index
GET       | `/photos/create`       | create       | photos.create
POST      | `/photos`              | store        | photos.store
GET       | `/photos/{photo}`      | show         | photos.show
GET       | `/photos/{photo}/edit` | edit         | photos.edit
PUT/PATCH | `/photos/{photo}`      | update       | photos.update
DELETE    | `/photos/{photo}`      | destroy      | photos.destroy

<a name="customizing-missing-model-behavior"></a>
#### Налаштування поведінки при відсутності моделі

Як правило, HTTP-відповідь 404 буде згенерована, якщо модель ресурсу з неявним прив’язуванням не знайдено. Однак ви можете налаштувати цю поведінку, викликавши `missing` метод під час визначення маршруту вашого ресурсу. Метод `missing` приймає замикання, яке буде викликано, якщо неявно пов’язану модель не можна знайти для жодного з маршрутів ресурсу:

```php
use App\Http\Controllers\PhotoController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redirect;

Route::resource('photos', PhotoController::class)
		->missing(function (Request $request) {
			return Redirect::route('photos.index');
		});
```

<a name="specifying-the-resource-model"></a>
#### Визначення моделі ресурсу

Якщо ви використовуєте [зв’язування моделі](routing.md#route-model-binding) з маршрутом та бажаєте, щоб методи контроллера ресурсів містили типізацію екземпляра моделі, ви можете використовувати параметр `--model` під час генерації контроллера:

```shell
php artisan make:controller PhotoController --model=Photo --resource
```

<a name="generating-form-requests"></a>
#### Генерація запитів форми

Ви можете вказати прапор `--requests` при створенні ресурсного контроллера, щоб вказати Artisan про попутне створення [form request classes](validation.md#form-request-validation) для методів store та update контроллера:


```shell
php artisan make:controller PhotoController --model=Photo --resource --requests
```

<a name="restful-partial-resource-routes"></a>
### Вибіркові ресурсні маршрути

При оголошенні маршрута ресурсу ви можете вказати підмножину дій, які має виконувати контроллер, замість повного набору дій за замовчуванням:

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class)->only([
	'index', 'show'
]);

Route::resource('photos', PhotoController::class)->except([
	'create', 'store', 'update', 'destroy'
]);
```

<a name="api-resource-routes"></a>
#### Ресурсні API-маршрути

Оголошуючи маршрути ресурсів, які використовуватимуться API, як правило, виникає необхідність виключити маршрути, які візуалізують шаблони HTML, такі як `create` та `edit`. Для зручності ви можете використовувати метод `apiResource` для автоматичного виключення цих двох маршрутів:

```php
use App\Http\Controllers\PhotoController;

Route::apiResource('photos', PhotoController::class);
```
Ви можете зареєструвати багато ресурсних контроллерів API одночасно, передавши масив методу `apiResources`:

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\PostController;

Route::apiResources([
	'photos' => PhotoController::class,
	'posts' => PostController::class,
]);
```
Щоб швидко створити ресурсний контроллер API, який вже не містить методів `create` або `edit`, використовуйте перемикач `--api` під час виконання команди `make:controller`:

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### Вкладені ресурси

Іноді може знадобитися визначити маршрути до вкладеного ресурсу. Наприклад, фоторесурс може мати кілька коментарів, які можна прикріпити до фото. Щоб вкласти ресурсні контроллери, ви можете використовувати «крапкову нотацію» в декларації маршруту:

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class);
```
Цей маршрут зареєструє вкладений ресурс, до якого можна отримати доступ за такими URI:

```php
    // /photos/{photo}/comments/{comment}
```

<a name="scoping-nested-resources"></a>
#### Обмеження вкладених ресурсів

Функціонал [неявного прив’язування моделі](routing.md#implicit-model-binding-scoping) Laravel може автоматично обмежувати вкладені прив'язки для підтвердження приналежності дочірньої моделі по відношенню до батьківської моделі. Використовуючи метод `scoped` при визначенні вашого вкладеного ресурсу, можна включити автоматичне обмеження, а також вказати Laravel, через яке поле дочірній ресурс повинен бути отриманий. Для отримання додаткових відомостей про те, як це зробити, дивіться документацію щодо [обмеження ресурсних маршрутів](#restful-scoping-resource-routes).


<a name="shallow-nesting"></a>
#### Спрощене вкладення

Часто немає необхідності мати в URI і батьківський, і дочірній ідентифікатори, оскільки дочірній ідентифікатор є унікальним ідентифікатором. При використанні унікальних ідентифікаторів, таких як автоінкрементні первинні ключі для ідентифікації ваших моделей в сегментах URI, ви можете використовувати «спрощене вкладення»:

```php
    use App\Http\Controllers\CommentController;

    Route::resource('photos.comments', CommentController::class)->shallow();
```

Це визначення маршруту визначить наступні маршрути:

Verb      | URI                               | Action       | Route Name
----------|-----------------------------------|--------------|---------------------
GET       | `/photos/{photo}/comments`        | index        | photos.comments.index
GET       | `/photos/{photo}/comments/create` | create       | photos.comments.create
POST      | `/photos/{photo}/comments`        | store        | photos.comments.store
GET       | `/comments/{comment}`             | show         | comments.show
GET       | `/comments/{comment}/edit`        | edit         | comments.edit
PUT/PATCH | `/comments/{comment}`             | update       | comments.update
DELETE    | `/comments/{comment}`             | destroy      | comments.destroy

<a name="restful-naming-resource-routes"></a>
### Перейменування ресурсних маршрутів

За замовчуванням усі дії ресурсного контроллера мають ім'я маршруту; однак, ви можете перевизначити ці імена, передавши масив імен із бажаними іменами маршрутів:

```php
    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->names([
        'create' => 'photos.build'
    ]);
```

<a name="restful-naming-resource-route-parameters"></a>
### Перейменування параметрів ресурсних маршрутів

За замовчуванням `Route::resource` створить параметри маршруту для ваших ресурсних маршрутів на основі "singularized" версії імені ресурсу. Ви можете легко перевизначити це для кожного ресурсу, використовуючи метод `parameters`. Масив, що передається в метод `parameters`, повинен бути асоціативним масивом імен ресурсів та імен параметрів:

```php
use App\Http\Controllers\AdminUserController;

Route::resource('users', AdminUserController::class)->parameters([
	'users' => 'admin_user'
]);
```
У наведеному вище прикладі створюється наступний URI для маршруту `show` ресурсу:

```php
	// /users/{admin_user}
```

<a name="restful-scoping-resource-routes"></a>
### Обмеження ресурсних маршрутів

Функціонал [обмеження неявної прив'язки моделі](routing.md#implicit-model-binding-scoping) може автоматично обмежувати вкладені прив'язки для підтвердження приналежності витягнутої дочірньої моделі по відношенню до батьківської моделі. Використовуючи метод `scoped` при визначенні вашого вкладеного ресурсу, ви можете увімкнути автоматичне обмеження, а також вказати Laravel, через яке поле дочірній ресурс має бути отриманий:

```php
use App\Http\Controllers\PhotoCommentController;

Route::resource('photos.comments', PhotoCommentController::class)->scoped([
	'comment' => 'slug',
]);
```
Цей маршрут зареєструє обмежений вкладений ресурс, до якого можна отримати доступ за допомогою таких URI, як:

```php
   // /photos/{photo}/comments/{comment:slug}
```
При використанні неявної прив'язки з ключем як параметр вкладеного маршруту, Laravel автоматично задає обмеження для отримання вкладеної моделі своїм батьком, використовуючи угоди, щоб вгадати ім'я відношення батьківського елемента. У цьому випадку передбачається, що модель `Photo` має відношення до імені `comments` (множина від імені параметра маршруту), яке можна використовувати для отримання моделі `Comment`.

<a name="restful-localizing-resource-uris"></a>
### Локалізація URI ресурсів

За замовчуванням `Route::resource` створює URI ресурсів з використанням англійських дієслів та мовних правил множини. Якщо вам потрібно локалізувати команди методів `create` і `edit`, ви можете використовувати метод `Route::resourceVerbs`. Це можна зробити на початку метода boot провайдера `App\Providers\RouteServiceProvider`:

```php
/**
 * Визначити зв'язування моделі та маршруту, фільтри шаблонів тощо.
 *
 * @return void
 */
public function boot()
{
	Route::resourceVerbs([
		'create' => 'crear',
		'edit' => 'editar',
	]);

	// ...
}
```
Побудовник слів у множині Laravel підтримує [кілька різних мов, які ви можете змінити в залежності від ваших потреб.](localization.md#pluralization-language). Після того, як дієслова були скориговані, а також змінені мовні правила множини, реєстрація маршруту ресурсу, наприклад Route::resource('publicacion', PublicacionController::class), створить наступні URI:

```php
publicacion/crear

/publicacion/{publicaciones}/editar

```

<a name="restful-supplementing-resource-controllers"></a>
### Додаткові маршрути ресурсних контролерів

Якщо вам потрібно додати додаткові маршрути ресурсного контроллера, крім набору ресурсних маршрутів за замовчуванням, ви повинні визначити ці маршрути перед викликом методу `Route::resource`; в іншому випадку маршрути, визначені методом `resource`, можуть ненавмисно мати пріоритет над вашими додатковими маршрутами:


```php
use App\Http\Controller\PhotoController;

Route::get('/photos/popular', [PhotoController::class, 'popular']);
Route::resource('photos', PhotoController::class);
```


> {tip} Пам'ятайте, що ваші контроллери мають бути зосередженими. Якщо вам постійно потрібні методи, що виходять за рамки типового набору дій з ресурсами, розгляньте можливість поділу вашого контроллера на два менші контроллери.

<a name="dependency-injection-and-controllers"></a>
## Впровадження залежностей та контролери

<a name="constructor-injection"></a>
#### Впровадження залежностей у конструкторі контроллера

[Контейнер служб](container.md) Laravel використовується для отримання всіх контроллерів. В результаті ви можете оголосити будь-які залежності, які можуть знадобитися вашому контроллеру у його конструкторі. Оголошені залежності будуть автоматично вилучені та впроваджені в екземпляр контроллера:

```php
<?php

namespace App\Http\Controllers;

use App\Repositories\UserRepository;

class UserController extends Controller
{
	/**
	 * Екземпляр сховища користувача.
	 */
	protected $users;

	/**
	 * Створіть новий екземпляр контроллера.
	 *
	 * @param  \App\Repositories\UserRepository  $users
	 * @return void
	 */
	public function __construct(UserRepository $users)
	{
		$this->users = $users;
	}
}
```

<a name="method-injection"></a>
#### Впровадження залежностей у методах контроллера

На додаток до ін’єкції конструктора, ви також можете вводити підказки залежно від методів вашого контроллера. Загальним випадком використання ін’єкції методу є ін’єкція екземпляра `Illuminate\Http\Request` у ваші методи контроллера:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class UserController extends Controller
{
	/**
	 * Збережіть нового користувача.
	 *
	 * @param  \Illuminate\Http\Request  $request
	 * @return \Illuminate\Http\Response
	 */
	public function store(Request $request)
	{
		$name = $request->name;

		//
	}
}
```
Якщо ваш метод контроллера також очікує вхідних даних від параметра маршруту, перелічіть аргументи маршруту після інших залежностей. Наприклад, якщо ваш маршрут визначено так:

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```
Ви все ще можете оголосити тип залежності `Illuminate\Http\Request` і отримати доступ до вашого параметра маршруту `id`, визначивши метод контроллера таким чином:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class UserController extends Controller
{
	/**
	 * Оновіть даного користувача.
	 *
	 * @param  \Illuminate\Http\Request  $request
	 * @param  string  $id
	 * @return \Illuminate\Http\Response
	 */
	public function update(Request $request, $id)
	{
		//
	}
}
```