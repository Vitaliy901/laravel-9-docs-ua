# Планування завдань

- [Вступ](#introduction)
- [Визначення розкладів](#defining-schedules)
  - [Планування команд Artisan](#scheduling-artisan-commands)
  - [Планування завдань у черзі](#scheduling-queued-jobs)
  - [Планування команд операційної системи](#scheduling-shell-commands)
  - [Методи періодичності розкладу](#schedule-frequency-options)
  - [Часові пояси](#timezones)
  - [Попередження дублюючих завдань](#preventing-task-overlaps)
  - [Виконання завдань на одному сервері](#running-tasks-on-one-server)
  - [Фонові завдання](#background-tasks)
  - [Режим обслуговування](#maintenance-mode)
- [Запуск планувальника](#running-the-scheduler)
  - [Запуск планувальника локально](#running-the-scheduler-locally)
- [Вивід завдань](#task-output)
- [Гачки завдань](#task-hooks)
- [Події](#events)

<a name="introduction"></a>

## Вступ

У минулому ви, можливо, писали запис конфігурації cron для кожного завдання, яке потрібно було запланувати на вашому сервері. Однак це може швидко стати проблемою, оскільки ваш розклад завдань більше не знаходиться в системі керування версіями, і вам потрібно підключитися через SSH до свого сервера, щоб переглянути наявні записи cron або додати додаткові записи.

Планувальник команд Laravel пропонує новий підхід до керування запланованими завданнями на вашому сервері. Планувальник дозволяє вам вільно та чітко визначати свій розклад команд у самому додатку Laravel. Під час використання планувальника на вашому сервері потрібен лише один запис cron. Ваш розклад завдань визначається в методі `schedule` файла `app/Console/Kernel.php`. Щоб допомогти вам розпочати, у методі визначено простий приклад.

<a name="defining-schedules"></a>

## Визначення розкладів

Ви можете визначити всі заплановані завдання в методі `schedule` класа `App\Console\Kernel` вашого дадатка. Для початку розглянемо приклад. У цьому прикладі ми заплануємо замикання, яке буде викликатись щодня опівночі. Всередині замикання ми виконаємо запит до бази даних, щоб очистити таблицю:

```php
<?php

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;
use Illuminate\Support\Facades\DB;

class Kernel extends ConsoleKernel
{
    /**
     * Визначте розклад виконання команд додатка.
     *
     * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->call(function () {
            DB::table('recent_users')->delete();
        })->daily();
    }
}
```

Окрім планування за допомогою замикання, ви також можете використовувати [викликаємі об’єкти](https://secure.php.net/manual/en/language.oop5.magic.php#object.invoke). Об’єкти, які можна викликати, — це прості класи PHP, які містять метод `__invoke`:

```php
$schedule->call(new DeleteRecentUsers)->daily();
```

Якщо ви хочете переглянути список ваших запланованих завдань а також, коли їх заплановано запустити наступного разу, то ви можете використовувати команду `schedule:list` Artisan:

```bash
php artisan schedule:list
```

<a name="scheduling-artisan-commands"></a>

### Планування команд Artisan

Окрім планування замикань, ви також можете планувати [команди Artisan](artisan.md) і системні команди. Наприклад, ви можете використовувати метод `command` для планування виконання команди Artisan, використовуючи назву або клас команди.

Під час планування команд Artisan за допомогою назви класу команди, ви можете передати масив додаткових аргументів командного рядка, які слід надати команді під час її виклику:

```php
use App\Console\Commands\SendEmailsCommand;

$schedule->command('emails:send Taylor --force')->daily();

$schedule->command(SendEmailsCommand::class, ['Taylor', '--force'])->daily();
```

<a name="scheduling-queued-jobs"></a>

### Планування завдань у черзі

Метод `job` можна використовувати для планування [завдання в черзі](queues.md). Цей метод забезпечує зручний спосіб планування таких завдань без використання метода `call` із замиканням:

```php
use App\Jobs\Heartbeat;

$schedule->job(new Heartbeat)->everyFiveMinutes();
```

Необов’язкові другий і третій аргументи можуть бути надані методу `job`, який визначає ім’я черги та підключення до черги, які слід використовувати для постановки завдання в чергу:

```php
use App\Jobs\Heartbeat;

// Відправлення завдання до черги "heartbeats" на з'єднанні "sqs"...
$schedule->job(new Heartbeat, 'heartbeats', 'sqs')->everyFiveMinutes();
```

<a name="scheduling-shell-commands"></a>

### Планування команд операційної системи

Метод `exec` можна використовувати для передачі команди операційній системі:

```php
$schedule->exec('node /home/forge/script.js')->daily();
```

<a name="schedule-frequency-options"></a>

### Методи періодичності розкладу

Ми вже бачили декілька прикладів того, як ви можете налаштувати завдання на виконання через визначені проміжки часу. Однак існує набагато більше параметрів планування, які можна призначити:

| Метод                             | Опис                                              |
| --------------------------------- | ------------------------------------------------- |
| `->cron('* * * * *');`            | Запустіть завдання за спеціальним розкладом cron  |
| `->everyMinute();`                | Виконувати завдання щохвилини                     |
| `->everyTwoMinutes();`            | - кожні дві хвилини                               |
| `->everyThreeMinutes();`          | - кожні три хвилини                               |
| `->everyFourMinutes();`           | - кожні чотири хвилини                            |
| `->everyFiveMinutes();`           | - кожні п'ять хвилини                             |
| `->everyTenMinutes();`            | - кожні десять хвилини                            |
| `->everyFifteenMinutes();`        | - кожні п'ятнадцять хвилини                       |
| `->everyThirtyMinutes();`         | - кожні тридцять хвилини                          |
| `->hourly();`                     | - кожну годину                                    |
| `->hourlyAt(17);`                 | - о 17 хвилині кожної години                      |
| `->everyTwoHours();`              | - кожні 2 години                                  |
| `->everyThreeHours();`            | - кожні 3 години                                  |
| `->everyFourHours();`             | - кожні 4 години                                  |
| `->everySixHours();`              | - кожні 6 години                                  |
| `->daily();`                      | – щодня опівночі                                  |
| `->dailyAt('13:00');`             | – щоденно о 13:00 год                             |
| `->twiceDaily(1, 13);`            | – щоденно двічі на день: о 1:00 та 13:00          |
| `->weekly();`                     | – щотижня в неділю о 00:00                        |
| `->weeklyOn(1, '8:00');`          | – щотижня у понеділок о 8:00                      |
| `->monthly();`                    | – щомісячно першого числа о 00:00                 |
| `->monthlyOn(4, '15:00');`        | – щомісячно 4 числа о 15:00                       |
| `->twiceMonthly(1, 16, '13:00');` | – щомісяця двічі на місяць: 1 та 16 числа о 13:00 |
| `->lastDayOfMonth('15:00');`      | - щомісячно в останній день місяця о 15:00        |
| `->quarterly();`                  | – щокварталу першого дня о 00:00                  |
| `->yearly();`                     | - щорічно першого дня о 00:00                     |
| `->yearlyOn(6, 1, '17:00');`      | – щорічно у червні першого числа о 17:00          |
| `->timezone('America/New_York');` | Встановити часовий пояс для завдання              |

Ці методи можна поєднувати з додатковими обмеженнями для створення ще більш точно налаштованих розкладів, які виконуються лише в певні дні тижня. Наприклад, ви можете запланувати виконання команди щотижня в понеділок:

```php
// Запуск раз на тиждень у понеділок о 13:00...
$schedule->call(function () {
    //
})->weekly()->mondays()->at('13:00');

// Щогодини з 8:00 до 17:00 у будні...
$schedule->command('foo')
    ->weekdays()
    ->hourly()
    ->timezone('America/Chicago')
    ->between('8:00', '17:00');
```

Список додаткових обмежень розкладу можна знайти нижче:

| Метод                                    | Опис                                                        |
| ---------------------------------------- | ----------------------------------------------------------- |
| `->weekdays();`                          | Обмежте завдання буднями                                    |
| `->weekends();`                          | - вихідними                                                 |
| `->sundays();`                           | - неділею                                                   |
| `->mondays();`                           | - понеділком                                                |
| `->tuesdays();`                          | - вівторком                                                 |
| `->wednesdays();`                        | - середою                                                   |
| `->thursdays();`                         | - четвергом                                                 |
| `->fridays();`                           | - п'ятницею                                                 |
| `->saturdays();`                         | - суботою                                                   |
| `->days(array\|mixed);`                  | - Певними днями                                             |
| `->between($startTime, $endTime);`       | – часовими інтервалами початку та закінчення                |
| `->unlessBetween($startTime, $endTime);` | – через виключення часових інтервалів початку та закінчення |
| `->when(Closure);`                       | – на основі істинності результата виконаного замикання      |
| `->environments($env);`                  | – оточенням виконання                                       |

<a name="day-constraints"></a>

#### Денні обмеження

Метод `days` можна використовувати для обмеження виконання завдання певними днями тижня. Наприклад, ви можете запланувати виконання команди щогодини по неділях і середах:

```php
$schedule->command('emails:send')
            ->hourly()
            ->days([0, 3]);
```

Крім того, ви можете використовувати константи, доступні в класі `Illuminate\Console\Scheduling\Schedule`, під час визначення днів, у які має виконуватися завдання:

```php
use Illuminate\Console\Scheduling\Schedule;

$schedule->command('emails:send')
        ->hourly()
        ->days([Schedule::SUNDAY, Schedule::WEDNESDAY]);
```

<a name="between-time-constraints"></a>

#### Між часовими обмеженнями

Метод `between` можна використовувати для обмеження виконання завдання на основі часу доби:

```php
$schedule->command('emails:send')
        ->hourly()
        ->between('7:00', '22:00');
```

Так само метод `unlessBetween` може використовуватися для виключення певних періодів часу виконання завдання:

```php
$schedule->command('emails:send')
        ->hourly()
        ->unlessBetween('23:00', '4:00');
```

<a name="truth-test-constraints"></a>

#### Умовні обмеження істинності

Метод `when` може бути використаний для обмеження виконання завдання на основі результата даного тесту істинності. Іншими словами, якщо задане замикання повертає `true`, завдання виконуватиметься до тих пір, поки жодні інші обмежувальні умови не заважатимуть виконанню завдання:

```php
$schedule->command('emails:send')->daily()->when(function () {
    return true;
});
```

Метод `skip` можна розглядати як зворотний метод `when`. Якщо метод `skip` повертає `true`, заплановане завдання не буде виконано:

```php
$schedule->command('emails:send')->daily()->skip(function () {
    return true;
});
```

Якщо використовуються ланцюгові методи `when`, запланована команда виконуватиметься, лише якщо всі умови `when` повертатимуть `true`.

<a name="environment-constraints"></a>

#### Обмеження середовища

Метод `environments` можна використовувати для виконання завдань лише в заданих середовищах (as defined by the `APP_ENV` [змінній середовища](configuration.md#environment-configuration)):

```php
$schedule->command('emails:send')
        ->daily()
        ->environments(['staging', 'production']);
```

<a name="timezones"></a>

### Часові пояси

Використовуючи метод `timezone`, ви можете вказати, що час запланованого завдання має інтерпретуватися в межах заданого часового поясу:

```php
$schedule->command('report:generate')
        ->timezone('America/New_York')
        ->at('2:00')
```

Якщо ви неодноразово призначаєте той самий часовий пояс для всіх своїх запланованих завдань, ви можете визначити метод `scheduleTimezone` у своєму класі `App\Console\Kernel`. Цей метод повинен повертати часовий пояс, який призначається за змовчанням для всіх запланованих завдань:

```php
/**
 * Отримати часовий пояс, який має використовуватися за змовчанням для запланованих подій.
 *
 * @return \DateTimeZone|string|null
 */
protected function scheduleTimezone()
{
    return 'America/Chicago';
}
```

> **Warning**  
> Пам’ятайте, що в деяких часових поясах використовується літній час. Коли відбувається зміна літнього часу, ваше заплановане завдання може виконуватися двічі або навіть не виконуватися взагалі. З цієї причини ми рекомендуємо уникати планування за часовими поясами, коли це можливо.

<a name="preventing-task-overlaps"></a>

### Попередження дублюючих завдань

За замовчуванням заплановані завдання виконуватимуться, навіть якщо попередній екземпляр завдання все ще виконується. Щоб запобігти цьому, ви можете використовувати метод `withoutOverlapping`:

```php
$schedule->command('emails:send')->withoutOverlapping();
```

У цьому прикладі команда `emails:send` [Artisan command](artisan.md) виконуватиметься щохвилини, якщо вона ще не запущена. Метод `withoutOverlapping` особливо корисний, якщо у вас є завдання, час виконання яких відрізняється, що не дозволяє вам точно передбачити, скільки часу займе конкретне завдання.

Якщо необхідно, ви можете вказати, скільки хвилин має пройти до закінчення терміну дії блокування «дубюлюючих» завдань. За замовчуванням блокування закінчується через 24 години:

```php
$schedule->command('emails:send')->withoutOverlapping(10);
```

За лаштунками метод `withoutOverlapping` використовує [кеш](cache.md) вашого додатка для отримання блокувань. Якщо необхідно, ви можете очистити ці блокування кеша за допомогою команди `schedule:clear-cache` Artisan. Зазвичай це необхідно, лише якщо завдання зависає через неочікувану проблему на сервері.

<a name="running-tasks-on-one-server"></a>

### Виконання завдань на одному сервері

> **Warning**  
> Щоб скористатися цією функцією, ваш додаток має використовувати драйвер кеша `database`, `memcached`, `dynamodb`, or `redis` як драйвер кеша за замовчуванням. Крім того, всі сервери мають спілкуватися з одним центральним сервером кеша.

Якщо планувальник вашого додатка працює на кількох серверах, ви можете обмежити виконання запланованого завдання лише на одному сервері. Наприклад, припустимо, що у вас є заплановане завдання, яке створює новий звіт щоп’ятниці ввечері. Якщо планувальник завдань запущено на трьох робочих серверах, заплановане завдання буде запущено на всіх трьох серверах і тричі створюватиме звіт. Це не добре!

Щоб вказати, що завдання має виконуватися лише на одному сервері, використовуйте метод `onOneServer` під час визначення запланованого завдання. Перший сервер, який отримає завдання, забезпечить атомарне блокування завдання, щоб запобігти одночасному запуску того самого завдання іншими серверами:

```php
$schedule->command('report:generate')
        ->fridays()
        ->at('17:00')
        ->onOneServer();
```

<a name="naming-unique-jobs"></a>

#### Іменування завдань одного сервера

Іноді вам може знадобитися запланувати відправку однакового завдання з різними параметрами, при цьому вказуючи Laravel виконувати кожну перестановку завдання на одному сервері. Щоб досягти цього, ви можете призначити кожному визначенню планування унікальне ім’я за допомогою методу `name`:

```php
$schedule->job(new CheckUptime('https://laravel.com'))
            ->name('check_uptime:laravel.com')
            ->everyFiveMinutes()
            ->onOneServer();

$schedule->job(new CheckUptime('https://vapor.laravel.com'))
            ->name('check_uptime:vapor.laravel.com')
            ->everyFiveMinutes()
            ->onOneServer();
```

<a name="background-tasks"></a>

### Фонові завдання

За замовчуванням декілька завдань, запланованих одночасно, виконуватимуться послідовно відповідно до порядку, визначеного у вашому методі `schedule`. Якщо у вас є довгострокові завдання, це може призвести до того, що наступні завдання розпочнуться набагато пізніше, ніж очікувалося. Якщо ви хочете запускати завдання у фоновому режимі, щоб всі вони могли виконуватися одночасно, ви можете скористатися методом `runInBackground`:

```php
$schedule->command('analytics:report')
        ->daily()
        ->runInBackground();
```

> **Warning**  
> Метод `runInBackground` можна використовувати лише, коли заплановані завдання використовують методи `command` і `exec`.

<a name="maintenance-mode"></a>

### Режим обслуговування

Заплановані завдання вашого додатка не виконуватимуться, коли додаток перебуває в [режимі обслуговування](configuration.md#maintenance-mode), оскільки ми не хочемо, щоб ваші завдання заважали будь-якому незавершеному технічному обслуговуванню, яке ви можете виконувати на своєму сервері. Однак, якщо ви хочете змусити завдання виконуватися навіть у режимі обслуговування, ви можете викликати метод `evenInMaintenanceMode` під час визначення завдання:

```php
$schedule->command('emails:send')->evenInMaintenanceMode();
```

<a name="running-the-scheduler"></a>

## Запуск планувальника

Тепер, коли ми навчилися визначати заплановані завдання, давайте обговоримо, як насправді їх запускати на нашому сервері. Команда `schedule:run` Artisan оцінить всі ваші заплановані завдання та визначить, чи потрібно їх виконувати, на основі поточного часу сервера.

Отже, коли використовується планувальник Laravel, нам потрібно лише додати один запис конфігурації Cron на наш сервер, який щохвилини запускає команду `schedule:run`. Якщо ви не знаєте, як додати записи cron на свій сервер, спробуйте скористатися такою службою, як [Laravel Forge](https://forge.laravel.com), яка може керувати записами Cron за вас:

```shell
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

<a name="running-the-scheduler-locally"></a>

## Локальний запуск планувальника

Як правило, ви не додаєте запис cron планувальника до вашої локальної машини розробки. Замість цього ви можете використовувати команду `schedule:work` Artisan. Ця команда виконуватиметься на передньому плані та запускатиме планувальник щохвилини, доки ви не завершите команду:

```shell
php artisan schedule:work
```

<a name="task-output"></a>

## Вивід завдань

Планувальник Laravel надає декілька зручних методів для роботи з результатами, створеними запланованими завданнями. По-перше, використовуючи метод `sendOutputTo`, ви можете надіслати вивід у файл для подальшої перевірки:

```php
$schedule->command('emails:send')
        ->daily()
        ->sendOutputTo($filePath);
```

Якщо ви бажаєте додати результат до даного файлу, ви можете використати метод `appendOutputTo`:

```php
$schedule->command('emails:send')
        ->daily()
        ->appendOutputTo($filePath);
```

Використовуючи метод `emailOutputTo`, ви можете відправити вихідні дані електронною поштою на адресу електронної пошти за вашим вибором. Перш ніж надсилати результат завдання електронною поштою, вам слід налаштувати [служби електронної пошти](mail.md) Laravel:

```php
$schedule->command('report:generate')
        ->daily()
        ->sendOutputTo($filePath)
        ->emailOutputTo('taylor@example.com');
```

Якщо ви хочете надіслати вивід електронною поштою, лише якщо запланована команда (Artisan або системна) команда закінчується ненульовим кодом, скористайтеся методом `emailOutputOnFailure`:

```php
$schedule->command('report:generate')
        ->daily()
        ->emailOutputOnFailure('taylor@example.com');
```

> **Warning**  
> Методи `emailOutputTo`, `emailOutputOnFailure`, `sendOutputTo`, та `appendOutputTo` є ексклюзивними для методів `command` and `exec`.

<a name="task-hooks"></a>

## Гачки завдань

Використовуючи методи `before` і `after`, ви можете вказати код, який буде виконано до і після виконання запланованого завдання:

```php
$schedule->command('emails:send')
        ->daily()
        ->before(function () {
            // The task is about to execute...
        })
        ->after(function () {
            // The task has executed...
        });
```

Методи `onSuccess` and `onFailure` дозволяють вказати код, який буде виконано, якщо заплановане завдання буде успішним або невдалим. Помилка вказує на те, що запланована команда (Artisan або системна) команда завершилася поверненням ненульового коду:

```php
$schedule->command('emails:send')
        ->daily()
        ->onSuccess(function () {
            // завдання успішно виконано...
        })
        ->onFailure(function () {
            // Невдалось виконати задачу...
        });
```

Якщо вивід доступний у вашій команді, ви можете отримати до нього доступ у хуках `after`, `onSuccess` or `onFailure`, вказавши тип екземпляра `Illuminate\Support\Stringable` як аргумент `$output` у визначенні замикання вашого хука:

```php
use Illuminate\Support\Stringable;

$schedule->command('emails:send')
        ->daily()
        ->onSuccess(function (Stringable $output) {
            // завдання успішно виконано...
        })
        ->onFailure(function (Stringable $output) {
           // Невдалось виконати задачу...
        });
```

<a name="pinging-urls"></a>

#### Пінгування URL-адрес

Використовуючи методи `pingBefore` and `thenPing`, планувальник може автоматично перевіряти певну URL-адресу до або після виконання завдання. Цей метод корисний для сповіщення зовнішньої служби, такої як [Envoyer](https://envoyer.io), про те, що ваше заплановане завдання починає або закінчило виконання:

```php
$schedule->command('emails:send')
        ->daily()
        ->pingBefore($url)
        ->thenPing($url);
```

Методи `pingBeforeIf` and `thenPingIf` можна використовувати для пінгування заданої URL-адреси, лише якщо задана умова `$condition` істинна:

```php
$schedule->command('emails:send')
        ->daily()
        ->pingBeforeIf($condition, $url)
        ->thenPingIf($condition, $url);
```

Методи `pingOnSuccess` and `pingOnFailure` можна використовувати для пінгування певної URL-адреси лише в разі успішного або невдалого виконання завдання. Помилка вказує на те, що запланована команда (Artisan або системна) команда завершилася з ненульовим кодом:

```php
$schedule->command('emails:send')
        ->daily()
        ->pingOnSuccess($successUrl)
        ->pingOnFailure($failureUrl);
```

Для всіх методів пінгування потрібна HTTP-бібліотека Guzzle. Guzzle зазвичай встановлюється в усіх нових проектах Laravel за замовчуванням, але ви можете вручну встановити Guzzle у свій проект за допомогою менеджера пакетів Composer, якщо його було випадково видалено:

```shell
composer require guzzlehttp/guzzle
```

<a name="events"></a>

## Події

За потреби ви можете прослухати [події](events.md), які розсилає планувальник. Як правило, реєстрація слухачів подій визначаються в класі `App\Providers\EventServiceProvider` вашого додатка:

```php
/**
 * Мапа слухачів подій додатка.
 *
 * @var array
 */
protected $listen = [
    'Illuminate\Console\Events\ScheduledTaskStarting' => [
        'App\Listeners\LogScheduledTaskStarting',
    ],

    'Illuminate\Console\Events\ScheduledTaskFinished' => [
        'App\Listeners\LogScheduledTaskFinished',
    ],

    'Illuminate\Console\Events\ScheduledBackgroundTaskFinished' => [
        'App\Listeners\LogScheduledBackgroundTaskFinished',
    ],

    'Illuminate\Console\Events\ScheduledTaskSkipped' => [
        'App\Listeners\LogScheduledTaskSkipped',
    ],

    'Illuminate\Console\Events\ScheduledTaskFailed' => [
        'App\Listeners\LogScheduledTaskFailed',
    ],
];
```
