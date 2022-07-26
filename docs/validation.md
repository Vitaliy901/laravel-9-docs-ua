# Laravel 9 · Валідація

- [Вступ](#introduction)
- [Швидкий старт](#validation-quickstart)
    - [Визначення маршрутів](#quick-defining-the-routes)
    - [Створення контроллера](#quick-creating-the-controller)
    - [Написання логіки валідації](#quick-writing-the-validation-logic)
    - [Відображення помилок валідації](#quick-displaying-the-validation-errors)
    - [Повторне заповнення форми](#repopulating-forms)
    - [Примітка не обов'язкових полів](#a-note-on-optional-fields)
    - [Формат відповіді валідаційних помилок](#validation-error-response-format)
- [Валідація запита форми](#form-request-validation)
    - [Створення запита форми](#creating-form-requests)
    - [Авторизація запитів](#authorizing-form-requests)
    - [Корегування повідомлень про помилки](#customizing-the-error-messages)
    - [Підготовка вхідних данних для валідації](#preparing-input-for-validation)
- [Створення валідатора на вимогу](#manually-creating-validators)
    - [Автоматичне перенаправлення](#automatic-redirection)
    - [Іменування колекції помилок](#named-error-bags)
    - [Корегування повідомлень про помилки](#manual-customizing-the-error-messages)
    - [Хук валидатора After](#after-validation-hook)
- [Робота з провалідованими вхідними даними](#working-with-validated-input)
- [Робота з повідомленнями про помилки](#working-with-error-messages)
    - [Визначення користувацьких повідомлень в мовних файлах](#specifying-custom-messages-in-language-files)
    - [Визначення атрибутів в мовних файлах](#specifying-attribute-in-language-files)
    - [Визначення користувацьких імен для атрибутів в мовних файлах](#specifying-values-in-language-files)
- [Доступні правила валідації](#available-validation-rules)
- [Умовне додавання правил](#conditionally-adding-rules)
- [Валідація масивів](#validating-arrays)
    - [Валідація вкладених масивів](#validating-nested-array-input)
    - [Індекси та позиції повідомлень про помилки](#error-message-indexes-and-positions)
- [Валідація паролей](#validating-passwords)
- [Користувацькі правила валідації](#custom-validation-rules)
    - [Використання класа Rule](#using-rule-objects)
    - [Використання замикання](#using-closures)
    - [Неявні правила](#implicit-rules)

<a name="introduction"></a>
## Вступ

Laravel пропонує декілька підходів для перевірки вхідних даних вашого додатка. Наприклад, метод `validate`, доступний для всіх вхідних HTTP-запитів. Однак ми обговоримо й інші підходи до валідації.
Laravel містить корисні правила валідації, які застосовуються до даних, включаючи валідацію на унікальність значення в конкретній таблиці бази даних. Ми детально розглянемо кожне з цих правил валідації, щоб ви були ознайомленні з усіма особливостями валідації в Laravel.

<a name="validation-quickstart"></a>
## Швидкий старт

Щоб дізнатися про потужні функції валідації Laravel, давайте розглянемо повний приклад валідації форми і відображень повідомлень про помилки кінцевому користувачеві. Прочитавши цей загальний огляд, ви зможите отримати уявлення про те, як перевіряти дані вхідного запита за допомогою Laravel:

<a name="quick-defining-the-routes"></a>
### Визначення маршрутів

По-перше, припустимо, що в нашому файлі `routes/web.php` визначені наступні маршрути:
```php
use App\Http\Controllers\PostController;

Route::get('/post/create', [PostController::class, 'create']);
Route::post('/post', [PostController::class, 'store']);
```
Маршрут `GET` покаже користувачеві форму для створення нового повідомлення в блозі, а маршрут `POST` збереже нове повідомлення в базі даних.

<a name="quick-creating-the-controller"></a>
### Створення контроллера

Тепер, давайте розглянемо звичайний контроллер, який обробляє вхідні запити на ці маршрути. Поки що залишимо метод `store` порожнім:
```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;

class PostController extends Controller
{
	/** 
	 * Показати форму для створення нового повідомлення в блозі
	 *
	 * @return \Illuminate\View\View
	 */
	public function create()
	{
		return view('post.create');
	}

	/** 
	 * Зберегти новий запис в блозі
	 *
	 * @param  \Illuminate\Http\Request  $request
	 * @return \Illuminate\Http\Response
	 */
	public function store(Request $request)
	{	
		// Виконати валідацію та зберегти повідомлення в блозі ...
	}
}
```
<a name="quick-writing-the-validation-logic"></a>
### Написання логіки валідації

Тепер ми готові заповнити наш метод `store` логікою для валідації нового повідомлення в блозі. Для цього ми будемо вживати базовий метод `validate`, який надається об'єктом `Illuminate\Http\Request`. Якщо правила валідації будуть пройдені, то ваш код продовжить нормальне виконання; однак, якщо перевірка не пройдена, то буде викинуто виняток `Illuminate\Validation\ValidationException`, і автоматично буде додана відповідна відповідь про помилку до змінної `$errors` яка є екземпляром `Illuminate\Support\MessageBag`, яку можно показати відправивши назад користувачеві через шаблон.

Також якщо валідація не пройдена під час традиційного HTTP-запита, то буде сгенеровано відповідь-перенаправлення на попередню URL-адресу. Якщо вхідний запит є XMLHttpRequest-запитом, то буде повернуто [JSON-відповідь, яка містить повідомлення про помилки валідації](#validation-error-response-format).

Щоб краще зрозуміти метод `validate`, давате повернемося до метода `store`:
```php
/**
 * Зберегти новий запис в блозі.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Http\Response
 */
public function store(Request $request)
{
	$validatedData = $request->validate([
		'title' => 'required|unique:posts|max:255',
		'body' => 'required',
	]);

	// Запис блога корректний ...
}
```
Як бачите, правила валідації передаються в метод `validate`. Не хвилюйтесь - всі доступні правила валідації [задокументовані](#available-validation-rules). Знов-таки, якщо перевірка не пройдена, то буде автоматично згенеровано корректну відповідь. Якщо перевірку пройдено успішно, то наш контроллер продовжить нормальну роботу, а метод `validate` поверне масив провалідованих даних.

В якості альтернативи правила валідації можуть бути вказані як масиви правил замість одного рядка з дільником `|`:
```php
$validatedData = $request->validate([
	'title' => ['required', 'unique:posts', 'max:255'],
	'body' => ['required'],
]);
```
Крім того, ви можете використовувати метод `validateWithBag` для валідації запита зі збереженням будь-яких повідомлень про помилки у [іменовану колекцію помилок](#named-error-bags), вказавши першим аргументом назву цій колекції:
```php
$validatedData = $request->validateWithBag('post', [
	'title' => ['required', 'unique:posts', 'max:255'],
	'body' => ['required'],
]);
```
<a name="stopping-on-first-validation-failure"></a>
#### Припинення валідації при виникненні першої помилки

За бажанням можна припинити виконання правил валідації для атрибута (атрибутом називається ім'я поля форми) після першої помилки. Для цього потрібно встановити атрибуту правило `bail`:
```php
$request->validate([
	'title' => 'bail|required|unique:posts|max:255',
	'body' => 'required',
]);
```
В цьому прикладі, якщо правило `unique` для атрибута `title` не буде пройдено, то наступне правило `max` не виконається. Правила перевірятимуться по порядку їх призначення.

<a name="a-note-on-nested-attributes"></a>
#### Примітка про вкладені атрибути

Якщо вхідний HTTP-запит містить дані «вкладених» полей `<input type="user" name="user[name]">`, то ви можете вказати ці поля в своїх правилах валідації, використовуючи «крапкову нотацію»:
```php
$request->validate([
	'title' => 'required|unique:posts|max:255',
	'author.name' => 'required',
	'author.description' => 'required',
]);
```
З іншого боку, якщо ім'я вашого поля буквально містить крапку, то ви можете явно заборонити її інтерпритацію, як частину «крапкової нотації», екранувавши крапку за допомогою зворотної косої риски `\` <span style="color:red">(чомусь не працює!).</span>:
```php
$request->validate([
	'title' => 'required|unique:posts|max:255',
	'v1\.0' => 'required',
]);
```
<a name="quick-displaying-the-validation-errors"></a>
### Відображення помилок валідації

А що, якщо поля вхідних запитів не проходять зазначені правила валідації? Як згадувалось раніше, Laravel автоматично пренаправить користувача назад у вихідне положення. Крім того, всі помилки валідації разом з [вхідними даними запита](requests.md#retrieving-old-input) будуть автоматично [записані в сессію](session.md#flash-data).

Змінна `$errors` використовується у всіх шаблонах вашого додатка завдяки посереднику `Illuminate\View\Middleware\ShareErrorsFromSession`, який занесений до групи посередників `web` у класі `Kernel.php`. Поки застосовується цей посередник, у ваших шаблонах завжи буде доступна така змінна як `$errors`, яка дозволить вам припускати, що змінна `$errors` завжи визначена і може безпечно використовуватися. Змінна `$errors` як зазначалося вище є екземпляром `Illuminate\Support\MessageBag`. Для отримання додаткової інформації по роботі з цим об'єктом [ознайомтесь з його документацією](#working-with-error-messages).

Отже, в нашому прикладі користувач буде перенаправлений на метод нашого контроллера `create`, у випадко, якщо валідація завершиться невдало, що дозволяє нам відобразити повідомлення про помилку в шаблоні:
```blade
<!-- /resources/views/post/create.blade.php -->

<h1>Створення поста в блозі</h1>

@if ($errors->any())
<div class="alert alert-danger">
	<ul>
		@foreach ($errors->all() as $error)
			<li>{{ $error }}</li>
		@endforeach
	</ul>
</div>
@endif

<!-- Форма для створення поста в блозі -->
```
<a name="quick-customizing-the-error-messages"></a>
#### Корегування повідомлень про помилки

Кожне вмонтоване правило валідації Laravel містить повідомлення про помилку, яке знаходиться у файлі `lang/en/validation.php` вашого додатка. В цьому файлі ви знайдете записи з перекладом для кожного правила вілідації. Також ви можете зміюват або модифікувати ці повідомлення в залежності від потреб вашого додатка.

Крім того, ви можете скопіювати цей файл до каталогу перекладу іншої мови, щоб перекласти повідомлення на мову вашого додатка. Щоб дізнатися більше про локалізацію Laravel, ознайомтеся з повною [документацією про локалізацію](localization.md).

<a name="quick-xhr-requests-and-validation"></a>
#### XHR-запити і валідація

У наведених вище прикладах ми використовували форму для відправки даних в додаток. Однак, багато додатків отримують запити XHR з фронтенду з використанням JavaScript. При використанні метода `validate`, під час виконання XHR-запита, Laravel не буде генерувати відповідь-перенаправлення. Замість цього Laravel генерує [JSON-відповідь, яка міститиме всі помилки валідації](#validation-error-response-format). Ця JSON відповідь буде відправлена з кодом 422 HTTP статуса.

<a name="the-at-error-directive"></a>
#### Директива `@error`

Ви можете використовувати директиву [`@error` Blade](blade.md#validation-errors), щоб швидко виявити, чи існують повідомлення про помилки валідації для окремо взятого атрибута, включаючи повідомлення про помилки в іменованій колекції помилок. В директиві `@error` ви можите вивести вміст змінної `$message` для відображення повідомлення про помилку:
```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input id="title"
type="text"
name="title"
class="@error('title') is-invalid @enderror">

@error('title')
<div class="alert alert-danger">{{ $message }}</div>
@enderror
```
Якщо ви використовуєте [іменовані колекції помилок](#named-error-bags), то ви забов'язані передати ім'я колекції в якості другого аргумента директиви `@error`:
```blade
<input ... class="@error('title', 'post') {{ $message }} @enderror">
```
<a name="repopulating-forms"></a>
### Повторне заповнення полей форми

Коли Laravel генерує відповідь-перенаправлення через помилку валідації, фреймворк автоматично [одноразово записує всі віходні дані запита до сессії](session.md#flash-data). Це зроблено для того, щоб ви мали змогу зручно отримати доступ до вхідних даних під час наступного запита і повторно заповнити поля форми, яку користувачь намагався відправити.

Щоб отримати вхідні дані попереднього запита, викличте метод `old` екземпляра `Illuminate\Http\Request`. Метод `old` дістане попередньо записані вхідні данні з [сессії](session.md):
```php
// поверне масив з усіма даними попередньо отриманого запита
$title = $request->old();
// поверне вміст окремо взятого поля форми
$title = $request->old('title');
```
Laravel також містить гломального помічника `old`. Якщо вам потрібно показати вхідні дані минулого запита в [шаблоні Blade](blade.md), то варто використовувати моічник `old` для повторного заповнення полів форми. Якщо для якогось поля не були надані дані у минулому запиті, то буде повернуто `null`:
```blade
<input type="text" name="title" value="{{ old('title') }}">
```
<a name="a-note-on-optional-fields"></a>
### Примітка про необов'язкові поля

По дефолту Laravel містить посередників `App\Http\Middleware\TrimStrings` і `App\Http\Middleware\ConvertEmptyStringsToNull`
в глобальному стеку посередників вашого додатка. Ці посередники перераховані в класі `App\Http\Kernel`. Перший буде автоматично обрізати пробіли всіх вхідних рядкових полів запита на початку та в кінці рядка, а другий - конвертувати будь-які порожні рядкові поля в `null`. Через це вам необхідно буде позначати ваші «необов'язкові» поля запита як `nullable`, якщо ви не хочите, щоб валідатор не вважав такі поля недійсними. Наприклад:
```php
$request->validate([
	'title' => 'required|unique:posts|max:255',
	'body' => 'required',
	'publish_at' => 'nullable|date',
]);
```
В цьому прикладі ми вказали, що поле `publish_at` може бути `null` або допустимим поданням дати. Якщо модифікатор `nullable` не додано до визначення правил, валідатор вважатиме `null` неприпустимою датою.

<a name="validation-error-response-format"></a>
### Формат відповіді помилок валідації

Коли ваш додаток викидає виняток `Illuminate\Validation\ValidationException`, і на вхідний HTTP-запит очікується відповідь JSON, то Laravel автоматично відформатує для вас повідомлення про помилки і поверне HTTP-відповідь `422 Unprocessable Entity`.

Нижче ви можите переглянути приклад формата ответа JSON для помилок валідації. зверніть увагу, що вкладені ключі помилок зведені до «крапкової нотації»:
```json
{
"message": "The team name must be a string. (and 4 more errors)",
"errors": {
	"team_name": [
		"The team name must be a string.",
		"The team name must be at least 1 characters."
	],
	"authorization.role": [
		"The selected authorization.role is invalid."
	],
	"users.0.email": [
		"The users.0.email field is required."
	],
	"users.2.email": [
		"The users.2.email must be a valid email address."
	]
}
}
```
<a name="form-request-validation"></a>
## Валідація запита форми (FormRequest)

<a name="creating-form-requests"></a>
### Створення запитів фоми

Для більш складних сценаріїв валідації ви можете створити «form request». Form request - це ваш клас запита, який інкапсулює свою власну логіку валідації та авторизації. Щоб сгенерувати новий запит форми, використовуйте команду `make:request` [Artisan](artisan.md):

```shell
php artisan make:request StorePostRequest
```
Ця команда розмістить новий клас запита форми в каталог `app/Http/Requests` вашого додатка. Якщо такий каталог відсутній у вашому додатку, то laravel попередньо створить його, після того як ви запустите команду `make:request`. Кожний запит форми,створений Laravel, матиме два методи: `authorize` і `rules`.

Як ви могли здогадатися, метод `authorize` відповідає за визначення того, чи може поточний аутентифікований користувач виконувати дію, представлену запитом, а метод `rules` повертає правила валідації, які повинні застосуватися до даних запита.
```php
/**
 * Отримати масив правил валідації, які будуть накладатися на запит
 *
 * @return array
 */
public function rules()
{
	return [
		'title' => 'required|unique:posts|max:255',
		'body' => 'required',
	];
}
```

> {tip} Ви можите оголосити будь-які залежності, які вам потрібні, в сигнатурі метода `rules`. Вони будуть автоматично отримані за допомогою [контейнера служб](container.md) Laravel.

Отже як аналізуються правила валідації? Все, що вам потрібно зробити - це оголосити залежність від створеного класа «form request» у методі вашого контроллера. Вхідний запит форми перевірятиметься до виклику метода контроллера - це означає, що вам не доведеться переповнювати ваш контроллер будь-якою логікою валідації. Метод `safe` доступний лише для класа «form request»:
```php
/**
 * Зберегти новий запис в блозі
 *
 * @param  \App\Http\Requests\StorePostRequest  $request
 * @return Illuminate\Http\Response
 */
public function store(StorePostRequest $request)
{
	// Вхідний запит пройшов валідацію ...

	// Отримати провалідовані вхідні дані ...
	$validated = $request->validated(): array;
	// Отримати вміст окремо взятого провалідованого поля ...
	$validated = $request->validated('email');
	// Отримати частину підтверджених вхідних даних...
	$validated = $request->safe()->only(['name', 'email']);
	$validated = $request->safe()->except(['name', 'email']);
}
```
При невдалій валідації також буде сгенеровано відповідь-перенаправлення, щоб повернути користувача в його вихідне положення. Помилки також будуть одноразово додані до сессії, щоб вони були доступні для відображення. Якщо запит був XHR, то користувачеві буде повернуто HTTP-відповідь з кодом статуса 422, включаючи [JSON-подання помилок валідації](#validation-error-response-format).

<a name="adding-after-hooks-to-form-requests"></a>
#### Додавання гачків `after` для запитів форми

Якщо ви бажаєте додати гачок валідації для запита форми, то ви можете використовувати метод `withValidator`. Цей метод отримує повністю ініційований валідатор, що дозволяє вам викликати будь-який з його методів до того як, правила валідації будуть фактично проаналазовані:
```php
/**
 * Надстройка экземпляра валидатора.
 *
 * @param  \Illuminate\Validation\Validator  $validator
 * @return void
 */
public function withValidator($validator)
{
	$validator->after(function ($validator) {
		if ($this->somethingElseIsInvalid()) {
			$validator->errors()->add('field', 'Щось не так з цим полем!');
		}
	});
}
```
<a name="request-stopping-on-first-validation-rule-failure"></a>

#### Припинення валідації після першої невдалої перевірки

Додавши властивість `$stopOnFirstFailure` до вашого класу запитів, ви можете повідомити валідатору, що він повинен припинити валідацію всіх атрибутів після виникнення першої помилки валідації:
```php
/**
 * Припинення валідації після першої невдалої перевірки.
 *
 * @var bool
 */
protected $stopOnFirstFailure = true;
```
<a name="customizing-the-redirect-location"></a>

#### Зміна адреси відповіді-перенаправлення

Як обговорювалось раніше, буде згенерована відповідь-перенаправлення, щоб повернути користувача в його вихідне положення, якщо валідація запита форми невдала. Однак, ви можите налаштувати цю поведінку. Для цього визначте властивість `$redirect` у вашому запиті форми:
```php
/**
 * URI перенаправлення користувачів у випадку невдалої валідації.
 *
 * @var string
 */
protected $redirect = '/dashboard';
```
Або, якщо ви бажаєте перенаправити користувачів на іемнований маршрут, то вам потрібно замість `$redirect` визначити властивість `$redirectRoute`:
```php
/**
 * Маршрут перенаправлення користувачів у випадку невдалої валідації.
 *
 * @var string
 */
protected $redirectRoute = 'dashboard';
```
<a name="authorizing-form-requests"></a>

### Авторизація запитів

Клас запита форми також містить метод `authorize`. В рамках цього метода ви можете визначити, чи дійсно аутентифікований користувач має право на зміну поточного ресурса. Наприклад, ви можете визначити, чи дійсно користувач володіє коментарем у блозі, який він намагається оновити. Швидше за все, ви взаємодіятимете зі своїми [шлюзами та політиками авторизації](authorization.md) за допомогою цього методу:

```php
use App\Models\Comment;

/**
 * Визначте, чи має користувач право робити цей запит.
 *
 * @return bool
 */
public function authorize()
{
	$comment = Comment::find($this->route('comment'));

	return $comment && $this->user()->can('update', $comment);
}
```
Оскільки всі запити форми розширюють базовий клас запитів Laravel, ми можемо використовувати метод `user` для доступу до поточного аутентифікованого користувача. Також зверніть увагу на виклик метода `route` в наведеному вище прикладі. Цей метод надає вам доступ до параметрів URL, які визначені для викликаємого маршрута, такі як `{comment}` у наведеному нижче прикладі:

```php
Route::post('/comment/{comment}');
```
Таким чином, якщо ваша програма використовує переваги, [зв’язування моделі маршруту](routing.md#route-model-binding), то ваш код може бути ще більш стислим за допомогою доступу до витягнутої моделі як властивості класа запиту форми:

```php
return $this->user()->can('update', $this->comment);
```
Якщо метод `authorize` повертає `false`, то буде автоматично повернуто HTTP-відповідь з кодом статуса 403, і метод вашего контроллера не виконається.

Якщо ви плануєте обробляти логіку авторизації для запита в іншій частині вашого додатка, то ви можете просто повернути `true` з метода `authorize`:

```php
/**
 * Визначте, чи має користувач право робити цей запит.
 *
 * @return bool
 */
public function authorize()
{
	return true;
}
```

> {tip} Ви можете оголосити будь-які залежності які вам потрібні в сигнатурі методу `authorize`. Вони будуть автоматично витягнуті з [контейнера служб](container.md) Laravel.

<a name="customizing-the-error-messages"></a>
### Корегування повідомлень про помилки

Ви можете змінювати повідомлення про помилки використовуючи запити форм, перевизначивши метод `messages`. Цей метод повинен повертати масив пар атрибут / правило і відповідні їм повідомлення про помилки. Якщо ключем буде лише правило, то повідомлення про помилку буде росповсюджуватися тільки на правило. Можна визначити пару атрибут / правило через «крапкову нотацію» тоді, повідомлення про помилку буде втановлено для конкретного атрибута з правилом:
```php
/**
 * Отримати повідомлення про помилку для визначених правил перевірки.
 *
 * @return array
 */
public function messages()
{
	return [
		'title.required' => 'A title is required',
		'title.min' => 'A title min lenght is ...',
		'body.required' => 'A message is required',
	];
}
```
<a name="customizing-the-validation-attributes"></a>

#### Коригування атрибутів валідації

Багато повідомлень про помилку вмонтованого правила валідації Laravel містять заповнювач :attribute. Якщо ви бажаєте, щоб заповнювач `:attribute` вашого повідомлення про помилку валідації був замінений іншим іменем атрибута, то ви можите вказати власні імена перевизначивши метод `attributes`. Цей метод повинен повертати масив пар атрибут (ім'я інпута) / нове ім'я:
```php
/**
 * Отримайти власні атрибути для помилок валідатора.
 *
 * @return array
 */
public function attributes()
{
	return [
		'email' => 'email address',
	];
}
```
<a name="preparing-input-for-validation"></a>
### Підготовка вхідних данних для валідації

Якщо вам необхідно підготувати або обробити будь-які дані із запиту перш ніж застосувати правила валідації, то вам варто використати метод `prepareForValidation`:

```php
use Illuminate\Support\Str;

/**
 * Підготовка даних для валідації.
 *
 * @return void
 */
protected function prepareForValidation()
{
	$this->merge([
		'slug' => Str::slug($this->slug),
	]);
}
```

<a name="manually-creating-validators"></a>
## Створення валідатора на замовлення

Якщо ви не бажаєте використовувати метод `validate` запита, ви можете створити екземпляр валідатора вручну, використовуючи [фасад](facades.md) `Validator`. Метод `make` фасада генерує новий екземпляр валідатора:
```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class PostController extends Controller
{
	/**
	 * Зберегти новий запис в блозі.
	 *
	 * @param  Request  $request
	 * @return Response
	 */
	public function store(Request $request)
	{
		$validator = Validator::make($request->all(), [
			'title' => 'required|unique:posts|max:255',
			'body' => 'required',
		]);

		if ($validator->fails()) {
			return redirect('post/create')
						->withErrors($validator)
						->withInput();
		}

		// Отримати масив з провалідованими вхідними даними ...
		$validated = $validator->validated();

		// Отримати вміст окремо взятого провалідованого поля ...
		$validated = $request->validated('email');

		// Зберегти повідомлення в блозі ...
	}
}
```
Перший аргумент, який передається методу `make`, — це дані, які перевіряються. Другий аргумент - це масив правил валідації, які слід застосовувати до даних.

Після визначення того, що запит не пройшов валідацію за допомогою метода `fails`, ви можете здійснити перенаправленя користувача у вихідне положення, використавши глобального помічника `redirect`, а за допомогою метода `withErrors` можна одноразово занести повідомлення про помилки в сесію. При використанні цього метода змінна `$errors` буде автоматично передана до ваших шаблонів після перенаправлення, що дозволить вам з легкістю відобразити їх користувачеві. Метод `withErrors` приймає екземпляр валідатора, екземпляр `MessageBag` або звичайний масив PHP. Метод `withInput` одноразово заносить дані полів форми до сесії.

#### Припинення валідації пілся першої невдалої перевірки

Методо `stopOnFirstFailure` проінформує валідотор про те, що він має зупинити валідацію всіх атрибутів після виникнення першої помилки валідації.
```php
if ($validator->stopOnFirstFailure()->fails()) {
	// ...
}
```
<a name="automatic-redirection"></a>
### Автоматичне перенаправлення

Якщо ви бажаєте створити екземпляр валідатора в ручну, але як і раніше скористатися перевагами автоматичного перенаправлення з занесенням помилок та даних полів форми, пропонованого методом `validate` HTTP-запита, ви можете викликати метод `validate` створеного екземпляра валідатора. Якщо валідація пройдена успішно, то метод `validate` вашого екземпляра валідатора поверне масив провалідованих даних. Користувач буде автоматично перенаправлений або у випадку запита XHR, [буде повернуто відповідь JSON](#validation-error-response-format), якщо валідація невдала:
```php
$validatedData = Validator::make($request->all(), [
	'title' => 'required|unique:posts|max:255',
	'body' => 'required',
])->validate();
```
Ви також можете використовувати метод `validateWithBag` для збереження повідомлень про помилки до [іменованої колекції помилок](#named-error-bags), якщо валідація буде невдалою:
```php
$validatedData = Validator::make($request->all(), [
	'title' => 'required|unique:posts|max:255',
	'body' => 'required',
])->validateWithBag('post');
```
<a name="named-error-bags"></a>
### Іменовані колекції помилок

Якщо ви маєте декілька форм на одній сторінці, то вам варто встановити ім'я екземпляру `MessageBag`, який містить помилки валідації, що дозволить вам отримати повідомлення про помилки для окремо взятої форми. Щоб досягти цього, передайте ім'я у якості другого аргумента метода `withErrors`:
```php
return redirect('register')->withErrors($validator, 'login');
```
Потім, ви можете отримати доступ до іменованого екземпляра MessageBag зі змінної $errors через властивість:

```blade
{{ $errors->login->first('email') }}
```

<a name="manual-customizing-the-error-messages"></a>
### Коригування повідомлень про помилки

За необхідністю ви також маєте можливість визначати власні повідомлення про помилки, які повинен використовувати екземпляр валідатора замість повідомлень про помилки за замовчуванням, наданими Laravel. Є декілька способів вказати власні повідомлення. По-перше, ви можете передати власні повідомлення в якості третього аргумента метода `Validator::make`:
```php
$validator = Validator::make($input, $rules, $messages = [
	'required' => 'The :attribute field is required.',
]);
```
В цьому прикладі заповнювач `:attribute` буде замінений фактичним ім'ям поля, яке перевіряється. Ви також можете використовувати інші заповнювачі у повідомленнях валідатора. Наприклад
```php
$messages = [
	'same' => 'The :attribute and :other must match.',
	'size' => 'The :attribute must be exactly :size.',
	'between' => 'The :attribute value :input is not between :min - :max.',
	'in' => 'The :attribute must be one of the following types: :values',
];
```
<a name="specifying-a-custom-message-for-a-given-attribute"></a>

#### Визначення користувацьких повідомленнь для заданого атрибута
За бажанням можна визначити власне повідомлення про помилки тільки для окремого атрибута. Ви можете це зробити використовуючи «крапкову нотацію». Спочатку вкажіть ім'я атрибута, а потім правило:
```php
$messages = [
	'email.required' => 'We need to know your email address!',
];
```
<a name="specifying-custom-attribute-values"></a>

#### Визначення користувацьких імен для атрибутів

Багато повідомлень про помилку вмонтованого правила валідації Laravel містять заповнювач :attribute, який замінюється іменем перевіряємого поля або атрибута. Щоб визначити власні значення, які використовуються для заміни цих заповнювачів для конкретних полів, ви можете передати масив ваших атрибутів у якості четвертого аргумента методу `Validator::make`:
```php
$validator = Validator::make($input, $rules, $messages, [
	'email' => 'email address',
]);
```
<a name="after-validation-hook"></a>
### Гачок валідатора After

Ви також маєте можливість визначити замикання, які будуть запускатися після завершення валідації. Це дозволяє легко виконувати подальшу валідацію і навіть додавати повідомлення про помилки до колекції помилок. Викличте метод `after` екземпляра валідатора:
```php
$validator = Validator::make(/* ... */);

$validator->after(function ($validator) {
	if ($this->somethingElseIsInvalid()) {
		$validator->errors()->add(
			'field', 'Something is wrong with this field!'
		);
	}
});

if ($validator->fails()) {
	//
}
```
<a name="working-with-validated-input"></a>
## Робота з провалідованими вхідними даними

Після валідації даних вхідного запиту за допомогою запита форми (FormRequest), ви можете отримати дані вхідного запита, які дійсно пройшли валідацію. Це робиться кількома способами. По перше, ви можете викликати метод `validated` запита форми. Цей метод поверне масив даних, які були провалідовані:

```php
$validated = $request->validated(): array;
```

У якості альтернативи ви моете викликати метод `safe` запита форми. Цей метод повертає екземпляр `Illuminate\Support\ValidatedInput`. Цей об'єкт надає методи `only`, `except` та `all` для отримання вибіркових даних або цілого масива провалідованих даних:

```php
$validated = $request->safe()->only(['name', 'email']): array;

$validated = $request->safe()->except(['name', 'email']): array;

$validated = $request->safe()->all(): array;
```

Крім того, екземпляр `Illuminate\Support\ValidatedInput` може бути ітерованим та доступним, як масив:

```php
// Провалідовані дані можуть бути ітеровані ...
foreach ($request->safe() as $key => $value) {
	//
}
// До провалідованих даних можна отримати доступ, як до звичайного масива ...
$validated = $request->safe();

$email = $validated['email'];
```

Якщо є необхідність додати додаткові поля до провалідованих даних, то ви можете викликати додатковий метод `merge`:

```php
$validated = $request->safe()->merge(['name' => 'Taylor Otwell']);
```

Якщо ви хочите отримати провалідовані дані, як екземпляр [колекції](collections.md), то ви можите викликати метод `collect`:

```php
$collection = $request->safe()->collect();
```

<a name="working-with-error-messages"></a>
## Робота з повідомленнями про помилки

Після виклику метода `errors` екземпляра `Validator`, ви отримаєте екземпляр `Illuminate\Support\MessageBag`, який містить безліч зручних методів для роботи з повідомленнями про помилки. Змінна `$errors`, яка автоматично стає доступною для всіх шаблонів, також є екземпляром класу `MessageBag`.

<a name="retrieving-the-first-error-message-for-a-field"></a>
#### Отримання одного повідомлення про помилку для поля

Щоб отримати одне повідомлення про помилку для вказаного поля, використовуйте метод `first`:

```php
$errors = $validator->errors();

echo $errors->first('email'): string;
echo $errors->first('user.name'): string;
```

<a name="retrieving-all-error-messages-for-a-field"></a>
#### Отримання всіх повідомлень про помилки для поля

Якщо вам потрібно отримати масив всіх повідомлень для певного поля, використайте метод `get`:

```php
foreach ($errors->get('email') as $message) {
	//
}
```

Якщо ви перевіряєте масив полей форми, то ви можете отримати всі повідомлення для кожного з елементів масива, використавши символ `*`:

```php
foreach ($errors->get('attachments.*') as $message) {
	//
}
```

<a name="retrieving-all-error-messages-for-all-fields"></a>
#### Отримання всіх повідомлень про помилки для всіх полів

Щоб отримати масив всіх повідомлень для всіх полів, використовутей метод `all`:

```php
foreach ($errors->all() as $message) {
	//
}
```

<a name="determining-if-messages-exist-for-a-field"></a>

#### Визначення наявності повідомлень для поля

Метод `has` використовується для визначення наявності повідомлень про помилки для вказаного поля:

```php
if ($errors->has('email')) {
	//
}
```

<a name="specifying-custom-messages-in-language-files"></a>

### Визначення користувацьких повідомлень в мовних файлах валідації

Кожне вмонтоване правило валідації Laravel містить повідомлення про помилку, яке знаходиться у файлі `lang/en/validation.php` вашого додатка. В цьому файлі ви знайдете запис з перекладом для кожного правила валідації. Ви можете змінювати або модифікувати ці повідомлення в залежності від потреб вашого додатка.

Крім того, ви можете зробити копію цього файла і додати його до каталогу з іншою мовою, щоб перекласти повідомлення на іншу мову вашого додатка. Щоб дізнатися більше про локалізацію Laravel, ознайомтеся з повною [документацією про локалізацію](localization.md).

<a name="custom-messages-for-specific-attributes"></a>

#### Визначення користувацького повідомлення для певного атрибута

Ви можете налаштувати повідомлення про помилки, які використовуються для вказаних комбінацій атрибутів і правил у файлах мови перевірки вашого додатка. Для цього додайте повідомлення в масив `custom` мовного файла `lang/xx/validation.php` вашого додатка.

```php
'custom' => [
	'email' => [
		'required' => 'We need to know your email address!',
		'max' => 'Your email address is too long!'
	],
],
```

<a name="specifying-attribute-in-language-files"></a>

### Визначення атрибутів в мовних файлах валідації

Багато повідомлень про помилку вмонтованого правила валідації Laravel містять заповнювач :attribute, який замінюється іменем перевіряємого поля або атрибута. Якщо ви бажаєте, щоб частина `:attribute` вашого повідомлення валідації була змінена власним значенням, то вам варто визначити ім'я атрибута в масиві `attributes` вашого мовного файла `lang/xx/validation.php`:

```php
'attributes' => [
	'email' => 'email address',
],
```
<a name="specifying-values-in-language-files"></a>

### Визначення користувацьких імен для атрибутів в мовних файлах валідації

Деякі повідомлення про помилки вмонтованих правил валідації Laravel містять заповнювач `:value`, який замінюється поточним значенням атрибута запиту. Іноді потрібно замінити частину `:value` вашого повідомлення валідації на власне значення. Наприклад, розглянемо наступне правило, яке вказує, що номер кредитної карти вимагається обов'язково, якщо для параметра `payment_type` встановленно значення `cc`:
```php
Validator::make($request->all(), [
	'credit_card_number' => 'required_if:payment_type,cc'
]);
```
Якщо дане правило не пройде перевірки, то буде видано наступне повідомлення про помилку:

```none
The credit card number field is required when payment type is cc.
```
Замість того, щоб відобразити `cc` в якості значення типа платежа, ви можете вказати більш зручне представлення значення для користувача у вашому язиковому файлі`lang/xx/validation.php`, визначив масив `values`:
```php
'values' => [
	'payment_type' => [
		'cc' => 'credit card'
	],
],
```
Після визначення цього значення правило валідації видасть наступне повідомлення про помилку:

```none
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## Доступні правила валідації

Нижче наведено список всіх доступних правил валідації та опис їх функцій:

- [Accepted](#rule-accepted)
- [Accepted If](#rule-accepted-if)
- [Active URL](#rule-active-url)
- [After (Date)](#rule-after)
- [After Or Equal (Date)](#rule-after-or-equal)
- [Alpha](#rule-alpha)
- [Alpha Dash](#rule-alpha-dash)
- [Alpha Numeric](#rule-alpha-num)
- [Array](#rule-array)
- [Bail](#rule-bail)
- [Before (Date)](#rule-before)
- [Before Or Equal (Date)](#rule-before-or-equal)
- [Between](#rule-between)
- [Boolean](#rule-boolean)
- [Confirmed](#rule-confirmed)
- [Current Password](#rule-current-password)
- [Date](#rule-date)
- [Date Equals](#rule-date-equals)
- [Date Format](#rule-date-format)
- [Declined](#rule-declined)
- [Declined If](#rule-declined-if)
- [Different](#rule-different)
- [Digits](#rule-digits)
- [Digits Between](#rule-digits-between)
- [Dimensions (Image Files)](#rule-dimensions)
- [Distinct](#rule-distinct)
- [Email](#rule-email)
- [Ends With](#rule-ends-with)
- [Enum](#rule-enum)
- [Exclude](#rule-exclude)
- [Exclude If](#rule-exclude-if)
- [Exclude Unless](#rule-exclude-unless)
- [Exclude With](#rule-exclude-with)
- [Exclude Without](#rule-exclude-without)
- [Exists (Database)](#rule-exists)
- [File](#rule-file)
- [Filled](#rule-filled)
- [Greater Than](#rule-gt)
- [Greater Than Or Equal](#rule-gte)
- [Image (File)](#rule-image)
- [In](#rule-in)
- [In Array](#rule-in-array)
- [Integer](#rule-integer)
- [IP Address](#rule-ip)
- [JSON](#rule-json)
- [Less Than](#rule-lt)
- [Less Than Or Equal](#rule-lte)
- [MAC Address](#rule-mac)
- [Max](#rule-max)
- [MIME Types](#rule-mimetypes)
- [MIME Type By File Extension](#rule-mimes)
- [Min](#rule-min)
- [Multiple Of](#multiple-of)
- [Not In](#rule-not-in)
- [Not Regex](#rule-not-regex)
- [Nullable](#rule-nullable)
- [Numeric](#rule-numeric)
- [Password](#rule-password)
- [Present](#rule-present)
- [Prohibited](#rule-prohibited)
- [Prohibited If](#rule-prohibited-if)
- [Prohibited Unless](#rule-prohibited-unless)
- [Prohibits](#rule-prohibits)
- [Regex (regular expression)](#rule-regex)
- [Required](#rule-required)
- [Required If](#rule-required-if)
- [Required Unless](#rule-required-unless)
- [Required With](#rule-required-with)
- [Required With All](#rule-required-with-all)
- [Required Without](#rule-required-without)
- [Required Without All](#rule-required-without-all)
- [Required Array Keys](#rule-required-array-keys)
- [Same](#rule-same)
- [Size](#rule-size)
- [Sometimes](#validating-when-present)
- [Starts With](#rule-starts-with)
- [String](#rule-string)
- [Timezone](#rule-timezone)
- [Unique (Database)](#rule-unique)
- [URL](#rule-url)
- [UUID](#rule-uuid)

<!-- </div> -->

<a name="rule-accepted"></a>
#### accepted
Поле, яке перевіряється, повинно мати значення `"yes"`, `"on"`, `1` или `true`. Це корисно для підтвердження прийняття «Умов використання» або подібних полів.

<a name="rule-accepted-if"></a>
#### accepted_if:anotherfield,value,...

Поле, яке перевіряється, повинно мати значення `"yes"`, `"on"`, `1` или `true` якщо інше поле, яке перевіряється, дорівнює вказаному значенню. Це корисно для підтвердження прийняття «Умов використання» або подібних полів.

<a name="rule-active-url"></a>
#### active_url

Поле, яке перевіряється, повинно містити дійсний запис A або AAAA відповідно до функції `dns_get_record` PHP. Ім’я хоста зазначеного URL витягується за допомогою функції PHP parse_url перед передачею в dns_get_record.

<a name="rule-after"></a>
#### after:_date_

Поле, яке перевіряється, повинно мати значення після зазначеної дати. Дати будуть передані до функції `strtotime` PHP для перетворення в дийсний екземпляр `DateTime`:

```php
'start_date' => 'required|date|after:tomorrow'
```
Замість передачі рядка дати, яка буде проаналізована за допомогою `strtotime`, ви можете вказати інше поле для порівняння з датою.

```php
'finish_date' => 'required|date|after:start_date'
```
<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_


Поле, яке перевіряється, повинно мати значення після зазаначеної дати або дорівнювати їй. Для дотримання додатково інформації див. правило [after](#rule-after).

<a name="rule-alpha"></a>
#### alpha

Поле, яке перевіряється, повинно складатися лише з літер.

<a name="rule-alpha-dash"></a>
#### alpha_dash

Поле, яке перевіряється, може містити в собі символи чисел та літер, а також дефіси і підкреслення.

<a name="rule-alpha-num"></a>
#### alpha_num

Поле, яке перевіряється повинно складатися лише з літер та чисел.

<a name="rule-array"></a>
#### array

Поле, яке перевіряється повинно бути масивом PHP.

Коли для правила `array` надаються додаткові значення, кожний ключ у вхідному масиві повинен бути присутній у списку значень, які надані правилу. В наступному прикладі ключ `admin` у вхідному масиві буде недійсним тому, що він не присутній у списку значень, які надані правилу `array`. Виникне помилка з наступним повідомленням 'The user must be an array.':

```php
use Illuminate\Support\Facades\Validator;

$input = [
	'user' => [
		'name' => 'Taylor Otwell',
		'username' => 'taylorotwell',
		'admin' => true,
	],
];

Validator::make($input, [
	'user' => 'array:username,name,locale',
]);
```

Взагалі, ви завжди маєте вказувати ключі масива, які можуть бути присутні у вашому масиві.

<a name="rule-bail"></a>
#### bail

Зупунити подальше застосування правил валідації атрибута пілся першої невдалої перевірки.

Порівняно з правилом `bail`, яке зупиняє подальшу валідацію тільки окремо взятого поля, метод `stopOnFirstFailure` повідомить валідатору, що він повинен припинити подальшу валідацію всіх атрибутів при виникненні першої помилки:

```php
if ($validator->stopOnFirstFailure()->fails()) {
	// ...
}
```

<a name="rule-before"></a>
#### before:_date_

Поле, яке перевіряється, має бути значенням, що передує вказаній даті. Дати будуть надані функції PHP `strtotime` для перетворення в дійсний екземпляр `DateTime`. Крім того, як і в правилі [`after`](#rule-after), ім'я іншого поля, яке перевіряється, може бути вказане у якості значення `date`.

<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

Поле, що перевіряється, має бути значенням, що передує вказаній даті або дорівнювати їй. Дати будуть надані функції PHP `strtotime` для перетворення в дійсний екземпляр `DateTime`. Крім того, як і в правилі [`after`](#rule-after), ім'я іншого поля, яке перевіряється, може бути вказане у якості значення `date`.

<a name="rule-between"></a>
#### between:_min_,_max_

Розмір поля, що перевіряється, має бути між вказаним _min_ і _max_. Рядки, числа, масиви та файли обчислюються так само, як і правило [`size`](#rule-size)..

<a name="rule-boolean"></a>
#### boolean

Поле, яке перевіряється, має бути приведено до логічного значення. Поля, які прийматимуться, повинні мати значення: `true`, `false`, `1`, `0`, `"1"` та `"0"`.

<a name="rule-confirmed"></a>
#### confirmed

Поле, яке перевіряється, повинно бути однаковим з полем `{field}_confirmation`. Наприклад, якщо поле, яке перевіряється – `password`, то обов'язково у віхідних даних повинно бути присутне поле `password_confirmation`.

<a name="rule-current-password"></a>
#### current_password

Поле підтвердження має бути однаковим з паролем автентифікованого користувача. Також ви можете вказати [захист автентифікації](authentication.md), за допомогою першого параметра правила:

```php
'password' => 'current_password:api'
```

<a name="rule-date"></a>
#### date

Поле, яке перевіряється, має відповідати формату функції `strtotime` PHP, який вона приймає. Наприклад: 2021-07-09/09-07-2021 13:30:00

<a name="rule-date-equals"></a>
#### date_equals:_date_

Поле, яке перевіряється, повинно дорівнювати зазначеній даті. Дати будуть надані функції `strtotime` PHP для перетворення в дійсний екземпляр `DateTime`.

<a name="rule-date-format"></a>
#### date_format:_format_

Поле, яке перевіряється, повинно відповідати переданому _format_. При валідації поля слід використовувати `date` або `date_format`, але не обидва одразу. Це правило валідації підтримує всі формати, які підтримує клас [`DateTime`](https://www.php.net/manual/ru/class.datetime.php) PHP.

<a name="rule-declined"></a>
#### declined

Поле, яке перевіряється повинно містити значення `"no"`, `"off"`, `0` або `false`.

<a name="rule-declined-if"></a>
#### declined_if:anotherfield,value,...

Поле, яке перевіряється, повинно містити значення `"no"`, `"off"`, `0` або `false`, якщо інше поле, яке перевіряється, дорівнює зазначеному.

<a name="rule-different"></a>
#### different:_field_

Поле, що перевіряється, має відрізнятися від зазначеного _field_.

<a name="rule-digits"></a>
#### digits:_value_

Поле, яке перевіряється, має бути цілим і повинно мати зазначену довжину _value_.

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

Поле, яке перевіряється, має бути цілим і повинно мати довжину в зазначеному діапазоні _min_ и _max_.

<a name="rule-dimensions"></a>
#### dimensions

Файл, який перевіряється, має бути зображенням з зазначиними розмірами у параметрах:

```php
'avatar' => 'dimensions:min_width=100,min_height=200'
```

Доступні обмеження:_min\_width_, _max\_width_, _min\_height_, _max\_height_, _width_, _height_, _ratio_.

Обмеження _ratio_ має бути представлене як ширина, поділена на висоту. Це можна вказати дробом, наприклад 3/2, або числом з плаваючою точкою, наприклад 1,5:

```php
'avatar' => 'dimensions:ratio=3/2'
```
Оскільки це правило потребує декількох аргументів, ви можете використати метод `Rule::dimensions` для довільної побудови правила:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
	'avatar' => [
		'required',
		Rule::dimensions()->maxWidth(1000)->maxHeight(500)->ratio(3 / 2),
	],
]);
```

<a name="rule-distinct"></a>
#### distinct

Під час перевірки масивів поле, яке перевіряється, не повинно мати дубльованих значень:

```php
'foo.*.id' => 'distinct'

$validator = Validator::make([
	'first' => ['id'=> 1],
	'second' => ['id' => 1]
	], [
		'*.id' => 'distinct',
	]);
```

Distinct за замовчуванням використовує не суворе порівняння змінних. Щоб використовувати суворі порівняння, ви можете додати параметр strict до правила перевірки:


```php
'foo.*.id' => 'distinct:strict'
```
Ви можете додати ignore_case до аргументів правила перевірки, щоб правило ігнорувало відмінності у використанні великих літер:

```php
'foo.*.id' => 'distinct:ignore_case'
```

<a name="rule-email"></a>
#### email

Поле, яке перевіряється, повинно мати формат адреси електронної пошти. Це правило валідації використоює пакет [`egulias/email-validator`](https://github.com/egulias/EmailValidator) для перевірки адреси електронної пошти. За замовченням застосовується валідатор `RFCValidation`, але ви можете застосовувати інші стилі валідації:

```php
'email' => 'email:rfc,dns'
```
У наведеному вище прикладі будуть застосовані перевірки `RFCValidation` та `DNSCheckValidation`. Ось повний перелік стилів перевірки, які ви можете застосовувати:

<!-- <div class="content-list" markdown="1"> -->

- `rfc`: `RFCValidation`
- `strict`: `NoRFCWarningsValidation`
- `dns`: `DNSCheckValidation`
- `spoof`: `SpoofCheckValidation`
- `filter`: `FilterEmailValidation`

<!-- </div> -->

Валідатор `filter`, який використовує функцію `filter_var` PHP, постачається з Laravel і застосовувався за замовченням до Laravel версії 5.8.

> {note} Валідатори `dns` і `spoof` потребують розширення `intl` PHP.

<a name="rule-ends-with"></a>
#### ends_with:_foo_,_bar_,...

Поле, яке перевіряється, повинно закінчуватися на одне з вказаних значень.

<a name="rule-enum"></a>
#### enum

Правило `Enum` — це правило на основі класу, яке перевіряє, чи містить поле, що перевіряється, дійсне значення enum. Правило `Enum` приймає назву `Enum` як єдиний аргумент конструктора:

```php
use App\Enums\ServerStatus;
use Illuminate\Validation\Rules\Enum;

$request->validate([
	'status' => [new Enum(ServerStatus::class)],
]);
```

> {note} Перераховані типи доступні тільки в [PHP 8.1+](https://www.php.net/manual/ru/language.enumerations.php).

<a name="rule-exclude"></a>
#### exclude

Поле, яке перевіряється, буде виключене з даних запита, які повертаються методами `validate` і `validated`.

<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

Поле, яке перевіряється, буде виключене з даних запита, які повертаються методами `validate` і `validated`, якщо поле _anotherfield_ дорівнює _value_.

Якщо потрібна складна логіка умовного виключення, ви можете використати метод `Rule::excludeIf`. Цей метод приймає логічне значення або замикання. Якщо надати замикання, то воно має повертати `true` або `false`, щоб вказати, чи потрібно виключити поле, що перевіряється:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
	'role_id' => Rule::excludeIf($request->user()->is_admin),
]);

Validator::make($request->all(), [
	'role_id' => Rule::excludeIf(fn () => $request->user()->is_admin),
]);
```

<a name="rule-exclude-unless"></a>
#### exclude_unless:_anotherfield_,_value_

Поле, яке перевіряється, буде виключене з даних запита, які повертаються методами `validate` і `validated`, якщо поле _anotherfield_ не дорівнює _value_. Якщо _value_ дорівнюватиме `null` (тобто `exclude_unless:name,null`), то поле, яке перевіряється, буде виключено, якщо поле порівняння не має значення `null` або, якщо поле для порівняння взагалі відсутне.

<a name="rule-exclude-with"></a>
#### exclude_with:_anotherfield_

Поле, яке перевіряється, буде виключене з даних запита, які повертаються методами `validate` і `validated`, якщо буде присутне зазначене поле _anotherfield_.


<a name="rule-exclude-without"></a>
#### exclude_without:_anotherfield_

Поле, яке перевіряється, буде виключене з даних запита, які повертаються методами `validate` і `validated`, якщо буде відсутнє зазначене поле _anotherfield_.

<a name="rule-exists"></a>
#### exists:_table_,_column_

Поле, яке перевіряється, має бути присутнє у зазначеній таблиці і колонці.

<a name="basic-usage-of-exists-rule"></a>
#### Основи використання правила Exists

```php
'state' => 'exists:states'
```

Якщо параметр `column` не вказано, буде використано назву поля. Таким чино, у даному прикладі правило буде перевіряти, чи містить колонка `state` значення атрибута `state`, у таблиці з іменем `states`.

<a name="specifying-a-custom-column-name"></a>
#### Зазначення користувацьго імені колонки

Ви можете явно вказати ім’я стовпця бази даних, яке має використовуватися правилом перевірки, розмістивши його після імені таблиці:

```php
'state' => 'exists:states,abbreviation'
```
Іноді вам може знадобитися вказати конкретне підключення до бази даних, яке буде використовуватися для запиту `exists`. Ви можете зробити це, додавши назву з’єднання перед назвою таблиці:

```php
'email' => 'exists:connection.tableName,email'
```

Замість того, щоб вказувати назву таблиці напряму, ви можете вказати модель Eloquent, яка має використовуватися для визначення імені таблиці:

```php
'user_id' => 'exists:App\Models\User,id'
```

Якщо ви бажаєте використовувати власний запит, який виконується правилом валідації, то вам варто використовувати клас `Rule` для довільного визначення правила. В цьому прикладі ми також вкажемо правила валідації у вигляді масиву замість використання символа `|` для їх розмежування.

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
	'email' => [
		'required',
		Rule::exists('tableName')->where(function ($query) {
			return $query->where('account_id', 1);
		}),
	],
]);
```
Ви можете явно вказати ім'я колонки бази даних, яке повинно використовуватися правилом `exists`, сгенероване методом `Rule::exists`, вказавши им'як колонки у якості другого аргумента метода `exists`:

```php
'state' => Rule::exists('tableName', 'columnName'),
```

<a name="rule-file"></a>
#### file

Поле, яке перевіряється, повинно бути успішно завантаженим файлом на сервер.

<a name="rule-filled"></a>
#### filled

Поле, яке перевіряється, не повинно бути порожнім, якщо воно присутне.

<a name="rule-gt"></a>
#### gt:_field_

Поле, яке перевіряється, повинно містити значення більше заданого у _field_. Два поля повинні бути одного типу. Рядки, числа, масиви і файли оцінюються з використанням тих самих угод, що й правило [`size`](#rule-size).

<a name="rule-gte"></a>
#### gte:_field_

Поле, яке перевіряється, повинно містити значення більше заданого або бути рівним йому у _field_. Два поля повинні бути одного типу. Рядки, числа, масиви і файли оцінюються з використанням тих самих угод, що й правило [`size`](#rule-size).

<a name="rule-image"></a>
#### image

Файл, який перевіряється, повинен бути зображенням форматів (jpg, jpeg, png, bmp, gif, svg или webp).

<a name="rule-in"></a>
#### in:_foo_,_bar_,...

Поле, яке перевіряється, повинно містити одне значення з перелічених у списку значеннь. Оскільки це правило часто вимагає від вас розгортання масиву, метод Rule::in можна використовувати для довільної побудови правила:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
	'zones' => [
		'required',
		Rule::in(['first-zone', 'second-zone']),
	],
]);
```

Коли правило `in` комбінується з правилом `array`, в такому випадку, кожне значення у віходному масиві повинно бути присутнім у списку значень, які надані правилу `in`. У наступному прикладі код аеропорту `LAS` у вхідному масиві є не дійсним, оскільки він відсутній у списку аеропортів, наданому правилу `in`:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

$input = [
	'airports' => ['NYC', 'LAS'],
];

Validator::make($input, [
	'airports' => [
		'required',
		'array',
	],
	'airports.*' => Rule::in(['NYC', 'LIT']),
]);
```

<a name="rule-in-array"></a>
#### in_array:_anotherfield_.*

Значення поля, яке перевіряється, повинно існувати у значеннях _anotherfield_. Поле _anotherfield_ не обов'язково може бути масивом.

<a name="rule-integer"></a>
#### integer

Поле, яке перевіряється, повинно містити ціле число.

> {note} Це правило валідації не перевіряє те, що значення поля відноситься до типу `integer`, а виконує перевірку за правилами `FILTER_VALIDATE_INT` PHP. Якщо вам потрібно перевірити значення поля у якості числа, то використовуйте це правило у поєднанні з [правилом валідації `numeric`](#rule-numeric).

<a name="rule-ip"></a>
#### ip

Поле, яке перевіряється, повинно містити IP-адресу.

<a name="ipv4"></a>
#### ipv4

Поле, яке перевіряється, повинно містити адресу IPv4.

<a name="ipv6"></a>
#### ipv6

Поле, яке перевіряється, повинно містити адресу ipv6.

<a name="rule-json"></a>
#### json

Значення поля, яке перевіряється, повинно відповідати формату JSON.

<a name="rule-lt"></a>
#### lt:_field_

Значення поля, яке перевіряється, повинно мати меньшу кількість символів ніж у _field_. Два поля повинні бути одного типу. Рядки, числа, масиви і файли оцінюються з використанням тих самих угод, що й правило [`size`](#rule-size).

<a name="rule-lte"></a>
#### lte:_field_

Значення поля, яке перевіряється, повинно мати меньшу кількість символів або дорівнювати _field_. Два поля повинні бути одного типу. Рядки, числа, масиви і файли оцінюються з використанням тих самих угод, що й правило [`size`](#rule-size).

<a name="rule-mac"></a>
#### mac_address

Поле, яке перевіряється, повинно містити адресу MAC-адресу.

<a name="rule-max"></a>
#### max:_value_

Значення поля, яке перевіряється, повинно мати меньшу кількість символів або дорівнювати _value_. Рядки, числа, масиви і файли оцінюються з використанням тих самих угод, що й правило [`size`](#rule-size).

<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

Файл, який перевіряється, повинен відповідати одному з перелічених MIME-типів:

```php
'video' => 'mimetypes:video/avi,video/mpeg,video/quicktime',
```
Щоб визначити MIME-тип завантаженого файла, вміст файла буде прочитано фреймворком. Він спробує вгадати MIME-тип, який може відрізнятися від наданого клієнтом типу MIME.

<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

Файл, який перевіряється, повинен мати тип MIME, який відповідає одному з розширень у списку.

```php
'file' => 'required|file|mimes:ppt,pptx,doc,docx,pdf,xls,xlsx|max:204800'
```

<a name="basic-usage-of-mime-rule"></a>
#### Основи використання правила MIME

```php
'photo' => 'mimes:jpg,bmp,png'
```

Незважаючи на те, що вам потрібно лише вказати розширення, це правило фактично перевіряє тип MIME файлу, читаючи вміст файлу та вгадуючи його тип MIME. Повний перелік типів MIME та їхніх відповідних розширень можна знайти за таким місцем:
[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="rule-min"></a>
#### min:_value_

Значення поля, яке перевіряється, повинно мати більшу кількість символів або дорівнювати _value_. Рядки, числа, масиви і файли оцінюються з використанням тих самих угод, що й правило [`size`](#rule-size).

<a name="multiple-of"></a>
#### multiple_of:_value_

Поле, яке перевіряється, має бути кратним _value_.

> {note} Для використання правила `multiple_of` потрібне [розширення `bcmath` PHP.](https://www.php.net/manual/ru/book.bc.php).

<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

Значення поля, яке перевіряється, не повинно бути присутнім у списку визначених значень. Метод `Rule::notIn` використовується для довільної побудови правила:

```php
use Illuminate\Validation\Rule;

Validator::make($data, [
	'toppings' => [
		'required',
		Rule::notIn(['sprinkles', 'cherries']),
	],
]);
```

<a name="rule-not-regex"></a>
#### not_regex:_pattern_

Значення поля, яке перевіряється, не повинно відповідати регулярному виразу

Всередині цього правила використовується функція `preg_match` PHP. Визначений шаблон має відповідати тому ж форматуванню, яке вимагає `preg_match`, відповідно повинен містити доступні роздільники. Наприклад: `'email' => 'not_regex:/^.+$/i'`.

> {note} Під час використання шаблонів `regex` / `not_regex` може знадобитися вказати правила перевірки за допомогою масиву замість використання `|` роздільників, особливо якщо регулярний вираз містить символ `|`.

<a name="rule-nullable"></a>
#### nullable

Поле, яке перевіряється може містити `null`.

<a name="rule-numeric"></a>
#### numeric

Поле, яке перевіряється має містити тип [число](https://www.php.net/manual/ru/function.is-numeric.php).

<a name="rule-password"></a>
#### password

Поле, що перевіряється, має бути однаковим з паролем автентифікованого користувача.

> {note} Це правило було перейменовано в `current_password` з його наступним видаленням в Laravel 9. Замість цього правила використовуте правило [Current Password](#rule-current-password).

<a name="rule-present"></a>
#### present

Поле, що перевіряється, має бути присутнім у вхідних даних, але може бути порожнім.

<a name="rule-prohibited"></a>
#### prohibited

Поле, що перевіряється, повинно бути порожнім або відсутнім.

<a name="rule-prohibited-if"></a>
#### prohibited_if:_anotherfield_,_value_,...

Поле, що перевіряється, повинно бути порожнім або відсутнім, якщо інше поле _anotherfield_ дорівнює будь-якому _value_.

Якщо потрібна складна логіка умовної заборони, ви можете використати метод `Rule::prohibitedIf`. Цей метод приймає логічне значення або замикання. Якщо надати замикання, воно має повернути `true` або `false`, щоб вказати, чи має бути заборонено поле, що перевіряється:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
	'role_id' => Rule::prohibitedIf($request->user()->is_admin),
]);

Validator::make($request->all(), [
	'role_id' => Rule::prohibitedIf(fn () => $request->user()->is_admin),
]);
```

<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_anotherfield_,_value_,...

Поле, яке перевіряється, повинно бути порожнім або відсутнім, якщо інше поле _anotherfield_ не дорівнює будь-якому _value_.

<a name="rule-prohibits"></a>
#### prohibits:_anotherfield_,...

Якщо поле, яке перевіряється, присутнє, поля в _anotherfield_ не можуть бути присутніми, навіть якщо вони пусті.

<a name="rule-regex"></a>
#### regex:_pattern_

Значення поля, яке перевіряється, повинно відповідати регулярному виразу

Всередині цього правила використовується функція `preg_match` PHP. Визначений шаблон має відповідати тому ж форматуванню, яке вимагає `preg_match`, відповідно повинен містити доступні роздільники. Наприклад: `'email' => 'not_regex:/^.+$/i'`.

> {note} Під час використання шаблонів `regex` / `not_regex` може знадобитися вказати правила перевірки за допомогою масиву замість використання `|` роздільників, особливо якщо регулярний вираз містить символ `|`.

<a name="rule-required"></a>
#### required

Поле, яке перевіряється, повинно бути присутнім у вхідних даних і не бути порожнім. Поле вважатиметься пустим, якщо виконується одне з наступних вимог:

<!-- <div class="content-list" markdown="1"> -->

- Значення поля дорівнює `null`.
- Значення поля - це порожній рядок.
- Значення поля - це порожній масив або порожній об'єктом, що реалізує інтерфейс `Countable`.
- Значення поля - це завантажений файл з відсутнім шлахом.

<!-- </div> -->

<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

Поле, яке перевіряється, повинно бути присутнім у вхідних даних і не бути порожнім, якщо поле _anotherfield_ дорівнює будь-якому _value_.

Якщо ви бажаєте створити більш складу умову для правила `required_if`, то вам варто використовувати метод `Rule::requiredIf`. Цей метод приймає логічне значення або замикання. При використанні замикання воно повинно повертати `true` или `false`, щоб вказати, чи обов'язкове поле, яке перевіряється:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
	'role_id' => Rule::requiredIf($request->user()->is_admin),
]);

Validator::make($request->all(), [
	'role_id' => Rule::requiredIf(fn () => $request->user()->is_admin),
]);
```

<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

Поле, яке перевіряється, повинно бути присутнім в вхідних даних і не бути порожнім, якщо поле _anotherfield_ не дорівнює будь-якому _value_. Це також означає, що _anotherfield_ повинно бути присутнім у даних запиту, якщо _value_ не має значення `null`. Якщо _value_ дорівнює null` (тобто `required_unless:name,null`), то поле, яке перевіряється, має бути обов'язковим, якщо тільки поле порівняння не дорівнює `null` або поле порівняння відсутнє в даних запита.

<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

Поле, яке перевіряється, повинно бути присутнім і не порожнім _тільки, якщо_ будь-яке із зазначиних інших полів присутнє і не порожнє.

<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

Поле, яке перевіряється, повинно бути присутнім і не порожнім _тільки, якщо_ всі інші зазначені поля присутні і не порожні.

<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

Поле, яке перевіряється, повинно бути присутнім і не порожнім _тільки, якщо_ будь-яке із зазначених інших полів порожнє або відсутнє.

<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

Поле, яке перевіряється, повинно бути присутнім і не порожнім _тільки, якщо_ всі інші зазначені поля порожні або відсутні.

<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

Поле, яке перевіряється, повинно бути масивом і містити принаймні зазначені ключі.

<a name="rule-same"></a>
#### same:_field_

Вміст наданого _field_ повиннен відповідати полю, яке перевіряється.

<a name="rule-size"></a>
#### size:_value_

Поле, яке перевіряється, повинно мати розмір, який зазначений _value_. Для рядкових даних _value_відповідатиме кількості символів. Для числових даних _value_ відповідатиме заданому цілому значенню (додатково атрибут також повинен мати `numeric` чи `integer` правила). Для масиву _size_ відповідає `count` масиву. Для файлів _size_ відповідатиме розміру файла в кілобайтах. Давайте розглянемо декіль прикладів:

```php
// Перевіряємо, що рядок містить 12 символів ...
'title' => 'size:12';

// Перевіряємо, що передано ціле число, і воно дорівнює 10 ...
'seats' => 'integer|size:10';

// Перевіряємо, що в масиві рівно 5 элементів ...
'tags' => 'array|size:5';

// Перевіряємо, що розмір завантаженого файла складає рівно 512 кілобайт ...
'image' => 'file|size:512';
```

<a name="rule-starts-with"></a>
#### starts_with:_foo_,_bar_,...

Поле, яке перевіряється, повинно починатися на одне з вказаних значень.

<a name="rule-string"></a>
#### string

Поле, яке перевіряється, повинно бути рядком. Якщо ви бажаєте, щоб дане поле також могло бути `null`, тоді вам потрібно назначити цьому полю правило `nullable`.

<a name="rule-timezone"></a>
#### timezone

Поле, яке перевіряється, має бути дійсним ідентифікатором часового поясу відповідно до функції `timezone_identifiers_list` PHP.

<a name="rule-unique"></a>
#### unique:_table_,_column_

Поле, яке перевіряється, не повинно існувати в даній таблиці бази даних.

**Визначення користувацького імені таблиці / імені колонки:**

Замість того, щоб вказувати назву таблиці безпосередньо, ви можете вказати модель Eloquent, яка має використовуватися для визначення імені таблиці:

```php
'email' => 'unique:App\Models\User,email_address'
```
Параметр `column` можна використовувати для визначення відповідного поля колонки бази даних. Якщо параметр `column` не вказано, використовуватиметься ім’я поля, яке перевіряється.

```php
'email' => 'unique:users,email_address'
```

**Визначення користувацького з'єднання з базою даних**

Іноді вам може знадобитися встановити спеціальне з’єднання для запитів до бази даних, які виконує валідатор. Щоб досягти цього, ви можете додати назву з’єднання перед назвою таблиці

```php
'email' => 'unique:connection.users,email_address'
```

**Навмисне ігнорування правилом Unique конкретного ідентифікатора:**

Іноді ви можете проігнорувати даний ідентифікатор під час `unique` перевірки. Наприклад, розглянемо сторінку «оновлення профілю», який містить ім’я користувача, електронну адресу та місцезнаходження. Ймовірно, ви захочете переконатися, що адреса електронної пошти унікальна. Однак якщо користувач змінює лише поле імені, а не поле електронної пошти, то ви не захочете, щоб виникала помилка валідації, оскільки користувач вже є власником відповідної адреси електронної пошти.

Щоб наказати валідатору ігнорувати ідентифікатор користувача, ми використаємо клас `Rule` для довільного визначення правила. У цьому прикладі ми також вкажемо правила перевірки як масива замість використання `|` символа для розмежування правил:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
	'email' => [
		'required',
		Rule::unique('users')->ignore($user->id),
	],
]);
```

> {note} Ви ніколи не повинні передавати будь-яке введене користувачем значення з вхідного запита у метод `ignore`. Натомість вам слід передавати лише унікальний ідентифікатор, згенерований системою, наприклад ідентифікатор із автоматичним збільшенням або UUID з екземпляра моделі Eloquent. Інакше ваша програма буде вразливою до SQL-атаки..

Замість того, щоб передавати значення ключа моделі в метод `ignore`, ви також можете передати весь екземпляр моделі. Laravel автоматично витягне ключ із моделі:

```php
Rule::unique('users')->ignore($user)
```
Якщо ваша таблиця використовує ім’я колонки первинного ключа, яке відрізняється від id, ви можете вказати другим аргументом ім’я колонки під час виклику методу ігнорування:

```php
Rule::unique('users')->ignore($user->id, 'user_id')
```

За замовчуванням правило `unique` перевіряє унікальність колонки, яка відповідає імені атрибута, який перевіряється. Однак ви можете передати іншу назву колонки другим аргументом `unique` методу:

```php
Rule::unique('users', 'email_address')->ignore($user->id),
```

**Додавання додаткових виразів Where:**

Ви можете вказати додаткові умови запиту, змінивши запит за допомогою методу `where`. Наприклад, давайте додамо умову запиту, яка обмежує запит лише для пошуку записів, які мають значення стовпця account_id `1`:

```php
'email' => Rule::unique('users')->where(fn ($query) => $query->where('account_id', 1))
```

<a name="rule-url"></a>
#### url

Поле, яке перевіряється, повинно мати дійсний URL.

<a name="rule-uuid"></a>
#### uuid

Поле, яке перевіряється, має бути дійсним універсальним унікальним ідентифікатором (UUID) RFC 4122 (версія 1, 3, 4 або 5).

<a name="conditionally-adding-rules"></a>
## Умовне додавання правил

<a name="skipping-validation-when-fields-have-certain-values"></a>
#### Пропуск валідації, якщо поля мають певні значення

За бажанням ви можете не перевіряти дане поле, якщо інше поле має задане значення. Це можна зробити за допомогою правила перевірки exclude_if. У цьому прикладі поля `appointment_date` і `doctor_name` не перевірятимуться, якщо поле `has_appointment` має значення `false`:

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, [
	'has_appointment' => 'required|boolean',
	'appointment_date' => 'exclude_if:has_appointment,false|required|date',
	'doctor_name' => 'exclude_if:has_appointment,false|required|string',
]);
```
Крім того, ви можете використати правило `exclude_unless`, щоб не перевіряти дане поле, якщо інше поле не дорівнює вказаного значення:

```php
$validator = Validator::make($data, [
	'has_appointment' => 'required|boolean',
	'appointment_date' => 'exclude_unless:has_appointment,true|required|date',
	'doctor_name' => 'exclude_unless:has_appointment,true|required|string',
]);
```

<a name="validating-when-present"></a>
#### Валідація за наявністі

У деяких випадках ви можете запустити перевірку поля **лише**, якщо це поле присутнє в даних, які перевіряються. Щоб швидко досягти цього, додайте правило `sometimes` до свого списку правил:

```php
$validator = Validator::make($data, [
	'email' => 'sometimes|required|email',
]);
```

У наведеному вище прикладі поле `email` буде перевірено, лише якщо воно присутнє в масиві $data або `$request->all()`.

> {tip} Якщо ви намагаєтеся перевірити поле, яке завжди має бути присутнім, але може бути порожнім, перегляньте цю [примітку щодо необов’язкових полів](#a-note-on-optional-fields).

<a name="complex-conditional-validation"></a>
#### Комплексна умовна перевірка

Іноді ви можете додати правила перевірки на основі складнішої умовної логіки. Наприклад, ви можете вимагати певне поле, лише якщо інше поле має значення більше 100. Або вам може знадобитися, щоб два поля мали задане значення лише тоді, коли присутнє інше поле. Додавання цих правил перевірки не повинно бути проблемою. Спочатку створіть екземпляр Validator` зі своїми _статическими правилами_, які ніколи не змінюються:

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
	'email' => 'required|email',
	'games' => 'required|numeric',
]);
```

Припустімо, що наша веб-програма призначена для колекціонерів ігор. Якщо колекціонер ігор реєструється в нашій програмі та володіє понад 100 ігор, ми хочемо, щоб він пояснив, чому він володіє такою кількістю ігор. Наприклад, можливо, вони тримають магазин із перепродажу ігор, а може, їм просто подобається колекціонувати ігри. Щоб умовно додати цю вимогу, ми можемо використати метод `sometimes` в екземплярі `Validator`:

```php
$validator->sometimes('reason', 'required|max:500', function ($input) {
	return $input->games >= 100;
});
```
Перший аргумент, який передається методу `sometimes` — це ім'я поля, яке ми умовно перевіряємо. Другий аргумент — це список правил, які ми хочемо додати. Якщо передано замикання третім аргументом, який повертає true, правила будуть додані. Цей метод полегшує створення складних умовних перевірок. Ви навіть можете додати умовні перевірки для кількох полів одночасно:

```php
$validator->sometimes(['reason', 'cost'], 'required', function ($input) {
	return $input->games >= 100;
});
```

> {tip} Параметр `$input`, переданий вашому замиканню, буде екземпляром `Illuminate\Support\Fluent` і може використовуватися для доступу до ваших введених даних і файлів, що перевіряються.

<a name="complex-conditional-array-validation"></a>
#### Комплексна перевірка умовного масиву

Іноді вам може знадобитися перевірити поле на основі іншого поля в тому ж вкладеному масиві, індекс якого ви не знаєте. У таких ситуаціях ви можете використовувати другий аргумент вашого замикання, який буде поточним окремим елементом у масиві, що перевіряється:

```php
$input = [
	'channels' => [
		[
			'type' => 'email',
			'address' => 'abigail@example.com',
		],
		[
			'type' => 'url',
			'address' => 'https://example.com',
		],
	],
];

$validator->sometimes('channels.*.address', 'email', function ($input, $item) {
	return $item->type === 'email';
});

$validator->sometimes('channels.*.address', 'url', function ($input, $item) {
	return $item->type !== 'email';
});
```
Подібно до параметра `$input`, переданого до замикання, параметр `$item` є екземпляром Illuminate\Support\Fluent, коли дані атрибута є масивом; інакше це рядок.

<a name="validating-arrays"></a>
## Валідація масивів

Як зазначено в документації [правил перевірки масиву](#rule-array), правило масиву приймає список дозволених ключів масиву. Якщо в масиві присутні додаткові ключі, перевірка буде невдалою:

```php
use Illuminate\Support\Facades\Validator;

$input = [
	'user' => [
		'name' => 'Taylor Otwell',
		'username' => 'taylorotwell',
		'admin' => true,
	],
];

Validator::make($input, [
	'user' => 'array:username,locale',
]);
```
Загалом, ви завжди повинні вказувати ключі масиву, які можуть бути присутніми у вашому масиві. Інакше методи `validate` і `validated` засобу перевірки повернуть усі провалідовані дані, включаючи масив і всі його ключі, навіть якщо ці ключі не перевірено іншими правилами валідації вкладених масивів.

<a name="validating-nested-array-input"></a>
### Валідація вкладених масивів

Перевірка полів у формі на основі вкладених масивів не повинна бути проблемою. Ви можете використовувати «крапкову нотацію» для перевірки атрибутів у масиві. Наприклад, якщо вхідний HTTP-запит містить поле `photos[profile]`, ви можете перевірити його так:

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
	'photos.profile' => 'required|image',
]);
```
Ви також можете перевірити кожен елемент масиву. Наприклад, щоб підтвердити унікальність кожного електронного листа в заданому полі введення масиву, ви можете зробити наступне:

```php
$validator = Validator::make($request->all(), [
	'person.*.email' => 'email|unique:users',
	'person.*.first_name' => 'required_with:person.*.last_name',
]);
```

Так само ви можете використовувати символ *, коли вказуєте [спеціальні повідомлення про помилки валідації у ваших мовних файлах](#custom-messages-for-specific-attributes), що полегшить використання одного повідомлення перевірки для полів на основі масиву:

```php
'custom' => [
	'person.*.email' => [
		'unique' => 'Each person must have a unique email address',
	]
],
```

<a name="accessing-nested-array-data"></a>
#### Доступ до даних вкладеного масиву

Іноді вам може знадобитися отримати доступ до значення для заданого елемента вкладеного масиву під час призначення правил перевірки атрибуту. Це можна зробити за допомогою методу `Rule::forEach`. Метод `forEach` приймає замикання, яке буде викликано для кожної ітерації атрибута масиву під час перевірки і отримає значення атрибута та явне, повністю розгорнуте ім’я атрибута. Замикання має повернути масив правил призначених елементу масиву:

```php
use App\Rules\HasPermission;
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

$validator = Validator::make($request->all(), [
	'companies.*.id' => Rule::forEach(function ($value, $attribute) {
		return [
			Rule::exists(Company::class, 'id'),
			new HasPermission('manage-company', $value),
		];
	}),
]);
```

<a name="error-message-indexes-and-positions"></a>
### Індекси та позиції повідомлень про помилки

Під час перевірки масивів ви можете посилатися на індекс або позицію певного елемента, який не пройшов перевірку, у повідомленні про помилку, яке відображається вашою програмою. Щоб досягти цього, ви можете включити заповнювачі `:index` і `:position` у своеє [власне повідомлення про помилку](#manual-customizing-the-error-messages):

```php
use Illuminate\Support\Facades\Validator;

$input = [
	'photos' => [
		[
			'name' => 'BeachVacation.jpg',
			'description' => 'A photo of my beach vacation!',
		],
		[
			'name' => 'GrandCanyon.jpg',
			'description' => '',
		],
	],
];

Validator::validate($input, [
	'photos.*.description' => 'required',
], [
	'photos.*.description.required' => 'Please describe photo #:position.',
]);
```
Враховуючи наведений вище приклад, перевірка не вдасться, і користувачеві буде запропоновано таку помилку _"Please describe photo №2."_


<a name="validating-passwords"></a>
## Валідація паролей

Щоб переконатися, що паролі мають адекватний рівень складності, ви можете використовувати клас правила `Password` Laravel:

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\Password;

$validator = Validator::make($request->all(), [
	'password' => ['required', 'confirmed', Password::min(8)],
]);
```
Клас правила `Password` дозволяє легко налаштувати вимоги до складності пароля для вашого додатку, наприклад, вказати, що паролі потребують принаймні однієї літери, цифри, символу або символів зі змішаним регістром:

```php
// Потрібно принаймні 8 символів...
Password::min(8)

// Потрібна хоча б одна буква...
Password::min(8)->letters()

// Вимагайте принаймні одну велику та одну малу літери...
Password::min(8)->mixedCase()

// Потрібна принаймні одна цифра...
Password::min(8)->numbers()

// Потрібен хоча б один символ...
Password::min(8)->symbols()
```

Крім того, ви можете переконатися, що пароль не було [зкомпрометовано](https://uk.wikipedia.org/wiki/Компрометація_(криптографія)) в результаті витоку публічних даних про пароль, використовуючи метод `uncompromised`:

```php
Password::min(8)->uncompromised()
```
Внутрішньо об’єкт правила `Password` використовує модель [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity), щоб визначити, чи стався витік пароля через службу [haveibeenpwned.com](https://haveibeenpwned.com) не жертвуючи конфіденційністю чи безпекою користувача.

За замовчуванням, якщо пароль з’являється принаймні один раз у витоку даних, він вважатиметься скомпрометованим. Ви можете налаштувати цей поріг за допомогою першого аргументу `uncompromised` методу:

```php
// Будьте впевнені, що пароль з’являється менше 3 разів під час одного витоку даних...
Password::min(8)->uncompromised(3);
```

Також методи можна викликати ланцюжком:

```php
Password::min(8)
	->letters()
	->mixedCase()
	->numbers()
	->symbols()
	->uncompromised()
```

<a name="defining-default-password-rules"></a>
#### Визначення правил валідації паролей за замовчуванням

Можливо вам буде зручніше вказати стандартні правила перевірки паролів в одному місці вашого додатка. Це можна легко зробити за допомогою методу `Password::defaults`, який приймає замикання. Замикання передане методу `defaults`, має повертати стандартну конфігурацію правила пароля. Зазвичай виклик метода `defaults` здійснюється у методі `boot` одного з постачальників служб вашого додатка `app/Http/Providers/AppServiceProvider.php`:

```php
use Illuminate\Validation\Rules\Password;

/**
 * Завантаження будь-яких служб додатка.
 *
 * @return void
 */
public function boot()
{
Password::defaults(function () {
	$rule = Password::min(8);

	return $this->app->isProduction()
				? $rule->mixedCase()->uncompromised()
				: $rule;
});
}
```
Далі, якщо ви хочете застосувати власні стандартні правила до певного пароля, який проходить перевірку, ви можете викликати метод `defaults` без аргументів:

```php
'password' => ['required', Password::defaults()],
```

За бажанням можна додати додаткові правила валідації до ваших стандартних правил валідації пароля. Для цього вам потрібно використати метод `rules`:

```php
use App\Rules\ZxcvbnRule;

Password::defaults(function () {
	$rule = Password::min(8)->rules([new ZxcvbnRule]);

	// ...
});
```

<a name="custom-validation-rules"></a>
## Власні правила валідації

<a name="using-rule-objects"></a>
### Використання класа Rule

Laravel надає різноманітні корисні правила перевірки: проте ви можете вказати деякі власні. Одним із методів реєстрації власних правил перевірки є використання об’єктів правил. Щоб створити новий об’єкт правила, ви можете використати команду `make:rule` [Artisan](artisan.md). Давайте використаємо цю команду для створення правила, яке перевіряє, що рядок є у верхньому регістрі. Laravel розмістить нове правило в каталозі `app/Rules`. Якщо цей каталог не існує, Laravel створить його, коли ви виконаєте команду Artisan для створення свого правила:

```shell
php artisan make:rule Uppercase --invokable
```
> {tip} Починаючи з версії Laravel: 9.19.0, параметр `--invokable` доступний.

Коли правило створено, ми готові визначити його поведінку. Об’єкт правила містить один метод: `__invoke`. Цей метод отримує ім’я атрибута, його значення та зворотний виклик, який має бути викликаний у разі помилки з повідомленням про помилку перевірки:

```php
<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\InvokableRule;

class Uppercase implements InvokableRule
{
	/**
	* Run the validation rule.
	*
	* @param  string  $attribute
	* @param  mixed  $value
	* @param  \Closure  $fail
	* @return void
	*/
	public function __invoke($attribute, $value, $fail)
	{
		if (strtoupper($value) !== $value) {
			$fail('The :attribute must be uppercase.');
		}
	}
}
```
Після визначення правила ви можете приєднати його до валідатора, передавши екземпляр об’єкта правила з іншими правилами перевірки:
Вы можете вызвать помощник `trans` в методе `message`, если хотите вернуть сообщение об ошибке из ваших файлов перевода:

```php
use App\Rules\Uppercase;

$request->validate([
	'name' => ['required', 'string', new Uppercase],
]);
```

### Переклад валідаційних повідомлень

Замість надання буквального повідомлення про помилку для замикання `$fail`, ви також можете надати _ключ рядка перекладу_ та вказати Laravel перекласти повідомлення про помилку за допомогою метода `translate`:

```php
if (strtoupper($value) !== $value) {
	$fail('validation.uppercase')->translate();
}
```
Якщо необхідно, ви можете вказати замінники для заповнювачів і бажану мову як першим і другим аргументами методу `translate`:

```php
$fail('validation.location')->translate([
	'value' => $this->value,
], 'fr')
```
### Доступ до додаткових даних

Якщо вашому спеціальному класу правил перевірки потрібен доступ до всіх інших даних, які проходять перевірку, ваш клас правил може реалізувати інтерфейс `Illuminate\Contracts\Validation\DataAwareRule`. Цей інтерфейс вимагає від вашого класу визначення методу setData. Цей метод буде автоматично викликано Laravel (перед тим як почнеться валідація) з усіма даними, що перевіряються:

```php
<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\InvokableRule;
use Illuminate\Contracts\Validation\DataAwareRule;

class Uppercase implements DataAwareRule, InvokableRule
{
	/**
	 * Всі дані, які перевіряються.
	 *
	 * @var array
	 */
	protected $data = [];

	// ...

	/**
	 * Встановіть дані для вілідації.
	 *
	 * @param  array  $data
	 * @return $this
	 */
	public function setData($data)
	{
		$this->data = $data;

		return $this;
	}
}
```
Або, якщо ваше правило перевірки вимагає доступу до екземпляра валідатора, який виконує перевірку, ви можете застосувати інтерфейс `ValidatorAwareRule`:

```php
<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\InvokableRule;
use Illuminate\Contracts\Validation\ValidatorAwareRule;

class Uppercase implements InvokableRule, ValidatorAwareRule
{
/**
 * Екземпляр валідатора.
 *
 * @var \Illuminate\Validation\Validator
 */
protected $validator;

// ...

/**
 * Встановити поточний валідатор.
 *
 * @param  \Illuminate\Validation\Validator  $validator
 * @return $this
 */
public function setValidator($validator)
{
	$this->validator = $validator;

	return $this;
}
}
```

<a name="using-closures"></a>
### Використання замикань

Якщо вам потрібні функції спеціального правила лише один раз у вашому додатку, ви можете використати анонімне правило замість об’єкта правила. Анонімне правило отримує ім’я атрибута, значення атрибута та зворотний виклик $fail, який слід викликати, якщо перевірка невдала:

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
'title' => [
	'required',
	'max:255',
	function ($attribute, $value, $fail) {
		if ($value === 'foo') {
			$fail('The '.$attribute.' is invalid.');
		}
	},
],
]);
```

<a name="implicit-rules"></a>
### Неявні правила

За замовчуванням, коли атрибут, який перевіряється, відсутній або містить порожній рядок, звичайні правила перевірки, включаючи правила створені вами, не виконуються. Наприклад, правило [`unique`](#rule-unique) не буде виконано для порожнього рядка:

```php
use Illuminate\Support\Facades\Validator;

$rules = ['name' => 'unique:users,name'];

$input = ['name' => ''];

Validator::make($input, $rules)->passes(); // true
```

Щоб ваше правило було застосоване, навіть якщо атрибут порожній, правило має означати, що атрибут є обов'язкоми. Щоб створити «неявне» правило, реалізуйте інтерфейс `Illuminate\Contracts\Validation\ImplicitRule`. Цей інтерфейс виконує роль «маркера» для валідатора; отже, він не містить жодних додаткових методів, які потрібно реалізувати, окрім методів, необхідних для типового інтерфейсу `Rule`.

Щоб створити новий об’єкт неявного правила, ви можете використати команду `make:rule` Artisan із параметром `--implicit`:

```shell
php artisan make:rule Uppercase --invokable --implicit
```

> {note} Неявне правило лише _передбачає_, що атрибут є обов'язковим до валідації. Насправді, вирішувати тільки вам, порожній або відсутній атрибут вважатиметься невалідним.
