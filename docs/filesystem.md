# Файлове сховище

- [Вступ](#introduction)
- [Налаштування](#configuration)
  - [Локальний драйвер](#the-local-driver)
  - [Публічний диск](#the-public-disk)
  - [Попередня підготовка дравера](#driver-prerequisites)
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

#### Failed Writes

If the `put` method (or other "write" operations) is unable to write the file to disk, `false` will be returned:

    if (! Storage::put('file.jpg', $contents)) {
        // The file could not be written to disk...
    }

If you wish, you may define the `throw` option within your filesystem disk's configuration array. When this option is defined as `true`, "write" methods such as `put` will throw an instance of `League\Flysystem\UnableToWriteFile` when write operations fail:

    'public' => [
        'driver' => 'local',
        // ...
        'throw' => true,
    ],

<a name="prepending-appending-to-files"></a>

### Prepending & Appending To Files

The `prepend` and `append` methods allow you to write to the beginning or end of a file:

    Storage::prepend('file.log', 'Prepended Text');

    Storage::append('file.log', 'Appended Text');

<a name="copying-moving-files"></a>

### Copying & Moving Files

The `copy` method may be used to copy an existing file to a new location on the disk, while the `move` method may be used to rename or move an existing file to a new location:

    Storage::copy('old/file.jpg', 'new/file.jpg');

    Storage::move('old/file.jpg', 'new/file.jpg');

<a name="automatic-streaming"></a>

### Automatic Streaming

Streaming files to storage offers significantly reduced memory usage. If you would like Laravel to automatically manage streaming a given file to your storage location, you may use the `putFile` or `putFileAs` method. This method accepts either an `Illuminate\Http\File` or `Illuminate\Http\UploadedFile` instance and will automatically stream the file to your desired location:

    use Illuminate\Http\File;
    use Illuminate\Support\Facades\Storage;

    // Automatically generate a unique ID for filename...
    $path = Storage::putFile('photos', new File('/path/to/photo'));

    // Manually specify a filename...
    $path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');

There are a few important things to note about the `putFile` method. Note that we only specified a directory name and not a filename. By default, the `putFile` method will generate a unique ID to serve as the filename. The file's extension will be determined by examining the file's MIME type. The path to the file will be returned by the `putFile` method so you can store the path, including the generated filename, in your database.

The `putFile` and `putFileAs` methods also accept an argument to specify the "visibility" of the stored file. This is particularly useful if you are storing the file on a cloud disk such as Amazon S3 and would like the file to be publicly accessible via generated URLs:

    Storage::putFile('photos', new File('/path/to/photo'), 'public');

<a name="file-uploads"></a>

### File Uploads

In web applications, one of the most common use-cases for storing files is storing user uploaded files such as photos and documents. Laravel makes it very easy to store uploaded files using the `store` method on an uploaded file instance. Call the `store` method with the path at which you wish to store the uploaded file:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;

    class UserAvatarController extends Controller
    {
        /**
         * Update the avatar for the user.
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

There are a few important things to note about this example. Note that we only specified a directory name, not a filename. By default, the `store` method will generate a unique ID to serve as the filename. The file's extension will be determined by examining the file's MIME type. The path to the file will be returned by the `store` method so you can store the path, including the generated filename, in your database.

You may also call the `putFile` method on the `Storage` facade to perform the same file storage operation as the example above:

    $path = Storage::putFile('avatars', $request->file('avatar'));

<a name="specifying-a-file-name"></a>

#### Specifying A File Name

If you do not want a filename to be automatically assigned to your stored file, you may use the `storeAs` method, which receives the path, the filename, and the (optional) disk as its arguments:

    $path = $request->file('avatar')->storeAs(
        'avatars', $request->user()->id
    );

You may also use the `putFileAs` method on the `Storage` facade, which will perform the same file storage operation as the example above:

    $path = Storage::putFileAs(
        'avatars', $request->file('avatar'), $request->user()->id
    );

> **Warning**  
> Unprintable and invalid unicode characters will automatically be removed from file paths. Therefore, you may wish to sanitize your file paths before passing them to Laravel's file storage methods. File paths are normalized using the `League\Flysystem\WhitespacePathNormalizer::normalizePath` method.

<a name="specifying-a-disk"></a>

#### Specifying A Disk

By default, this uploaded file's `store` method will use your default disk. If you would like to specify another disk, pass the disk name as the second argument to the `store` method:

    $path = $request->file('avatar')->store(
        'avatars/'.$request->user()->id, 's3'
    );

If you are using the `storeAs` method, you may pass the disk name as the third argument to the method:

    $path = $request->file('avatar')->storeAs(
        'avatars',
        $request->user()->id,
        's3'
    );

<a name="other-uploaded-file-information"></a>

#### Other Uploaded File Information

If you would like to get the original name and extension of the uploaded file, you may do so using the `getClientOriginalName` and `getClientOriginalExtension` methods:

    $file = $request->file('avatar');

    $name = $file->getClientOriginalName();
    $extension = $file->getClientOriginalExtension();

However, keep in mind that the `getClientOriginalName` and `getClientOriginalExtension` methods are considered unsafe, as the file name and extension may be tampered with by a malicious user. For this reason, you should typically prefer the `hashName` and `extension` methods to get a name and an extension for the given file upload:

    $file = $request->file('avatar');

    $name = $file->hashName(); // Generate a unique, random name...
    $extension = $file->extension(); // Determine the file's extension based on the file's MIME type...

<a name="file-visibility"></a>

### File Visibility

In Laravel's Flysystem integration, "visibility" is an abstraction of file permissions across multiple platforms. Files may either be declared `public` or `private`. When a file is declared `public`, you are indicating that the file should generally be accessible to others. For example, when using the S3 driver, you may retrieve URLs for `public` files.

You can set the visibility when writing the file via the `put` method:

    use Illuminate\Support\Facades\Storage;

    Storage::put('file.jpg', $contents, 'public');

If the file has already been stored, its visibility can be retrieved and set via the `getVisibility` and `setVisibility` methods:

    $visibility = Storage::getVisibility('file.jpg');

    Storage::setVisibility('file.jpg', 'public');

When interacting with uploaded files, you may use the `storePublicly` and `storePubliclyAs` methods to store the uploaded file with `public` visibility:

    $path = $request->file('avatar')->storePublicly('avatars', 's3');

    $path = $request->file('avatar')->storePubliclyAs(
        'avatars',
        $request->user()->id,
        's3'
    );

<a name="local-files-and-visibility"></a>

#### Local Files & Visibility

When using the `local` driver, `public` [visibility](#file-visibility) translates to `0755` permissions for directories and `0644` permissions for files. You can modify the permissions mappings in your application's `filesystems` configuration file:

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

## Deleting Files

The `delete` method accepts a single filename or an array of files to delete:

    use Illuminate\Support\Facades\Storage;

    Storage::delete('file.jpg');

    Storage::delete(['file.jpg', 'file2.jpg']);

If necessary, you may specify the disk that the file should be deleted from:

    use Illuminate\Support\Facades\Storage;

    Storage::disk('s3')->delete('path/file.jpg');

<a name="directories"></a>

## Directories

<a name="get-all-files-within-a-directory"></a>

#### Get All Files Within A Directory

The `files` method returns an array of all of the files in a given directory. If you would like to retrieve a list of all files within a given directory including all subdirectories, you may use the `allFiles` method:

    use Illuminate\Support\Facades\Storage;

    $files = Storage::files($directory);

    $files = Storage::allFiles($directory);

<a name="get-all-directories-within-a-directory"></a>

#### Get All Directories Within A Directory

The `directories` method returns an array of all the directories within a given directory. Additionally, you may use the `allDirectories` method to get a list of all directories within a given directory and all of its subdirectories:

    $directories = Storage::directories($directory);

    $directories = Storage::allDirectories($directory);

<a name="create-a-directory"></a>

#### Create A Directory

The `makeDirectory` method will create the given directory, including any needed subdirectories:

    Storage::makeDirectory($directory);

<a name="delete-a-directory"></a>

#### Delete A Directory

Finally, the `deleteDirectory` method may be used to remove a directory and all of its files:

    Storage::deleteDirectory($directory);

<a name="custom-filesystems"></a>

## Custom Filesystems

Laravel's Flysystem integration provides support for several "drivers" out of the box; however, Flysystem is not limited to these and has adapters for many other storage systems. You can create a custom driver if you want to use one of these additional adapters in your Laravel application.

In order to define a custom filesystem you will need a Flysystem adapter. Let's add a community maintained Dropbox adapter to our project:

```shell
composer require spatie/flysystem-dropbox
```

Next, you can register the driver within the `boot` method of one of your application's [service providers](/docs/{{version}}/providers). To accomplish this, you should use the `extend` method of the `Storage` facade:

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
         * Register any application services.
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * Bootstrap any application services.
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

The first argument of the `extend` method is the name of the driver and the second is a closure that receives the `$app` and `$config` variables. The closure must return an instance of `Illuminate\Filesystem\FilesystemAdapter`. The `$config` variable contains the values defined in `config/filesystems.php` for the specified disk.

Once you have created and registered the extension's service provider, you may use the `dropbox` driver in your `config/filesystems.php` configuration file.
