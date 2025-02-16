# Підтвердження адреси електронної пошти

- [Вступ](#introduction)
  - [Підготовлення моделі](#model-preparation)
  - [Підготовлення бази даних](#database-preparation)
- [Маршрутизація](#verification-routing)
  - [Повідомлення про підтвердження електронної пошти](#the-email-verification-notice)
  - [Виконавець перевірки електронної пошти](#the-email-verification-handler)
  - [Повторне надсилання електронного листа для підтвердження](#resending-the-verification-email)
  - [Захист маршрутів](#protecting-routes)
- [Налаштування](#customization)
- [Події](#events)

<a name="introduction"></a>

## Вступ

Багато веб-додатків вимагають від користувачів перевірки своїх електронних адрес перед використанням додатків. Замість того, щоб змушувати вас повторно впроваджувати цей функціонал вручну для кожного створеного додатка, Laravel надає зручні вбудовані служби для надсилання та перевірки запитів на підтвердження електронної пошти.

> **Note**  
> Хочете швидко почати? Встановіть один із [стартових наборів](starter-kits.md) Laravel в новий додаток Laravel. Стартові набори подбають про створення всієї вашої системи автентифікації, включаючи підтримку перевірки електронної пошти.

<a name="model-preparation"></a>

### Підготовлення моделі

Перш ніж почати, переконайтеся, що ваша модель `App\Models\User` реалізує контракт `Illuminate\Contracts\Auth\MustVerifyEmail`:

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable implements MustVerifyEmail
{
    use Notifiable;

    // ...
}
```

Як тільки буде додано цей інтерфейс до вашої моделі, щойно зареєстрованим користувачам автоматично буде надіслано електронний лист із посиланням для підтвердження електронної пошти. Як ви можете побачити, провайдер `App\Providers\EventServiceProvider` вашого додатка Laravel, вже містить [слухача](events.md) `SendEmailVerificationNotification`, який додано до події `Illuminate\Auth\Events\Registered`. Цей слухач подій надішле користувачеві посилання для підтвердження електронної пошти.

Якщо ви вручну реалізуєте функціонал у своєму додатку замість використання [стартового набору](starter-kits.md), вам слід переконатися, що ви надсилаєте подію `Illuminate\Auth\Events\Registered` після успішної реєстрації користувача:

```php
use Illuminate\Auth\Events\Registered;

event(new Registered($user));
```

<a name="database-preparation"></a>

### Підготування бази данних

Далі ваша таблиця `users` має містити стовпець `email_verified_at` для зберігання дати й часу підтвердження електронної адреси користувача. За замовчуванням міграція таблиці користувачів, яка входить до складу Laravel, вже містить цей стовпець. Отже, все, що вам потрібно зробити, це запустити міграцію бази даних:

```shell
php artisan migrate
```

<a name="verification-routing"></a>

## Маршрутизація

Щоб належним чином запровадити перевірку електронної пошти, необхідно визначити три маршрути. По-перше, знадобиться маршрут для відображення повідомлення для користувача про те, що він повинен натиснути посилання для підтвердження електронної пошти в електронному листі для підтвердження, який Laravel надіслав йому після реєстрації.

По-друге, буде потрібен маршрут для орпацювання запитів, створених, коли користувач натискає посилання для підтвердження електронної пошти в електронному листі.

По-третє, знадобиться маршрут для повторного надсилання посилання підтвердження, якщо користувач випадково втратить перше посилання підтвердження.

<a name="the-email-verification-notice"></a>

### Повідомлення про підтвердження електронної пошти

Як згадувалося раніше, слід визначити маршрут, який повертатиме візуалізацію із вказівками для користувача натиснути посилання, для підтвердження електронної пошти, надіслане йому електронною поштою від Laravel після реєстрації. Цей шаблон відображатиметься для користувачів, коли вони намагатимуться отримати доступ до інших частин додатка без попередньої перевірки своєї електронної адреси. Пам’ятайте, що посилання автоматично надсилається електронною поштою користувачеві, якщо ваша модель `App\Models\User` реалізує інтерфейс `MustVerifyEmail`:

```php
Route::get('/email/verify', function () {
    return view('auth.verify-email');
})->middleware('auth')->name('verification.notice');
```

Маршрут, який повертає шаблон повідомлення про підтвердження електронної пошти, повинен мати назву `verification.notice`. Важливо, щоб маршруту було призначено саме це ім’я, оскільки посередник `verified`, [який входить до складу Laravel](#protecting-routes), автоматично перенаправлятиме на це ім’я маршрута, якщо користувач не підтвердив свою електронну адресу.

> **Note**  
> Під час ручного впровадження перевірки електронної пошти ви повинні самостійно визначити вміст повідомлення про перевірку. Якщо ви бажаєте готовий каркас, який включає всі необхідні шаблони автентифікації та перевірки, ознайомтеся зі [стартовими наборами додатка Laravel](starter-kits.md).

<a name="the-email-verification-handler"></a>

### Виконавець перевірки електронної пошти

Далі нам потрібно визначити маршрут, який буде опрацьовувати запити, створені, коли користувач натискає посилання для підтвердження електронної пошти, надіслане йому електронною поштою. Цей маршрут потрібно назвати `verification.verify` і йому призначити посередників `auth` та `signed`:

```php
use Illuminate\Foundation\Auth\EmailVerificationRequest;

Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {
    $request->fulfill();

    return redirect('/home');
})->middleware(['auth', 'signed'])->name('verification.verify');
```

Перш ніж рухатися далі, розглянемо докладніше цей маршрут. По-перше, ви помітите, що ми використовуємо тип запита `EmailVerificationRequest` замість типового екземпляра `Illuminate\Http\Request`. `EmailVerificationRequest` — це [запит форми](validation.md#form-request-validation), який включено до Laravel. Цей запит автоматично подбає про перевірку `id` запита та `hash` параметрів.

Далі ми можемо перейти безпосередньо до виклика метода `fulfill` в запиті. Цей метод викличе метод `markEmailAsVerified` для автентифікованого користувача та надішле подію `Illuminate\Auth\Events\Verified`. Метод `markEmailAsVerified` доступний для моделі `App\Models\User` за замовчуванням через базовий клас `Illuminate\Foundation\Auth\User`. Після підтвердження електронної адреси користувача ви можете перенаправити його, куди забажаєте.

<a name="resending-the-verification-email"></a>

### Повторне надсилання електронного листа для підтвердження

Іноді користувач може загубити або випадково видалити електронний лист для підтвердження електронної адреси. Для цього ви можете визначити маршрут, який дозволить користувачеві запитувати повторне надсилання електронного листа для підтвердження. Потім ви можете зробити запит на цей маршрут, розмістивши просту кнопку надсилання форми на [сторінці повідомлення про підтвердження](#the-email-verification-notice):

```php
use Illuminate\Http\Request;

Route::post('/email/verification-notification', function (Request $request) {
    $request->user()->sendEmailVerificationNotification();

    return back()->with('message', 'Verification link sent!');
})->middleware(['auth', 'throttle:6,1'])->name('verification.send');
```

<a name="protecting-routes"></a>

### Захист маршрутів

[Посередник маршрута](middleware.md) може використовуватися для надання доступу до певного маршрута лише перевіреним користувачам. Laravel поставляється з посередником `verified`, який посилається на клас `Illuminate\Auth\Middleware\EnsureEmailIsVerified`. Оскільки цей посередник вже зареєстровано в HTTP `kernel` вашого додатка, все, що вам потрібно зробити, це приєднати посередник до визначеного маршрута. Як правило, цей посередник поєднується з посередником `auth`:

```php
Route::get('/profile', function () {
    // Тільки користувачі з перевіреним имейлом можуть отримати доступ до цього маршруту...
})->middleware(['auth', 'verified']);
```

Якщо неперевірений користувач намагається отримати доступ до маршрута, якому було призначено цей посередник, він буде автоматично перенаправлений на [маршрут під назвою](routing.md#named-routes) `verification.notice`.

<a name="customization"></a>

## Налаштування

<a name="verification-email-customization"></a>

#### Налаштування підтвердження електронної пошти

Хоча повідомлення про підтвердження електронної пошти за замовчуванням повинно задовольняти вимоги більшості додатків, Laravel дозволяє вам змінити повідомлення підтвердження електронної пошти.

Щоб розпочати, передайте замикання методу `toMailUsing`, повідомлення `Illuminate\Auth\Notifications\VerifyEmail`. Замикання отримає екземпляр моделі, яка повідомляється, а також підписану URL-адресу підтвердження електронної пошти, яку користувач має відвідати, щоб підтвердити свою адресу електронної пошти. Замикання має повернути екземпляр `Illuminate\Notifications\Messages\MailMessage`. Як правило, ви повинні викликати метод `toMailUsing`із метода `boot`класа`App\Providers\AuthServiceProvider` вашого додатка:

```php
use Illuminate\Auth\Notifications\VerifyEmail;
use Illuminate\Notifications\Messages\MailMessage;

/**
 * Зареєструйте будь-які служби аутентифікації / авторизації.
 *
 * @return void
 */
public function boot()
{
    // ...

    VerifyEmail::toMailUsing(function ($notifiable, $url) {
        return (new MailMessage)
            ->subject('Verify Email Address')
            ->line('Click the button below to verify your email address.')
            ->action('Verify Email Address', $url);
    });
}
```

> **Note**  
> Щоб дізнатися більше про повідомлення електронною поштою, перегляньте [документацію щодо повідомлень електронною поштою](notifications.md#mail-notifications).

<a name="events"></a>

## Події

Під час [використання стартових наборів Laravel](starter-kits.md), він надсилає [події](events.md) протягом процесу перевірки електронної пошти. Якщо ви вручну бажаєте опрацьовати перевірку електронної пошти для свого додатка, то ви повинні запускати ці події після завершення перевірки. Ви можете приєднати слухачів до цих подій в `EventServiceProvider` вашого додатка:

```php
    /**
     * Карта слухачів подій додатка.
     *
     * @var array
     */
    protected $listen = [
        'Illuminate\Auth\Events\Verified' => [
            'App\Listeners\LogVerifiedUser',
        ],
    ];
```
