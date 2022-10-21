# Авторизація

- [Вступ](#introduction)
- [Шлюзи](#gates)
  - [Написання шлюзів](#writing-gates)
  - [Авторизація дій через шлюзи](#authorizing-actions-via-gates)
  - [Відповіді шлюза](#gate-responses)
  - [Перехоплення шлюзів](#intercepting-gate-checks)
  - [Спрощена авторизація](#inline-authorization)
- [Створення політик](#creating-policies)
  - [Генерація політик](#generating-policies)
  - [Реєстрація політики](#registering-policies)
- [Написання політики](#writing-policies)
  - [Методи політики](#policy-methods)
  - [Відповіді політики](#policy-responses)
  - [Методи політики без моделей](#methods-without-models)
  - [Гостьові користувачі](#guest-users)
  - [Фільтри політики](#policy-filters)
- [Авторизація дій за допомогою політик](#authorizing-actions-using-policies)
  - [Через модель User](#via-the-user-model)
  - [Через додаткові методи контроллера](#via-controller-helpers)
  - [Через Middleware](#via-middleware)
  - [Через шаблони Blade](#via-blade-templates)
  - [Надання додаткового контексту](#supplying-additional-context)

<a name="introduction"></a>

## Вступ

Окрім надання вбудованих служб [автентифікації](authentication.md), Laravel також надає простий спосіб авторизації дій користувача щодо певного ресурса. Наприклад, навіть якщо користувач автентифікований, він може не мати права оновлювати або видаляти певні моделі Eloquent або записи бази даних, якими керує ваш додаток. Функціонал авторизації Laravel забезпечує простий і організований спосіб керування такими типами перевірок авторизації.

Laravel надає два основні способи авторизації дій: [шлюзи](#gates) та [політики](#creating-policies). Сприймайте шлюзи та політики, як маршрути та контролери. Шлюзи пропонують простий, заснований на замиканні підхід до авторизації, тоді як політики, як контролери, групують логіку навколо конкретної моделі або ресурса. В цій документації ми спочатку розглянемо шлюзи, а потім розглянемо політики.

Вам не потрібно обирати між виключно використанням шлюзів або виключно використанням політик під час створення додатка. Більшість додатків, швидше за все, міститиме деяку суміш шлюзів і політик, і це цілком нормально! Шлюзи найбільш застосовуються до дій, які не пов’язані з жодною моделлю чи ресурсом, наприклад перегляду інформаційної панелі адміністратора. Політики навпаки, слід використовувати, коли ви хочете дозволити дію для певної моделі чи ресурса.

<a name="gates"></a>

## Шлюзи

<a name="writing-gates"></a>

### Написання шлюзів

> **Warning**  
> Шлюзи — чудовий спосіб вивчити основи функціонала авторизації Laravel; однак, створюючи надійні додатки Laravel, вам слід розглянути можливість використання [політик](#creating-policies) для організації правил авторизації.

Шлюзи — це просто замикання, які визначають, чи має користувач право виконувати певну дію. Як правило, шлюзи визначаються в методі `boot` класа `App\Providers\AuthServiceProvider` за допомогою фасада `Gate`. Шлюз завжди отримує екземпляр користувача першим аргументом і за бажанням може отримати додаткові аргументи, наприклад відповідну модель Eloquent.

У цьому прикладі ми визначимо шлюз, щоб з'ясувати, чи може користувач оновлювати певну модель `App\Models\Post`. Шлюз зробить це шляхом порівняння `id` користувача з `user_id`, який створив публікацію:

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Support\Facades\Gate;

/**
 * Реєстрація будь-яких служб автентифікації/авторизації.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Gate::define('update-post', function (User $user, Post $post) {
        return $user->id === $post->user_id;
    });
}
```

Подібно до контролерів, шлюзи також можуть бути визначені за допомогою callback-масива:

```php
use App\Policies\PostPolicy;
use Illuminate\Support\Facades\Gate;

/**
 * Реєстрація будь-яких служб автентифікації/авторизації.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Gate::define('update-post', [PostPolicy::class, 'update']);
}
```

<a name="authorizing-actions-via-gates"></a>

### Авторизація дій через шлюзи

Щоб авторизувати дію з використанням шлюзів, ви повинні використовувати методи `allows` або `denies`, надані фасадом `Gate`. Зауважте, що вам не потрібно передавати поточного автентифікованого користувача цим методам. Laravel автоматично подбає про передачу користувача до замикання шлюза. Зазвичай методи авторизації шлюза викликаються в контролерах вашого додатка перед виконанням дії, яка вимагає авторизації:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Post;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

class PostController extends Controller
{
    /**
     * Оновити передану публікацію...
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, Post $post)
    {
        if (! Gate::allows('update-post', $post)) {
            abort(403);
        }

        // Оновлення публікації...
    }
}
```

Якщо ви бажаєте перевірити, чи інший користувач, який відрізняється від поточного автентифікованого користувача, має право виконувати дію? Ви можете використати метод `forUser` фасада `Gate`:

```php
if (Gate::forUser($user)->allows('update-post', $post)) {
    // Користувач може оновити публікацію...
}

if (Gate::forUser($user)->denies('update-post', $post)) {
    // Користувач не може оновити публікацію...
}
```

Ви можете дозволити декілька дій одночасно за допомогою методів `any` або `none`:

```php
if (Gate::any(['update-post', 'delete-post'], $post)) {
    // Користувач може оновити або видалити публікацію...
}

if (Gate::none(['update-post', 'delete-post'], $post)) {
    // Користувач не може оновити або видалити публікацію...
}
```

<a name="authorizing-or-throwing-exceptions"></a>

#### Авторизація або створення винятків

Якщо ви бажаєте спробувати авторизувати дію та автоматично викликати `Illuminate\Auth\Access\AuthorizationException`, якщо користувачеві не дозволено виконувати дану дію, ви можете використати метод `authorize` фасада `Gate`. Екземпляри `AuthorizationException` автоматично перетворюються на HTTP-відповідь `403` виконацем винятків Laravel:

```php
Gate::authorize('update-post', $post);

// Дію дозволено...
```

<a name="gates-supplying-additional-context"></a>

#### Надання додаткового контекста шлюзам

Методи шлюза для авторизації повноважень (`allows`, `denies`, `check`, `any`, `none`, `authorize`, `can`, `cannot`) і [директиви Blade](#via-blade-templates) авторизації (`@can`, `@cannot`, `@canany`) можуть отримувати масив другим аргументом. Ці елементи масива передаються як параметри в замикання шлюза і можуть використовуватися для додаткового контексту під час прийняття рішень щодо авторизації:

```php
use App\Models\Category;
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::define('create-post', function (User $user, Category $category, $pinned) {
    if (! $user->canPublishToGroup($category->group)) {
        return false;
    } elseif ($pinned && ! $user->canPinPosts()) {
        return false;
    }

    return true;
});

if (Gate::check('create-post', [$category, $pinned])) {
   // Користувач може створити публікацію...
}
```

<a name="gate-responses"></a>

### Відповіді шлюза

Поки що ми досліджували лише Шлюзи, які повертають прості булеві значення. Однак іноді ви можете повернути більш детальну відповідь, включаючи повідомлення про помилку. Для цього ви можете повернути `Illuminate\Auth\Access\Response` зі свого шлюза:

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
        ? Response::allow()
        : Response::deny('You must be an administrator.');
});
```

Навіть коли ви повертаєте відповідь авторизації зі свого шлюза, метод `Gate::allows` повертатиме просте логічне значення; однак ви можете використати метод `Gate::inspect`, щоб отримати повну авторизаційну відповідь, повернуту шлюзом:

```php
$response = Gate::inspect('edit-settings');

if ($response->allowed()) {
    // Дія дозволена...
} else {
    echo $response->message();
}
```

Під час використання метода `Gate::authorize`, який створює `AuthorizationException`, якщо дія не авторизована, повідомлення про помилку, надане відповіддю авторизації, буде передано в HTTP-відповідь:

```php
Gate::authorize('edit-settings');

// Дія дозволена ...
```

<a name="customising-gate-response-status"></a>

#### Налаштування статусу HTTP-відповіді

Коли дію відхилено через шлюз, повертається відповідь HTTP `403`; однак іноді може бути корисним повернути альтернативний код стану HTTP. Ви можете налаштувати код статусу HTTP, який повертається для невдалої перевірки авторизації, використовуючи статичний конструктор `denyWithStatus` в класі `Illuminate\Auth\Access\Response`:

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
        ? Response::allow()
        : Response::denyWithStatus(404);
});
```

Оскільки приховування ресурсів за допомогою відповіді `404` є типовим шаблоном для веб-додатків, для зручності пропонується метод `denyAsNotFound`:

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
        ? Response::allow()
        : Response::denyAsNotFound();
});
```

<a name="intercepting-gate-checks"></a>

### Перехоплення шлюзів

Іноді, за бажанням ви можете надати всі повноваження певному користувачеві. Ви можете використовувати метод `before` для визначення замикання, яке виконується перед всіма іншими перевірками авторизації:

```php
use Illuminate\Support\Facades\Gate;

Gate::before(function ($user, $ability) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

Якщо замикання `before` повертає не `null` результат, цей результат вважатиметься результатом перевірки авторизації.

Ви можете використовувати метод `after`, щоб визначити замикання, яке буде виконано після всіх інших перевірок авторизації:

```php
Gate::after(function ($user, $ability, $result, $arguments) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

Подібно до методу `before`, якщо замикання `after` повертає не `null` результат, цей результат вважатиметься результатом перевірки авторизації.

<a name="inline-authorization"></a>

### Спрощена авторизація

Час від часу ви можете визначити, чи має поточний автентифікований користувач виконувати певну дію без написання спеціального шлюза, який відповідає цій дії. Laravel дозволяє виконувати ці типи «спрощених» перевірок авторизації за допомогою методів `Gate::allowIf` і `Gate::denyIf`:

```php
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn ($user) => $user->isAdministrator());

Gate::denyIf(fn ($user) => $user->banned());
```

Якщо дія не авторизована або якщо наразі жоден користувач не автентифікований, Laravel автоматично створить виняток `Illuminate\Auth\Access\AuthorizationException`. Екземпляри `AuthorizationException` автоматично перетворюються на HTTP-відповідь `403` виконавцем винятків Laravel.

<a name="creating-policies"></a>

## Створення політик

<a name="generating-policies"></a>

### Генерація політик

Політики — це класи, які організовують логіку авторизації навколо певної моделі чи ресурса. Наприклад, якщо ваш додаток є блогом, ви можете мати модель `App\Models\Post` і відповідну `App\Policies\PostPolicy` для авторизації дій користувача, таких як створення або оновлення публікацій.

Ви можете створити політику за допомогою команди `make:policy` Artisan. Створену політику буде розміщено в каталозі `app/Policies`. Якщо цей каталог не існує у вашому додатку, Laravel створить його для вас:

```shell
php artisan make:policy PostPolicy
```

Команда `make:policy` створить порожній клас політики. Якщо ви хочете створити клас із прикладами методів політики, пов’язаних із переглядом, створенням, оновленням і видаленням ресурса, ви можете вказати параметр `--model` під час виконання команди:

```shell
php artisan make:policy PostPolicy --model=Post
```

<a name="registering-policies"></a>

### Реєстрації політики

Після створення класа політики його потрібно зареєструвати. Реєстрації політики — це те, як ми можемо повідомити Laravel, яку політику використовувати під час авторизації дій щодо певного типу моделі.

`App\Providers\AuthServiceProvider`, який входить до нових додатків Laravel, містить властивість `policies` , яка зіставляє ваші моделі Eloquent з відповідними політиками. Реєстрація політики вкаже Laravel, яку політику використовувати під час авторизації дій щодо певної моделі Eloquent:

```php
<?php

namespace App\Providers;

use App\Models\Post;
use App\Policies\PostPolicy;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * Карта політик додатка.
     *
     * @var array
     */
    protected $policies = [
        Post::class => PostPolicy::class,
    ];

    /**
     * Реєстрація будь-яких служб аутентифікації / авторизації.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        //
    }
}
```

<a name="policy-auto-discovery"></a>

#### Автоматичне виявлення політики

Замість того, щоб вручну реєструвати політики моделі, Laravel може автоматично виявляти політики, якщо модель і політика відповідають стандартним іменуванням Laravel. Зокрема, політики мають бути в каталозі `Policies`, а каталог в середині або вище каталогу, який містить ваші моделі. Так, наприклад, моделі можна розмістити в каталозі `app/Models`, тоді як політики можна розмістити в каталозі `app/Policies`. В цій ситуації Laravel перевірить політики в `app/Models/Policies`, а потім в `app/Policies`. Крім того, назва політики має відповідати назві моделі та мати суфікс `Policy`. Таким чином, модель `User` буде відповідати класу політики `UserPolicy`.

Якщо ви хочете визначити власну логіку виявлення політики, ви можете зареєструвати зворотний виклик виявлення спеціальної політики за допомогою метода `Gate::guessPolicyNamesUsing`. Як правило, цей метод слід викликати з метода `boot` `AuthServiceProvider` вашого додатка:

```php
use Illuminate\Support\Facades\Gate;

Gate::guessPolicyNamesUsing(function ($modelClass) {
    // Повертає назву класа політики для даної моделі...
});
```

> **Warning**  
> Будь-які політики, явно зіставлені у вашому `AuthServiceProvider` , матимуть **перевагу** над усіма потенційно автоматично виявленими політиками.

<a name="writing-policies"></a>

## Написання політики

<a name="policy-methods"></a>

### Методи політики

Після реєстрації класа політики ви можете додати методи для кожної дії, яку він авторизує. Наприклад, давайте визначимо метод оновлення в нашій `PostPolicy`, який визначає, чи може певний `App\Models\User` оновлювати даний екземпляр `App\Models\Post`.

Метод `update` отримає в якості своїх аргументів екземпляр `User` і `Post` і повинен повернути `true` або `false`, вказуючи, чи має користувач право оновлювати даний `Post`. Отже, у цьому прикладі ми перевіримо, чи `id` користувача збігається з `user_id` в публікації:

```php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Визначте, чи може дана публікація бути оновлена ​​користувачем.
     *
     * @param  \App\Models\User  $user
     * @param  \App\Models\Post  $post
     * @return bool
     */
    public function update(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }
}
```

За потреби ви можете продовжувати визначати додаткові методи в політиці для різних дій, які вона авторизує. Наприклад, ви можете визначити методи `view` або `delete` для авторизації різних дій, пов’язаних з `Post`, але пам’ятайте, що ви можете давати своїм методам політики будь-яку назву.

Якщо ви використовували параметр `--model` під час створення політики через консоль Artisan, вона вже міститиме методи для дій `viewAny`, `view`, `create`, `update`, `delete`, `restore` та `forceDelete`.

> **Note**  
> Всі політики вилучаються за допомогою [контейнера служби](container.md) Laravel, що дозволяє вам оголошувати будь-яких необхідних залежності в конструкторі політики, щоб вони автоматично додавались.

<a name="policy-responses"></a>

### Відповіді політики

Поки що ми розглядали лише методи політики, які повертають прості булеві значення. Однак іноді ви можете повернути більш детальну відповідь, включаючи повідомлення про помилку. Для цього, ви можете повернути екземпляр `Illuminate\Auth\Access\Response` із метода політики:

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

/**
 * Визначте, чи може дана публікація бути оновлена ​​користувачем.
 *
 * @param  \App\Models\User  $user
 * @param  \App\Models\Post  $post
 * @return \Illuminate\Auth\Access\Response
 */
public function update(User $user, Post $post)
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::deny('You do not own this post.');
}
```

При поверненні відповіді авторизації з вашої політики метод `Gate::allows` повертатиме просте логічне значення; однак ви можете використати метод `Gate::inspect`, щоб отримати повну авторизаційну відповідь, повернуту шлюзом:

```php
use Illuminate\Support\Facades\Gate;

$response = Gate::inspect('update', $post);

if ($response->allowed()) {
    // Дія дозволена....
} else {
    echo $response->message();
}
```

Під час використання метода `Gate::authorize`, який створює `AuthorizationException`, якщо дія не авторизована, повідомлення про помилку, надане відповіддю авторизації, буде передано в HTTP-відповідь:

```php
Gate::authorize('update', $post);

// Дія дозволена...
```

<a name="customising-policy-response-status"></a>

#### Налаштування статусу HTTP-відповіді

Коли дію заборонено за допомогою метода політики, повертається HTTP-відповідь `403`; однак іноді може бути корисним повернути альтернативний код стану HTTP. Ви можете налаштувати код статусу HTTP, який повертається для невдалої перевірки авторизації, використовуючи статичний конструктор `denyWithStatus` в класі `Illuminate\Auth\Access\Response`:

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

/**
 * Визначте, чи може дана публікація бути оновлена ​​користувачем.
 *
 * @param  \App\Models\User  $user
 * @param  \App\Models\Post  $post
 * @return \Illuminate\Auth\Access\Response
 */
public function update(User $user, Post $post)
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::denyWithStatus(404);
}
```

Оскільки приховування ресурсів за допомогою відповіді `404` є типовим шаблоном для веб-додатків, для зручності пропонується метод `denyAsNotFound`:

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

/**
 * Визначте, чи може дана публікація бути оновлена ​​користувачем.
 *
 * @param  \App\Models\User  $user
 * @param  \App\Models\Post  $post
 * @return \Illuminate\Auth\Access\Response
 */
public function update(User $user, Post $post)
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::denyAsNotFound();
}
```

<a name="methods-without-models"></a>

### Методи політики без моделей

Деякі методи політики отримують лише екземпляр поточного автентифікованого користувача. Ця ситуація найчастіше зустрічається під час авторизації `create` дій. Наприклад, якщо ви створюєте блог, ви можете визначити, чи має користувач взагалі право створювати будь-які повідомлення. У цих ситуаціях ваш метод політики повинен очікувати лише отримання екземпляра користувача:

```php
/**
 * Визначте, чи може даний користувач створювати дописи.
 *
 * @param  \App\Models\User  $user
 * @return bool
 */
public function create(User $user)
{
    return $user->role == 'writer';
}
```

<a name="guest-users"></a>

### Гостьові користувачі

За замовчуванням всі шлюзи та політики автоматично повертають `false`, якщо вхідний HTTP-запит не був ініційований автентифікованим користувачем. Однак ви можете дозволити проходження цих перевірк авторизації через ваші шлюзи та політики, оголосивши `User` як тип [«який може бути `null`»](https://www.php.net/manual/ru/language.types.declarations.php#language.types.declarations.nullable), шляхом додавання префікса як знака питання (?) або надавши значення `null` за замовчуванням для аргумента `User`:

```php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Визначити, чи може користувач оновити пост.
     *
     * @param  \App\Models\User  $user
     * @param  \App\Models\Post  $post
     * @return bool
     */
    public function update(?User $user, Post $post)
    {
        return optional($user)->id === $post->user_id;
    }
}
```

<a name="policy-filters"></a>

### Фільтри політики

Для деяких користувачів ви можете дозволити всі дії в рамках даної політики. Для цього визначте метод `before` в політиці. Метод `before` буде виконано перед будь-якими іншими методами політики, що дає вам можливість авторизувати дію до фактичного виклику призначеного метода політики. Цей функціонал найчастіше використовується для авторизації адміністраторів додатка на виконання будь-яких дій:

```php
use App\Models\User;

/**
 * Виконайте перевірку перед авторизацією.
 *
 * @param  \App\Models\User  $user
 * @param  string  $ability
 * @return void|bool
 */
public function before(User $user, $ability)
{
    if ($user->isAdministrator()) {
        return true;
    }
}
```

Якщо ви бажаєте заборонити всі перевірки авторизації для певного типу користувача, ви можете повернути `false` з метода `before`. А якщо повертається значення `null`, перевірка авторизації переходить до наступного метода політики.

> **Warning**  
> Метод `before` класа політики не буде викликаний, якщо клас не містить метода з іменем, який збігається з іменем повноваження, яке перевіряється.

<a name="authorizing-actions-using-policies"></a>

## Авторизація дій за допомогою політик

<a name="via-the-user-model"></a>

### Через модель User

Модель `App\Models\User`, яка входить до вашого додатка Laravel, включає два корисні методи авторизації дій: `can` та `cannot`. Методи `can` і `cannot` отримують назву дії, яку ви бажаєте авторизувати, і відповідну модель. Наприклад, давайте визначимо, чи має користувач право оновлювати певну модель `App\Models\Post`. Як правило, це буде зроблено за допомогою метода контролера:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * Оновіть даний пост.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, Post $post)
    {
        if ($request->user()->cannot('update', $post)) {
            abort(403);
        }

        // Оновити публікацію...
    }
}
```

Якщо для даної моделі `Post` [зареєстровано політику](#registering-policies), метод `can` автоматично викличе відповідну політику та поверне логічний результат. Якщо для моделі не зареєстровано жодної політики, метод `can` спробує викликати шлюз на основі замикання, що відповідає даній назві дії.

<a name="user-model-actions-that-dont-require-models"></a>

#### Дії, які не потребують моделей

Пам’ятайте, що деякі дії можуть відповідати методам політики, таким як `create`, для яких не потрібен екземпляр моделі. У таких ситуаціях ви можете передати назву класа методу `can`. Ім’я класа використовуватиметься для визначення того, яку політику використовувати під час авторизації дії:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * Оновіть даний пост.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request)
    {
        if ($request->user()->cannot('create', Post::class)) {
            abort(403);
        }

        // Оновити публікацію...
    }
}
```

<a name="via-controller-helpers"></a>

### Через додаткові методи контроллера

На додаток до корисних методів, наданих для моделі `App\Models\User`, Laravel надає корисний метод `authorize` для будь-якого з ваших контролерів, який розширює базовий клас `App\Http\Controllers\Controller`.

Як і метод `can` , цей метод приймає назву дії, яку ви бажаєте авторизувати, і відповідну модель. Якщо дія не авторизована, метод `authorize` створить виняток `Illuminate\Auth\Access\AuthorizationException`, який виконавець винятків Laravel автоматично перетворить на HTTP-відповідь з кодом статуса `403`:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * Оновіть вказану публікацію в блозі.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     *
     * @throws \Illuminate\Auth\Access\AuthorizationException
     */
    public function update(Request $request, Post $post)
    {
        $this->authorize('update', $post);

        // Поточний користувач може оновити публікацію в блозі...
    }
}
```

<a name="controller-actions-that-dont-require-models"></a>

#### Дії, які не потребують моделей

Як обговорювалося раніше, деякі методи політики, такі як `create`, не потребують екземпляра моделі. У таких ситуаціях ви повинні передати назву класа методу `authorize`. Ім’я класа використовуватиметься для визначення того, яку політику використовувати під час авторизації дії:

```php
use App\Models\Post;
use Illuminate\Http\Request;

/**
 * Створіть нову публікацію в блозі.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Http\Response
 *
 * @throws \Illuminate\Auth\Access\AuthorizationException
 */
public function create(Request $request)
{
    $this->authorize('create', Post::class);

    // Поточний користувач може створювати дописи в блозі...
}
```

<a name="authorizing-resource-controllers"></a>

#### Авторизація ресурсних контролерів

Якщо ви використовуєте [контролери ресурсів](controllers.md#resource-controllers), ви можете використовувати метод `authorizeResource` в конструкторі вашого контролера. Цей метод приєднає відповідні визначення посередника `can` до методів ресурсного контроллера.

Метод `authorizeResource` приймає ім’я класа моделі першим аргументом, а другим аргументом приймає ім’я параметра маршрута/запита, який міститиме ID моделі. Ви повинні переконатися, що ваш [ресурсний контроллер ](controllers.md#resource-controllers) створено за допомогою прапорця `--model`, щоб він мав необхідні сигнатури методів і оголошені типи:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * Створіть екземпляр контролера.
     *
     * @return void
     */
    public function __construct()
    {
        $this->authorizeResource(Post::class, 'post');
    }
}
```

Наступні методи контролера буде порівняно з відповідним методом політики. Коли запити направляються до даного метода контролера, відповідний метод політики буде автоматично викликаний перед виконанням метода контролера:

| Controller Method | Policy Method |
| ----------------- | ------------- |
| index             | viewAny       |
| show              | view          |
| create            | create        |
| store             | create        |
| edit              | update        |
| update            | update        |
| destroy           | delete        |

> **Note**  
> Ви можете використати команду `make:policy` з опцією `--model`, щоб швидко створити клас політики для певної моделі: `php artisan make:policy PostPolicy --model=Post`.

<a name="via-middleware"></a>

### Через Middleware

Laravel містить посередника, який може авторизувати дії ще до того, як вхідний запит досягне ваших маршрутів або контролерів. За замовчуванням посереднику `Illuminate\Auth\Middleware\Authorize` призначається ключ `can` у вашому класі `App\Http\Kernel`. Давайте розглянемо приклад використання посередника `can` для дозволу користувачеві оновлювати публікацію:

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // Поточний користувач може оновити публікацію...
})->middleware('can:update,post');
```

У цьому прикладі ми передаємо посереднику `can` два аргументи. Перший — це назва дії, яку ми хочемо авторизувати, а другий — параметр маршрута, який ми хочемо передати методу політики. У цьому випадку, оскільки ми використовуємо [неявне прив’язування моделі](routing.md#implicit-binding), модель `App\Models\Post` буде передана методу політики. Якщо користувач не авторизований для виконання даної дії, посередник поверне HTTP-відповідь з кодом статуса `403.`

Для зручності ви також можете приєднати посередник `can` до свого маршрута за допомогою метода `can`:

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // Поточний користувач може оновити публікацію...
})->can('update', 'post');
```

<a name="middleware-actions-that-dont-require-models"></a>

#### Дії, які не потребують моделей

Знову ж таки, деякі методи політики, такі як `create`, не потребують екземпляра моделі. У таких ситуаціях ви можете передати назву класа посереднику. Ім’я класа використовуватиметься для визначення того, яку політику використовувати під час авторизації дії:

```php
Route::post('/post', function () {
    // Поточний користувач може оновити публікацію...
})->middleware('can:create,App\Models\Post');
```

Зазначення рядка повної назви класа у визначенні посиредника може бути незручним. З цієї причини ви можете приєднати посередник `can` до свого маршрута за допомогою метода `can`:

```php
use App\Models\Post;

Route::post('/post', function () {
    // Поточний користувач може оновити публікацію...
})->can('create', Post::class);
```

<a name="via-blade-templates"></a>

### Через шаблони Blade

Під час написання шаблонів Blade ви можете відображати частину сторінки, лише якщо користувач має право виконувати певну дію. Наприклад, ви можете показати форму оновлення для публікації в блозі, лише якщо користувач дійсно може оновити публікацію. У цій ситуації ви можете використовувати директиви `@can` і `@cannot`:

````
```blade
@can('update', $post)
    <!-- Поточний користувач може оновити публікацію... -->
@elsecan('create', App\Models\Post::class)
    <!-- Поточний користувач може створювати нові повідомлення... -->
@else
    <!-- ... -->
@endcan

@cannot('update', $post)
    <!-- Поточний користувач не може оновити публікацію... -->
@elsecannot('create', App\Models\Post::class)
    <!-- Поточний користувач не може створювати нові повідомлення... -->
@endcannot
````

Ці директиви є зручним скороченням для написання операторів `@if` та `@unless`. Наведені вище оператори `@can` і `@cannot` еквівалентні наступним операторам:

```blade
@if (Auth::user()->can('update', $post))
    <!-- Поточний користувач може оновити публікацію... -->
@endif

@unless (Auth::user()->can('update', $post))
    <!-- Поточний користувач не може оновити публікацію... -->
@endunless
```

Ви також можете визначити, чи має користувач право виконувати будь-які дії з заданого масива дій. Для цього скористайтеся директивою `@canany`:

```blade
@canany(['update', 'view', 'delete'], $post)
    <!-- Поточний користувач може оновити, переглянути або видалити публікацію... -->
@elsecanany(['create'], \App\Models\Post::class)
    <!-- Поточний користувач може створити публікацію... -->
@endcanany
```

<a name="blade-actions-that-dont-require-models"></a>

#### Дії, які не потребують моделей

Як і більшість інших методів авторизації, ви можете передати ім’я класа директивам `@can` і `@cannot` якщо для дії не потрібен екземпляр моделі:

```blade
@can('create', App\Models\Post::class)
    <!-- Поточний користувач може створювати публікації... -->
@endcan

@cannot('create', App\Models\Post::class)
    <!-- Поточний користувач не може створювати публікації... -->
@endcannot
```

<a name="supplying-additional-context"></a>

### Надання додаткового контексту

Під час авторизації дій за допомогою політик ви можете передати масив другим аргументом до різних функцій авторизації та помічників. Перший елемент в масиві використовуватиметься для визначення того, яку політику слід викликати, а решта елементів масиву передаються як параметри методу політики та можуть використовуватися для додаткового контексту під час прийняття рішень щодо авторизації. Наприклад, розглянемо `PostPolicy` і таке визначення метода, яке містить додатковий параметр `$category`:

```php
/**
 * Визначити, чи може користувач оновити пост.
 *
 * @param  \App\Models\User  $user
 * @param  \App\Models\Post  $post
 * @param  int  $category
 * @return bool
 */
public function update(User $user, Post $post, int $category)
{
    return $user->id === $post->user_id &&
            $user->canUpdateCategory($category);
}
```

Під час спроби визначити, чи може автентифікований користувач оновити певну публікацію, ми можемо викликати цей метод політики таким чином:

```php
/**
 * Оновіть вказану публікацію в блозі.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  \App\Models\Post  $post
 * @return \Illuminate\Http\Response
 *
 * @throws \Illuminate\Auth\Access\AuthorizationException
 */
public function update(Request $request, Post $post)
{
    $this->authorize('update', [$post, $request->category]);

    // Поточний користувач може оновити публікацію в блозі...
}
```
