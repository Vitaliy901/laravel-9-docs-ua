# Поштові відправлення

- [Вступ](#introduction)
  - [Налаштування](#configuration)
  - [Попередня підготовка драйверів](#driver-prerequisites)
  - [Аварійне конфігурування](#failover-configuration)
- [Генерація розсилки](#generating-mailables)
- [Написання розсилки](#writing-mailables)
  - [Налаштування відправника](#configuring-the-sender)
  - [Налаштування візуалізації](#configuring-the-view)
  - [Дані шаблона](#view-data)
  - [Вкладення](#attachments)
  - [Вбудовані вкладення](#inline-attachments)
  - [Прикріплення Об'єктів](#attachable-objects)
  - [Теги та метадані](#tags-and-metadata)
  - [Налаштування повідомлення Symfony](#customizing-the-symfony-message)
- [Розсилка з розміткою Markdown](#markdown-mailables)
  - [Створення розсилки Markdown](#generating-markdown-mailables)
  - [Написання повідомлень Markdown](#writing-markdown-messages)
  - [Налаштування компонентів](#customizing-the-components)
- [Відправлення Пошти](#sending-mail)
  - [Пошта в черзі](#queueing-mail)
- [Візуалізація розсилки](#rendering-mailables)
  - [Попередній перегляд листів у браузері](#previewing-mailables-in-the-browser)
- [Локалізація розсилки](#localizing-mailables)
- [Тестування розсилки](#testing-mailables)
- [Пошта та локальна розробка](#mail-and-local-development)
- [Події](#events)
- [Користувальницькі драйвери](#custom-transports)
  - [Додаткові драйвери Symfony](#additional-symfony-transports)

<a name="introduction"></a>

## Вступ

Надсилання електронної пошти не повинно бути складною справою. Laravel надає чистий і простий API електронної пошти на основі популярного компонента [Symfony Mailer](https://symfony.com/doc/6.0/mailer.html). Laravel і «Symfony Mailer» надають драйвери для надсилання електронної пошти через SMTP, Mailgun, Postmark, Amazon SES і sendmail, що дозволяє вам швидко розпочати надсилання пошти через локальну або хмарну службу на ваш вибір.

<a name="configuration"></a>

### Налаштування

Служби електронної пошти Laravel можна налаштувати за допомогою конфігураційного файлу програми `config/mail.php`. Кожен поштомат, налаштований в цьому файлі, може мати власну унікальну конфігурацію та навіть власний унікальний «транспорт», що дозволяє вашому додатку використовувати різні служби електронної пошти для надсилання певних повідомлень електронної пошти. Наприклад, ваш додаток може використовувати Postmark для надсилання транзакційних електронних листів, а Amazon SES — для масового надсилання електронних листів.

У вашому конфігураційному файлі `mail` ви знайдете масив конфігурації `mailers`. Цей масив містить зразок конфігурації для кожного основного поштового драйвера / транспорту, які підтримуються Laravel, тоді як значення конфігурації `default` визначає, який поштомат використовуватиметься за замовчуванням, коли вашому додатку потрібно буде надіслати повідомлення електронної пошти.

<a name="driver-prerequisites"></a>

### Попередня підготовка драйверів

Драйвери на основі API, такі як Mailgun і Postmark, часто простіші та швидші, ніж надсилання пошти через сервери SMTP. За можливості рекомендуємо використовувати один із цих драйверів.

<a name="mailgun-driver"></a>

#### Драйвер Mailgun

Щоб використовувати драйвер Mailgun, встановить Symfony Mailgun Mailer через Composer:

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

Далі встановіть параметр `default` у конфігураційному файлі додатка `config/mail.php` на `mailgun`. Після налаштування пошти за замовчуванням у вашому додатку переконайтеся, що файл конфігурації `config/services.php` містить такі параметри:

```php
'mailgun' => [
'domain' => env('MAILGUN_DOMAIN'),
'secret' => env('MAILGUN_SECRET'),
],
```

Якщо ви не використовуєте United States [Mailgun region](https://documentation.mailgun.com/en/latest/api-intro.html#mailgun-regions), ви можете визначити кінцеву точку свого регіону у файлі конфігурації `services`:

```php
'mailgun' => [
'domain' => env('MAILGUN_DOMAIN'),
'secret' => env('MAILGUN_SECRET'),
'endpoint' => env('MAILGUN_ENDPOINT', 'api.eu.mailgun.net'),
],
```

<a name="postmark-driver"></a>

#### Драйвер Postmark

Щоб використовувати драйвер Postmark, інсталюйте Symfony Postmark Mailer через Composer:

```shell
composer require symfony/postmark-mailer symfony/http-client
```

Далі встановіть параметр `default` у конфігураційному файлі додатка `config/mail.php` на `postmark`. Після налаштування пошти за замовчуванням у вашому додатку переконайтеся, що файл конфігурації `config/services.php` містить такі параметри:

```php
'postmark' => [
    'token' => env('POSTMARK_TOKEN'),
],
```

Якщо ви бажаєте вказати потік повідомлень Postmark, який має використовуватися певним поштоматом, ви можете додати параметр конфігурації `message_stream_id` до масива конфігураційної поштомата. Цей масив конфігурації можна знайти у конфігураційному файлі додатка `config/mail.php`:

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
],
```

Таким чином ви також можете налаштувати декілька поштоматів Postmark з різними потоками повідомлень.

<a name="ses-driver"></a>

#### Драйвер SES

Щоб використовувати драйвер Amazon SES, спочатку потрібно встановити Amazon AWS SDK для PHP. Ви можете встановити цю бібліотеку через менеджер пакетів Composer:

```shell
composer require aws/aws-sdk-php
```

Далі встановіть параметр `default` у файлі конфігурації `config/mail.php` на `ses` і переконайтеся, що файл конфігурації `config/services.php` містить такі параметри:

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

Щоб використовувати [тимчасові облікові дані AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html) через токен сеансу, ви можете додати ключ `token` до конфігурації SES вашого додатка:

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

Якщо ви хочете визначити [додаткові параметри](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail), які Laravel повинен передавати методу `SendEmail` AWS SDK під час надсилання електронного листа, ви можете визначити масив `options` у своїй конфігурації `ses`:

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'options' => [
        'ConfigurationSetName' => 'MyConfigurationSet',
        'EmailTags' => [
            ['Name' => 'foo', 'Value' => 'bar'],
        ],
    ],
],
```

<a name="failover-configuration"></a>

### Аварійне конфігурування

Іноді зовнішня служба, яку ви налаштували для надсилання пошти у вашому додатку, може не працювати. У цих випадках може бути корисним визначити одну або декілька резервних конфігурацій доставки пошти, які використовуватимуться, якщо основний драйвер доставки не працює.

Щоб досягти цього, вам слід визначити поштомат у файлі конфігурації `config/mail.php` вашого додатка, який використовує транспорт `failover`. Масив конфігурації для поштомата `failover` вашого додатка має містити масив `mailers`, який посилається на порядок, у якому поштові драйвери слід вибирати для доставки:

```php
'mailers' => [
    'failover' => [
        'transport' => 'failover',
        'mailers' => [
            'postmark',
            'mailgun',
            'sendmail',
        ],
    ],

    // ...
],
```

Після того, як ваш аварійний поштомат було визначена, ви повинні встановити його як поштомат за замовчуванням, який використовує ваш додаток, вказавши його назву як значення конфігураційного ключа `default` у конфігураційному файлі `config/mail.php` вашого додатка:

```php
'default' => env('MAIL_MAILER', 'failover'),
```

<a name="generating-mailables"></a>

## Генерація розсилки

Під час створення додатків Laravel, кожен тип електронних листів, надісланих вашим додатком, представлений як клас `Illuminate\Mail\Mailable`. Ці класи зберігаються в каталозі `app/Mail`. Не хвилюйтеся, якщо ви не бачите цей каталог у своєму додатку, оскільки він буде згенерований для вас, коли ви створите свій перший поштовий клас для відправки за допомогою команди `make:mail` Artisan:

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>

## Написання розсилки

Після того, як ви створили поштовий клас, відкрийте його, щоб ми могли дослідити його вміст. По-перше, зверніть увагу, що вся конфігурація поштового класа, виконується в методі `build`. У цьому методі ви можете викликати різні методи, такі як `from`, `subject`, `view`, і `attach`, щоб налаштувати візуалізацію та доставку електронного листа.

> **Note**  
> Ви можете визначити тип залежностей в методі `build` поштового повідомлення. [Контейнер служби](container.md) Laravel автоматично додає ці залежності.

<a name="configuring-the-sender"></a>

### Налаштування відправника

<a name="using-the-from-method"></a>

#### Використання метода `from`

Спочатку розглянемо налаштування відправника електронного листа. Або, іншими словами, «від кого» буде електронний лист. Існує два способи налаштування відправника. По-перше, ви можете використати метод `from` у методі `build` свого поштового класа:

```php
/**
 * Створито повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->from('example@example.com', 'Example')
                ->view('emails.orders.shipped');
}
```

<a name="using-a-global-from-address"></a>

#### Використання глобальної адреси `from`

Однак, якщо ваш додаток використовує ту саму адресу `from` для всіх своїх електронних листів, виклик метода `from` у кожному створеному вами класі може втомлювати. Натомість ви можете вказати глобальну адресу відправника у файлі конфігурації `config/mail.php`. Ця адреса буде використана, якщо в класі для надсилання не вказано іншу адресу `from`:

```php
'from' => ['address' => 'example@example.com', 'name' => 'App Name'],
```

Крім того, ви можете визначити глобальну адресу «reply_to» у конфігураційному файлі `config/mail.php`:

```php
'reply_to' => ['address' => 'example@example.com', 'name' => 'App Name'],
```

<a name="configuring-the-view"></a>

### Налаштування візуалізації

У методі `build` поштового класа, ви можете використовувати метод `view`, щоб вказати, який шаблон слід використовувати при візуалізації вмісту електронного листа. Оскільки кожен електронний лист зазвичай використовує [шаблон Blade](blade.md) для візуалізації свого вмісту, ви маєте повну силу і зручність механізма створення шаблонів Blade під час створення HTML-розмітки свого електронного листа:

```php
/**
 * Створито повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped');
}
```

> **Note**  
> За бажанням ви можете створити каталог `resources/views/emails` для всіх ваших шаблонів електронної пошти; однак ви вільні у виборі де саме розміщувати їх у своєму каталозі `resources/views`.

<a name="plain-text-emails"></a>

#### Електронні листи з простим текстом

Якщо ви хочете створити звичайну текстову версію свого електронного листа, ви можете скористатися методом `text`. Подібно до метода `view`, текстовий метод приймає ім’я шаблона, яке використовуватиметься для відтворення вмісту електронного листа. Ви можете визначити як HTML, так і текстову версію вашого повідомлення:

```php
/**
 * Створито повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
                ->text('emails.orders.shipped_plain');
}
```

<a name="view-data"></a>

### Дані шаблона

<a name="via-public-properties"></a>

#### Передача даних шаблону через публічні властивості

Як правило, ви бажаєте передати деякі дані до свого шаблона, які можна використовувати під час відтворення HTML-розмітки електронного листа. Дані для шаблона можна зробити доступними двома способами. По-перше, будь-яка публічна властивість, визначена у вашому поштовому класі, автоматично стане доступною для шаблона. Так, наприклад, ви можете передати дані до конструктора свого поштового класа, і встановити для цих даних публічні властивості, визначені в класі:

```php
<?php

namespace App\Mail;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class OrderShipped extends Mailable
{
    use Queueable, SerializesModels;

    /**
     * Екземпляр замовлення.
     *
     * @var \App\Models\Order
     */
    public $order;

    /**
     * Створити новий екземпляр повідомлення.
     *
     * @param  \App\Models\Order  $order
     * @return void
     */
    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    /**
     * Створити повідомлення.
     *
     * @return $this
     */
    public function build()
    {
        return $this->view('emails.orders.shipped');
    }
}
```

Після встановлення публічної властивостей, дані автоматично стануть доступними у вашому шаблоні, тож ви зможете отримати до них доступ, як до будь-яких інших даних у своїх шаблонах Blade:

```php
<div>
    Price: {{ $order->price }}
</div>
```

<a name="via-the-with-method"></a>

#### Передача даних шаблону через метод `with`

Якщо ви бажаєте налаштувати формат даних електронної пошти перед тим, як її буде надіслано до шаблону, ви можете вручну передати свої дані до шаблона за допомогою методу `with`. Як правило, ви все одно передаватимете дані через конструктор поштового класа; однак ви повинні встановити для цих даних `protected` або `private` властивості, щоб дані не ставали автоматично доступними для шаблона. Потім під час виклику метода `with` передайте масив даних, які ви хочете зробити доступними для шаблона:

```php
<?php

namespace App\Mail;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class OrderShipped extends Mailable
{
    use Queueable, SerializesModels;

    /**
     * Екземпляр замовлення.
     *
     * @var \App\Models\Order
     */
    protected $order;

    /**
     * Створити новий екземпляр повідомлення.
     *
     * @param  \App\Models\Order  $order
     * @return void
     */
    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    /**
     * Створити повідомлення.
     *
     * @return $this
     */
    public function build()
    {
        return $this->view('emails.orders.shipped')
            ->with([
                'orderName' => $this->order->name,
                'orderPrice' => $this->order->price,
            ]);
    }
}
```

Після того, як дані були передані методу `with`, вони автоматично стануть доступними у вашому шаблоні, тому ви можете отримати до них доступ так само, як і до будь-яких інших даних у ваших шаблонах Blade:

```php
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>

### Вкладення

Щоб додати вкладення до електронного листа, використовуйте метод `attach` у методі `build` поштового класа. Метод `attach` приймає повний шлях до файлу першим аргументом:

```php
/**
 * Створити повідомлення
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
        ->attach('/path/to/file');
}
```

Прикріплюючи файли до повідомлення, ви також можете вказати ім’я або тип MIME для відображення, передавши масив другим аргументом методу `attach`:

```php
/**
 * Створити повідомлення
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
        ->attach('/path/to/file', [
            'as' => 'name.pdf',
            'mime' => 'application/pdf',
        ]);
}
```

<a name="attaching-files-from-disk"></a>

#### Прикріплення файлів з диска

Якщо ви зберегли файл на одному з [дисків файлової системи](filesystem.md), ви можете прикріпити його до електронного листа за допомогою метода `attachFromStorage`:

```php
/**
 * Створити повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
        ->attachFromStorage('/path/to/file');
}
```

Якщо необхідно, ви можете вказати назву вкладеного файла та додаткові параметри, використовуючи другий і третій аргументи метода `attachFromStorage`:

```php
/**
 * Створити повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
        ->attachFromStorage('/path/to/file', 'name.pdf', [
            'mime' => 'application/pdf'
        ]);
}
```

Метод `attachFromStorageDisk` можна використовувати, якщо вам потрібно вказати диск зберігання, який відрізняється від диска за замовчуванням:

```php
/**
 * Створити повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
        ->attachFromStorageDisk('s3', '/path/to/file');
}
```

<a name="raw-data-attachments"></a>

#### Вкладення необроблених даних

Метод `attachData` можна використовувати для прикріплення необробленого рядка в якості вкладення. Наприклад, ви можете скористатися цим методом, якщо ви створили PDF-файл у пам’яті та хочете прикріпити його до електронного листа, не записуючи на диск. Метод `attachData` приймає байти необроблених даних першим аргументом, назву файла другим аргументом і масив параметрів третім аргументом:

```php
/**
 * Створити повідомлення
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
        ->attachData($this->pdf, 'name.pdf', [
            'mime' => 'application/pdf',
        ]);
}
```

<a name="inline-attachments"></a>

### Вбудовані вкладення

Додавання зображень у ваші електронні листи зазвичай є складним завданням; однак Laravel надає зручний спосіб додавати зображення до ваших електронних листів. Щоб вставити зображення, використовуйте метод `embed` змінної `$message` у вашому шаблоні електронної пошти. Laravel автоматично робить змінну `$message` доступною для всіх ваших шаблонів електронної пошти, тому вам не потрібно турбуватися про її передачу вручну:

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> **Warning**  
> Змінна `$message` недоступна в шаблонах простих текстових повідомлень, оскільки текстові повідомлення не використовують вбудовані вкладення.

<a name="embedding-raw-data-attachments"></a>

#### Додавання вкладень необроблених даних

Якщо у вас вже є рядок необроблених даних зображення, який ви хочете вставити в шаблон електронної пошти, ви можете викликати метод `embedData` для змінної `$message`. Під час виклику метода `embedData` вам потрібно буде вказати ім’я файла, яке слід призначити доданому зображенню:

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```

<a name="attachable-objects"></a>

### Прикріплення об'єктів

Хоча зазвичай достатньо прикріплення файлів до повідомлень за допомогою простих рядкових шляхів, у багатьох випадках сутності, які можна приєднати, у вашому додатку представлені класами. Наприклад, якщо ваш додаток додає фотографію до повідомлення, ваш додаток також може мати `Photo` модель, яка представляє це фото. Коли це так, чи не було б зручніше просто передати `Photo` модель методу `attach`? Приєднані об’єкти дозволяють це зробити.

Для початку реалізуйте інтерфейс `Illuminate\Contracts\Mail\Attachable` на об’єкті, який можна буде приєднати до повідомлень. Цей інтерфейс вимагає, щоб ваш клас визначав метод `toMailAttachment`, який повертає екземпляр `Illuminate\Mail\Attachment`:

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Mail\Attachable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Mail\Attachment;

class Photo extends Model implements Attachable
{
    /**
     * Отримайте приєднану модель зображення.
     *
     * @return \Illuminate\Mail\Attachment
     */
    public function toMailAttachment()
    {
        return Attachment::fromPath('/path/to/file');
    }
}
```

Після того, як ви визначили свій об’єкт, який можна приєднати, ви можете просто передати екземпляр цього об’єкта методу `attach` під час створення повідомлення електронної пошти:

```php
/**
 * Створити повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->view('photos.resized')
        ->attach($this->photo);
}
```

Звичайно, дані вкладень можуть зберігатися у віддаленій службі зберігання файлів, наприклад Amazon S3. Отже, Laravel також дозволяє генерувати екземпляри вкладень із даних, які зберігаються на одному з [дисків файлової системи](filesystem.md) вашого додатка:

```php
// Створіть вкладення файла з диска за замовчуванням...
return Attachment::fromStorage($this->path);

// Створити вкладення файла з певного диска...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

Крім того, ви можете створювати екземпляри вкладень за допомогою даних, які є у вашій пам’яті. Щоб досягти цього, забезпечте замикання методу `fromData`. Замикання має повернути необроблені дані, які представляють вкладення:

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel також надає додаткові методи, які ви можете використовувати для налаштування своїх вкладень. Наприклад, ви можете використовувати методи `as` і `withMime` для налаштування назви файла та типу MIME:

```php
return Attachment::fromPath('/path/to/file')
        ->as('Photo Name')
        ->withMime('image/jpeg');
```

<a name="tags-and-metadata"></a>

### Теги та метадані

Деякі сторонні постачальники електронної пошти, такі як Mailgun і Postmark, підтримують «теги» та «метадані» повідомлень, які можуть використовуватися для групування та відстеження електронних листів, надісланих вашим додатком. Ви можете додати теги та метадані до повідомлення електронної пошти за допомогою методів `tag` і `metadata`:

```php
/**
 * Створити повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->view('emails.orders.shipped')
        ->tag('shipment')
        ->metadata('order_id', $this->order->id);
}
```

Якщо ваш додаток використовує драйвер Mailgun, ви можете переглянути документацію Mailgun для отримання додаткової інформації про [теги](https://documentation.mailgun.com/en/latest/user_manual.html#tagging-1) та [метадані](https://documentation.mailgun.com/en/latest/user_manual.html#attaching-data-to-messages). Подібним чином можна переглянути документацію Postmark для отримання додаткової інформації про підтримку [тегів](https://postmarkapp.com/blog/tags-support-for-smtp) і [метаданих](https://postmarkapp.com/support/article/1125-custom-metadata-faq).

Якщо ваш додаток використовує Amazon SES для надсилання електронних листів, ви повинні використовувати метод `metadata`, щоб додати [«теги» SES](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) до повідомлення.

<a name="customizing-the-symfony-message"></a>

### Налаштування повідомлення Symfony

Метод `withSymfonyMessage` базового класа `Mailable` дозволяє зареєструвати замикання, яке буде викликано екземпляром Symfony Message перед надсиланням повідомлення. Це дає вам можливість глибше налаштувати повідомлення перед його доставкою:

```php
use Symfony\Component\Mime\Email;

/**
 * Створити повідомлення.
 *
 * @return $this
 */
public function build()
{
    $this->view('emails.orders.shipped');

    $this->withSymfonyMessage(function (Email $message) {
        $message->getHeaders()->addTextHeader(
            'Custom-Header', 'Header Value'
        );
    });

    return $this;
}
```

<a name="markdown-mailables"></a>

## Розсилка з розміткою Markdown

Поштові повідомлення Markdown дозволяють скористатися перевагами попередньо створених шаблонів і компонентів [поштових повідомлень](notifications.md#mail-notifications) у ваших поштових повідомленнях. Оскільки повідомлення написані у Markdown, Laravel може відтворювати красиві адаптивні HTML-шаблони для повідомлень, а також автоматично генерувати аналог простого тексту.

<a name="generating-markdown-mailables"></a>

### Створення розсилки Markdown

Щоб створити поштовий лист із відповідним шаблоном Markdown, ви можете використати параметр `--markdown` команди `make:mail` Artisan:

```shell
php artisan make:mail OrderShipped --markdown=emails.orders.shipped
```

Потім, в методі `build` поштового класу викличте метод `markdown` замість метода `view`. Метод `markdown` приймає ім'я шаблону Markdown та необов'язковий масив даних, які мають бути доступні для шаблона:

```php
/**
 * Створити повідомлення.
 *
 * @return $this
 */
public function build()
{
    return $this->from('example@example.com')
        ->markdown('emails.orders.shipped', [
            'url' => $this->orderUrl,
        ]);
}
```

<a name="writing-markdown-messages"></a>

### Написання повідомлень Markdown

У поштових повідомленнях Markdown використовується комбінація компонентів Blade і синтаксису Markdown, які дозволяють легко створювати поштові повідомлення, використовуючи попередньо створені компоненти електронного інтерфейсу Laravel:

```blade
@component('mail::message')
# Order Shipped

Your order has been shipped!

@component('mail::button', ['url' => $url])
View Order
@endcomponent

Thanks,<br>
{{ config('app.name') }}
@endcomponent
```

> **Note**  
> Не використовуйте надмірні відступи під час написання електронних листів Markdown. Відповідно до стандартів Markdown аналізатори Markdown відображатимуть вміст із відступами як блоки коду.

<a name="button-component"></a>

#### Компонент Button

Компонент кнопки відображає центроване посилання кнопки. Компонент приймає два аргументи: `url` та необов’язковий `color`. Підтримуються кольори `primary`, `success`, and `error`. До повідомлення можна додати скільки завгодно компонентів кнопки:

```blade
@component('mail::button', ['url' => $url, 'color' => 'success'])
View Order
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

#### Налаштування CSS

Після експорту компонентів каталог `resources/views/vendor/mail/html/themes` міститиме файл default.css. Ви можете налаштувати CSS у цьому файлі, і ваші стилі будуть автоматично перетворені на вбудовані стилі CSS у HTML-представленнях ваших поштових повідомлень Markdown.

Якщо ви хочете створити абсолютно нову тему для компонентів Laravel Markdown, ви можете розмістити файл CSS у каталозі `html/themes`. Після назви та збереження файла CSS оновіть параметр `theme` у файлі конфігурації `config/mail.php` вашого додатка відповідно до назви нової теми.

Щоб налаштувати тему для окремого повідомлення, ви можете встановити властивість `$theme` поштовому класу, який надсилається, на назву теми, яку слід використовувати під час надсилання цього листа.

<a name="sending-mail"></a>

## Відправлення Пошти

Щоб надіслати повідомлення, використовуйте метод `to` [фасада](facades.md) `Mail`. Метод `to` приймає адресу електронної пошти, екземпляр користувача або колекцію користувачів. Якщо ви передаєте об’єкт або набір об’єктів, поштомат автоматично використовуватиме їхні властивості `email` і `name` під час визначення одержувачів електронного листа, тому переконайтеся, що ці атрибути доступні для ваших об’єктів. Після того, як ви вказали одержувачів, ви можете передати екземпляр вашого поштового класа для відправки в метод `send`:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Mail\OrderShipped;
use App\Models\Order;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Mail;

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

        // Відправити замовлення...

        Mail::to($request->user())->send(new OrderShipped($order));
    }
}
```

Ви не обмежені простим визначенням одержувачів під час надсилання повідомлення. Ви можете вказати одержувачів `to`, `cc`, та `bcc`, зв'язавши їх відповідні методи разом:

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```

<a name="looping-over-recipients"></a>

#### Ітерація списка одержувачів

Іноді вам може знадобитися надіслати лист до списку одержувачів, перебираючи масив одержувачів/електронних адрес. Однак, оскільки метод `to` додає адреси електронної пошти до списку одержувачів поштових листів, кожна ітерація циклу надсилатиме інший електронний лист кожному попередньому одержувачу. Отже, ви завжди повинні повторно створювати поштовий екземпляр для кожного одержувача:

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```

<a name="sending-mail-via-a-specific-mailer"></a>

#### Визначення поштового ​​драйвера під час відправлення пошти

За замовчуванням Laravel надсилатиме електронну пошту за допомогою поштомата, налаштований як поштомат `default` у файлі конфігурації `config/mail.php` вашого додатка. Однак ви можете використовувати метод `mailer`, щоб надіслати повідомлення за допомогою іншої конфігурації поштомата:

```php
Mail::mailer('postmark')
        ->to($request->user())
        ->send(new OrderShipped($order));
```

<a name="queueing-mail"></a>

### Пошта в черзі

<a name="queueing-a-mail-message"></a>

#### Постановка поштового повідомлення в чергу

Оскільки надсилання повідомлень електронної пошти може негативно вплинути на час відповіді вашого додатка, багато розробників обирають варіант поставити поштове повідомлення в чергу для надсилання у фоновому режимі. Laravel полегшує це за допомогою вбудованого [API уніфікованої черги](queues.md). Щоб поставити поштове повідомлення в чергу, скористайтеся методом `queue` фасада `Mail` після визначення одержувачів повідомлення:

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

Цей метод автоматично подбає про надсилання завдання до черги, щоб повідомлення надсилалося у фоновому режимі. Перед використанням цього функціоналу вам потрібно буде [налаштувати свої черги](queues.md).

<a name="delayed-message-queueing"></a>

#### Черга відкладених повідомлень

Якщо ви бажаєте відкласти доставку повідомлення електронної пошти в черзі, ви можете скористатися методом `later`. Першим аргументом `later` метод приймає екземпляр `DateTime`, який вказує, коли повідомлення має бути надіслано:

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->addMinutes(10), new OrderShipped($order));
```

<a name="pushing-to-specific-queues"></a>

#### Постановка поштового повідомлення у певну чергу

Оскільки всі поштові класи, створені за допомогою команди `make:mail`, за замовчуванням вони використовують трейт `Illuminate\Bus\Queueable`, ви можете викликати методи `onQueue` і `onConnection` у будь-якому екземплярі поштового класа, дозволяючи вказати підключення та назву черги для поштового повідомлення:

```php
$message = (new OrderShipped($order))
        ->onConnection('sqs')
        ->onQueue('emails');

Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue($message);
```

<a name="queueing-by-default"></a>

#### Постановка в чергу за замовчуванням

Якщо у вас є поштові класи, які ви хочете завжди ставити в чергу, ви можете реалізувати контракт `ShouldQueue` для класу. Тепер, навіть якщо ви викликаєте метод `send` під час надсилання, поштове повідомлення, все одно буде поставлено до черги, оскільки в ньому присутня реалізація контракта:

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    //
}
```

<a name="queued-mailables-and-database-transactions"></a>

#### Поштові повідомлення в черзі та транзакції в базі даних

Коли поштові повідомлення додані до черги відправляються в рамках транзакцій бази даних, вони можуть бути оброблені чергою до того, як транзакція бази даних буде зафіксована. Коли це станеться, будь-які оновлення, які ви внесли в моделі або записи бази даних під час транзакції бази даних, можуть ще не відображатися в базі даних. Крім того, будь-які моделі або записи бази даних, створені в рамках транзакції, можуть не існувати в базі даних. Якщо ваше поштове повідомлення залежить від цих моделей, під час виконання завдання, яке надсилає поштові повідомлення в черзі, можуть виникнути неочікувані помилки.

Якщо для параметра конфігурації `after_commit` вашого підключення з чергою встановлено значення `false`, ви все ще можете вказати, що конкретне поштове повідомлення в черзі має бути відправлено після того, як всі транзакції відкритої бази даних будуть зафіксовані, викликавши метод `after_commit` під час надсилання поштового повідомлення:

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

Крім того, ви можете викликати метод `afterCommit` з конструктора вашого поштового повідомлення:

```php
<?php

namespace App\Mail;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class OrderShipped extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;

    /**
     * Створити новий екземпляр повідомлення.
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
> Щоб дізнатися більше про вирішення цих проблем, перегляньте документацію щодо [завдань у черзі та транзакцій бази даних](queues.md#jobs-and-database-transactions).

<a name="rendering-mailables"></a>

## Візуалізація розсилки

Іноді вам може знадобитися захопити HTML-вміст листа, який надсилається, не надсилаючи його. Щоб досягти цього, ви можете викликати метод `render` поштового повідомлення. Цей метод поверне оцінений HTML-вміст поштового повідомлення у вигляді рядка:

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```

<a name="previewing-mailables-in-the-browser"></a>

### Попередній перегляд поштових повідомлень у браузері

Під час розробки шаблону поштового повідомлення зручніше переглядати відтворений поштове повідомлення у вашому браузері, як типовий шаблон Blade. З цієї причини Laravel дозволяє повертати будь-яку пошту безпосередньо з замикання маршрута або контролера. Коли лист повертається, він буде відтворений і відображений у браузері, що дозволить вам швидко переглянути його дизайн без необхідності надсилати його на справжню адресу електронної пошти:

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```

> **Warning**  
> Вбудовані вкладення не відображатимуться під час попереднього перегляду поштового повідомлення у вашому браузері. Щоб переглянути ці листи, ви повинні надіслати їх до додатка для тестування електронної пошти, наприклад [MailHog](https://github.com/mailhog/MailHog) або [HELO](https://usehelo.com).

<a name="localizing-mailables"></a>

## Локалізація розсилки

Laravel дозволяє надсилати поштові повідомлення використовуючи мову, яка відрізняється від поточної мови запита, і навіть запам’ятовує цю мову, якщо пошта стоїть у черзі.

Щоб досягти цього, фасад `Mail` пропонує метод `locale` для встановлення потрібної мови. Додаток змінить цю мову під час оцінювання шаблону листа, а потім повернеться до попередньої мови після завершення оцінки:

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```

<a name="user-preferred-locales"></a>

### Бажана мова користовача

Іноді додатки зберігають бажану мову кожного користувача. Реалізуючи контракт `HasLocalePreference` на одній або кількох ваших моделях, ви можете наказати Laravel використовувати цю збережену мову під час надсилання пошти:

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
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>

## Тестування розсилки

Laravel пропонує кілька зручних методів для перевірки того, що ваші поштові повідомлення мають вміст, який ви очікуєте. Ці методи: `assertSeeInHtml`, `assertDontSeeInHtml`, `assertSeeInOrderInHtml`, `assertSeeInText`, `assertDontSeeInText`, і `assertSeeInOrderInText`.

Як і можна було очікувати, твердження «HTML» стверджують, що HTML-версія вашого поштового повідомлення містить певний рядок, тоді як твердження «текст» стверджують, що звичайна текстова версія вашого поштового повідомлення містить переданий рядок:

```php
use App\Mail\InvoicePaid;
use App\Models\User;

public function test_mailable_content()
{
    $user = User::factory()->create();

    $mailable = new InvoicePaid($user);

    $mailable->assertSeeInHtml($user->email);
    $mailable->assertSeeInHtml('Invoice Paid');
    $mailable->assertSeeInOrderInHtml(['Invoice Paid', 'Thanks']);

    $mailable->assertSeeInText($user->email);
    $mailable->assertSeeInOrderInText(['Invoice Paid', 'Thanks']);
}
```

<a name="testing-mailable-sending"></a>

#### Тестування відправлення поштових повідомлень

Ми пропонуємо тестувати вміст ваших поштових листів окремо від тестів, які підтверджують, що дане поштове повідомлення було «надіслано» конкретному користувачеві. Щоб дізнатися, як перевірити, чи було надіслано листи, перегляньте нашу документацію про [підробку Mail](mocking.md#mail-fake).

<a name="mail-and-local-development"></a>

## Пошта та локальна розробка

Розробляючи додаток, який надсилає електронну пошту, ви, ймовірно, не захочите надсилати електронні листи на реальні адреси електронної пошти. Laravel надає декілька способів «відключити» фактичне надсилання електронних листів під час локальної розробки.

<a name="log-driver"></a>

#### Драйвер Log

Замість того, щоб надсилати ваші електронні листи, драйвер `log` пошти записуватиме всі повідомлення електронної пошти у файли журналів для перевірки. Як правило, цей драйвер використовується лише під час локальної розробки. Щоб отримати додаткові відомості про налаштування додатка для кожного середовища, перегляньте [документацію щодо налаштування](configuration.md#environment-configuration).

<a name="mailtrap"></a>

#### HELO / Mailtrap / MailHog

Крім того, ви можете скористатися службою на зразок [HELO](https://usehelo.com) або [Mailtrap](https://mailtrap.io) і драйвером `smtp` для надсилання повідомлень електронної пошти до «фіктивної» поштової скриньки, де ви можете переглядати їх у справжньому поштовому клієнті. Перевага цього підходу полягає в тому, що ви можете фактично перевіряти остаточні електронні листи безпосередньо у поштових службах Mailtrap.

Якщо ви використовуєте [Laravel Sail](sail.md), ви можете переглянути свої повідомлення за допомогою [MailHog](https://github.com/mailhog/MailHog). Коли Sail запущено, ви можете отримати доступ до інтерфейсу MailHog за адресою: `http://localhost:8025`.

<a name="using-a-global-to-address"></a>

#### Використання єдиної адреси одержувача

Нарешті, ви можете вказати глобальну адресу "одержувача", викликавши метод `alwaysTo`, запропонований фасадом `Mail`. Як правило, цей метод слід викликати з метода `boot` одного з постачальників служб вашого додатка:

```php
use Illuminate\Support\Facades\Mail;

/**
 * Завантаження будь-яких служб додатка.
 *
 * @return void
 */
public function boot()
{
    if ($this->app->environment('local')) {
        Mail::alwaysTo('taylor@example.com');
    }
}
```

<a name="events"></a>

## Події

Laravel запускає дві події під час процесу відправлення поштових повідомлень. Подія `MessageSending` запускається перед відправленням повідомлення, тоді як подія `MessageSent` запускається після того, як повідомлення було відправлено. Пам’ятайте, що ці події запускаються під час _відправлення_ пошти, а не під час її черги. Ви можете зареєструвати слухачів подій для цієї події у вашому постачальнику служб `App\Providers\EventServiceProvider`:

```php
/**
 * Мапа слухачів подій для додатка.
 *
 * @var array
 */
protected $listen = [
    'Illuminate\Mail\Events\MessageSending' => [
        'App\Listeners\LogSendingMessage',
    ],
    'Illuminate\Mail\Events\MessageSent' => [
        'App\Listeners\LogSentMessage',
    ],
];
```

<a name="custom-transports"></a>

## Користувальницькі драйвери

Laravel містить різноманітні засоби транспортування пошти; однак ви можете написати власні транспорти для доставки електронної пошти через інші служби, які Laravel не підтримує з коробки. Для початку визначте клас, який розширюватиме клас `Symfony\Component\Mailer\Transport\AbstractTransport`. Потім реалізуйте методи `doSend` і `__toString()` у вашому класі транспорта:

```php
use MailchimpTransactional\ApiClient;
use Symfony\Component\Mailer\SentMessage;
use Symfony\Component\Mailer\Transport\AbstractTransport;
use Symfony\Component\Mime\MessageConverter;

class MailchimpTransport extends AbstractTransport
{
    /**
     * Клієнт API Mailchimp
     *
     * @var \MailchimpTransactional\ApiClient
     */
    protected $client;

    /**
     * Створіть новий транспортний екземпляр Mailchimp.
     *
     * @param  \MailchimpTransactional\ApiClient  $client
     * @return void
     */
    public function __construct(ApiClient $client)
    {
        $this->client = $client;
    }

    /**
     * {@inheritDoc}
     */
    protected function doSend(SentMessage $message): void
    {
        $email = MessageConverter::toEmail($message->getOriginalMessage());

        $this->client->messages->send(['message' => [
            'from_email' => $email->getFrom(),
            'to' => collect($email->getTo())->map(function ($email) {
                return ['email' => $email->getAddress(), 'type' => 'to'];
            })->all(),
            'subject' => $email->getSubject(),
            'text' => $email->getTextBody(),
        ]]);
    }

    /**
     * Отримайте рядкове представлення транспорту.
     *
     * @return string
     */
    public function __toString(): string
    {
        return 'mailchimp';
    }
}
```

Після того, як ви визначили свій спеціальний транспорт, ви можете зареєструвати його за допомогою метода `extend`, наданого фасадом `Mail`. Як правило, це слід робити в рамках метода `boot` постачальника служб `AppServiceProvider` вашого додатка. Аргумент `$config` буде передано до замикання, наданого методу `extend`. Цей аргумент міститиме масив конфігурації, визначений для поштомата у конфігураційному файлі `config/mail.php` вашого додатка:

```php
use App\Mail\MailchimpTransport;
use Illuminate\Support\Facades\Mail;

/**
 * Завантажте будь-які служби додатків.
 *
 * @return void
 */
public function boot()
{
    Mail::extend('mailchimp', function (array $config = []) {
        return new MailchimpTransport(/* ... */);
    })
}
```

Після того як ваш спеціальний транспорт буде визначено та зареєстровано, для використання нового транспорта ви можете створити визначення поштомата у конфігураційному файлі `config/mail.php` вашого додатка:

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    // ...
],
```

<a name="additional-symfony-transports"></a>

### Додаткові драйвери Symfony

Laravel включає підтримку деяких існуючих поштових транспортів, які підтримує Symfony, наприклад Mailgun і Postmark.
Однак ви можете розширити Laravel підтримкою додаткових транспортів які підтримує Symfony. Ви можете зробити це, запитавши необхідну поштову програму Symfony через Composer і зареєструвавши транспорт у Laravel. Наприклад, ви можете встановити та зареєструвати пощтомат Symfony "Sendinblue":

```shell
composer require symfony/sendinblue-mailer
```

Після встановлення пакета Sendinblue ви можете додати запис для своїх облікових даних API Sendinblue до файлу конфігурації `config/services.php` вашого додатка:

```php
'sendinblue' => [
    'key' => 'your-api-key',
],
```

Нарешті, ви можете використовувати метод `extend` фасада `Mail` для реєстрації транспорту в Laravel. Як правило, це слід зробити в рамках метода `boot` постачальника служб:

```php
use Illuminate\Support\Facades\Mail;
use Symfony\Component\Mailer\Bridge\Sendinblue\Transport\SendinblueTransportFactory;
use Symfony\Component\Mailer\Transport\Dsn;

/**
 * Завантажте будь-які служби додатків.
 *
 * @return void
 */
public function boot()
{
    Mail::extend('sendinblue', function () {
        return (new SendinblueTransportFactory)->create(
            new Dsn(
                'sendinblue+api',
                'default',
                config('services.sendinblue.key')
            )
        );
    });
}
```
