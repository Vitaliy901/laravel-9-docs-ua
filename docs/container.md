# Службовий Контейнер

- [Вступ](#introduction)
  - [Неконфігуроване впровадження](#zero-configuration-resolution)
  - [Коли використовувати контейнер](#when-to-use-the-container)
- [Зв'язування](#binding)
  - [Основи зв'язування](#binding-basics)
  - [Зв'язування інтерфейсів з реалізаціями](#binding-interfaces-to-implementations)
  - [Контекстне зв'язування](#contextual-binding)
  - [Примітивні зв'язування](#binding-primitives)
  - [Зв'язування типізованих варіацій](#binding-typed-variadics)
  - [Додавання тегів](#tagging)
  - [Резширення зв'язувань](#extending-bindings)
- [Вилучення](#resolving)
  - [Метод `make`](#the-make-method)
  - [Автоматичне впровадження залежностей](#automatic-injection)
- [Виклик та впровадження метода](#method-invocation-and-injection)
- [Події контейнера](#container-events)
- [PSR-11](#psr-11)

<a name="introduction"></a>

## Вступ

Службовий контейнер Laravel - це потужний інструмент для керування залежностями класів і виконання впроваджень залежностей. Впровадження залежностей - це дивовижний вислів, який по суті означає наступне: класова залежність "впроваджується" в клас через конструктор або, в деяких випадках, через методи "сетери".

Давайте розглянемо простий приклад:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Repositories\UserRepository;
use App\Models\User;

class UserController extends Controller
{
    /**
     * Реалізація сховища користувача.
     *
     * @var UserRepository
     */
    protected $users;

    /**
     * Створити новий екземпляр контроллера.
     *
     * @param  UserRepository  $users
     * @return void
     */
    public function __construct(UserRepository $users)
    {
        $this->users = $users;
    }

    /**
     * Показати профіль переданого користувача.
     *
     * @param  int  $id
     * @return Response
     */
    public function show($id)
    {
        $user = $this->users->find($id);

        return view('user.profile', ['user' => $user]);
    }
}
```

В цьому прикладі, `UserController` потрібно отримати користувачів з джерела даних. Отже, ми **запровадимо** службу, яка здатна отримати користувачів. В цьому контексті, наш `UserRepository`, швидше за все використовує [Eloquent](eloquent.md), щоб отримати інформацію про користувача з бази даних. Однак, оскільки сховище впроваджено, ми здатні легко підмінити його іншою реалізацією. Також ми здатні легко "імітувати", або створювати штучну реалізацію `UserRepository` коли тестуємо наш додаток.

Глибоке розуміння службового контейнера Laravel, має велике значення для створення потужного, великого додатка, а також для внесення вкладу в саме ядро ​​Laravel.

<a name="zero-configuration-resolution"></a>

### Неконфігуроване впровадження

Якщо клас не має заложностей або залежить лише від інших конкретних класів (не інтерфейсів), контейнеру не потрібно вказувати на те як потрібно створювати цей клас. Наприклад, ви можете розмістити наступний код у вашому файлі `routes/web.php`:

```php
<?php

class Service
{
    //
}

Route::get('/', function (Service $service) {
    die(get_class($service));
});
```

В цьому прикладі, постукавшись до маршрута `/` вашого додатка, буде автоматично визначенно клас `Service` та впроваджено до вашого виконавця маршрутів. Це змінює правила гри. Це означає, що ви можете розробляти ваш додаток і користуватися залежністю впровадження без турботи про роздуті конфігураційні файли.

На щастя, багато класів, які ви будете писати під час створення додатка Laravel, автоматично отримають свої залежності через контейнер, включаючи [контролери](controllers.md), [слухачі подій](events.md), [посередник](middleware.md) тощо. Крім того, ви можете оголосити типізацію залежності методу `handle` [завданню в черзі](queues.md). Як тільки ви спробуєте силу автоматичного неконфігурованого впровадження залежностей, вам здаватиметься, що розробка без цього функціоналу неможлива.

<a name="when-to-use-the-container"></a>

### Коли використовувати контейнер

Завдяки неконфігурованому впровадженню, ви часто будете типізувати залежності в маршрутах, контроллерах, слухачах подій, та в інших місцях без жодної ручної взаємодії з контейнером. Наприклад, ви можете оголосити типізацію об'єкта `Illuminate\Http\Request` у вашому визначеному маршруті, щоб ви можете з легкістю отримати доступ до поточно запита. Незважаючи на те, що ми ніколи не взаємодіяли з контейнером, щоб написати цей код, він автоматично керуватиме впровадженням цих залежностей за лаштунками:

```php
use Illuminate\Http\Request;

Route::get('/', function (Request $request) {
    // ...
});
```

В багатьох випадках, завдяки автоматичному впровадженню залежностей та [фасадам](facades.md), ви можете створювати додаток Laravel наівть без необхідності **коли-небудь** вручну щось зв'язувати чи вилучати з контейнера. **Отже, коли б ви вручну спробували взаємодіяти з контейнером?** Давайте розглянемо дві ситуації.

По-перше, якщо ви пишете клас, який реалізує інтерфейс і ви бажаєте оголосити типізацію цього інтерфейса вашому маршруту або конструктору класа, ви повинні [вказати контейнеру, як отримати цей інтерфейс](#binding-interfaces-to-implementations). По-друге, якщо ви [пишете пакунок Laravel](packages.md), яким бажаєте поділитися з іншими Laravel розробниками, вам потрібно буде зв'язати ваш пакунок в службовому контейнері.

<a name="binding"></a>

## Зв'язування

<a name="binding-basics"></a>

### Основи зв'язування

<a name="simple-bindings"></a>

#### Просте зв'язування

Майже всі ваші зв'язування з службовим контейнером будуть реєструватись в [постачальнику служб](providers.md), тому більшість з цих прикладів продемонструють використання контейнера в цьому контексті.

В постачальнику служб, ви завжди маєте доступ до контейнера за допомогою влативості `$this->app`. Ми можемо зареєструвати зв'язування використовуючи метод `bind`, передавши ім'я класа або інтерфейса, які ми бажаємо зареєструвати разом з замиканям, яке повертає екземпляр класа.

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$this->app->bind(Transistor::class, function ($app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

Зверніть увагу на те, що ми отримуємо сам контейнер, як аргумент. Потім, ми можемо використати контейнер для вилучення підзалежностей об'єкта, який ми створюємо.

Як вже згадувалось, ви зазвичай будете взаємодіяти з контейнером в средині постачальника служб; проте, якщо ви бажаєте взаємодіяти з контейнером за межами постачальника служб, ви можете використати [фасад](facades.md) `App`:

```php
use App\Services\Transistor;
use Illuminate\Support\Facades\App;

App::bind(Transistor::class, function ($app) {
    // ...
});
```

> **Note**
> Немає неохбхідності зв'язувати класи в контейнері, якщо вони не мають залежності з будь-якими інтерфейсами. Контейнер не потребує вказівок, що до створення цих об'єктів, оскільки він автоматично буде вилучати об'єкти за допомогою [API рефлексії](https://www.php.net/manual/en/book.reflection.php)

<a name="binding-a-singleton"></a>

#### Зв'язування `Singleton`

Метод `singleton` зв'язує клас або інтерфейс в контейнері, який може бути вилучений лише один раз. Щойно буде отримано зв'язаний "Singleton", той самий екземпляр об'єкта буде повертатись під час наступних викликів контейнера:

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$this->app->singleton(Transistor::class, function ($app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="binding-scoped"></a>

#### Контекстне зв'язування `Singletons`

Метод `scoped` зв'язує клас або інтерфейс в контейнері, який може бути вилучений лише один раз протягом поточного життевого циклу запита або завдання Laravel. Незважаючи на те, що цей метод схожий на `singleton`, зареєстровані екземпляри, які використовують метод `scoped` будуть оновлені щоразу, коли додаток Laravel розпочне новий "життевий цикл". Наприклад, коли виконавець [Laravel Octane](octane.md) опрацьовуватиме новий запит або коли [виконавець черги](queues.md) Laravel опрацьовує нове завдання:

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$this->app->scoped(Transistor::class, function ($app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="binding-instances"></a>

#### Звязування екземплярів

Ви також можете зв'язати існуючий екземпляр об'єкта з контейнером, використовуючи метод `instance`. Наданий екземпляр, завжи буде повертатись під час наступних викликів в контейнері:

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$service = new Transistor(new PodcastParser);

$this->app->instance(Transistor::class, $service);
```

<a name="binding-interfaces-to-implementations"></a>

### Зв'язування інтерфейсів з реалізаціями

Дуже потужним функціоналом службового контейнера - це його здатність зв'язувати інтерфейст з певною реалізацією. Наприклад, припустиму, що ми маємо інтерфейс `EventPusher` і реалізацію `RedisEventPusher`. Після того, як ми написали нашу реалізацію `RedisEventPusher` цього інтерфейса, ми можемо зареєструвати його в службовому контейнері таким чином:

```php
    use App\Contracts\EventPusher;
    use App\Services\RedisEventPusher;

    $this->app->bind(EventPusher::class, RedisEventPusher::class);
```

Це визначення говорить контейнеру, що йому потрібно впровадити `RedisEventPusher` коли клас протребує реалізацію `EventPusher`. Тепер ми можемо оголосити тип інтерейса `EventPusher` в конструкторі будь'якого класа, який роспізнається контейнером. Пам'ятайте, що контроллери, слухачі подій, посередники, та різні інші типи класів додатка Laravel завжди виконуються за допомогою службового контейнера:

```php
use App\Contracts\EventPusher;

/**
 * Створити новий екземпляр класа.
 *
 * @param  \App\Contracts\EventPusher  $pusher
 * @return void
 */
public function __construct(EventPusher $pusher)
{
    $this->pusher = $pusher;
}
```

<a name="contextual-binding"></a>

### Контекстне зв'язування

Іноді ви можете мати два класи, які використовують однаковий інтерфейс, але ви бажаєте впровадити різні реалізації цього інтерфейса в кожний клас. Наприклад, два контроллера можуть залежати від різних реалізацій [контракта](contracts.md) `Illuminate\Contracts\Filesystem\Filesystem`. Laravel пропонує простий та зрозумілий інтерфейс для визначення такої поведінки:

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\UploadController;
use App\Http\Controllers\VideoController;
use Illuminate\Contracts\Filesystem\Filesystem;
use Illuminate\Support\Facades\Storage;

$this->app->when(PhotoController::class)
        ->needs(Filesystem::class)
        ->give(function () {
            return Storage::disk('local');
        });

$this->app->when([VideoController::class, UploadController::class])
        ->needs(Filesystem::class)
        ->give(function () {
            return Storage::disk('s3');
        });
```

<a name="binding-primitives"></a>

### Зв'язування примітивів

Іноді ви можете мати клас, який отримує деякі впровадження класів, але також потрібне впровадження примітивного значення, такого як ціле число.

```php
use App\Http\Controllers\UserController;

$this->app->when(UserController::class)
        ->needs('$variableName')
        ->give($value);
```

Іноді клас боже залежати від масива екземплярів, пов'язаних [тегом](#tagging). Використовуючи метод `giveTagged`, ви можете з легкістю впровадити їх в контейнер:

```php
$this->app->when(ReportAggregator::class)
        ->needs('$reports')
        ->giveTagged('reports');
```

Якщо вам потрібно впровадити значення одного з ваших конфігураційних файлі, ви можете викорстати метод `giveConfig`:

```php
$this->app->when(ReportAggregator::class)
        ->needs('$timezone')
        ->giveConfig('app.timezone');
```

<a name="binding-typed-variadics"></a>

### Зв'язування типізованих варіацій

Іноді, ви можете мати клас, який отримує, масив типізованих об'єктів з використанням змінної кількості аргументів конструктора:

```php
<?php

use App\Models\Filter;
use App\Services\Logger;

class Firewall
{
    /**
     * Екземпляр реєстратора.
     *
     * @var \App\Services\Logger
     */
    protected $logger;

    /**
     * Масив фільтрів.
     *
     * @var array
     */
    protected $filters;

    /**
     * Створити новий екземпляр класа.
     *
     * @param  \App\Services\Logger  $logger
     * @param  array  $filters
     * @return void
     */
    public function __construct(Logger $logger, Filter ...$filters)
    {
        $this->logger = $logger;
        $this->filters = $filters;
    }
}
```

Використовуючи контексне зв'язування, ви можете визначити цю залежність надавши методу `give` замикання, яке повертає масив впроваджених екземплярів `Filter`:

```php
$this->app->when(Firewall::class)
    ->needs(Filter::class)
    ->give(function ($app) {
        return [
            $app->make(NullFilter::class),
            $app->make(ProfanityFilter::class),
            $app->make(TooLongFilter::class),
        ];
    });
```

Для зручності, ви також можете просто надати масив імен класів, які будуть впроваджені контейнером щоразу, коли для `Firewall` потрібні будуть екземпляри `Filter`:

```php
$this->app->when(Firewall::class)
        ->needs(Filter::class)
        ->give([
            NullFilter::class,
            ProfanityFilter::class,
            TooLongFilter::class,
        ]);
```

<a name="variadic-tag-dependencies"></a>

#### Варіативні залежності тегів

Іноді клас може мати варіативну залежність, яка вказує на тип наданого класа (`Report ...$reports`). Використовуючи методи `needs` і `giveTagged`, ви можете з легкістю запровадити всі залежності контейнера, за допомогою [тега](#tagging), для вказаної залежності:

```php
$this->app->when(ReportAggregator::class)
    ->needs(Report::class)
    ->giveTagged('reports');
```

<a name="tagging"></a>

### Додавання тегів

Іноді, вам може знадобитись отримати певні "категорії" зв'язування. Наприклад, як варіант, ви створюєте аналізатор звітності, який отримує масив багатьох різних реалізацій інтерфейса `Report`. Після реєестрації реалізацій `Report`, ви можете підписати їх використовуючи метод `tag`:

```php
$this->app->bind(CpuReport::class, function () {
    //
});

$this->app->bind(MemoryReport::class, function () {
    //
});

$this->app->tag([CpuReport::class, MemoryReport::class], 'reports');
```

Щойно службам буде призначено тег, ви можете з легкістю їх отримати та впровадити за допомогою метода `tagged`:

```php
$this->app->bind(ReportAnalyzer::class, function ($app) {
    return new ReportAnalyzer($app->tagged('reports'));
});
```

<a name="extending-bindings"></a>

### Розширення Зв'язувань

Метод `extend` дозволяє модифікувати вилучені служби. Наприклад, коли сервіс отримано, ви можете запустити додатковий код, щоб прикрасити або налаштувати службу. Метод `extend` приймає два аргументи, службовий клас, який ви розширюєте та замикання, яке повинно повернути змінену службу. Замикання отримує вилучену службу та екземпляр контейнера:

```php
$this->app->extend(Service::class, function ($service, $app) {
    return new DecoratedService($service);
});
```

<a name="resolving"></a>

## Вилучення

<a name="the-make-method"></a>

### Метод `make`

Ви можете використовувати метод `make`, щоб вилучити екземпляр класа з контейнера. Метод `make` приймає ім'я класа або інтерфейса, який ви бажаєте отримати:

```php
use App\Services\Transistor;

$transistor = $this->app->make(Transistor::class);
```

Якщо якісь зележністі вашого класа не можуть бути отримані за допомогою контейнера, ви можете впровадити їх, передавши їх як асоціативний маси в метод `makeWith`. Наприклад, ми можемо вручну передати в конструктор аргумент `$id`, який вимагає сервіс `Transistor`:

```php
use App\Services\Transistor;

$transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);
```

Якщо ви за межами вашого постачальника служб і в місці вашого коду, який не має доступу до змінної `$app`, ви можете використовувати [фасад](facades.md) `App` для вилучення екземпляра класа з контейнера:

```php
use App\Services\Transistor;
use Illuminate\Support\Facades\App;

$transistor = App::make(Transistor::class);
```

Якщо ви бажаєте, щоб саме екземпляр контейнера Laravel був запроваджений в клас, який вилучається контейнером, вам потрібно буде оголосити тип `Illuminate\Container\Container` в конструкторі вашого класа:

```php
use Illuminate\Container\Container;

/**
 * Створити новий екземпляр класа.
 *
 * @param  \Illuminate\Container\Container  $container
 * @return void
 */
public function __construct(Container $container)
{
    $this->container = $container;
}
```

<a name="automatic-injection"></a>

### Автоматична ін'єкція

Крім того, і що важлово, ви можете оголосити тип залежності в конструкторі класа, який вилучається контейнером, включаючи [контроллери](controllers.md), [слухачі подій](events.md), [посередники](middleware.md), та інші. Крім того, ви можете оголосити тип залежності в методі `handle` [завдань в черзі](queues.md). На практиці, саме так повинен контейнер вилучати більшість ваших об'єктіви.

Наприклад, ви оголосили тип сховища, який визначений вашим додатком в конструкторі контроллера. Сховище буде автоматично вилучено та впровадженно в клас:

```php
<?php

namespace App\Http\Controllers;

use App\Repositories\UserRepository;

class UserController extends Controller
{
    /**
     * Екземпляр сховища користувача.
     *
     * @var \App\Repositories\UserRepository
     */
    protected $users;

    /**
     * Створити новий екземпляр контроллера.
     *
     * @param  \App\Repositories\UserRepository  $users
     * @return void
     */
    public function __construct(UserRepository $users)
    {
        $this->users = $users;
    }

    /**
     * Показати корисувача за отриманим ID.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function show($id)
    {
        //
    }
}
```

<a name="method-invocation-and-injection"></a>

## Виклик та впровадження метода

Іноді вам може знадобитись викликати метод екземпляра об’єкта, дозволяючи контейнеру автоматично впровадити залежності цього метода. Наприклад, наданий наступний клас:

```php
<?php

namespace App;

use App\Repositories\UserRepository;

class UserReport
{
    /**
     * Створити новий звіт користувача.
     *
     * @param  \App\Repositories\UserRepository  $repository
     * @return array
     */
    public function generate(UserRepository $repository)
    {
        // ...
    }
}
```

Ви можете викликати метод `generate` за допомогою контейнера наступним чином:

```php
use App\UserReport;
use Illuminate\Support\Facades\App;

$report = App::call([new UserReport, 'generate']);
```

Метод `call` приймає будь-який PHP [callable](https://www.php.net/manual/ru/language.types.callable.php). Метод `call` контейнера може навіть використовуватись для виклику замикання під час автоматичного впровадження його залежностей:

```php
use App\Repositories\UserRepository;
use Illuminate\Support\Facades\App;

$result = App::call(function (UserRepository $repository) {
    // ...
});
```

<a name="container-events"></a>

## Події контейнера

Службовий контейнер ініціює подію кожного разу коли він вилучає об'єкт. Ви можете прослухати цю подію використовуючи метод `resolving`:

```php
use App\Services\Transistor;

$this->app->resolving(Transistor::class, function ($transistor, $app) {
    // Викликається, коли контейнер вилучає об’єкти типу «Транзистор»...
});

$this->app->resolving(function ($object, $app) {
    // Викликається, коли контейнер вилучає об’єкт будь-якого типу...
});
```

Як ви можете бачити, вилучений об'єкт буде передано до замикання, що дозволяє вам встановити будь-які додаткові властивості об'єкту,
перш ніж він буде наданий своєму споживачу.

<a name="psr-11"></a>

## PSR-11

Контейнер служб Laravel реалізує інтерфейс [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container. Тому, ви можете оголостити тип інтерфейсу контейнера PSR-11, щоб отримати екземпляр кеонтейнера Laravel.

```php
use App\Services\Transistor;
use Psr\Container\ContainerInterface;

Route::get('/', function (ContainerInterface $container) {
    $service = $container->get(Transistor::class);

    //
});
```

Буде створено виняток, якщо заданий ідетифікатор не може бути вилучений. Виняток буде екземпляром `Psr\Container\NotFoundExceptionInterface`, якщо ідентифікатор ніколи не був зв'язаний. Якщо ідентифікатор був зв'язаний, але не вдалось вилучити, буде створено екземпляр `Psr\Container\ContainerExceptionInterface`.
