# Файлове сховище

- [Вступ](#introduction)
- [Налаштування](#configuration)
  - [Локальний драйвер](#the-local-driver)
  - [Публічний диск](#the-public-disk)
  - [Попередня підготовка драйвера](#driver-prerequisites)
    - [Обмеженний шлях і Файлові системи лише для читання](#scoped-and-read-only-filesystems)
    - [Файлові системи сумісні з Amazon S3](#amazon-s3-compatible-filesystems)
- [Доступ до дискових екземплярів](#obtaining-disk-instances)
  - [Диски на вимогу](#on-demand-disks)
- [Отримання файлів](#retrieving-files)
  - [Завантаження файлів](#downloading-files)
  - [URL-адреси файлів](#file-urls)
  - [Метадані файла](#file-metadata)
- [Зберігання файлів](#storing-files)
  - [Додавання інформації до файлів](#prepending-appending-to-files)
  - [Копіювання та переміщення файлів](#copying-moving-files)
  - [Автоматична потокова передача](#automatic-streaming)
  - [Завантаження файлів](#file-uploads)
  - [Видимість файлів](#file-visibility)
- [Видалення файлів](#deleting-files)
- [Каталоги](#directories)
- [Власна файлова система](#custom-filesystems)

<a name="introduction"></a>

## Вступ

Завдяки чудовому PHP-пакунку [Flysystem](https://github.com/thephpleague/flysystem) від Франка де Йонга (Frank de Jonge),Laravel забезпечує потужну абстракцію файлової системи. Інтеграція Laravel Flysystem надає прості драйвери для роботи з локальними файловими системами, SFTP і Amazon S3. Навіть краще, надзвичайно просто перемикатися між цими параметрами зберігання між вашою локальною машиною розробки та робочим сервером, оскільки API залишається незмінним для кожної системи.

<a name="configuration"></a>

## Налаштування

Конфігурація файлової системи Laravel розташована в `config/filesystems.php`. В цьому файлі ви можете налаштувати всі "диски" вашої файлової системи. Кожен диск представляє певний драйвер сховища та місце зберігання.
Приклади конфігурацій для кожного підтримуваного драйвера включено в конфігураційний файл, тож ви можете змінити конфігурацію, відповідно до ваших вимог зберігання та облікових даних.

Драйвер `local` взаємодіє з файлами, які зберігаються локально на сервері, на якому запущено додаток Laravel, тоді як драйвер `s3` використовується для запису в службу хмарного зберігання даних Amazon S3.

> **Note**
> Ви можете налаштувати скільки завгодно дисків і навіть мати декілька дисків, які використовують той самий драйвер.

<a name="the-local-driver"></a>

### Локальний драйвер

Під час використання `local` драйвера всі файлові операції відносяться до `root` каталогу, визначеного в конфігураційному файлі `filesystems.php` вашого додатка. За замовчуванням це значення встановлено на каталог `storage/app`. Таким чином, наступний метод буде записувати в `storage/app/example.txt`:

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('example.txt', 'Contents');
```

<a name="the-public-disk"></a>

### Публічний диск

Диск `public`, включений в конфігураційному файлі `filesystems.php` вашого додатка, призначений для файлів, які будуть загальнодоступними. За замовчуванням диск `public` використовує `local` драйвер і зберігає свої файли в `storage/app/public`.

Щоб зробити ці файли доступними з Інтернету, вам слід створити символічне посилання від `public/storage` до `storage/app/public`. Застосування цієї угоди про папки зберігатиме ваші загальнодоступні файли в одному каталозі, якими можна легко ділитися між розгортаннями під час використання систем розгортання без простоїв, таких як [Envoyer](https://envoyer.io/).

Щоб створити символічне посилання, ви можете використати команду Artisan `storage:link`:

```shell
php artisan storage:link
```

Після того, як файл збережено та створено символічне посилання, ви можете сшенерувати URL-адресу для файлів за допомогою помічника `asset`:

```php
echo asset('storage/file.txt');
```

Ви можете налаштувати додаткові символічні посилання в конфігураційному файлі `filesystems.php` вашого додатка. Кожне з налаштованих посилань буде створено, коли ви запустите команду `storage:link`:

```php
'links' => [
    public_path('storage') => storage_path('app/public'),
    public_path('images') => storage_path('app/images'),
],
```

<a name="driver-prerequisites"></a>

### Попередня підготовка дравера

<a name="s3-driver-configuration"></a>

#### Налаштування драйвера S3

Перед використанням драйвера S3 вам потрібно буде встановити пакет Flysystem S3 за допомогою менеджера пакунків Composer:

```shell
composer require league/flysystem-aws-s3-v3 "^3.0"
```

Інформація про налаштування драйвера S3 міститься в конфігураційному файлі `config/filesystems.php`. Цей файл містить приклад масива конфігурації для драйвера S3. Ви можете змінювати цей масив за допомогою власної конфігурації S3 та облікових даних. Для зручності ці змінні середовища відповідають правилам іменування, які використовуються в AWS CLI.

<a name="ftp-driver-configuration"></a>

#### Налаштування драйверів FTP

Перед використанням FTP-драйвера вам потрібно буде встановити пакет Flysystem FTP за допомогою менеджера пакунків Composer:

```shell
composer require league/flysystem-ftp "^3.0"
```

Інформація про налаштування драйвера S3 міститься в конфігураційному файлі `config/filesystems.php`. Цей файл містить приклад масиву конфігурації для драйвера S3. Ви можете змінювати цей масив за допомогою власної конфігурації S3 та облікових даних. Для зручності ці змінні середовища відповідають правилам іменування, які використовуються в AWS CLI.

```php
'ftp' => [
    'driver' => 'ftp',
    'host' => env('FTP_HOST'),
    'username' => env('FTP_USERNAME'),
    'password' => env('FTP_PASSWORD'),

    // Додаткові налаштування FTP...
    // 'port' => env('FTP_PORT', 21),
    // 'root' => env('FTP_ROOT'),
    // 'passive' => true,
    // 'ssl' => true,
    // 'timeout' => 30,
],
```

<a name="sftp-driver-configuration"></a>

#### Налаштування драйвера SFTP

Перед використанням драйвера SFTP вам потрібно буде встановити пакет SFTP Flysystem через менеджер пакетів Composer:

```shell
composer require league/flysystem-sftp-v3 "^3.0"
```

Інтеграція Flysystem від Laravel чудово працює з SFTP; однак зразок конфігурації не надається у конфігураційному файлі фреймворка за замовчуванням `filesystems.php`. Якщо вам потрібно налаштувати файлову систему SFTP, ви можете скористатися наведеним нижче прикладом конфігурації:

```php
'sftp' => [
    'driver' => 'sftp',
    'host' => env('SFTP_HOST'),

    // Налаштування базової автентифікації...
    'username' => env('SFTP_USERNAME'),
    'password' => env('SFTP_PASSWORD'),

    // Налаштування автентифікації на основі ключа SSH із паролем шифрування...
    'privateKey' => env('SFTP_PRIVATE_KEY'),
    'password' => env('SFTP_PASSWORD'),

    // Додаткові налаштування SFTP...
    // 'hostFingerprint' => env('SFTP_HOST_FINGERPRINT'),
    // 'maxTries' => 4,
    // 'passphrase' => env('SFTP_PASSPHRASE'),
    // 'port' => env('SFTP_PORT', 22),
    // 'root' => env('SFTP_ROOT', ''),
    // 'timeout' => 30,
    // 'useAgent' => true,
],
```

<a name="scoped-and-read-only-filesystems"></a>

### Обмеженний шлях і Файлові системи лише для читання

Ви можете створити екземпляр будь-якого існуючого диска файлової системи з обмеженим шляхом, визначивши диск, який використовує драйвер `scoped`. Диски з обмеженою областю дозволяють визначити файлову систему, де всі шляхи автоматично матимуть префікс заданого шляху. Наприклад, ви можете створити диск, який обмежуватиме ваш існуючий диск `s3` певним префіксом шляху, і тоді кожна операція з файлами, яка використовує ваш обмежений диск, використовуватиме вказаний префікс:

```php
's3-videos' => [
    'driver' => 'scoped',
    'disk' => 's3',
    'prefix' => 'path/to/videos',
],
```

Якщо ви бажаєте вказати, що будь-який диск файлової системи має бути «тільки для читання», ви можете додати параметр конфігурації `read-only` в масив конфігурації диска:

```php
's3-videos' => [
    'driver' => 's3',
    // ...
    'read-only' => true,
],
```

<a name="amazon-s3-compatible-filesystems"></a>

### Сумісні файлові системи Amazon S3

За замовчуванням конфігураційний файл `filesystems.php` вашого додатка, містить конфігурацію для диска `s3`. Окрім використання цього диска для взаємодії з Amazon S3, ви можете використовувати його для взаємодії з будь-яким S3-сумісним сервісом зберігання файлів, таким як [MinIO](https://github.com/minio/minio або [DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/).

Як правило, після оновлення облікових даних диска відповідно до облікових даних служби, яку ви плануєте використовувати, вам потрібно лише оновити значення `endpoint` параметра конфігурації. Значення цього параметра зазвичай визначається через змінну середовища `AWS_ENDPOINT`:

```php
'endpoint' => env('AWS_ENDPOINT', 'https://minio:9000'),
```

<a name="obtaining-disk-instances"></a>

## Отримання екземплярів диска

Фасад `Storage` можна використовувати для взаємодії з будь-яким із налаштованих дисків. Наприклад, ви можете використовувати метод `put` фасаду, щоб зберегти аватар на диску за замовчуванням. Якщо ви викликаєте методи фасада `Storage` без попереднього виклику метода `disk`, метод буде автоматично передано на диск за замовчуванням:

```php
use Illuminate\Support\Facades\Storage;

Storage::put('avatars/1', $content);
```

Якщо ваш додаток взаємодіє з декількома дисками, ви можете використовувати метод `disk` фасада `Storage` для роботи з файлами на певному диску:

```php
Storage::disk('s3')->put('avatars/1', $content);
```

<a name="on-demand-disks"></a>

### Диски на вимогу

Іноді вам може знадобитись створити диск під час виконання, використовуючи задану конфігурацію, за фактичної відсутності цієї конфігурації в конфігураційному файлі `filesystems` вашого додатка. Щоб досягти цього, ви можете передати масив конфігурації методу `build` фасада `Storage`:

```php
use Illuminate\Support\Facades\Storage;

$disk = Storage::build([
    'driver' => 'local',
    'root' => '/path/to/root',
]);

$disk->put('image.jpg', $content);
```

<a name="retrieving-files"></a>

## Отримання файлів

Метод `get` можна використовувати для отримання вмісту файла. Необроблений рядковий вміст файла буде повернено методом. Пам’ятайте, що всі шляхи до файлів мають бути вказані відносно «root» розташування диска:

```php
$contents = Storage::get('file.jpg');
```

Щоб визначити, чи існує файл на диску, можна використовувати метод `exists`:

```php
if (Storage::disk('s3')->exists('file.jpg')) {
    // ...
}
```

Щоб визначити, чи відсутній файл на диску, можна використати метод `missing`:

```php
if (Storage::disk('s3')->missing('file.jpg')) {
    // ...
}
```

<a name="downloading-files"></a>

### Завантаження файлів

Метод `download` може бути використаний для створення відповіді, яка змушує браузер користувача завантажити файл за вказаним шляхом. Метод `download` приймає ім’я файла другим аргументом, який визначатиме ім’я файла, яке бачить користувач, під час завантаження файла. Нарешті, ви можете передати масив HTTP-заголовків третім аргументом:

```php
return Storage::download('file.jpg');

return Storage::download('file.jpg', $name, $headers);
```

<a name="file-urls"></a>

### URL-адреси файлів

Ви можете використовувати метод `url`, щоб отримати певну URL-адресу файла. Якщо ви використовуєте `local` драйвер, це зазвичай просто додає `/storage` до заданого шляху та повертає відносну URL-адресу до файла. Якщо ви використовуєте драйвер `s3`, буде повернуто повну віддалену URL-адресу:

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::url('file.jpg');
```

Під час використання `local` драйвера всі файли, які мають бути загальнодоступними, слід розташовувати в каталозі`storage/app/public`. Крім того, ви повинні [створити символічне посилання](#the-public-disk) на `public/storage`, яке вказує на каталог `storage/app/public`.

> **Warning**  
> Під час використання драйвера `local`, значення `url`, яке повертається, не кодується URL-адресою. З цієї причини ми рекомендуємо завжди зберігати ваші файли, використовуючи імена, які створять дійсні URL-адреси.

<a name="temporary-urls"></a>

#### Тимчасові URL-адреси

Використовуючи метод `temporaryUrl`, ви можете створювати тимчасові URL-адреси для файлів, які зберігаються за допомогою драйвера `s3`. Цей метод приймає шлях і екземпляр `DateTime`, який визначатиме, термін дії URL:

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::temporaryUrl(
    'file.jpg', now()->addMinutes(5)
);
```

Якщо вам потрібно вказати додаткові [параметри запита S3](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectGET.html#RESTObjectGET-requests), ви можете передати масив параметрів запита як третій аргумент методу `temporaryUrl`:

```php
$url = Storage::temporaryUrl(
    'file.jpg',
    now()->addMinutes(5),
    [
        'ResponseContentType' => 'application/octet-stream',
        'ResponseContentDisposition' => 'attachment; filename=file2.jpg',
    ]
);
```

Якщо вам потрібно налаштувати спосіб створення тимчасових URL-адрес для певного диска зберігання, ви можете скористатися методом `buildTemporaryUrlsUsing`. Наприклад, це може бути корисним, якщо у вас є контролер, який дозволяє завантажувати файли, які зберігаються на диску, який зазвичай не підтримує тимчасові URL-адреси. Зазвичай цей метод слід викликати з методу `boot` постачальника служб:

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\URL;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Завантажте будь-які служби додатка.
     *
     * @return void
     */
    public function boot()
    {
        Storage::disk('local')->buildTemporaryUrlsUsing(function ($path, $expiration, $options) {
            return URL::temporarySignedRoute(
                'files.download',
                $expiration,
                array_merge($options, ['path' => $path])
            );
        });
    }
}
```

<a name="url-host-customization"></a>

#### Налаштування хосту URL

Якщо ви бажаєте попередньо визначити хост для URL-адрес, згенерованих за допомогою фасаду `Storage`, ви можете додати параметр `url` до масиву конфігурації диска:

```php
'public' => [
    'driver' => 'local',
    'root' => storage_path('app/public'),
    'url' => env('APP_URL').'/storage',
    'visibility' => 'public',
],
```

<a name="file-metadata"></a>

### Метадані файла

Окрім читання та запису файлів, Laravel також може надавати інформацію про самі файли. Наприклад, метод `size` можна використовувати для отримання розміру файла в байтах:

```php
use Illuminate\Support\Facades\Storage;

$size = Storage::size('file.jpg');
```

Метод `lastModified` повертає мітку часу UNIX останньої зміни файлу:

```php
$time = Storage::lastModified('file.jpg');
```

<a name="file-paths"></a>

#### Шляхи до файлів

Ви можете використати метод `path`, щоб отримати шлях до певного файлу. Якщо ви використовуєте драйвер `local`, буде повернено абсолютний шлях до файлу. Якщо ви використовуєте драйвер `s3`, цей метод поверне відносний шлях до файлу в сегменті S3:

```php
use Illuminate\Support\Facades\Storage;

$path = Storage::path('file.jpg');
```

<a name="storing-files"></a>

## Зберігання файлів

Метод `put` може бути використаний для зберігання вмісту файла на диску. Ви також можете передати `resource` PHP методу `put`, який використовуватиме базову підтримку потоку Flysystem. Пам’ятайте, що всі шляхи до файлів мають бути вказані відносно «root» розташування, налаштованого для диска:

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents);

Storage::put('file.jpg', $resource);
```

<a name="failed-writes"></a>

#### Невдалий запис

Якщо метод `put` (або інші операції «запису») не можуть записати файл на диск, буде повернуто `false`:

```php
if (!Storage::put('file.jpg', $contents)) {
    // Не вдалося записати файл на диск...
}
```

Якщо ви бажаєте, ви можете визначити параметр `throw` в конфігураційному масиві файлової системи вашого диска. Якщо цей параметр визначено як `true`, методи «запису», такий як `put`, створюватимуть екземпляр `League\Flysystem\UnableToWriteFile`, коли операції запису завершаться помилкою:

```php
'public' => [
    'driver' => 'local',
    // ...
    'throw' => true,
],
```

<a name="prepending-appending-to-files"></a>

### Додавання інформації до файлів

Методи `prepend` і `append` дозволяють писати на початку або в кінці файлу:

```php
Storage::prepend('file.log', 'Prepended Text');

Storage::append('file.log', 'Appended Text');
```

<a name="copying-moving-files"></a>

### Копіювання та переміщення файлів

Метод `copy` можна використовувати для копіювання існуючого файлу в нове місце на диску, тоді як метод `move` можна використовувати для перейменування або переміщення існуючого файлу в нове місце:

```php
Storage::copy('old/file.jpg', 'new/file.jpg');

Storage::move('old/file.jpg', 'new/file.jpg');
```

<a name="automatic-streaming"></a>

### Автоматичне потокове передавання

Потокова передача файлів у сховище значно зменшує використання пам’яті. Якщо ви бажаєте, щоб Laravel автоматично керував потоковим передаванням даного файлу у ваше місце зберігання, ви можете використати метод `putFile` або `putFileAs`. Цей метод приймає екземпляр `Illuminate\Http\File` або `Illuminate\Http\UploadedFile` і автоматично передає файл у потрібне місце:

```php
use Illuminate\Http\File;
use Illuminate\Support\Facades\Storage;

// Автоматично генерувати унікальний ID для імені файлу...
$path = Storage::putFile('photos', new File('/path/to/photo'));

// Вручну вказати назву файла...
$path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');
```

Є декілька важливих речей, які слід зазначити про метод `putFile`. Зверніть увагу, що ми вказали лише назву каталогу, а не назву файла. За замовчуванням метод `putFile` створить унікальний ID, який буде використовуватися як ім’я файла. Розширення файла буде визначено шляхом перевірки типу файла MIME. Шлях до файла буде повернено методом `putFile`, щоб ви могли зберегти шлях, включаючи згенероване ім’я файла, у своїй базі даних.

Методи `putFile` і `putFileAs` також приймають аргумент для визначення «видимості» збереженого файлу. Це особливо корисно, якщо ви зберігаєте файл на хмарному диску, наприклад Amazon S3, і хочете, щоб файл був загальнодоступним через згенеровані URL-адреси:

```php
Storage::putFile('photos', new File('/path/to/photo'), 'public');
```

<a name="file-uploads"></a>

### Завантаження файлів

У веб-додатках одним із найпоширеніших випадків зберігання файлів є зберігання завантажених користувачами файлів, таких як фотографії та документи. Laravel дозволяє дуже легко зберігати завантажені файли за допомогою метода `store` на екземплярі завантаженого файла. Викличте метод `store` зі шляхом, за яким ви бажаєте зберегти завантажений файл:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;

class UserAvatarController extends Controller
{
    /**
     * Оновіть аватар користувача.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request)
    {
        $path = $request->file('avatar')->store('avatars');

        return $path;
    }
}
```

Є декілька важливих речей, які слід зазначити щодо цього прикладу. Зверніть увагу, що ми вказали лише назву каталога, а не назву файла. За замовчуванням метод `store` генеруватиме унікальний ID, який буде використовуватися як ім’я файлу. Розширення файлу буде визначено шляхом перевірки типу файлу MIME. Шлях до файлу буде повернено методом `store`, щоб ви могли зберегти шлях, включаючи згенероване ім’я файлу, в своїй базі даних.

Ви також можете викликати метод `putFile` фасада `Storage`, щоб виконати таку саму операцію збереження файлів, як у прикладі вище:

```php
$path = Storage::putFile('avatars', $request->file('avatar'));
```

<a name="specifying-a-file-name"></a>

#### Визначення імені файла

Якщо ви не хочете, щоб ім’я файла автоматично призначалось вашому збереженому файлу, ви можете використати метод `storeAs`, який отримує шлях, ім’я файла та (необов’язковий) диск як аргументи:

```php
$path = $request->file('avatar')->storeAs(
    'avatars', $request->user()->id
);
```

Ви також можете використовувати метод `putFileAs` фасада `Storage`, який виконає ту саму операцію зберігання файлів, що й у прикладі вище:

```php
$path = Storage::putFileAs(
    'avatars', $request->file('avatar'), $request->user()->id
);
```

> **Warning**  
> Недруковані та недійсні символи Unicode будуть автоматично видалені зі шляхів до файлів. Тому ви можете очистити ваші шляхи до файлів перед тим, як передати їх до методів зберігання файлів Laravel. Шляхи до файлів нормалізуються за допомогою метода `League\Flysystem\WhitespacePathNormalizer::normalizePath`.

<a name="specifying-a-disk"></a>

#### Визначення диска

За замовчуванням метод `store` цього завантаженого файла використовуватиме ваш диск за замовчуванням. Якщо ви хочете вказати інший диск, передайте ім’я диска другим аргументом методу `store`:

```php
$path = $request->file('avatar')->store(
    'avatars/' . $request->user()->id, 's3'
);
```

Якщо ви використовуєте метод `storeAs`, ви можете передати методу назву диска третім аргументом:

```php
$path = $request->file('avatar')->storeAs(
    'avatars',
    $request->user()->id,
    's3'
);
```

<a name="other-uploaded-file-information"></a>

#### Інша інформація завантаженого файла

Якщо ви хочете отримати оригінальну назву та розширення завантаженого файла, ви можете зробити це за допомогою методів `getClientOriginalName` і `getClientOriginalExtension`:

```php
$file = $request->file('avatar');

$name = $file->getClientOriginalName();
$extension = $file->getClientOriginalExtension();
```

Однак майте на увазі, що методи `getClientOriginalName` і `getClientOriginalExtension` вважаються небезпечними, оскільки зловмисник може підробити назву файла та розширення. З цієї причини зазвичай ви повинні віддавати перевагу методам `hashName` і `extension`, щоб отримати ім’я та розширення для даного завантаженого файла:

Однак майте на увазі, що методи `getClientOriginalName` і `getClientOriginalExtension` вважаються небезпечними, оскільки зловмисник може підробити назву файла та розширення. З цієї причини зазвичай ви повинні віддавати перевагу методам `hashName` і `extension`, щоб отримати ім’я та розширення для даного завантаженого файла:

```php
$file = $request->file('avatar');

$name = $file->hashName(); // Згенеруйте унікальне випадкове ім'я...
$extension = $file->extension(); // З'ясуйте розширення файла на основі типу MIME файла...
```

<a name="file-visibility"></a>

### Видимість файла

В інтеграції Laravel Flysystem «видимість» - абстракція прав доступу до файлів на декількох платформах. Файли можуть бути оголошені `public` або `private`. Коли файл оголошено `public`, ви вказуєте на те, що він має бути загальнодоступним для інших. Наприклад, використовуючи драйвер S3, ви можете отримати URL-адреси `public` файлів.

Ви можете встановити видимість під час запису файла за допомогою метода `put`:

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents, 'public');
```

Якщо файл вже збережено, його видимість можна отримати та встановити за допомогою методів `getVisibility` та `setVisibility`:

```php
$visibility = Storage::getVisibility('file.jpg');

Storage::setVisibility('file.jpg', 'public');
```

Під час взаємодії із завантаженими файлами ви можете використовувати методи `storePublicly` і `storePubliclyAs`, щоб зберегти завантажений файл у `public` видимості:

```php
$path = $request->file('avatar')->storePublicly('avatars', 's3');

$path = $request->file('avatar')->storePubliclyAs(
    'avatars',
    $request->user()->id,
    's3'
);
```

<a name="local-files-and-visibility"></a>

#### Локальні файли та видимість

Під час використання `local` драйвера `public` [видимість](#file-visibility) перетворюється на дозволи `0755` для каталогів і дозволи `0644` для файлів. Ви можете змінити зіставлення дозволів у кофігураційному файлі `filesystems` вашого додатка:

    'local' => [
        'driver' => 'local',
        'root' => storage_path('app'),
        'permissions' => [
            'file' => [
                'public' => 0644,
                'private' => 0600,
            ],
            'dir' => [
                'public' => 0755,
                'private' => 0700,
            ],
        ],
    ],

<a name="deleting-files"></a>

## Видалення файлів

Метод `delete` приймає одне ім’я файла або масив файлів для видалення:

```php
use Illuminate\Support\Facades\Storage;

Storage::delete('file.jpg');

Storage::delete(['file.jpg', 'file2.jpg']);
```

Якщо необхідно, ви можете вказати диск, з якого потрібно видалити файл:

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('s3')->delete('path/file.jpg');
```

<a name="directories"></a>

## Каталоги

<a name="get-all-files-within-a-directory"></a>

#### Отримати всі файли в каталозі

Метод `files` повертає масив всіх файлів у заданому каталозі. Якщо ви хочете отримати список всіх файлів у даному каталозі, включаючи всі підкаталоги, ви можете скористатися методом `allFiles`:

```php
use Illuminate\Support\Facades\Storage;

$files = Storage::files($directory);

$files = Storage::allFiles($directory);
```

<a name="get-all-directories-within-a-directory"></a>

#### Отримати всі каталоги в одному каталозі

Метод `directories` повертає масив всіх каталогів у заданому каталозі. Крім того, ви можете використовувати метод `allDirectories`, щоб отримати список всіх каталогів у даному каталозі та всі його підкаталоги:

```php
$directories = Storage::directories($directory);

$directories = Storage::allDirectories($directory);
```

<a name="create-a-directory"></a>

#### Створіть каталог

Метод `makeDirectory` створить вказаний каталог, включаючи всі необхідні підкаталоги:

```php
Storage::makeDirectory($directory);
```

<a name="delete-a-directory"></a>

#### Видалити каталог

Нарешті, метод `deleteDirectory` можна використовувати для видалення каталога та всіх його файлів:

```php
Storage::deleteDirectory($directory);
```

<a name="custom-filesystems"></a>

## Власна файлова система

Інтеграція Laravel Flysystem забезпечує підтримку декількох «драйверів» із коробки; однак Flysystem не обмежується цим і має адаптери для багатьох інших систем зберігання. Ви можете створити спеціальний драйвер, якщо хочете використовувати один з цих додаткових адаптерів у своєму додатку Laravel.

Щоб визначити власну файлову систему, вам знадобиться адаптер Flysystem. Давайте додамо до нашого проекту адаптер Dropbox, який підтримується спільнотою:

```shell
composer require spatie/flysystem-dropbox
```

Далі ви можете зареєструвати драйвер у методі `boot` одного з [постачальників служб](providers.md) вашого додатка. Щоб досягти цього, ви повинні використовувати метод `extend` фасада `Storage`:

```php
<?php

namespace App\Providers;

use Illuminate\Filesystem\FilesystemAdapter;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\ServiceProvider;
use League\Flysystem\Filesystem;
use Spatie\Dropbox\Client as DropboxClient;
use Spatie\FlysystemDropbox\DropboxAdapter;

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
        Storage::extend('dropbox', function ($app, $config) {
            $adapter = new DropboxAdapter(new DropboxClient(
                $config['authorization_token']
            ));

            return new FilesystemAdapter(
                new Filesystem($adapter, $config),
                $adapter,
                $config
            );
        });
    }
}
```

Перший аргумент метода `extend` — це ім’я драйвера, а другий — це замикання, яке отримує змінні `$app` і `$config`. Замикання має повертати екземпляр `Illuminate\Filesystem\FilesystemAdapter`. Змінна `$config` містить значення, визначені в `config/filesystems.php` для зазначеного диска.

Після того, як ви створили та зареєстрували постачальника служб розширення, ви можете використовувати драйвер `dropbox` у вашому конфігураційному файлі `config/filesystems.php`.
