# Повідомлення

- [Вступ](#introduction)
- [Створення повідомлень](#generating-notifications)
- [Відправлення повідомлень](#sending-notifications)
  - [Використання трейта `Notifiable`](#using-the-notifiable-trait)
  - [Використання фасада `Notification`](#using-the-notification-facade)
  - [Визначення каналів доставки](#specifying-delivery-channels)
  - [Повідомлення в черзі](#queueing-notifications)
  - [Повідомлення на вимогу](#on-demand-notifications)
- [Поштові повідомлення](#mail-notifications)
  - [Формування поштових повідомлень](#formatting-mail-messages)
  - [Налаштування відправника](#customizing-the-sender)
  - [Налаштування одержувача](#customizing-the-recipient)
  - [Налаштування теми повідомлення](#customizing-the-subject)
  - [Налаштування поштового драйвера](#customizing-the-mailer)
  - [Налаштування шаблонів](#customizing-the-templates)
  - [Поштові влкадення](#mail-attachments)
  - [Додавання тегів і метаданих](#adding-tags-metadata)
  - [Налаштування повідомлення Symfony](#customizing-the-symfony-message)
  - [Використання поштових відправлень](#using-mailables)
  - [Попередній перегляд поштових повідомлень](#previewing-mail-notifications)
- [Поштові повідомлення Markdown](#markdown-mail-notifications)
  - [Створення повідомлення](#generating-the-message)
  - [Написання повідомлення](#writing-the-message)
  - [Налаштування компонентів](#customizing-the-components)
- [Повідомлення через канал database](#database-notifications)
  - [Попередня підготовка каналу database](#database-prerequisites)
  - [Формування повідомлень каналу database](#formatting-database-notifications)
  - [Доступ до повідомлень](#accessing-the-notifications)
  - [Маркування прочитаних повідомлень](#marking-notifications-as-read)
- [Трансляція повідомлень](#broadcast-notifications)
  - [Попередня підготовка каналу broadcast](#broadcast-prerequisites)
  - [Формування повідомлень, які транслюються.](#formatting-broadcast-notifications)
  - [Прослуховування повідомлень, які транслюються.](#listening-for-notifications)
- [Повідомлення через SMS](#sms-notifications)
  - [Попередня підготовка канала SMS](#sms-prerequisites)
  - [Формування повідомлень через SMS](#formatting-sms-notifications)
  - [Використання шорткодів при формуванні SMS-повідомлень](#formatting-shortcode-notifications)
  - [Зміна номера відправника](#customizing-the-from-number)
  - [Додавання посилання на клієнта](#adding-a-client-reference)
  - [Маршрутизація SMS-повідомлень](#routing-sms-notifications)
- [Повідомлення через Slack](#slack-notifications)
  - [Попередня підготовка каналу Slack](#slack-prerequisites)
  - [Формування повідомлення через Slack](#formatting-slack-notifications)
  - [Вкладення Slack-повідомлень](#slack-attachments)
  - [Маршрутизація Slack-повідомлень](#routing-slack-notifications)
- [Локалізація повідомлень](#localizing-notifications)
- [Повідомлення подій](#notification-events)
- [Налаштування власних каналів повідомлень](#custom-channels)

<a name="introduction"></a>

## Вступ

На додачу до підтримки [відправлень електронної пошти](mail.md), Laravel забезпечує підтримку відправлення повідомлень через різноманітні канали доставки, включаючи електронну пошту, SMS (через [Vonage](https://www.vonage.com/communications-apis/), раніше відомий як Nexmo) і [Slack](https://slack.com). Крім того, [спільнотою було створено безліч каналів повідомлень](https://laravel-notification-channels.com/about/#suggesting-a-new-channel) для надсилання повідомлень десятками різних каналів! Повідомлення також можна зберегти в базі даних, щоб вони могли відображатися у вашому веб-інтерфейсі.

Як правило, повідомлення мають бути короткими інформаційними повідомленнями, які повідомляють користувачів про те, що сталося у вашому додатку. Наприклад, якщо ви пишете платіжний додаток, ви можете надіслати сповіщення «Рахунок оплачено» своїм користувачам електронною поштою та SMS-каналами.

<a name="generating-notifications"></a>

## Створення повідомлень

У Laravel кожне повідомлення представлено одним класом, який зазвичай зберігається в каталозі `app/Notifications`. Не хвилюйтеся, якщо ви не бачите цей каталог у своєму додатку – він буде створений для вас, коли ви запустите команду `make:notification` Artisan:

```shell
php artisan make:notification InvoicePaid
```

Ця команда розмістить новий клас повідомлень у вашому каталозі `app/Notifications`. Кожен клас повідомлень містить метод `via` та змінну кількість методів створення повідомлень, наприклад `toMail` або `toDatabase`, які перетворюють повідомлення адаптуючи його для цього конкретного каналу.

<a name="sending-notifications"></a>

## Відправлення повідомлень

<a name="using-the-notifiable-trait"></a>

### Використання трейта `Notifiable`

Повідомлення можна надсилати двома способами: за допомогою метода `notify` трейта `Notifiable` або за допомогою [фасада](facades.md) `Notification` . Трейт `Notifiable`, за замовчуванням присутній в моделі `App\Models\User` вашого додатка:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;
}
```

Метод `notify`, наданий цим трейтом, очікує отримання екземпляра повідомлення:

```php
use App\Notifications\InvoicePaid;

$user->notify(new InvoicePaid($invoice));
```

> **Note**  
> Пам’ятайте, що ви можете використовувати трейт `Notifiable` на будь-якій своїй моделі. Ви не обмежені лише включенням його у свою модель `User`.

<a name="using-the-notification-facade"></a>

### Використання фасада `Notification`

Крім того, ви можете надсилати повідомлення через [фасад](facades.md) `Notification`. Цей підхід корисний при відправленні повідомлення декільком об'єктам які потрібно повідомити, наприклад, групі користувачів. Щоб надіслати повідомлення за допомогою фасада, передайте всі об’єкти, які потрібно повідомити, та екземпляр повідомлення методу `send`:

```php
use Illuminate\Support\Facades\Notification;

Notification::send($users, new InvoicePaid($invoice));
```

Ви також можете негайно відправляти повідомлення за допомогою метода `sendNow`. Цей метод негайно надішле повідомлення, навіть якщо повідомлення реалізує інтерфейс `ShouldQueue`:

```php
Notification::sendNow($developers, new DeploymentCompleted($deployment));
```

<a name="specifying-delivery-channels"></a>

### Визначення каналів доставки

Кожен клас повідомлення має метод `via`, який визначає, на які канали повідомлення буде доставлено. Повідомлення можуть надсилатися на такі канали: `mail`, `database`, `broadcast`, `vonage`, and `slack`.

> **Note**  
> Якщо ви хочете використовувати інші канали доставки, такі як Telegram або Pusher, відвідайте веб-сайт спільноти [Laravel Notification Channels website](http://laravel-notification-channels.com).

Метод `via` отримує екземпляр `$notifiable`, який буде екземпляром класа, якому надсилається повідомлення. Ви можете використовувати `$notifiable`, щоб визначити, на які канали має бути доставлено повідомлення:

```php
/**
 * Отримати канали доставки повідомлень.
 *
 * @param  mixed  $notifiable
 * @return array
 */
public function via($notifiable)
{
    return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];
}
```

<a name="queueing-notifications"></a>

### Повідомлення в черзі

> **Warning**  
> Перш ніж поставити повідомлення в чергу, вам слід налаштувати чергу та [запустити виконавця черги](queues.md).

Відправлення повідомлень може зайняти деякий час, особливо якщо каналу потрібно зробити виклик зовнішнього API, щоб доставити повідомлення. Щоб прискорити час відповіді вашого додатка, додайте ваше повідомлення в чергу, використавши інтерфейс `ShouldQueue` і трейт `Queueable` до свого класу повідомлення. Інтерфейс і трейт вже імпортовані для всіх повідомлень, створених за допомогою команди `make:notification`, тому ви можете негайно додати їх до свого класу повідомлень:

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    // ...
}
```

Після додавання інтерфейса `ShouldQueue` до вашого повідомлення ви можете надіслати повідомлення як зазвичай. Laravel виявить інтерфейс `ShouldQueue` в класі та автоматично поставить в чергу доставки повідомлення:

```php
$user->notify(new InvoicePaid($invoice));
```

Під час постановки повідомлення в чергу буде створено завдання в черзі для кожної комбінації одержувача та канала. Наприклад, шість завдань буде відправлено в чергу, якщо ваше повідомлення має трьох одержувачів і два канали.

<a name="delaying-notifications"></a>

#### Затримка повідомлень в черзі

Якщо ви хочете відкласти доставку повідомлення, ви можете прив’язати метод `delay` до свого екземпляра повідомлення:

```php
$delay = now()->addMinutes(10);

$user->notify((new InvoicePaid($invoice))->delay($delay));
```

<a name="delaying-notifications-per-channel"></a>

#### Затримка повідомлень для певного канала

Ви можете передати масив до метода `delay`, щоб визначити величину затримки для певних каналів:

```php
$user->notify((new InvoicePaid($invoice))->delay([
    'mail' => now()->addMinutes(5),
    'sms' => now()->addMinutes(10),
]));
```

Крім того, ви можете визначити метод `withDelay` у самому класі повідомлення. Метод `withDelay` має повертати масив назв каналів і значень затримки:

```php
/**
 * Визначте затримку для доставки повідомлення.
 *
 * @param  mixed  $notifiable
 * @return array
 */
public function withDelay($notifiable)
{
    return [
        'mail' => now()->addMinutes(5),
        'sms' => now()->addMinutes(10),
    ];
}
```

<a name="customizing-the-notification-queue-connection"></a>

#### Налаштування підключення черги для повідомлення

За замовчуванням повідомлення в черзі ставитимуться в чергу за допомогою стандартного підключення черги вашого додатка. Якщо ви хочете вказати інше підключення, яке має використовуватися для певного повідомлення, ви можете визначити властивість `$connection` у класі повідомлення:

```php
/**
 * Ім’я підключення до черги, яке буде використовуватися під час постановки повідомлення в чергу.
 *
 * @var string
 */
public $connection = 'redis';
```

<a name="customizing-notification-channel-queues"></a>

#### Налаштування канала черги повідомленя

Якщо ви бажаєте вказати конкретну чергу, яка має використовуватися для кожного канала повідомлення, який підтримує повідомлення, ви можете визначити метод `viaQueues` у своєму повідомленні. Цей метод повинен повертати масив пар ім'я каналу/ім'я черги:

```php
/**
 * Визначте, які черги слід використовувати для кожного каналу повідомлення.
 *
 * @return array
 */
public function viaQueues()
{
    return [
        'mail' => 'mail-queue',
        'slack' => 'slack-queue',
    ];
}
```

<a name="queued-notifications-and-database-transactions"></a>

#### Повідомлення в черзі та транзакції в базі даних

Коли повідомлення в черзі відправляться в рамках транзакцій бази даних, вони можуть бути оброблені чергою до того, як транзакція бази даних буде зафіксована. Коли це станеться, будь-які оновлення, які ви внесли в моделі або записи бази даних під час транзакції бази даних, можуть ще не відображатися в базі даних. Крім того, будь-які моделі або записи бази даних, створені в рамках транзакції, можуть не існувати в базі даних. Якщо ваше повідомлення залежить від цих моделей, можуть виникнути непередбачувані помилки під час виконання завдання, яке надсилає повідомлення.

Якщо для параметра конфігурації `after_commit` підключення до черги встановлено значення `false`, ви все ще можете вказати, що конкретне повідомлення в черзі має бути відправлено після того, як всі транзакції відкритої бази даних будуть зафіксовані, викликавши метод `afterCommit` під час надсилання повідомлення:

```php
use App\Notifications\InvoicePaid;

$user->notify((new InvoicePaid($invoice))->afterCommit());
```

Крім того, ви можете викликати метод `afterCommit` з конструктора повідомлень:

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    /**
     * Створіть новий екземпляр повідомлення.
     *
     * @return void
     */
    public function __construct()
    {
        $this->afterCommit();
    }
}
```

> **Note**  
> Щоб дізнатися більше про вирішення цих проблем, перегляньте [документацію щодо завдань у черзі та транзакцій бази даних](queues.md#jobs-and-database-transactions).

<a name="determining-if-the-queued-notification-should-be-sent"></a>

#### Визначення необхідності надсилання повідомлення в черзі

Після відправки повідомлення в чергу для фонового виконання, воно зазвичай приймається виконавцем черги і далі надсилається призначеному одержувачу.

Однак, якщо ви бажаєте остаточно визначити, чи слід надсилати заплановане повідомлення після того, як воно виконано виконавцем черги, ви можете визначити метод `shouldSend` в класі повідомлення. Якщо цей метод повертає `false`, повідомлення не буде надіслано:

```php
    /**
     * Determine if the notification should be sent.
     *
     * @param  mixed  $notifiable
     * @param  string  $channel
     * @return bool
     */
    public function shouldSend($notifiable, $channel)
    {
        return $this->invoice->isPaid();
    }
```

<a name="on-demand-notifications"></a>

### Повідомлення на вимогу

Іноді вам може знадобитися відправляти повідомлення комусь, хто не зберігається як «користувач» вашого додатка. Використовуючи метод `route` фасада `Notification`, ви можете вказати інформацію про маршрутизацію спеціального повідомлення перед відправкою повідомлення:

```php
Notification::route('mail', 'taylor@example.com')
    ->route('vonage', '5555555555')
    ->route('slack', 'https://hooks.slack.com/services/...')
    ->notify(new InvoicePaid($invoice));
```

Якщо ви бажаєте вказати ім'я одержувача при надсиланні такого повідомлення на маршрут `mail`, ви можете надати масив, який містить адресу електронної пошти як ключ і ім’я як значення першого елемента в масиві:

```php
Notification::route('mail', [
    'barrett@example.com' => 'Barrett Blair',
])->notify(new InvoicePaid($invoice));
```

<a name="mail-notifications"></a>

## Поштові повідомлення

<a name="formatting-mail-messages"></a>

### Формування поштових повідомлень

Якщо повідомлення підтримує надсилання електронною поштою, вам слід визначити метод `toMail` в класі повідомлення. Цей метод отримає об'єкт сутності `$notifiable` і має повернути екземпляр `Illuminate\Notifications\Messages\MailMessage`.

Клас `MailMessage` містить кілька простих методів, які допоможуть створити транзакційні повідомлення електронної пошти. Поштові повідомлення можуть містити рядки тексту, а також "заклик до дії". Давайте розглянемо приклад метода `toMail`:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    $url = url('/invoice/' . $this->invoice->id);

    return (new MailMessage)
        ->greeting('Hello!')
        ->line('One of your invoices has been paid!')
        ->lineIf($this->amount > 0, "Amount paid: {$this->amount}")
        ->action('View Invoice', $url)
        ->line('Thank you for using our application!');
}
```

> **Note**  
> Зверніть увагу, що ми використовуємо `$this->invoice->id` у нашому методі `toMail`. Ви можете передати будь-які дані, необхідні вашому повідомленню для створення свого повідомлення, в конструкторі повідомлення.

У цьому прикладі ми реєструємо привітання, рядок тексту, заклик до дії, а потім інший рядок тексту. Ці методи, надані об’єктом `MailMessage`, дозволяють легко та швидко формувати невеликі транзакційні електронні листи. Потім поштовий канал перетворить компоненти повідомлення в красивий адаптивний HTML-шаблон електронної пошти з аналогом у вигляді звичайного тексту. Ось приклад електронного листа, створеного каналом `mail`:

<img src="https://laravel.com/img/docs/notification-example-2.png">

> **Note**  
> При надсиланні поштових повідомлень не забудьте встановити параметр `name` у конфігураційному файлі `config/app.php`. Це значення буде використовуватися у верхньому та нижньому колонтитулах ваших поштових повідомлень.

<a name="error-messages"></a>

#### Повідомлення про помилки

Деякі повідомлення інформують користувачів про помилки, наприклад про невдалу оплату рахунку. Ви можете вказати, що повідомлення електронної пошти стосується помилки, викликавши метод `error` під час створення повідомлення. Під час використання метода `error` в поштовому повідомленні кнопка заклику до дії буде червоною, а не чорною:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->error()
        ->subject('Invoice Payment Failed')
        ->line('...');
}
```

<a name="other-mail-notification-formatting-options"></a>

#### Інші параметри формування поштових повідомлень

Замість того, щоб визначати текстові «рядки» в класі повідомлень, ви можете використовувати метод `view`, щоб визначити власний шаблон, який слід використовувати для відтворення електронного листа повідомлення:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)->view(
        'emails.name', ['invoice' => $this->invoice]
    );
}
```

Ви можете визначити текстовий вміст для поштового повідомлення, вказавши ім'я шаблона як другий елемент масива, що передається методом `view`:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)->view(
        ['emails.name.html', 'emails.name.plain'],
        ['invoice' => $this->invoice]
    );
}
```

<a name="customizing-the-sender"></a>

### Зміна відправника

За замовчуванням адресу відправника електронного листа визначено у кофігураційному файлі `config/mail.php`. Однак ви можете вказати адресу відправника для певного повідомлення за допомогою метода `from`:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->from('barrett@example.com', 'Barrett Blair')
        ->line('...');
}
```

<a name="customizing-the-recipient"></a>

### Зміна одержувача

Під час відправлення повідомлень через `mail` канал, система повідомлень автоматично шукатиме властивість `email` об'єкта повідомлення. Ви можете налаштувати адресу електронної пошти, яка використовуватиметься для доставки повідомлень, визначивши метод `routeNotificationForMail` для об'єкта повідомлення:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Маршрут повідомлення для поштового каналу.
     *
     * @param  \Illuminate\Notifications\Notification  $notification
     * @return array|string
     */
    public function routeNotificationForMail($notification)
    {
        // Повернути лише адресу електронної пошти...
        return $this->email_address;

        // Повернути адресу електронної пошти та ім'я...
        return [$this->email_address => $this->name];
    }
}
```

<a name="customizing-the-subject"></a>

### Налаштування назви повідомлення

За замовчуванням назвою електронного листа є ім’я класу повідомлення у форматі «Title Case». Отже, якщо ваш клас повідомлення має назву `InvoicePaid`, назва електронного листа буде `Invoice Paid`. Якщо ви хочете вказати іншу назву для повідомлення, ви можете викликати метод `subject` під час створення повідомлення:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->subject('Notification Subject')
        ->line('...');
}
```

<a name="customizing-the-mailer"></a>

### Зміна поштового драйвера

За замовчуванням повідомлення електронної пошти надсилатиметься за допомогою стандартного поштомата, визначеного у конфігураційному файлі `config/mail.php`. Однак ви можете вказати інший поштомат під час виконання, викликавши метод `mailer` під час створення свого повідомлення:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->mailer('postmark')
        ->line('...');
}
```

<a name="customizing-the-templates"></a>

### Налаштування шаблонів

Ви можете змінити шаблон HTML і звичайний текст, який використовується для поштових повідомлень, опублікувавши необхідні ресурси повідомлення. Після виконання цієї команди шаблони поштових повідомлень будуть розташовані в каталозі `resources/views/vendor/notifications`:

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>

### Поштові вкладення

Щоб додати вкладення до поштового повідомлення, використовуйте метод `attach` під час створення повідомлення. Метод `attach` приймає абсолютний шлях до файлу першим аргументом:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attach('/path/to/file');
}
```

> **Note**  
> Метод `attach`, запропонований поштовим повідомленням, також приймає [об’єкти, які можна вкладати](mail.md#attachable-objects). Щоб дізнатися більше, зверніться до вичерпної [документації об’єкта, який можна вкладати](mail.md#attachable-objects)

При прикріпленні файлів до повідомлення ви також можете вказати ім'я / або MIME-тип, передавши масив другим аргумент методу `attach`:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attach('/path/to/file', [
            'as' => 'name.pdf',
            'mime' => 'application/pdf',
        ]);
}
```

На відміну від прикріплення файлів до поштових повідомлень, ви не можете прикріпити файл безпосередньо з диска файлового зберігання за допомогою `attachFromStorage`. Краще використовуйте метод `attach` з абсолютним шляхом до файлу на диску зберігання. Крім того, ви можете повернути [екземпляр поштового листа](mail.md#generating-mailables) із метода `toMail`:

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;

/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return Mailable
 */
public function toMail($notifiable)
{
    return (new InvoicePaidMailable($this->invoice))
        ->to($notifiable->email)
        ->attachFromStorage('/path/to/file');
}
```

За потреби до повідомлення можна прикріпити декілька файлів за допомогою методу `attachMany`:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attachMany([
            '/path/to/forge.svg',
            '/path/to/vapor.svg' => [
                'as' => 'Logo.svg',
                'mime' => 'image/svg+xml',
            ],
        ]);
}
```

<a name="raw-data-attachments"></a>

#### Поштові вкладення необроблених даних

Метод `attachData` можна використовувати для прикріплення необробленого рядка байтів як вкладення. Викликаючи метод `attachData`, ви повинні вказати ім’я файла, яке слід призначити вкладенню:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attachData($this->pdf, 'name.pdf', [
            'mime' => 'application/pdf',
        ]);
}
```

<a name="adding-tags-metadata"></a>

### Додавання тегів і метаданих

Деякі сторонні постачальники електронної пошти, такі як Mailgun і Postmark, підтримують «теги» та «метадані» повідомлень, які можуть використовуватися для групування та відстеження електронних листів, надісланих вашим додатком. Ви можете додати теги та метадані до поштового повідомлення за допомогою методів `tag` і `metadata` :

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->greeting('Comment Upvoted!')
        ->tag('upvote')
        ->metadata('comment_id', $this->comment->id);
}
```

Якщо ваш додаток використовує драйвер Mailgun, ви можете переглянути документацію Mailgun для отримання додаткової інформації про [теги](https://documentation.mailgun.com/en/latest/user_manual.html#tagging-1) та [метадані](https://documentation.mailgun.com/en/latest/user_manual.html#attaching-data-to-messages). Подібним чином можна переглянути документацію Postmark для отримання додаткової інформації про підтримку [тегів](https://postmarkapp.com/blog/tags-support-for-smtp) і [метаданих](https://postmarkapp.com/support/article/1125-custom-metadata-faq).

Якщо ваш додаток використовує Amazon SES для надсилання електронних листів, ви повинні використовувати метод `metadata`, щоб додати [«теги» SES](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) до повідомлення.

<a name="customizing-the-symfony-message"></a>

### Налаштування повідомлення Symfony

Метод `withSymfonyMessage` класа `MailMessage` дозволяє зареєструвати замикання, яке буде викликано екземпляром Symfony Message перед надсиланням повідомлення. Це дає вам можливість глибоко налаштувати повідомлення перед його доставкою:

```php
use Symfony\Component\Mime\Email;

/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->withSymfonyMessage(function (Email $message) {
            $message->getHeaders()->addTextHeader(
                'Custom-Header', 'Header Value'
            );
        });
}
```

<a name="using-mailables"></a>

### Використання поштових повідомлень

За потреби ви можете повернути [об’єкт поштового повідомлення](mail.md), з вашого метода `toMail` повідомлення. У разі повернення `Mailable` замість `MailMessage` вам потрібно буде вказати одержувача повідомлення за допомогою методу `to` об’єкта поштового повідомлення:

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;

/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return Mailable
 */
public function toMail($notifiable)
{
    return (new InvoicePaidMailable($this->invoice))
        ->to($notifiable->email);
}
```

<a name="mailables-and-on-demand-notifications"></a>

#### Поштові відправлення та повідомлення на вимогу

Якщо ви надсилаєте [повідомлення за вимогою](#on-demand-notifications), то екземпляр `$notifiable`, наданий методу `toMail`, буде екземпляром `Illuminate\Notifications\AnonymousNotifiable`, який надає метод `routeNotificationFor`, який можна використовувати для отримання адреси електронної пошти, на яку слід надіслати повідомлення за вимогою:

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;
use Illuminate\Notifications\AnonymousNotifiable;

/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return Mailable
 */
public function toMail($notifiable)
{
    $address = $notifiable instanceof AnonymousNotifiable
            ? $notifiable->routeNotificationFor('mail')
            : $notifiable->email;

    return (new InvoicePaidMailable($this->invoice))
                ->to($address);
}
```

<a name="previewing-mail-notifications"></a>

### Попередній перегляд поштових повідомлень

Під час розробки шаблону поштового повідомлення зручно та швидко переглядати візуалізацію поштового повідомлення можна у вашому браузері, як у типовий шаблоні Blade. З цієї причини Laravel дозволяє повертати будь-яке поштове повідомлення безпосередньо з замикання маршруту або контролера. Коли `MailMessage` повертається, воно буде відтворено та візуалізовано у браузері, що дозволить вам швидко переглянути його дизайн без необхідності надсилати його на справжню електронну адресу:

```php
use App\Models\Invoice;
use App\Notifications\InvoicePaid;

Route::get('/notification', function () {
    $invoice = Invoice::find(1);

    return (new InvoicePaid($invoice))
        ->toMail($invoice->user);
});
```

<a name="markdown-mail-notifications"></a>

## Поштові повідомлення з розміткою Markdown

Поштові повідомлення Markdown дозволяють скористатися перевагами попередньо створених шаблонів поштових повідомлень, одночасно надаючи вам більше свободи в написанні довших персоналізованих повідомлень. Оскільки повідомлення написані на Markdown, Laravel може відтворювати красиві адаптивні HTML-шаблони для повідомлень, а також автоматично генерувати аналог простого тексту.

<a name="generating-the-message"></a>

### Створення повідомлень

Щоб створити повідомлення з відповідним шаблоном Markdown, ви можете використовувати параметр `--markdown` команди `make:notification` Artisan:

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

Як і всі інші поштові повідомлення, повідомлення, які використовують шаблони Markdown, мають визначати метод `toMail` у своєму класі повідомлення. Однак замість використання методів `line` та `action` для створення повідомлення використовуйте метод `markdown`, щоб вказати назву шаблону Markdown, який слід використовувати. Масив даних, який ви хочете зробити доступним для шаблону, може бути переданий як другий аргумент метода:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    $url = url('/invoice/'.$this->invoice->id);

    return (new MailMessage)
        ->subject('Invoice Paid')
        ->markdown('mail.invoice.paid', ['url' => $url]);
}
```

<a name="writing-the-message"></a>

### Написання повідомлень

Поштові повідомлення Markdown використовують комбінацію компонентів Blade і синтаксису Markdown, які дозволяють легко створювати повідомлення, одночасно використовуючи попередньо створені компоненти повідомлень Laravel:

```blade
@component('mail::message')
# Invoice Paid

Your invoice has been paid!

@component('mail::button', ['url' => $url])
View Invoice
@endcomponent

Thanks,<br>
{{ config('app.name') }}
@endcomponent
```

<a name="button-component"></a>

#### Компонент Button

Компонент кнопки відображає центроване посилання кнопки. Компонент приймає два аргументи: `url` та необов’язковий `color`. Підтримуються кольори `primary`, `success`, and `error`. До повідомлення можна додати скільки завгодно компонентів кнопки:

```blade
@component('mail::button', ['url' => $url, 'color' => 'green'])
View Invoice
@endcomponent
```

<a name="panel-component"></a>

#### Компонент Panel

Компонент панелі відображає заданий блок тексту на панелі, колір фону якого дещо відрізняється від решти повідомлення. Це дозволяє привернути увагу до певного блоку текста:

```blade
@component('mail::panel')
This is the panel content.
@endcomponent
```

<a name="table-component"></a>

#### Компонент Table

Компонент таблиці дозволяє трансформувати таблицю Markdown у таблицю HTML. Компонент приймає таблицю Markdown як свій вміст. Вирівнювання стовпців таблиці підтримується за допомогою синтаксису вирівнювання таблиці Markdown за замовчанням:

```blade
@component('mail::table')
| Laravel       | Table         | Example  |
| ------------- |:-------------:| --------:|
| Col 2 is      | Centered      | $10      |
| Col 3 is      | Right-Aligned | $20      |
@endcomponent
```

<a name="customizing-the-components"></a>

### Налаштування компонентів

Ви можете експортувати всі поштові компоненти Markdown у власний додаток для налаштування. Щоб експортувати компоненти, скористайтеся командою `vendor:publish` Artisan, щоб опублікувати тег ресурса `laravel-mail`:

```shell
php artisan vendor:publish --tag=laravel-mail
```

Ця команда опублікує поштові компоненти Markdown у каталозі `resources/views/vendor/mail`. Каталог `mail`міститиме каталоги `html` і `text`, кожен з яких містить відповідні шаблони кожного доступного компонента. Ви можете налаштувати ці компоненти як вам завгодно.

<a name="customizing-the-css"></a>

#### Customizing The CSS

#### Налаштування CSS

Після експорту компонентів каталог `resources/views/vendor/mail/html/themes` міститиме файл default.css. Ви можете налаштувати CSS у цьому файлі, і ваші стилі будуть автоматично перетворені на вбудовані стилі CSS у HTML-представленнях ваших поштових повідомлень Markdown.

Якщо ви хочете створити абсолютно нову тему для компонентів Laravel Markdown, ви можете розмістити файл CSS у каталозі `html/themes`. Після назви та збереження файла CSS оновіть параметр `theme` у файлі конфігурації `config/mail.php` вашого додатка відповідно до назви нової теми.

Щоб налаштувати тему для окремого повідомлення, ви можете викликати метод `theme під час створення поштового повідомлення. Метод `theme приймає ім'я теми, яка повинна використовуватися при надсиланні повідомлення:

```php
/**
 * Отримати вміст поштового повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\MailMessage
 */
public function toMail($notifiable)
{
    return (new MailMessage)
        ->theme('invoice')
        ->subject('Invoice Paid')
        ->markdown('mail.invoice.paid', ['url' => $url]);
}
```

<a name="database-notifications"></a>

## Повідомлення через канал `database`

<a name="database-prerequisites"></a>

### Попередня підготовка бази даних

Канал повідомлень `database` зберігає інформацію повідомлень у таблиці бази даних. Ця таблиця міститиме таку інформацію, як тип повідомлення, а також структуру даних JSON, яка описує повідомлення.

Ви можете запитати таблицю, щоб відобразити повідомлення в користувацькому інтерфейсі вашого додатка. Але перш ніж ви зможете це зробити, вам потрібно буде створити таблицю бази даних для зберігання повідомлень. Ви можете використати команду `notifications:table` для створення [міграції](migrations.md) з належною схемою таблиці:

```shell
php artisan notifications:table

php artisan migrate
```

<a name="formatting-database-notifications"></a>

### Формування повідомлень канала `database`

Якщо повідомлення підтримує збереження в таблиці бази даних, вам слід визначити метод `toDatabase` або `toArray` у класі повідомлення. Цей метод отримає сутність `$notifiable` і має повернути звичайний масив PHP. Повернений масив буде закодовано як JSON і збережено в стовпці `data` вашої таблиці повідомлень. Давайте подивимося на приклад метода `toArray`:

```php
/**
 * Отримати масив даних сповіщення.
 *
 * @param  mixed  $notifiable
 * @return array
 */
public function toArray($notifiable)
{
    return [
        'invoice_id' => $this->invoice->id,
        'amount' => $this->invoice->amount,
    ];
}
```

<a name="todatabase-vs-toarray"></a>

#### Методи `toDatabase` і `toArray`

Метод `toArray` також використовується `broadcast` каналом, щоб визначити, які дані транслювати у ваш інтерфейс на основі JavaScript. Якщо ви хочете мати два різних представлення масива для `database` і `broadcast` каналів, вам слід визначити метод `toDatabase` замість метода `toArray`.

<a name="accessing-the-notifications"></a>

### Доступ до повідомлень

Після того як повідомлення зберігаються в базі даних, вам потрібен зручний спосіб доступу до них із ваших об’єктів, які повідомляються. Трейт `Illuminate\Notifications\Notifiable`, який за замовчуванням включений в модель `App\Models\User` Laravel, містить відношення `notifications` [Eloquent](eloquent-relationships.md), яке повертає повідомлення для сутності (об'єкта). Ви можете звернутися до цього метода, як до будь-якого іншого відношення Eloquent, щоб отримати повідомлення. За замовчуванням повідомлення будуть відсортовані за часовою міткою `created_at` з найновішими повідомленням на початку колекції:

```php
$user = App\Models\User::find(1);

foreach ($user->notifications as $notification) {
    echo $notification->type;
}
```

Якщо ви хочете отримати лише «непрочитані» повідомлення, ви можете використовувати відношення `unreadNotifications`. Знову ж таки, ці повідомлення будуть відсортовані за міткою часу `created_at` з найновішими повідомленнями на початку колекції:

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    echo $notification->type;
}
```

> **Note**  
> Щоб отримати доступ до повідомлень від клієнта JavaScript, вам слід визначити контролер повідомлень для вашого додатка, який повертає повідомлення для сутності (об’єкта), який повідомлятиметься, наприклад поточного користувача. Потім ви можете зробити HTTP-запит до URL-адреси цього контролера зі свого клієнта JavaScript.

<a name="marking-notifications-as-read"></a>

### Позначення повідомлень як прочитаних

Як правило, ви хочете позначити повідомлення як "прочитане", коли користувач переглядає його. Трейт `Illuminate\Notifications\Notifiable` надає метод `markAsRead`, який оновлює стовпець `read_at` у записі бази даних повідомлення:

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    $notification->markAsRead();
}
```

Однак замість циклічного перегляду кожного повідомлення ви можете використовувати метод `markAsRead` безпосередньо для колекції повідомлення:

```php
$user->unreadNotifications->markAsRead();
```

Ви також можете використовувати запит на масове оновлення, щоб позначити всі повідомлення як прочитані, не вилучаючи їх із бази даних:

```php
$user = App\Models\User::find(1);

$user->unreadNotifications()->update(['read_at' => now()]);
```

Ви можете використати метод `delete` для повідомлення, щоб повністю видалити їх з таблиці:

```php
$user->notifications()->delete();
```

<a name="broadcast-notifications"></a>

## Трансляція повідомлень

<a name="broadcast-prerequisites"></a>

### Попередня підготовка трансляції

Перш ніж транслювати повідомлення, вам слід налаштувати та ознайомитись із службами [трансляції подій](broadcasting.md) Laravel. Трансляція подій надає спосіб реагувати на серверні події Laravel із вашого інтерфейсу на основі JavaScript.

<a name="formatting-broadcast-notifications"></a>

### Формування повідомлень, які транслюються.

`broadcast` канал транслює повідомлення за допомогою служб [трансляції подій](broadcasting.md) Laravel, що дозволяє вашому інтерфейсу на основі JavaScript отримувати повідомлення в реальному часі. Якщо повідомлення підтримує трансляцію, ви можете визначити метод `toBroadcast` в класі повідомлення. Цей метод отримає сутність `$notifiable` і має повернути екземпляр `BroadcastMessage`. Якщо метод `toBroadcast` не існує, метод `toArray` буде використано для збору даних, які мають транслюватися. Повернені дані будуть закодовані як JSON і передані вашому JavaScript-додатку на стороні клієнта. Давайте подивимося на приклад метода `toBroadcast`:

```php
use Illuminate\Notifications\Messages\BroadcastMessage;

/**
 * Отримати вміст повідомлення, яке  транслюється.
 *
 * @param  mixed  $notifiable
 * @return BroadcastMessage
 */
public function toBroadcast($notifiable)
{
    return new BroadcastMessage([
        'invoice_id' => $this->invoice->id,
        'amount' => $this->invoice->amount,
    ]);
}
```

<a name="broadcast-queue-configuration"></a>

#### Налаштування черги трансляції

Всі транслюючі повідомлення можна поставити в чергу для трансляції. Якщо ви бажаєте налаштувати підключення до черги або назву черги, яка використовується для постановки в чергу трансляції, ви можете використовувати методи `onConnection` and `onQueue` класа `BroadcastMessage`:

```php
return (new BroadcastMessage($data))
    ->onConnection('sqs')
    ->onQueue('broadcasts');
```

<a name="customizing-the-notification-type"></a>

#### Налаштування типу повідомлення

Окрім вказаних вами даних, всі транслюючі повідомлення також мають поле `type`, яке містить повне ім’я класа повідомлення. Якщо ви хочете налаштувати `type` повідомлення, ви можете визначити метод `broadcastType` в класі повідомлення:

```php
use Illuminate\Notifications\Messages\BroadcastMessage;

/**
 * Отримати тип повідомлення, яке транслюється.
 *
 * @return string
 */
public function broadcastType()
{
    return 'broadcast.message';
}
```

<a name="listening-for-notifications"></a>

### Прослуховування повідомлень, які транслюються.

Повідомлення транслюватимуться на приватному каналі, відформатованому за правилами `{notifiable}.{id}`. Отже, якщо ви надсилаєте повідомлення екземпляра `App\Models\User` з ідентифікатором `1`, повідомлення транслюватиметься на приватному каналі `App.Models.User.1`. Використовуючи [Laravel Echo](broadcasting.md#client-side-installation), ви можете легко прослухати повідомлення на каналі за допомогою метода `notification`:

```js
Echo.private("App.Models.User." + userId).notification((notification) => {
  console.log(notification.type);
});
```

<a name="customizing-the-notification-channel"></a>

#### Налаштування канал повідомлень

Якщо ви бажаєте налаштувати, на якому каналі транслюватимуться повідомлення об’єкта, ви можете визначити метод `receivesBroadcastNotificationsOn` для сутності (об’єкта), який повідомляється:

```php
<?php

namespace App\Models;

use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Канали, якими користувач отримує розсилку повідомлень.
     *
     * @return string
     */
    public function receivesBroadcastNotificationsOn()
    {
        return 'users.' . $this->id;
    }
}
```

<a name="sms-notifications"></a>

## Повідомлення черезз SMS

<a name="sms-prerequisites"></a>

### Предварительная подготовка канала SMS

Відправлення SMS-повідомлень в Laravel підтримується за допомогою [Vonage](https://www.vonage.com/) (раніше відомий як Nexmo). Перш ніж ви зможете надсилати повідомлення через Vonage, вам потрібно встановити пакети `laravel/vonage-notification-channel` і `guzzlehttp/guzzle`:

```shell
composer require laravel/vonage-notification-channel guzzlehttp/guzzle
```

Пакет включає [конфігураційний файл](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php). Однак вам не потрібно експортувати цей файл конфігурації у власний додаток. Ви можете просто використовувати змінні середовища `VONAGE_KEY` і `VONAGE_SECRET`, щоб визначити свої відкритий і секретний ключі Vonage.

Після визначення ваших ключів ви можете встановити змінну середовища `VONAGE_SMS_FROM`, яка визначає номер телефона, з якого за замовчуванням мають надсилатися ваші SMS-повідомлення. Ви можете створити цей номер телефона на панелі керування Vonage:

```env
VONAGE_SMS_FROM=15556666666
```

<a name="formatting-sms-notifications"></a>

### Формування повідомлень через SMS

Якщо повідомлення підтримує надсилання як SMS, вам слід визначити метод `toVonage` в класі повідомлення. Цей метод отримає сутність `$notifiable` і має повернути екземпляр `Illuminate\Notifications\Messages\VonageMessage`:

```php
/**
 * Отримати SMS-подання повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\VonageMessage
 */
public function toVonage($notifiable)
{
    return (new VonageMessage)
        ->content('Your SMS message content');
}
```

<a name="unicode-content"></a>

#### Вміст Unicode

Якщо ваше SMS-повідомлення міститиме символи Unicode, вам слід викликати метод `unicode` під час створення екземпляра `VonageMessage`:

```php
/**
 * Отримати SMS-подання повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\VonageMessage
 */
public function toVonage($notifiable)
{
    return (new VonageMessage)
        ->content('Your unicode message')
        ->unicode();
}
```

<a name="customizing-the-from-number"></a>

### Зміна номера відправника

Якщо ви бажаєте надіслати деякі повідомлення з номера телефона, який відрізняється від номера телефона, зазначеного вашою змінною середовища VONAGE_SMS_FROM, ви можете викликати метод `from` в екземплярі `VonageMessage`:

```php
/**
 * Отримати SMS-подання повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\VonageMessage
 */
public function toVonage($notifiable)
{
    return (new VonageMessage)
        ->content('Your SMS message content')
        ->from('15554443333');
}
```

<a name="adding-a-client-reference"></a>

### Додавання посилання на клієнта

Якщо ви хочете відстежувати витрати кожного користувача, команди чи клієнта, ви можете додати до повідомлення «посилання на клієнта». Vonage дозволить вам створювати звіти, використовуючи це посилання клієнта, щоб ви могли краще зрозуміти використання SMS конкретним клієнтом. ПОсилання на клієнта може бути будь-яким рядком до 40 символів:

```php
/**
 * Отримати SMS-подання повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\VonageMessage
 */
public function toVonage($notifiable)
{
    return (new VonageMessage)
        ->clientReference((string) $notifiable->id)
        ->content('Your SMS message content');
}
```

<a name="routing-sms-notifications"></a>

### Маршрутизація SMS-повідомлень

Щоб скеровувати повідомлення Vonage на правильний номер телефона, визначте метод `routeNotificationForVonage` у своїй сутності, яка повідомляється:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Маршрутизація повідомлень для Vonage Channel.
     *
     * @param  \Illuminate\Notifications\Notification  $notification
     * @return string
     */
    public function routeNotificationForVonage($notification)
    {
        return $this->phone_number;
    }
}
```

<a name="slack-notifications"></a>

## Повідомлення через Slack

<a name="slack-prerequisites"></a>

### Попередня підготовка каналу Slack

Перш ніж ви зможете надсилати повідомлення через Slack, ви повинні встановити канал повідомлень Slack через Composer:

```shell
composer require laravel/slack-notification-channel
```

Вам також потрібно буде створити [додаток Slack](https://api.slack.com/apps?new_app=1) для вашої команди. Після створення додатка вам слід налаштувати «Incoming Webhook» для робочої області. Потім Slack надасть вам URL-адресу веб-гачка, який ви можете використовувати під час [маршрутизації повідомлень Slack](#routing-slack-notifications).

<a name="formatting-slack-notifications"></a>

### Форматування повідомлень Slack

Якщо повідомлення підтримує надсилання як повідомлення Slack, вам слід визначити метод `toSlack` у класі повідомлення. Цей метод отримає сутність `$notifiable` і має повернути екземпляр `Illuminate\Notifications\Messages\SlackMessage`. Slack-повідомлення можуть містити текстовий вміст, а також «вкладення», яке формує додатковий текст або масив полів. Давайте розглянемо базовий приклад `toSlack`:

```php
/**
 * Отримати подання Slack-повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\SlackMessage
 */
public function toSlack($notifiable)
{
    return (new SlackMessage)
        ->content('One of your invoices has been paid!');
}
```

<a name="slack-attachments"></a>

### Вкладення Slack-повідомлень

Ви також можете додавати «вкладення» до повідомлень Slack. Вкладення надають більші можливості форматування, ніж прості текстові повідомлення. У цьому прикладі ми надішлемо повідомлення про помилку винятка, який стався в додатку, включно з посиланням для перегляду додаткових відомостей про виняток:

```php
/**
 * Отримати подання Slack-повідомлення.
 *
 * @param  mixed  $notifiable
 * @return \Illuminate\Notifications\Messages\SlackMessage
 */
public function toSlack($notifiable)
{
    $url = url('/exceptions/'.$this->exception->id);

    return (new SlackMessage)
        ->error()
        ->content('Whoops! Something went wrong.')
        ->attachment(function ($attachment) use ($url) {
            $attachment->title('Exception: File Not Found', $url)
                ->content('File [background.jpg] was not found.');
        });
}
```

Вкладення також дозволяють вказати масив даних, які повинні бути представлені користувачеві. Надані дані будуть представлені у форматі таблиці для зручності читання:

```php
/**
 * Отримати подання Slack-повідомлення.
 *
 * @param  mixed  $notifiable
 * @return SlackMessage
 */
public function toSlack($notifiable)
{
    $url = url('/invoices/'.$this->invoice->id);

    return (new SlackMessage)
        ->success()
        ->content('One of your invoices has been paid!')
        ->attachment(function ($attachment) use ($url) {
            $attachment->title('Invoice 1322', $url)
                ->fields([
                    'Title' => 'Server Expenses',
                    'Amount' => '$1,234',
                    'Via' => 'American Express',
                    'Was Overdue' => ':-1:',
                ]);
        });
}
```

<a name="markdown-attachment-content"></a>

#### Вміст вкладень з розміткою Markdown

Якщо деякі з ваших полів вкладень містять Markdown, ви можете використати метод `markdown`, щоб наказати Slack проаналізувати та візуалізувати надані поля вкладень як текст у форматі Markdown. Цей метод приймає такі значення: `pretext`, `text`, і / або `fields`. Щоб дізнатися більше про формування вкладень Slack, перегляньте документацію [документацію Slack API](https://api.slack.com/docs/message-formatting#message_formatting):

```php
/**
 * Отримати подання Slack-повідомлення.
 *
 * @param  mixed  $notifiable
 * @return SlackMessage
 */
public function toSlack($notifiable)
{
    $url = url('/exceptions/' . $this->exception->id);

    return (new SlackMessage)
        ->error()
        ->content('Whoops! Something went wrong.')
        ->attachment(function ($attachment) use ($url) {
            $attachment->title('Exception: File Not Found', $url)
                ->content('File [background.jpg] was *not found*.')
                ->markdown(['text']);
        });
}
```

<a name="routing-slack-notifications"></a>

### Маршрутизація Slack-повідомлень

Щоб спрямувати повідомлення Slack до належної команди та каналу Slack, визначте метод `routeNotificationForSlack` у своїй сутності, яка повідомляється. Це метод має повернути URL-адресу веб-гачка, на яку має бути доставлено повідомлення. URL-адреси веб-гачків можна створити, додавши службу «Incoming Webhook» до вашої команди Slack:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * Route notifications for the Slack channel.
     *
     * @param  \Illuminate\Notifications\Notification  $notification
     * @return string
     */
    public function routeNotificationForSlack($notification)
    {
        return 'https://hooks.slack.com/services/...';
    }
}
```

<a name="localizing-notifications"></a>

## Локалізація повідомлень

Laravel дозволяє відправляти повідомлення, використовуючи мову, ка відрізняється від поточної мови запита, і навіть буде пам'ятати її, якщо повідомлення знаходиться в черзі.

Щоб досягти цього, клас `Illuminate\Notifications\Notification` пропонує метод `locale` для встановлення потрібної мови. Ваш додаток змінить цю мову під час оцінювання повідомлення, а потім повернеться до попередньої мови після завершення оцінки:

```php
$user->notify((new InvoicePaid($invoice))->locale('es'));
```

Локалізацію декількох записів, які повідомляються, також можна досягти за допомогою фасада `Notification`:

```php
Notification::locale('es')->send(
    $users, new InvoicePaid($invoice)
);
```

<a name="user-preferred-locales"></a>

### Бажана мова користовача

Іноді додатки зберігають бажану мову кожного користувача. Реалізуючи контракт `HasLocalePreference` у вашій моделі, яка повідомлятиметься, ви можете вказати Laravel використовувати цю збережену мову під час відправлення повідомлення:

```php
use Illuminate\Contracts\Translation\HasLocalePreference;

class User extends Model implements HasLocalePreference
{
    /**
     * Отримайте бажану мову користувача.
     *
     * @return string
     */
    public function preferredLocale()
    {
        return $this->locale;
    }
}
```

Щойно ви реалізуєте інтерфейс, Laravel автоматично використовуватиме бажану мову під час надсилання листів і повідомлень до моделі. Таким чином, немає необхідності викликати метод `locale` під час використання цього інтерфейсу:

```php
$user->notify(new InvoicePaid($invoice));
```

<a name="notification-events"></a>

## Події повідомлення

<a name="notification-sending-event"></a>

#### Подія відправлення повідомлення

Під час відправлення повідомлення система повідомлень надсилає [подію](events.md)`Illuminate\Notifications\Events\NotificationSending`. Подія містить сутність, яка «повідомляється» і сам екземпляр повідомлення. Ви можете зареєструвати слухачів для цієї події в `App\Providers\EventServiceProvider` вашого додатка:

```php
/**
 * Мапа слухачів подій додатка.
 *
 * @var array
 */
protected $listen = [
    'Illuminate\Notifications\Events\NotificationSending' => [
        'App\Listeners\CheckNotificationStatus',
    ],
];
```

Повідомленя не буде відправлено, якщо слухач події `NotificationSending` повертає `false` зі свого метода `handle`:

The notification will not be sent if an event listener for the `NotificationSending` event returns `false` from its `handle` method:

```php
use Illuminate\Notifications\Events\NotificationSending;

/**
 * Орпацювати дану подію.
 *
 * @param  \Illuminate\Notifications\Events\NotificationSending  $event
 * @return void
 */
public function handle(NotificationSending $event)
{
    return false;
}
```

В слухачі події ви можете отримати доступ до властивостей `notifiable`, `notification`, і `channel` події для отримання інформації про одержувача повідомлення або про саме повідомлення:

```php
/**
 * Орпацювати дану подію.
 *
 * @param  \Illuminate\Notifications\Events\NotificationSending  $event
 * @return void
 */
public function handle(NotificationSending $event)
{
    // $event->channel
    // $event->notifiable
    // $event->notification
}
```

<a name="notification-sent-event"></a>

#### Подія відправленого повідомлення

Після відправлення повідомлення система повідомлень запускає [подію](events.md)`Illuminate\Notifications\Events\NotificationSent`. Подія містить сутність, яка «повідомляється» і сам екземпляр повідомлення. Як правило, реєстрація слухачів цієї події здійснюється у постачальнику `App\Providers\EventServiceProvider`:

```php
/**
 * Мапа слухачів подій додатка.
 *
 * @var array
 */
protected $listen = [
    'Illuminate\Notifications\Events\NotificationSent' => [
        'App\Listeners\LogNotification',
    ],
];
```

> **Note**  
> Після реєстрації слухачів у вашому `EventServiceProvider` використовуйте команду `event:generate` Artisan, щоб швидко створити класи слухачів.

В слухачі події ви можете отримати доступ до властивостей `notifiable`, `notification`, і `channel` події для отримання інформації про одержувача повідомлення або про саме повідомлення:

```php
/**
 * Орпацювати дану подію.
 *
 * @param  \Illuminate\Notifications\Events\NotificationSent  $event
 * @return void
 */
public function handle(NotificationSent $event)
{
    // $event->channel
    // $event->notifiable
    // $event->notification
    // $event->response
}
```

<a name="custom-channels"></a>

## Налаштування власних каналів повідомлень

Laravel поставляється з кількома каналами повідомлень, але ви можете написати власні драйвери для доставки повідомлень через інші канали. Laravel полегшує цю роботу. Для початку визначте клас, який містить метод `send`. Метод повинен отримати два аргументи: `$notifiable` і `$notification`.

В методі `send` ви можете викликати методи повідомлень, щоб отримати об’єкт повідомлення, зрозумілий вашому каналу, а потім надіслати повідомлення необхідному вам екземпляру `$notifiable`:

```php
<?php

namespace App\Notifications;

use Illuminate\Notifications\Notification;

class VoiceChannel
{
    /**
     * Відправити передане повідомлення.
     *
     * @param  mixed  $notifiable
     * @param  \Illuminate\Notifications\Notification  $notification
     * @return void
     */
    public function send($notifiable, Notification $notification)
    {
        $message = $notification->toVoice($notifiable);

        // Надіслати повідомлення екземпляру $notifiable...
    }
}
```

Після визначення класа канала повідомлень ви можете повернути назву класа з метода `via` будь-якого зі своїх повідомлень. У цьому прикладі метод `toVoice` вашого повідомлення може повертати будь-який об’єкт, який ви оберете для формування голосових повідомлень. Наприклад, ви можете визначити власний клас `VoiceMessage` для формування цих повідомлень:

```php
<?php

namespace App\Notifications;

use App\Notifications\Messages\VoiceMessage;
use App\Notifications\VoiceChannel;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification
{
    use Queueable;

    /**
     * Отримати канали доставки повідомлень.
     *
     * @param  mixed  $notifiable
     * @return array|string
     */
    public function via($notifiable)
    {
        return [VoiceChannel::class];
    }

    /**
     * Отримати вміст голосового повідомлення.
     *
     * @param  mixed  $notifiable
     * @return VoiceMessage
     */
    public function toVoice($notifiable)
    {
        // ...
    }
}
```
