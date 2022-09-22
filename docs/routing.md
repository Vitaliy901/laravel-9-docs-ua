# Маршрутизація

- [Основи маршрутизації](#basic-routing)
  - [Маршрути перенаправлення](#redirect-routes)
  - [Маршрути візуалізації](#view-routes)
  - [Список маршрутів](#the-route-list)
- [Параметри маршруту](#route-parameters)
  - [Обов'язкові параметри](#required-parameters)
  - [Необов'язкові параметри](#parameters-optional-parameters)
  - [Обмеження регулярного виразу](#parameters-regular-expression-constraints)
- [Іменовані маршрути](#named-routes)
- [Групи маршрутів](#route-groups)
  - [Посередники](#route-group-middleware)
  - [Контроллери](#route-group-controllers)
  - [Маршрутизація піддоменів](#route-group-subdomain-routing)
  - [Префікси URI згрупованих маршрутів](#route-group-prefixes)
  - [Префікси імен згрупованих маршрутів](#route-group-name-prefixes)
- [Прив'язка моделі до маршруту](#route-model-binding)
  - [Неявне прив'язування](#implicit-binding)
  - [Неявне прив'язування із типізованими перерахуваннями](#implicit-enum-binding)
  - [Явне зв'язування](#explicit-binding)
- [Запасний маршрут](#fallback-routes)
- [Обмеження частоти запитів](#rate-limiting)
  - [Визначення обмежувачів частоти запитів](#defining-rate-limiters)
  - [Прив'язування обмежувачів частоти запитів до маршрутів](#attaching-rate-limiters-to-routes)
- [Підміна методів форми](#form-method-spoofing)
- [Доступ до поточного маршруту](#accessing-the-current-route)
- [Спільне використання ресурсів між джерелами (CORS)](#cors)
- [Кешування маршрутів](#route-caching)

<a name="basic-routing"></a>

## Основи маршрутизації

Найпростіші маршрути Laravel приймають URI та функцію зворотного виклику, забезпечуючи дуже простий та виразний метод визначення маршрутів і поведінку без складних конфігураційних файлів маршрутизації:

```php
use Illuminate\Support\Facades\Route;

Route::get('/greeting', function () {
	return 'Hello World';
});

```

<a name="the-default-route-files"></a>

#### Файли маршрутів за замовчуванням

Усі маршрути Laravel визначені у ваших файлах маршрутів, які знаходяться в каталозі `routes`. Ці файли автоматично завантажуються `App\Providers\RouteServiceProvider` вашої програми. Файл `routes/web.php` визначає маршрути для вашого веб-інтерфейсу. Цим маршрутам призначається група middleware `web`, яка наділяє всі маршрути у цій групі такими особливосями, як стан сеансу та захист CSRF. Маршрути в `routes/api.php` не мають стану сеанса і їм присвоєно групу middleware `api`.

Для більшості програм ви почнете з визначення маршрутів у файлі `routes/web.php`. Доступ до маршрутів, визначених у `routes/web.php`, можна отримати, ввівши URL-адресу визначеного маршруту у вашому браузері. Наприклад, ви можете отримати доступ до наступного маршруту, перейшовши за адресою `http://example.com/user` у своєму браузері:

```php
use App\Http\Controllers\UserController;

Route::get('/user', [UserController::class, 'index']);
```

Маршрути, визначені у файлі `routes/api.php` занесені в групу маршрутів `RouteServiceProvider`. У цій групі префікс `/api` URI застосовується автоматично, тому вам не потрібно вручну застосовувати його до кожного маршруту у файлі. Ви можете змінити префікс та інші параметри групи маршрутів, змінивши свій клас `RouteServiceProvider`.

<a name="available-router-methods"></a>

#### Доступні методи маршрутизатора

Маршрутизатор дозволяє реєструвати маршрути, які відповідають будь-якому методу HTTP:

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```

Іноді вам може знадобитися зареєструвати маршрут, який відповідає на декілька методів HTTP. Ви можете зробити це за допомогою методу `match`. Або ви навіть можете зареєструвати маршрут, який відповідає на всі методи HTTP за допомогою методу `any`:

```php
Route::match(['get', 'post'], '/', function () {
	//
});

Route::any('/', function () {
	//
});
```

> **Note**  
> При визначенні кількох маршрутів, які мають один і той самий URI, маршрути, що використовують методи `get`, `post`, `put`, `patch`, `delete`, і `options`, повинні бути визначені перед маршрутами, які використовують методи `any`, `match`, і `redirect`. Це гарантує, що вхідний запит узгоджується з правильним маршрутом.

<a name="dependency-injection"></a>

#### Ін'єкція залежності

Ви можете вказати будь-які залежності, необхідні для вашого маршруту, у функції зворотного виклику вашого маршруту. Оголошені залежності будуть автоматично вилучені та додані до функції зворотного виклику [контейнером служби](container.md) Laravel. Наприклад, ви можете оголосити клас `Illuminate\Http\Request`, щоб поточний HTTP-запит автоматично додавався до функції зворотного виклику вашого маршрута:

```php
use Illuminate\Http\Request;

Route::get('/users', function (Request $request) {
	// ...
});
```

<a name="csrf-protection"></a>

#### Захист від CSRF

Пам'ятайте, що будь-які HTML-форми, які вказують на маршрути POST, PUT, PATCH або DELETE, яким призначена група middleware `web`, повинні включати поле токена CSRF. В іншому випадку запит буде відхилено. Ви можете прочитати більше про захист від CSRF у [документації CSRF](csrf.md): документації CSRF:

```blade
<form method="POST" action="/profile">
	@csrf
	...
</form>
```

<a name="redirect-routes"></a>

### Маршрути перенаправлення

Якщо ви визначаєте маршрут, який перенаправляє до іншого URI, ви можете використовувати метод `Route::redirect`. Цей метод забезпечує зручний ярлик, щоб вам не довелось визначати повний маршрут або контролер для виконання простого перенаправлення:

```php
Route::redirect('/here', '/there');
```

За замовчуванням `Route::redirect` повертає код статусу 302. Ви можете налаштувати код статуса за допомогою додаткового третього параметра:

```php
Route::redirect('/here', '/there', 301);
```

Або ви можете використати метод `Route::permanentRedirect`, щоб повернути код статусу `301`:

```php
Route::permanentRedirect('/here', '/there');
```

> **Warning**  
> Під час використання параметрів маршрута в маршрутах перенаправлення такі параметри зарезервовані Laravel і не можуть бути використані: `destination` та `status`.

<a name="view-routes"></a>

### Маршрути візуалізації

Якщо ваш маршрут потребує лише [візуалізацію](views.md), HTML-шаблона, ви можете використати метод `Route::view`. Подібно до методу `redirect`, цей метод забезпечує простий ярлик, тому вам не потрібно визначати повний маршрут або контролер. Метод візуалізації приймає URI як перший аргумент, а назву HTML-шаблону як другий аргумент. Крім того, ви можете надати масив даних для передачі в середину візуалізації як необов’язковий третій аргумент:

```php
Route::view('/welcome', 'welcome');

Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

> **Warning**  
> При використанні параметрів маршрута у маршрутах візуалізації, наступні параметри зарезервовані Laravel і не можуть бути використані: `data`, `status`, та `headers`.

<a name="the-route-list"></a>

### Список маршрутів

Команда `route:list` Aartisan може легко надати весь список маршрутів, визначених у вашому додку:

```shell
php artisan route:list
```

За замовчуванням middleware, який призначений кожному маршруту, не відображатиметься у вихідних даних `route:list`; однак ви можете вказати Laravel відображати middleware маршруту, додавши до команди параметр -v:

```shell
php artisan route:list -v
```

Ви також можете вказати Laravel показувати лише маршрути, які починаються з заданого URI:

```shell
php artisan route:list --path=url
```

Крім того, ви можете вказати Laravel приховати будь-які маршрути, визначені пакетами сторонніх розробників, забезпечивши параметр `--except-vendor` під час виконання команди `route:list`:

```shell
php artisan route:list --except-vendor
```

Так само ви можете вказати Laravel показувати лише маршрути, які визначені пакетами сторонніх розробників, за допомогою параметра `--only-vendor` під час виконання команди `route:list`:

```shell
php artisan route:list --only-vendor
```

<a name="route-parameters"></a>

## Параметри маршруту

<a name="required-parameters"></a>

### Обов'язкові параметри

Іноді вам потрібно буде захопити сегменти URI вашого маршруту. Наприклад, вам може знадобитися отримати ідентифікатор користувача з URL-адреси. Це можна зробити, визначивши параметри маршруту:

```php
Route::get('/user/{id}', function ($id) {
	return 'User '.$id;
});
```

Ви можете визначити скільки завгодно параметрів маршруту:

```php
Route::get('/posts/{post}/comments/{comment}', function ($postId, $commentId) {
	//
});
```

Параметри маршруту завжди містяться у фігурних дужках `{}` і мають складатися з літер алфавіту. Підкреслення (`_`) також прийнятні в назвах параметрів маршруту. Параметри маршрута передаються функції зворотного виклику маршруту / контролера в залежності від їх порядку - назви аргументів зворотного виклику маршруту / контролера не мають значення.

<a name="parameters-and-dependency-injection"></a>

#### Параметри та впровадження залежності

Якщо ваш маршрут має залежності, які ви хочете, щоб контейнер служби Laravel автоматично додавав до зворотного виклику вашого маршруту / контролера, ви повинні вказати параметри маршруту після своїх залежностей:

```php
use Illuminate\Http\Request;

Route::get('/user/{id}', function (Request $request, $id) {
	return 'User '.$id;
});
```

<a name="parameters-optional-parameters"></a>

### Необов'язкові параметри

Іноді вам може знадобитися вказати параметр маршруту, який не завжди може бути присутнім в URI. Ви можете зробити це, розмістивши `?` після назви параметра. Обов’язково визначте аргументу маршруту значення за замовчуванням:

```php
Route::get('/user/{name?}', function ($name = null) {
	return $name;
});

Route::get('/user/{name?}', function ($name = 'John') {
	return $name;
    });
```

<a name="parameters-regular-expression-constraints"></a>

### Обмеження регулярного виразу

Ви можете обмежити формат параметрів маршруту за допомогою методу `where` екземпляра маршруту. Метод `where` приймає назву параметра та регулярний вираз, який визначає обмеження для параметра:

```php
Route::get('/user/{name}', function ($name) {
	//
})->where('name', '[A-Za-z]+');

Route::get('/user/{id}', function ($id) {
	//
})->where('id', '[0-9]+');

Route::get('/user/{id}/{name}', function ($id, $name) {
	//
})->where(['id' => '[0-9]+', 'name' => '[a-z]+']);
```

Для зручності деякі типові шаблони регулярних виразів мають допоміжні методи, які дозволяють швидко додавати шаблонні обмеження до ваших маршрутів:

```php
Route::get('/user/{id}/{name}', function ($id, $name) {
	//
})->whereNumber('id')->whereAlpha('name');

Route::get('/user/{name}', function ($name) {
	//
})->whereAlphaNumeric('name');

Route::get('/user/{id}', function ($id) {
	//
})->whereUuid('id');

Route::get('/category/{category}', function ($category) {
	//
})->whereIn('category', ['movie', 'song', 'painting']);
```

Якщо вхідний запит не відповідає обмеженням шаблону маршруту, буде повернено відповідь HTTP 404.

<a name="parameters-global-constraints"></a>

#### Глобальні обмеження

Якщо ви бажаєте, щоб параметр маршруту завжди обмежувався заданим регулярним виразом, ви можете використати метод `pattern`. Вам слід визначити ці шаблони в методі `boot` класу `App\Providers\RouteServiceProvider`:

```php
/**
 * Define your route model bindings, pattern filters, etc.
 *
 * @return void
 */
public function boot()
{
	Route::pattern('id', '[0-9]+');
}
```

Після визначення шаблону він автоматично застосовується до всіх маршрутів, які використовують цю назву параметра:

```php
Route::get('/user/{id}', function ($id) {
	// Виконується, лише якщо {id} є числом...
});
```

<a name="parameters-encoded-forward-slashes"></a>

#### Визначення похилого слеша у параметрі маршруту

Компонент маршрутизації Laravel дозволяє всім символам, крім `/`, бути присутніми в значеннях параметрів маршруту. Ви повинні явно дозволити `/` бути частиною вашого заповнювача за допомогою регулярного виразу у методі `where`:

```php
Route::get('/search/{search}', function ($search) {
	return $search;
})->where('search', '.*');
```

> **Warning**  
> Визначення похилого слеша підтримується лише в останньому сегменті маршруту.

<a name="named-routes"></a>

## Іменовані маршрути

Іменовані маршрути дозволяють зручно генерувати URL-адреси або перенаправляти на інші маршрути. Ви можете вказати назву для маршруту, приєднавши метод `name` додатково до визначення маршруту:

```php
Route::get('/user/profile', function () {
	//
})->name('profile');
```

Ви також можете вказати назви маршрутів для методів контролера:

You may also specify route names for controller actions:

```php
Route::get(
	'/user/profile',
	[UserProfileController::class, 'show']
)->name('profile');
```

> **Warning**  
> Назви маршрутів завжди мають бути унікальними.

<a name="generating-urls-to-named-routes"></a>

#### Створення URL-адрес для іменованих маршрутів

Після того, як ви призначили назву даному маршруту, ви можете використовувати назву маршруту під час генерації URL-адрес або перенаправлень за допомогою допоміжних функцій `route` та `redirect` Laravel:

```php
// Генерація URL-адрес...
$url = route('profile');

// Створення перенаправлень...
return redirect()->route('profile');
```

Якщо названий маршрут має параметри, ви можете передати їх другим аргументом `route` функції. Наведені параметри будуть автоматично вставлені у згенеровану URL-адресу у їх відповідних позиціях:

```php
Route::get('/user/{id}/profile', function ($id) {
	//
})->name('profile');

$url = route('profile', ['id' => 1]);
```

Якщо ви передасте додаткові параметри в масиві, ці пари ключ/значення буде автоматично додано до згенерованого рядка запиту URL-адреси у форматі GET параметрів:

```php
Route::get('/user/{id}/profile', function ($id) {
	//
})->name('profile');

$url = route('profile', ['id' => 1, 'photos' => 'yes']);

// /user/1/profile?photos=yes
```

```php
use Illuminate\Support\Facades\URL;

Route::get('/user/{id}/profile', function ($id) {
	//
})->name('profile');

$url = route('profile', URL::defaults(['id' => 1]));

// /user/1/profile
```

> **Note**  
> Іноді вам може знадобитися вказати значення за замовчуванням для параметрів URL-адреси на рівні запиту, як-от поточна мова. Щоб досягти цього, ви можете використати метод [`URL::defaults`](urls.md#default-values).

<a name="inspecting-the-current-route"></a>

#### Огляд поточного маршруту

Якщо ви бажаєте визначити, чи поточний запит було направлено саме на заданий іменований маршрут, ви можете використати метод `named` для екземпляра `Route`. Наприклад, ви можете перевірити поточну назву маршруту із middleware маршрута:

```php
/**
 * Обробляти вхідний запит.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  \Closure  $next
 * @return mixed
 */
public function handle($request, Closure $next)
{
	if ($request->route()->named('profile')) {
		//
	}

	return $next($request);
}
```

<a name="route-groups"></a>

## Групи маршрутів

Групи маршрутів дозволяють надавати спільний доступ до атрибутів маршруту (наприклад, middleware) для великої кількості маршрутів без потреби визначати ці атрибути для кожного маршруту окремо.

Вкладені групи намагаються розумно об'єднати атрибути зі своєю батьківською групою. Middleware і `where`, умови об’єднуються, а імена та префікси додаються. Розділювачі простору імен і `/` в префіксах URI автоматично додаються, де це необхідно.

<a name="route-group-middleware"></a>

### Middleware

Щоб призначити [middleware](middleware.md) всім маршрутам у групі, можна використовувати метод `middleware` перед визначенням групи. Посередники будуть виконуватися в порядку, в якому вони перераховані в масиві:

```php
Route::middleware(['first', 'second'])->group(function () {
	Route::get('/', function () {
		// Використовує middleware `first` і `second`...
	});

	Route::get('/user/profile', function () {
		// Використовує middleware `first` і `second`...
	});
});
```

<a name="route-group-controllers"></a>

### Controllers

Якщо група маршрутів використовує один [controller](controllers.md), ви можете використовувати метод `controller`, щоб визначити загальний контролер для всіх маршрутів у групі. Тоді, визначаючи маршрути, вам потрібно лише надати метод контролера, який вони викликають:

```php
use App\Http\Controllers\OrderController;

Route::controller(OrderController::class)->group(function () {
	Route::get('/orders/{id}', 'show');
	Route::post('/orders', 'store');
});
```

<a name="route-group-subdomain-routing"></a>

### Маршрутизація субдоменів

Групи маршрутів також можна використовувати для обробки субдоменної маршрутизації. Субдоменам можуть бути призначені параметри маршруту так само, як і URI маршруту, що дозволяє захопити частину субдомену для використання у вашому маршруті або контролері. Піддомен можна вказати викликом методу `domain` перед визначенням групи:

```php
Route::domain('{account}.example.com')->group(function () {
	Route::get('user/{id}', function ($account, $id) {
		//
	});
});
```

> **Warning**  
> Щоб забезпечити доступність маршрутів піддоменів, необхідно зареєструвати маршрути піддоменів перед реєстрацією маршрутів корінного домену. Це запобігає перезапису маршрутами корінного домену маршрутів піддоменів, які мають однаковий шлях URI.

<a name="route-group-prefixes"></a>

### Груповання маршрутів за URI префіксом

Метод `prefix` можна використовувати для визначення префікса кожному маршруту в групі заданим URI. Наприклад, ви можете додати до всіх URI маршрутів у групі префікс `admin`:

```php
Route::prefix('admin')->group(function () {
	Route::get('/users', function () {
		// Відповідає URL-адресі "/admin/users".
	});
});
```

<a name="route-group-name-prefixes"></a>

### Префікси імен маршрутів

Метод `name` може бути використаний для префікса кожному імені маршруту в групі з заданим рядком. Наприклад, ви можете додати до всіх імен згрупованих маршрутів префікс `name`. Визначений рядок має префікс до імені маршруту точно так, як це зазначено, тому ми обов’язково надамо завершальний символ `.` у префіксі:

```php
Route::name('admin.')->group(function () {
	Route::get('/users', function () {
		// Маршрут присвоєно ім'ям "admin.users"...
	})->name('users');
});
```

<a name="route-model-binding"></a>

## Прив'язка моделі до маршруту

При інєкції ID моделі в маршрут або метод контролера ви часто будете звертатися до бази даних, щоб отримати модель з відповідним ID. Зв’язування моделі з маршрутом Laravel забезпечує зручний спосіб автоматичного введення екземплярів моделі безпосередньо у ваші маршрути. Наприклад, замість впровадження ID користувача, ви можете впровадити весь екземпляр моделі `User` з відповідним ID.

<a name="implicit-binding"></a>

### Неявне зв'язування

Laravel автоматично розпізнає моделі Eloquent, визначені в маршрутах або діях контролера, чиї імена змінних оголошеного типу відповідають назві сегмента маршруту. Наприклад:

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
	return $user->email;
});
```

Оскільки змінна `$user` зазначена як модель `App\Models\User` Eloquent, а назва змінної відповідає сегменту URI `{user}`, Laravel автоматично введе екземпляр моделі, ID якої відповідає значенню з URI запита. Якщо відповідний екземпляр моделі не знайдено в базі даних, автоматично буде згенеровано HTTP-відповідь 404.

Звичайно, неявне зв'язування також можливе при використанні методів контролера. Знову зверніть увагу на те, що сегмент URI `{user}` відповідає змінній `$user` у контроллері, який містить типізацію `App\Models\User`:

```php
use App\Http\Controllers\UserController;
use App\Models\User;

// Визначення маршруту...
Route::get('/users/{user}', [UserController::class, 'show']);

// Визначення методу контролера...
public function show(User $user)
{
	return view('user.profile', ['user' => $user]);
}
```

<a name="implicit-soft-deleted-models"></a>

#### Програмне видалення моделей

Як правило, неявне прив’язування моделі не відновлює моделі, які було [програмно видалено](eloquent.md#soft-deleting). Однак ви можете вказати неявному зв’язуванню отримати ці моделі, приєднавши метод `withTrashed` до визначення вашого маршруту:

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
	return $user->email;
})->withTrashed();
```

<a name="customizing-the-key"></a>
<a name="customizing-the-default-key-name"></a>

#### Налаштування ключа

За бажанням ви можете вилучати модель Eloquent за допомогою стовпця, який відрізняється від `id`. Для цього ви можете вказати стовпець у визначенні параметра маршруту:

```php
use App\Models\Post;

Route::get('/posts/{post:slug}', function (Post $post) {
	return $post;
});
```

Якщо ви бажаєте, щоб зв’язування моделі завжди використовувало стовпець бази даних, який відрізняється від `id` за замовчуванням, під час отримання даного класу моделі, ви можете замінити метод `getRouteKeyName` у відповідній моделі Eloquent:

```php
/**
 * Отримання ключа маршруту для моделі.
 *
 * @return string
 */
public function getRouteKeyName()
{
	return 'slug';
}
```

<a name="implicit-model-binding-scoping"></a>

#### Змінення ключа та обмеження неявної прив'язки моделі

При неявному прив’язуванні кількох моделей Eloquent до одного визначення маршруту, можливо, ви забажаєте обмежити другу моделі Eloquent таким чином, щоб вона була дочірньою моделлю попередньої моделі Eloquent. Наприклад, розглянемо це визначення маршруту, який отримує пост блогу за допомогою slug для конкретного користувача:

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
	return $post;
});
```

У разі використання спеціального неявного зв’язування з ключем як вкладеного параметра маршруту, Laravel автоматично створить обмеження запиту для отримання вкладеної моделі своїм батьком за допомогою угод, щоб вгадати ім’я зв’язку батьківської моделі. У цьому випадку буде припущено, що модель `User` має відношення під назвою `posts` (форма множини імені параметра маршруту), яке можна використовувати для отримання моделі `Post`.

За бажанням можна вказати Laravel обмежити дочірні прив'язки, навіть якщо ключ не був наданий. Для цього ви можете викликати метод `scopeBindings` при визначенні вашого маршруту:

```php
use App\Models\Post;
use App\Models\User;

Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
	return $post;
})->scopeBindings();
```

Або ви можете вказати у визначенні групи маршрутів використовувати обмеження прив'язки:

```php
Route::scopeBindings()->group(function () {
	Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
		return $post;
	});
});
```

<a name="customizing-missing-model-behavior"></a>

#### Налаштування поведінки відсутньої моделі

Як правило, HTTP-відповідь 404 буде згенерована, якщо неявно пов’язана модель не знайдена. Однак ви можете налаштувати цю поведінку, викликавши `missing` метод під час визначення маршруту. `missing` метод приймає замикання, яке буде викликано, якщо неявно пов’язану модель не можна знайти:

```php
use App\Http\Controllers\LocationsController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redirect;

Route::get('/locations/{location:slug}', [LocationsController::class, 'show'])
		->name('locations.view')
		->missing(function (Request $request) {
			return Redirect::route('locations.index');
		});
```

<a name="implicit-enum-binding"></a>

### Неявне зв'язування із типізованим перерахуванням

PHP 8.1 запровадив підтримку [типізованих перерахувань](https://www.php.net/manual/en/language.enumerations.backed.php). Щоб доповнити цю функцію, Laravel дозволяє вказувати типізацію для підкріплених типізованих перерахувань у вашому визначенні маршруту, і Laravel викличе маршрут, лише якщо цей сегмент маршруту відповідає дійсному значенню перерахувань.
В іншому випадку відповідь HTTP 404 буде повернено автоматично. Наприклад, враховуючи такий перерахунок:

```php
<?php

namespace App\Enums;

enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}
```

Ви можете визначити маршрут, який буде викликатись лише в тому випадку, якщо сегмент маршруту `{category}` має значення `fruits` або `people`. В іншому випадку Laravel поверне 404-а відповідь HTTP:

```php
use App\Enums\Category;
use Illuminate\Support\Facades\Route;

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```

<a name="explicit-binding"></a>

### Явне зв'язування

Ви не зобов’язані використовувати неявне прив'язування моделі на основі конвенцій Laravel, щоб використовувати зв’язування моделі. Ви також можете явно визначити, як параметри маршруту відповідають моделям. Щоб зареєструвати явне зв’язування, використовуйте метод `model` маршрутизатора, щоб вказати клас для заданого параметра. Вам слід визначити явне зв'язування моделі на початку методу `boot` вашого класу `RouteServiceProvider`:

```php
use App\Models\User;
use Illuminate\Support\Facades\Route;

/**
 * Визначити зв'язування моделі та маршруту, шаблони обмежень регулярних виразів тощо.
 *
 * @return void
 */
public function boot()
{
	Route::model('user', User::class);

	// ...
}
```

Потім визначте маршрут, який містить параметр `{user}`:

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
	//
});
```

Оскільки ми пов'язали всі параметри `{user}` з моделлю `App\Models\User`, екземпляр цього класу буде впроваджено у маршрут. Наприклад, при запиті `users/1` буде впроваджено екземпляр `User` з бази даних, який має ID `1`.

Якщо відповідний екземпляр моделі не знайдено у базі даних, то автоматично буде згенеровано 404 HTTP-відповідь.

<a name="customizing-the-resolution-logic"></a>

#### Зміна логіки зв'язування

Якщо ви бажаєте визначити власну логіку зв’язування моделі, ви можете використати метод `Route::bind`. Замикання, яке ви передаєте методу `bind`, отримає значення сегмента URI та має повернути екземпляр класу, який слід додати до маршруту. Знову ж таки, це налаштування має відбутися в методі `boot` `RouteServiceProvider` вашого дадатка:

```php
use App\Models\User;
use Illuminate\Support\Facades\Route;

/**
 * Визначити зв'язування моделі та маршруту, шаблони обмежень регулярних виразів тощо.
 *
 * @return void
 */
public function boot()
{
	Route::bind('user', function ($value) {
		return User::where('name', $value)->firstOrFail();
	});

	// ...
}
```

Крім того, ви можете перевизначити метод `resolveRouteBinding` у своїй моделі Eloquent. Цей метод отримає значення сегмента URI і має повернути екземпляр класу, який слід додати до маршруту:

```php
/**
 * Отримати дочірню модель для зв’язаного значення.
 *
 * @param  mixed  $value
 * @param  string|null  $field
 * @return \Illuminate\Database\Eloquent\Model|null
 */
public function resolveRouteBinding($value, $field = null)
{
	return $this->where('name', $value)->firstOrFail();
}
```

Якщо у маршруті використовується [обмеження неявної прив'язки моделі](#implicit-model-binding-scoping), то для отримання пов'язаної дочірньої моделі буде використовуватися метод `resolveChildRouteBinding` батьківської моделі:

```php
/**
 * Отримати дочірню модель для зв’язаного значення.
 *
 * @param  string  $childType
 * @param  mixed  $value
 * @param  string|null  $field
 * @return \Illuminate\Database\Eloquent\Model|null
 */
public function resolveChildRouteBinding($childType, $value, $field)
{
	return parent::resolveChildRouteBinding($childType, $value, $field);
}
```

<a name="fallback-routes"></a>

## Запасний маршрут

Використовуючи метод `Route::fallback`, ви можете визначити маршрут, який буде виконано, якщо жоден інший маршрут не відповідає вхідному запиту. Як правило, необроблені запити автоматично відтворюють сторінку «404» через обробник винятків вашої програми. Однак, оскільки ви зазвичай визначаєте `fallback` маршрут у своєму файлі `routes/web.php`, всі посередники в групі `web` посередників застосовуватимуться до маршруту. За потреби ви можете додати до цього маршруту додаткове посередники:

```php
    Route::fallback(function () {
        //
    });
```

> {note} Запасним маршрутом завжди має бути останнім маршрутом, зареєстрований у вашому додатку.

<a name="rate-limiting"></a>

## Обмеження частоти запитів

<a name="defining-rate-limiters"></a>

### Визначення обмежувачів частоти запитів

Laravel містить потужні та керовані служби обмеження частоти запитів, які можна використовувати для обмеження обсягу трафіку для конкретного маршруту або групи маршрутів. Для початку ви повинні визначити конфігурації обмежувача, які відповідають потребам вашого дадатка. Як правило, це має виконуватися в методі `configureRateLimiting` постачальника `App\Providers\RouteServiceProvider` вашого додатка.

Обмежувачі визначаються за допомогою метода `for` фасада `RateLimiter`. Метод `for` приймає ім'я обмежувача та замикання, яке повертає конфігурацію обмежень, які застосовуються до призначених маршрутів через посередник `throttle:yourRateLimit`. Конфігурація обмежень - це екземпляри класа `Illuminate\Cache\RateLimiting\Limit`. Цей клас містить корисні методи "побудови", щоб ви могли швидко визначити свій "ліміт". Ім'я обмежувача може бути будь-яким рядком за вашим бажанням:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Настроить ограничители частоты запросов для приложения.
 *
 * @return void
 */
protected function configureRateLimiting()
{
	RateLimiter::for('global', function (Request $request) {
		return Limit::perMinute(1000);
	});
}
```

Якщо вхідний запит перевищує зазначений ліміт, Laravel автоматично поверне відповідь з 429 кодом стану HTTP. Якщо ви хочете визначити свою власну відповідь, яка має повертатися, то ви можете використовувати метод `response`:

```php
RateLimiter::for('global', function (Request $request) {
	return Limit::perMinute(1000)->response(function (Request $request, array $headers) {
		return response('Custom response...', 429, $headers);
	});
});
```

Оскільки функція зворотного виклику обмежувача запитів отримують екземпляр вхідного HTTP-запиту, ви можете створити відповідне обмеження запиту динамічно на основі вхідного запиту або автентифікованого користувача:

```php
RateLimiter::for('uploads', function (Request $request) {
	return $request->user()->vipCustomer()
		? Limit::none()
		: Limit::perMinute(100);
});
```

<a name="segmenting-rate-limits"></a>

#### Сегментація обмежень частоти запитів

Іноді потрібна сегментація обмежень за деякими довільними значеннями. Наприклад, ви можете дозволити користувачам отримувати доступ до зазначеного маршруту 100 разів на хвилину на кожну IP-адресу. Для цього можна використовувати метод `by` при побудові ліміту:

```php
RateLimiter::for('uploads', function (Request $request) {
	return $request->user()->vipCustomer()
		? Limit::none()
		: Limit::perMinute(100)->by($request->ip());
});
```

Щоб проілюструвати цю функцію на іншому прикладі, ми можемо обмежити доступ до вказаного маршруту до 100 разів на хвилину за ідентифікатором автентифікованого користувача або до 10 разів на хвилину за IP-адресою для кожного з гостей:

```php
RateLimiter::for('uploads', function (Request $request) {
	return $request->user()
		? Limit::perMinute(100)->by($request->user()->id)
		: Limit::perMinute(10)->by($request->ip());
});
```

<a name="multiple-rate-limits"></a>

#### Численні обмеження частоти запитів

За потреби ви можете повернути масив обмежень при визначенні конфігурації обмежувача. Кожне обмеження оцінюватиметься для маршруту в залежності від порядку, в якому вони розміщені у масиві:

```php
RateLimiter::for('login', function (Request $request) {
	return [
		Limit::perMinute(500),
		Limit::perMinute(3)->by($request->input('email')),
	];
});
```

<a name="attaching-rate-limiters-to-routes"></a>

### Прив'язка обмежувачів частоти запитів до маршрутів

Обмежувачі можуть бути закріплені за маршрутами або групами маршрутів за допомогою [посередника](middleware.md) `throttle`. Посередник `throttle` приймає ім'я обмежувача, який ви хочете призначити маршруту:

```php
Route::middleware(['throttle:uploads'])->group(function () {
	Route::post('/audio', function () {
		//
	});

	Route::post('/video', function () {
		//
	});
});
```

<a name="throttling-with-redis"></a>

#### Використання Redis для посередника `throttle`

Як правило, посередник `throttle` підставлений до класу `Illuminate\Routing\Middleware\ThrottleRequests`. Це підставлення визначено в (`App\Http\Kernel`) вашого додатка. Якщо ви використовуєте Redis як драйвер кеша вашого додатка, то ви можете змінити це порівняння, щоб використовувати клас `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis`. Цей клас ефективніший при керуванні обмеженнями запитів за допомогою Redis:

```php
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
```

<a name="form-method-spoofing"></a>

## Підміна методів форми

HTML-форми не підтримують дії `PUT`, `PATCH`, або `DELETE`. Таким чином, при визначенні маршрутів `PUT`, `PATCH`, або `DELETE`, які викликаються з HTML-форми, вам потрібно буде додати до форми приховане поле `_method`. Значення, відправлене з полем `_method`, використовуватиметься як метод HTTP-запиту:

```php
<form action="/example" method="POST">
	<input type="hidden" name="_method" value="PUT">
	<input type="hidden" name="_token" value="{{ csrf_token() }}">
</form>
```

Для зручності ви можете використовувати директиву шаблонизатора [Blade](blade.md) `@method` для створення поля введення `_method`:

```blade
<form action="/example" method="POST">
	@method('PUT')
	@csrf
</form>
```

<a name="accessing-the-current-route"></a>

## Доступ до поточного маршруту

Ви можете використовувати методи `current`, `currentRouteName`, і `currentRouteAction` фасаду `Route` для доступу до інформації про маршрут, що обробляє вхідний запит:

```php
    use Illuminate\Support\Facades\Route;

    $route = Route::current(); // Illuminate\Routing\Route
    $name = Route::currentRouteName(); // string
    $action = Route::currentRouteAction(); // string
```

Ви можете звернутися до документації API [основного класу фасаду Route](https://laravel.com/api/8.x/Illuminate/Routing/Router.html), і [екземпляра Route](https://laravel.com/api/8.x/Illuminate/Routing/Route.html), щоб переглянути всі методи, які доступні маршрутизаторі та класах маршрутів.

<a name="cors"></a>

## Спільне використання ресурсів між джерелами (CORS)

Laravel може автоматично відповідати на HTTP-запити CORS `OPTIONS` зі значеннями, які ви налаштували. Усі налаштування CORS можна налаштувати у конфігураційному файлі додатка `config/cors.php`. Запити `OPTIONS` автоматично оброблятимуться [посередником](middleware) `HandleCors`, яке за замовчуванням включено у ваш глобальний стек посередника. Ваш глобальний стек посередника розташований у HTTP-ядрі вашого додатка (`App\Http\Kernel`).

> **Note**  
> Для отримання додаткової інформації про CORS і заголовки CORS зверніться до [веб-документації MDN щодо CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#The_HTTP_response_headers).

<a name="route-caching"></a>

## Кешування маршруту

Коли ваш додаток розгортається на `prodation`, вам слід скористатися перевагами кешу маршрутів Laravel. Використання кешу маршрутів суттєво зменшить кількість часу, необхідного для реєстрації всіх маршрутів вашого додатку. Щоб створити кеш маршруту, виконайте команду `route:cache` Artisan:

```shell
php artisan route:cache
```

Після виконання цієї команди ваш кешований файл маршрутів завантажуватиметься під час кожного запиту. Пам’ятайте, якщо ви додаєте нові маршрути, вам потрібно буде створити новий кеш маршрутів. Через це вам слід запускати лише команду `route:cache` під час розгортання проекту.

Ви можете скористатися командою `route:clear`, щоб очистити кеш маршруту:

```shell
php artisan route:clear
```
