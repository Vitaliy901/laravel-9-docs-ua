# Події

- [Вступ](#introduction)
- [Реєстрація подій та слухачів](#registering-events-and-listeners)
  - [Генерація подій та слухачів](#generating-events-and-listeners)
  - [Реєстрація подій вручну](#manually-registering-events)
  - [Автоматичне виявлення подій](#event-discovery)
- [Визначення подій](#defining-events)
- [Визначення слухачів](#defining-listeners)
- [Слухачі подій у черзі](#queued-event-listeners)
  - [Взаємодія з чергою вручну](#manually-interacting-with-the-queue)
  - [Слухачі подій у черзі та транзакції бази даних](#queued-event-listeners-and-database-transactions)
  - [Обробка невдалих завдань](#handling-failed-jobs)
- [Відправка подій](#dispatching-events)
- [Підписники подій](#event-subscribers)
  - [Написання підписників на події](#writing-event-subscribers)
  - [Реєстрація підписників подій](#registering-event-subscribers)

<a name="introduction"></a>

## Вступ

Події Laravel забезпечують просту реалізацію шаблона спостерігача, що дозволяє вам підписуватися та слухати різні події, які відбуваються у вашому додатку. Класи подій зазвичай зберігаються в каталозі `app/Events`, тоді як їх слухачі зберігаються в `app/Listeners`. Не хвилюйтеся, якщо ви не бачите ці каталоги у своєму додатку, оскільки вони будуть створені для вас під час створення подій і слухачів за допомогою консольної команд Artisan.

Події служать чудовим способом відокремити різні аспекти вашого додатка, оскільки одна подія може мати кілька слухачів, які не залежать один від одного. Наприклад, ви можете надсилати повідомлення Slack своєму користувачеві кожного разу, коли замовлення відправляється. Замість того, щоб поєднувати код обробки замовлення з кодом повідомлення Slack, ви можете викликати подію `App\Events\OrderShipped`, яку слухач може отримати та використати для надсилання повідомлення Slack.

<a name="registering-events-and-listeners"></a>

## Реєстрація подій та слухачів

`App\Providers\EventServiceProvider`, що входить до вашого додатка Laravel, забезпечує зручне місце для реєстрації всіх слухачів подій вашого додатка. Властивість `listen` містить масив всіх подій (ключі) та їх слухачів (значення). Ви можете додати стільки подій до цього масиву, скільки вимагає ваш додаток. Наприклад, давайте додамо подію `OrderShipped`:

```php
use App\Events\OrderShipped;
use App\Listeners\SendShipmentNotification;

/**
 * Мапа слухачів подій для додатка.
 *
 * @var array
 */
protected $listen = [
    OrderShipped::class => [
        SendShipmentNotification::class,
    ],
];
```

> **Note**  
> Команда `event:list` може використовуватися для відображення списку всіх подій і слухачів, зареєстрованих вашим додатком.

<a name="generating-events-and-listeners"></a>

### Створення подій і слухачів

Звичайно, ручне створення файлів для кожної події та слухача може втомлювати. Замість цього додайте слухачів та події до свого `EventServiceProvider` і використовуйте команду `event:generate` Artisan. Ця команда генеруватиме будь-які події або слухачі, перелічені у вашому `EventServiceProvider`, які ще не існують:

```shell
php artisan event:generate
```

Крім того, ви можете використовувати команди `make:event` і `make:listener` Artisan для створення окремих подій і слухачів:

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

<a name="manually-registering-events"></a>

### Реєстрація подій вручну

Як правило, події слід реєструвати через масив `EventServiceProvider` `$listen`; однак ви також можете зареєструвати слухачів подій на основі класа або замикання в методі `boot` вашого `EventServiceProvider`:

```php
use App\Events\PodcastProcessed;
use App\Listeners\SendPodcastNotification;
use Illuminate\Support\Facades\Event;

/**
 * Реєстрація будь-яких інших подій вашого додатка.
 *
 * @return void
 */
public function boot()
{
    Event::listen(
        PodcastProcessed::class,
        [SendPodcastNotification::class, 'handle']
    );

    Event::listen(function (PodcastProcessed $event) {
        //
    });
}
```

<a name="queuable-anonymous-event-listeners"></a>

#### Анонімні слухачі подій, які ставляться в чергу

Реєструючи слухачів подій на основі замикання, ви можете загорнути замикання слухача у функцію `Illuminate\Events\queueable`, після чого Laravel зрозуміє, що потрібно виконати слухача за допомогою [черги](queues.md):

```php
use App\Events\PodcastProcessed;
use function Illuminate\Events\queueable;
use Illuminate\Support\Facades\Event;

/**
 * Реєстрація будь-яких інших подій вашого додатка.
 *
 * @return void
 */
public function boot()
{
    Event::listen(queueable(function (PodcastProcessed $event) {
        //
    }));
}
```

Подібно до завдань у черзі, ви можете використовувати `onConnection`, `onQueue`, та методи `delay` , щоб налаштувати виконання слухача в черзі:

```php
Event::listen(queueable(function (PodcastProcessed $event) {
    //
})->onConnection('redis')->onQueue('podcasts')->delay(now()->addSeconds(10)));
```

Якщо ви бажаєте обробляти анонімні збої слухачів у черзі, ви можете надати замикання методу `catch` під час визначення слухача в `queueable`. Це замикання отримає екземпляр події та екземпляр `Throwable`, який спричинив збій слухача:

```php
use App\Events\PodcastProcessed;
use function Illuminate\Events\queueable;
use Illuminate\Support\Facades\Event;
use Throwable;

Event::listen(queueable(function (PodcastProcessed $event) {
    //
})->catch(function (PodcastProcessed $event, Throwable $e) {
    // Помилка прослуховування в черзі...
}));
```

<a name="wildcard-event-listeners"></a>

#### Анонімні слухачі групи подій

Ви навіть можете зареєструвати слухачів, використовуючи `*`, це дозволить вам перехоплювати декілька подій на тому самому слухачі. Слухач, зареєстрований за допомогою даного синтаксису, отримує ім'я поточної події як перший аргумент і весь масив даних події другим аргументом:

```php
Event::listen('event.*', function ($eventName, array $data) {
    //
});
```

<a name="event-discovery"></a>

### Автоматичне виявлення подій

Замість того, щоб реєструвати події та слухачів вручну в масиві `$listen` `EventServiceProvider`, ви можете ввімкнути автоматичне виявлення подій. Якщо виявлення подій увімкнено, Laravel автоматично знайде та зареєструє ваші події та слухачів, скануючи каталог `Listeners` вашого додатка. Крім того, будь-які явно визначені події, перелічені в `EventServiceProvider`, все одно будуть зареєстровані.

Laravel знаходить слухачів подій, скануючи класи слухачів за допомогою служб рефлексії PHP. Коли Laravel знаходить будь-який метод класа слухача, який починається з `handle` або `__invoke`, Laravel зареєструє ці методи як слухачі подій для події, яка вказана в сигнатурі методу:

```php
use App\Events\PodcastProcessed;

class SendPodcastNotification
{
    /**
     * Орпацювати дану подію.
     *
     * @param  \App\Events\PodcastProcessed  $event
     * @return void
     */
    public function handle(PodcastProcessed $event)
    {
        //
    }
}
```

Виявлення подій вимкнено за замовчуванням, але ви можете ввімкнути його, перевизначивши метод `shouldDiscoverEvents` `EventServiceProvider` вашого додатка:

```php
/**
 * Визначте, чи слід автоматично виявляти події та слухачів.
 *
 * @return bool
 */
public function shouldDiscoverEvents()
{
    return true;
}
```

За замовчуванням всі слухачі в каталозі `app/Listeners` вашого додатка будуть скановані. Якщо ви бажаєте додати додаткові каталоги для сканування, ви можете визначити метод `discoverEventsWithin` у своєму `EventServiceProvider`:

```php
/**
 * Отримайте каталоги слухачів, які слід використовувати для виявлення подій.
 *
 * @return array
 */
protected function discoverEventsWithin()
{
    return [
        $this->app->path('Listeners'),
    ];
}
```

<a name="event-discovery-in-production"></a>

#### Кешування подій у експлуатаційному режимі

У експлуатаційному оточенні для фреймворка не ефективно сканувати всіх ваших слухачів за кожним запитом. Тому під час процесу розгортання вам слід запустити команду `event:cache` Artisan, щоб кешувати маніфест всіх подій вашого додатка та слухачів. Цей маніфест використовуватиметься фреймворком для прискорення процесу реєстрації події. Команда `event:clear` може бути використана для знищення кешу.

<a name="defining-events"></a>

## Визначення подій

Клас події — це, по суті, контейнер даних, який містить інформацію, пов’язану з подією. Наприклад, припустімо, що подія `App\Events\OrderShipped` отримує об’єкт [Eloquent ORM](eloquent.md):

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderShipped
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Екземпляр замовлення.
     *
     * @var \App\Models\Order
     */
    public $order;

    /**
     * Створіть новий екземпляр події..
     *
     * @param  \App\Models\Order  $order
     * @return void
     */
    public function __construct(Order $order)
    {
        $this->order = $order;
    }
}
```

Як бачите, цей клас подій не містить логіки. Це контейнер для екземпляра App\Models\Order, який було придбано. Трейт `SerializesModels`, який використовується подією, витончено серіалізує будь-які моделі Eloquent, якщо об’єкт події серіалізовано за допомогою функції `serialize` PHP, наприклад, коли використовуються [слухачі в черзі](#queued-event-listeners).

<a name="defining-listeners"></a>

## Визначення слухачів

Далі розглянемо слухача для нашого прикладу події. Слухачі подій отримують екземпляри подій у своєму методі `handle`. Команди `event:generate` і `make:listener` Artisan автоматично імпортують відповідний клас події і вставлять подію в методі `handle`. У межах метода `handle` ви можете виконувати будь-які дії, необхідні для відповіді на подію:

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;

class SendShipmentNotification
{
    /**
     * Створіть слухача події.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Опрацювати подією.
     *
     * @param  \App\Events\OrderShipped  $event
     * @return void
     */
    public function handle(OrderShipped $event)
    {
        // Доступ до замовлення за допомогою $event->order...
    }
}
```

> **Note**  
> У конструкторі слухачів подій можуть бути оголошені будь-які необхідні типи залежностей. Всі слухачі подій вирішуються через [контейнер служби](container.md) Laravel, тому залежності буде впроваджено автоматично.

<a name="stopping-the-propagation-of-an-event"></a>

#### Припинення розповсюдження події

Іноді, за бажанням ви можете зупинити розповсюдження події серед інших слухачів. Ви можете зробити це, повернувши `false` з метода `handle` вашого слухача.

<a name="queued-event-listeners"></a>

## Слухачі подій в черзі

Постановка слухачів в чергу може бути корисним, якщо ваш слухач збирається виконувати повільне завдання, наприклад надсилання електронного листа або виконання HTTP-запита. Перш ніж використовувати слухачів в черзі, переконайтеся, що ви [налаштували свою чергу](queues.md) та запустили виконавця черги на своєму сервері або в локальному середовищі розробки.

Щоб позначити, що слухач має бути поставлений в чергу, додайте інтерфейс `ShouldQueue` до класу слухача. Слухачі, згенеровані командами `event:generate` та `make:listener` Artisan, вже мають цей інтерфейс, імпортований у поточний простір імен, тому ви можете використовувати його негайно:

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue
{
    //
}
```

Ось і все! Тепер, коли подія, яку обробляє цей слухач, відправляється, цей слухач буде автоматично поставлено в чергу диспетчером подій за допомогою [системи черги](queues.md) Laravel. Якщо під час виконання слухача в черзі не виникає жодних винятків, завдання у черзі буде автоматично видалено після завершення обробки.

<a name="customizing-the-queue-connection-queue-name"></a>

#### Налаштування підключення до черги та назви черги

Якщо ви бажаєте налаштувати підключення до черги, назву черги або час затримки в черзі слухачу подій, ви можете визначити властивості `$connection`, `$queue` або `$delay` у своєму класі слухача:

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendShipmentNotification implements ShouldQueue
{
    /**
     * Ім’я підключення, до якого має бути надіслано завдання.
     *
     * @var string|null
     */
    public $connection = 'sqs';

    /**
     * Ім’я черги, до якої має бути надіслано завдання.
     *
     * @var string|null
     */
    public $queue = 'listeners';

    /**
     * Час у (секундах) перед тим як завдання повинно виконатись.
     *
     * @var int
     */
    public $delay = 60;
}
```

Якщо ви бажаєте визначити підключення черги слухача або назву черги під час виконання, ви можете визначити методи `viaConnection` або `viaQueue` для слухача:

```php
/**
 * Отримати назву підключення черги слухача.
 *
 * @return string
 */
public function viaConnection()
{
    return 'sqs';
}

/**
 * Отримати ім'я черги слухача.
 *
 * @return string
 */
public function viaQueue()
{
    return 'listeners';
}
```

<a name="conditionally-queueing-listeners"></a>

#### Умовне відправлення слухачів в чергу

Іноді вам може знадобитися визначити, чи потрібно поставити слухача в чергу на основі деяких даних, які доступні лише під час виконання. Щоб досягти цього, метод `shouldQueue` може бути доданий до слухача, щоб визначити, чи слід ставити слухача в чергу. Якщо метод `shouldQueue` повертає `false`, слухача не буде виконано:

```php
<?php

namespace App\Listeners;

use App\Events\OrderCreated;
use Illuminate\Contracts\Queue\ShouldQueue;

class RewardGiftCard implements ShouldQueue
{
    /**
     * Нагородити покупця подарунковою карткою.
     *
     * @param  \App\Events\OrderCreated  $event
     * @return void
     */
    public function handle(OrderCreated $event)
    {
        //
    }

    /**
     * Визначити, чи слід ставити слухача у чергу.
     *
     * @param  \App\Events\OrderCreated  $event
     * @return bool
     */
    public function shouldQueue(OrderCreated $event)
    {
        return $event->order->subtotal >= 5000;
    }
}
```

<a name="manually-interacting-with-the-queue"></a>

### Взаємодія з чергою вручну

Якщо вам потрібно вручну отримати доступ до методів `delete` та `release` основного завдання в черзі слухача, ви можете зробити це за допомогою трейта `Illuminate\Queue\InteractsWithQueue`. Ця властивість імпортується за замовчуванням у згенеровані слухачі та надає доступ до таких методів:

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * Опрацювати подію.
     *
     * @param  \App\Events\OrderShipped  $event
     * @return void
     */
    public function handle(OrderShipped $event)
    {
        if (true) {
            $this->release(30);
        }
    }
}
```

<a name="queued-event-listeners-and-database-transactions"></a>

### Слухачі подій в черзі та транзакції бази даних

Коли слухачі в черзі відправляються в рамках транзакцій бази даних, вони можуть бути опрацьовані чергою до того, як транзакція бази даних буде зафіксована. Коли це станеться, будь-які оновлення, які ви внесли в моделі або записи бази даних під час транзакції бази даних, можуть ще не відображатися в базі даних. Крім того, будь-які моделі або записи бази даних, створені в рамках транзакції, можуть не існувати в базі даних. Якщо ваш слухач залежить від цих моделей, можуть виникнути непередбачені помилки під час виконання завдання, яке надсилає поставлений у чергу слухач.

Якщо для параметра `after_commit` конфігурації вашого підключення до черги встановлено значення `false`, ви все ще можете вказати, що певний слухач у черзі має бути відправлений після того, як всі транзакції відкритої бази даних будуть зафіксовані, визначивши властивість `$afterCommit` у класі слухача:

```php
<?php

namespace App\Listeners;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    public $afterCommit = true;
}
```

> **Note**  
> Щоб дізнатися більше про вирішення цих проблем, перегляньте документацію щодо [завдань в черзі та транзакцій бази даних](queues.md#jobs-and-database-transactions).

<a name="handling-failed-jobs"></a>

### Обробка невдалих завдань

Іноді ваші слухачі подій у черзі можуть невиконатися. Якщо слухач у черзі перевищує максимальну кількість спроб, визначену вашим виконавцем черги, для вашого слухача буде викликано `failed` метод. `failed` метод отримує екземпляр події та `Throwable`, який спричинив збій:

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * Опрацвати подію.
     *
     * @param  \App\Events\OrderShipped  $event
     * @return void
     */
    public function handle(OrderShipped $event)
    {
        //
    }

    /**
     * Опрацювати невиконане завдання.
     *
     * @param  \App\Events\OrderShipped  $event
     * @param  \Throwable  $exception
     * @return void
     */
    public function failed(OrderShipped $event, $exception)
    {
        //
    }
}
```

<a name="specifying-queued-listener-maximum-attempts"></a>

#### Визначення максимальної кількості спроб слухача в черзі

Якщо один із ваших слухачів в черзі зустрічється з помилкою, ви, ймовірно, не хочете, щоб він продовжував повторюватись нескінченно довго. Таким чином, Laravel надає різні способи вказати, скільки разів або як довго може повторюватись слухач.

Ви можете визначити властивість `$tries` у своєму класі слухача, щоб вказати, скільки разів можна спробувати виконати слухача, перш ніж він буде вважатися невдалим:

```php
<?php

namespace App\Listeners;

use App\Events\OrderShipped;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendShipmentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * Кількість разів, які можна спробувати виконати слухача в черзі .
     *
     * @var int
     */
    public $tries = 5;
}
```

В якості альтернативи визначенню того, скільки разів можна спробувати виконати слухача, перш ніж він зазнає невдачі, ви можете визначити час, після якого слухач не повторюватиметься. Це дозволяє слухачу повторюватися будь-яку кількість разів протягом заданого періоду часу. Щоб визначити час, після якого слухач більше не повинен виконуватись, додайте метод `retryUntil` до свого класу слухача. Цей метод має повертати екземпляр `DateTime`:

```php
/**
 * Визначте час очікування слухача.
 *
 * @return \DateTime
 */
public function retryUntil()
{
    return now()->addMinutes(5);
}
```

<a name="dispatching-events"></a>

## Відправлення подій

Щоб відправити подію, ви можете викликати статичний метод `dispatch` події. Цей метод стає доступним у події за допомогою трейта `Illuminate\Foundation\Events\Dispatchable`. Будь-які аргументи, передані в метод `dispatch`, будуть передані конструктору події:

```php
<?php

namespace App\Http\Controllers;

use App\Events\OrderShipped;
use App\Http\Controllers\Controller;
use App\Models\Order;
use Illuminate\Http\Request;

class OrderShipmentController extends Controller
{
    /**
     * Відправити надане замовлення.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request)
    {
        $order = Order::findOrFail($request->order_id);

        // Order shipment logic...

        OrderShipped::dispatch($order);
    }
}
```

Якщо ви хочете відправити подію через умову, ви можете використовувати методи `dispatchIf` і `dispatchUnless`:

```php
OrderShipped::dispatchIf($condition, $order);

OrderShipped::dispatchUnless($condition, $order);
```

> **Note**
> Під час тестування може бути корисним стверджувати, що певні події були відправлені без фактичного запуску їх слухачів. [Вбудовані помічники тестування](mocking.md#event-fake) Laravel спрощують це.

<a name="event-subscribers"></a>

## Підписники подій

<a name="writing-event-subscribers"></a>

### Написання підписників на події

Підписники подій — це класи, які можуть підписуватися на декілька подій із самого класу підписника, дозволяючи вам визначати декілька виконавців подій в одному класі. Підписники повинні визначити метод `subscribe`, якому буде передано екземпляр диспетчера подій. Ви можете викликати метод `listen` у даному диспетчері, щоб зареєструвати слухачів подій:

```php
<?php

namespace App\Listeners;

use Illuminate\Auth\Events\Login;
use Illuminate\Auth\Events\Logout;

class UserEventSubscriber
{
    /**
     * Опрацювати події входу користувачів.
     */
    public function handleUserLogin($event) {}

    /**
     * Опрацювати події виходу користувачів.
     */
    public function handleUserLogout($event) {}

    /**
     * Реєстрація слухачів для підписника.
     *
     * @param  \Illuminate\Events\Dispatcher  $events
     * @return void
     */
    public function subscribe($events)
    {
        $events->listen(
            Login::class,
            [UserEventSubscriber::class, 'handleUserLogin']
        );

        $events->listen(
            Logout::class,
            [UserEventSubscriber::class, 'handleUserLogout']
        );
    }
}
```

Якщо ваші методи слухачів подій визначені в самому підписнику, вам може буде зручніше повертати масив подій та імен методів із метода `subscribe` пидписника. Laravel автоматично визначить ім’я класу підписника під час реєстрації слухачів подій:

```php
<?php

namespace App\Listeners;

use Illuminate\Auth\Events\Login;
use Illuminate\Auth\Events\Logout;

class UserEventSubscriber
{
    /**
     * Опрацювати події входу користувачів.
     */
    public function handleUserLogin($event) {}

    /**
     * Опрацювати події виходу користувачів.
     */
    public function handleUserLogout($event) {}

    /**
     * Реєстрація слухачів для підписника.
     *
     * @param  \Illuminate\Events\Dispatcher  $events
     * @return array
     */
    public function subscribe($events)
    {
        return [
            Login::class => 'handleUserLogin',
            Logout::class => 'handleUserLogout',
        ];
    }
}
```

<a name="registering-event-subscribers"></a>

### Реєстрація підписників подій

Написавши підписника, ви готові зареєструвати його у диспетчера подій. Ви можете зареєструвати підписників за допомогою властивості `$subscribe` у `EventServiceProvider`. Наприклад, давайте додамо `UserEventSubscriber` до списку:

```php
<?php

namespace App\Providers;

use App\Listeners\UserEventSubscriber;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    /**
     * Мапа слухачів подій для додатка.
     *
     * @var array
     */
    protected $listen = [
        //
    ];

    /**
     * Класи абонентів для реєстрації.
     *
     * @var array
     */
    protected $subscribe = [
        UserEventSubscriber::class,
    ];
}
```
