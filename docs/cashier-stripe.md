# Laravel Cashier (Stripe)

- [Вступ](#introduction)
- [Оновлення Cashier](#upgrading-cashier)
- [Встановлення](#installation)
  - [Міграція бази даних](#database-migrations)
- [Налаштування](#configuration)
  - [Модель оплати](#billable-model)
  - [API ключі](#api-keys)
  - [Налаштування валюти](#currency-configuration)
  - [Налаштування податку](#tax-configuration)
  - [Ведення журналу звітності фатальних помилок](#Logging)
  - [Використання власних моделей](#using-custom-models)
- [Клієнти](#customers)
  - [Отримання клієнтів](#retrieving-customers)
  - [Створення клієнтів](#creating-customers)
  - [Оновлення клієнтів](#updating-customers)
  - [Залишки](#balances)
  - [Податкові IDs](#tax-ids)
  - [Синхронізація даних клієнтів зі Stripe](#syncing-customer-data-with-stripe)
  - [Платіжний портал](#billing-portal)
- [Способи оплати](#payment-methods)
  - [Збереження способів оплати](#storing-payment-methods)
  - [Отримання способів оплати](#retrieving-payment-methods)
  - [Визначення того, чи має користувачспосіб оплати](#determining-if-a-user-has-a-payment-method)
  - [Оновлення способа оплати за замовчанням](#updating-the-default-payment-method)
  - [Додавання способа оплати](#adding-payment-methods)
  - [Видалення способа оплати](#deleting-payment-methods)
- [Підписки](#subscriptions)
  - [Створення підписок](#creating-subscriptions)
  - [Перевірка статусу підписки](#checking-subscription-status)
  - [Зміна цін](#changing-prices)
  - [Вартість підписаки](#subscription-quantity)
  - [Підписки з декількома продуктами](#subscriptions-with-multiple-products)
  - [Плата за лічильником](#metered-billing)
  - [Податки на підписку](#subscription-taxes)
  - [Прив’язана дата підписки](#subscription-anchor-date)
  - [Скасування підписок](#canceling-subscriptions)
  - [Відновлення підписок](#resuming-subscriptions)
- [Пробні версії підписки](#subscription-trials)
  - [З попередньою оплатою](#with-payment-method-up-front)
  - [Без попередньої оплати](#without-payment-method-up-front)
  - [Розширення випробувань](#extending-trials)
- [Опрацювання веб-гачків Stripe](#handling-stripe-webhooks)
  - [Визначення виконавців подій веб-гачків](#defining-webhook-event-handlers)
  - [Перевірка підписів веб-гачків](#verifying-webhook-signatures)
- [Одноразові платежі](#single-charges)
  - [Прості платежі](#simple-charge)
  - [Платежі за рахунком-фактурою](#charge-with-invoice)
  - [Створення платіжних намірів](#creating-payment-intents)
  - [Відшкодування платежів](#refunding-charges)
- [Перевірка](#checkout)
  - [Перевірка товару](#product-checkouts)
  - [Перевірка одноразовх платежів](#single-charge-checkouts)
  - [Перевірка підписників](#subscription-checkouts)
  - [Збір податкових IDs](#collecting-tax-ids)
  - [Гостьові перевірки](#guest-checkouts)
- [Рахунок-фактура](#invoices)
  - [Отримання рахунків](#retrieving-invoices)
  - [Майбутні рахунки](#upcoming-invoices)
  - [Попередній перегляд рахунків за підписку](#previewing-subscription-invoices)
  - [Створення рахунків у PDF-файлах](#generating-invoice-pdfs)
- [Опрацювання невдалих платежів](#handling-failed-payments)
- [Надійна автентифікація клієнта (SCA)](<#strong-customer-authentication-(SCA)>)
  - [Платежі, які вимагають додаткового підтвердження](#payments-requiring-additional-confirmation)
  - [Повідомлення про оплату поза сеансом](#off-session-payment-notifications)
- [Stripe SDK](#stripe-sdk)
- [Тестування](#testing)

<a name="introduction"></a>

## Вступ

[Laravel Cashier Stripe](https://github.com/laravel/cashier-stripe) надає виразний, гнучкий інтерфейс для абонентських оплачуваних послуг [Stripe](https://stripe.com/). Він опрацьовує майже весь стандартний платіжний код передплати, який вам страшно писати. На додаток до базового керування підписок, Cashier може керувати купонами, обмінювати підписки, «вартість» підписок, пільгові періоди скасування та навіть створювати PDF-файли рахунків.

<a name="upgrading-cashier"></a>

## Оновлення Cashier

Під час оновлення до нової версії Cashier важливо уважно переглянути [посібник з оновлення](https://github.com/laravel/cashier-stripe/blob/master/UPGRADE.md).

> **Warning**  
> Щоб запобігти помилковим змінам, Cashier використовує фіксовану версію API Stripe. Cashier 14 використовує Stripe API версії `2022-08-01`. Версія Stripe API буде оновлюватись у незначних випусках, щоб використовувати новий Stripe функціонал та вдосконалення.

<a name="installation"></a>

## Встановлення

Спочатку встановіть пакет Cashier для Stripe за допомогою менеджера пакетів Composer:

```shell
composer require laravel/cashier
```

> **Warning**  
> Щоб Cashier належним чином опрацьовував всі події Stripe, не забудьте [налаштувати опрацювання веб-гачка Cashier](#handling-stripe-webhooks).

<a name="database-migrations"></a>

### Міграція бази даних

Постачальник служб Cashier реєструє власний каталог міграції бази даних, тому не забудьте міграції після встановлення пакета. Міграція Cashier додасть декілька стовпців до вашої таблиці `users`, а також створить нову таблицю `підписок` для зберігання всіх підписок ваших клієнтів:

```shell
php artisan migrate
```

Якщо вам потрібно переписати міграції, які постачаються з Cashier, ви можете опублікувати їх за допомогою команди `vendor:publish Artisan`:

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

Якщо ви хочете повністю запобігти запуску міграцій Cashier, ви можете скористатися методом `ignoreMigrations`, наданим Cashier. Як правило, цей метод слід викликати в методі `register` вашого `AppServiceProvider`:

```php
use Laravel\Cashier\Cashier;

/**
 * Зареєструйте будь-які сервіси додатка.
 *
 * @return void
 */
public function register()
{
    Cashier::ignoreMigrations();
}
```

> **Warning**  
> Stripe рекомендує, щоб будь-який стовпець, який використовується для зберігання ідентифікаторів Stripe, були регістро-залежними. Таким чином, ви повинні переконатися, що сортування стовпців для стовпця `stripe_id` встановлено на `utf8_bin` під час використання MySQL. Більше інформації про це можна знайти в [документації Stripe](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible).

<a name="configuration"></a>

## Налаштування

<a name="billable-model"></a>

### Модель оплати

Перш ніж використовувати Cashier, додайте трейт `Billable` до визначення вашої моделі оплати. Як правило, це буде модель `App\Models\User`. Цей трейт надає різні методи, за допомогою яких можна виконувати звичайні завдання оплати, наприклад створювати підписку, застосовувати купони та оновлювати інформацію про метод оплати:

```php
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Cashier припускає, що вашою моделлю оплати буде клас `App\Models\User`, який присутній в Laravel. Якщо ви бажаєте це змінити, ви можете вказати іншу модель за допомогою метода `useCustomerModel`. Зазвичай цей метод слід викликати в методі `boot` вашого класа `AppServiceProvider`:

```php
use App\Models\Cashier\User;
use Laravel\Cashier\Cashier;

/**
 * Завантажте будь-які служби додатка.
 *
 * @return void
 */
public function boot()
{
    Cashier::useCustomerModel(User::class);
}
```

> **Warning**  
> Якщо ви використовуєте модель, яка відрізняється від наданої моделі Laravel `App\Models\User`, вам потрібно буде опублікувати та змінити надані [міграції Cashier](#installation), щоб відповідати назві таблиці вашої альтернативної моделі.

<a name="api-keys"></a>

### API Ключі

Далі вам слід налаштувати ключі API Stripe у файлі `.env` вашого додатка. Ви можете отримати свої ключі Stripe API з панелі керування Stripe:

```ini
STRIPE_KEY=your-stripe-key
STRIPE_SECRET=your-stripe-secret
STRIPE_WEBHOOK_SECRET=your-stripe-webhook-secret
```

Ви повинні переконатися, що змінну середовища `STRIPE_WEBHOOK_SECRET` визначено у файлі `.env` вашого додатка, оскільки ця змінна використовується, щоб переконатися, що вхідні веб-гачки насправді надходять від Stripe.

> **Warning**  
> Ви повинні переконатися, що змінну середовища `STRIPE_WEBHOOK_SECRET` визначено у файлі `.env` вашого додатка, оскільки ця змінна використовується, щоб переконатися, що вхідні веб-гачки насправді надходять від Stripe.

<a name="currency-configuration"></a>

### Налаштування валюти

Валюта Cashier за замовчуванням – долари США (USD). Ви можете змінити валюту за замовчуванням, перевизначивши змінну середовища `CASHIER_CURRENCY` у файлі `.env` вашого додатка:

```ini
CASHIER_CURRENCY=EUR
```

Окрім налаштування валюти Cashier, ви також можете вказати мову, яка використовуватиметься під час форматування грошових значень для відображення в рахунках. Внутрішньо Cashier використовує [PHP-клас `NumberFormatter`](https://www.php.net/manual/en/class.numberformatter.php) для встановлення локалі валюти:

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> **Warning**  
> Щоб використовувати локалі, які відрізняються від `en`, переконайтеся, що на вашому сервері встановлено та налаштовано розширення PHP `ext-intl`.

<a name="tax-configuration"></a>

### Налаштування податку

Завдяки [податку Stripe](https://stripe.com/tax) можна автоматично розраховувати податки для всіх рахунків, створених Stripe. Ви можете ввімкнути автоматичний розрахунок податку, викликавши метод `calculateTaxes` у методі `boot` класу `App\Providers\AppServiceProvider` вашого додатка:

```php
use Laravel\Cashier\Cashier;

/**
 * Завантажте будь-які служби додатка.
 *
 * @return void
 */
public function boot()
{
    Cashier::calculateTaxes();
}
```

Після ввімкнення розрахунку податків всі нові підписки та створені одноразові рахунки отримають автоматичний розрахунок податку.

Щоб цей функціонал працювала належним чином, платіжні дані вашого клієнта, такі як ім’я клієнта, адреса та податковий номер, мають бути синхронізовані зі Stripe. Для цього ви можете використовувати методи [синхронізації даних клієнта](#syncing-customer-data-with-stripe) та [ID платника податків](#tax-ids), які запропоновані Cashier.

> **Warning**  
> На жаль, від тепер, податок не розраховується для [одноразових зборів](#single-charges) або [перевірки одноразовх зборів](#single-charge-checkouts). Крім того, під час бета-тестування податок Stripe доступний лише за запрошеннями. Ви можете подати запит на доступ до оподаткування Stripe через [веб-сайт Stripe Tax](https://stripe.com/tax#request-access).

<a name="logging"></a>

### Ведення журналу звітності фатальних помилок

Cashier дозволяє вказати "log" канал, який використовуватиметься під час реєстрації фатальних помилок Stripe. Ви можете вказати "log" канал, визначивши змінну середовища `CASHIER_LOGGER` у файлі `.env` вашого додатка:

```ini
CASHIER_LOGGER=stack
```

Винятки, які генеруються викликами API до Stripe, реєструватимуться через стандартний "log" канал вашого додатка.

<a name="#using-custom-models"></a>

### Використання власних моделей

Ви можете розширювати моделі, які будуть використовуватися всередині Cashier, шляхом визначення вашої власної моделі та розширення відповідної моделі Cashier:

```php
use Laravel\Cashier\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

Після визначення вашої моделі ви можете повідомити Cashier про те, що потрібно використовувати вашу спеціальну модель через клас `Laravel\Cashier\Cashier`. Як правило, ви повинні повідомити Cashier про власні моделі в методі `boot` класа `App\Providers\AppServiceProvider` вашого додатка:

```php
use App\Models\Cashier\Subscription;
use App\Models\Cashier\SubscriptionItem;

/**
 * Завантажте будь-які служби додатка.
 *
 * @return void
 */
public function boot()
{
    Cashier::useSubscriptionModel(Subscription::class);
    Cashier::useSubscriptionItemModel(SubscriptionItem::class);
}
```

<a name="сustomers"></a>

## Клієнти

<a name="retrieving-customers"></a>

### Отримання клієнтів

Ви можете отримати клієнта за його Stripe ID за допомогою метода `Cashier::findBilable`. Цей метод повертає екземпляр оплачуваної моделі:

```php
use Laravel\Cashier\Cashier;

$user = Cashier::findBillable($stripeId);
```

<a name="creating-customers"></a>

### Створення клієнтів

Іноді ви можете захотіти створити клієнта Stripe, без попередньої підписки. Це можна зробити за допомогою метода `createAsStripeCustomer`:

```php
$stripeCustomer = $user->createAsStripeCustomer();
```

Після створення клієнта в Stripe ви можете почати підписку пізніше. Ви можете надати додатковий масив `$options` для передачі будь-яких додаткових [параметрів створення клієнта, які підтримуються Stripe API](https://stripe.com/docs/api/customers/create):

```php
$stripeCustomer = $user->createAsStripeCustomer($options);
```

Ви можете використовувати метод `asStripeCustomer`, якщо хочете повернути об’єкт клієнта Stripe для оплачуваної моделі:

```php
$stripeCustomer = $user->asStripeCustomer();
```

Метод `createOrGetStripeCustomer` можна використовувати, якщо ви бажаєте отримати об’єкт клієнта Stripe для даної оплачуваної моделі, але не впевнені, що оплачувана модель вже є клієнтом у Stripe. Цей метод створить нового клієнта в Stripe, якщо він ще не існує:

```php
$stripeCustomer = $user->createOrGetStripeCustomer();
```

<a name="updating-customers"></a>

### Оновлення клієнтів

Час від часу ви можете надати додаткову інформацію безпосередньо клієнту Stripe. Це можна зробити за допомогою метода `updateStripeCustomer`. Цей метод приймає набір параметрів [оновлення клієнта, які підтримуються Stripe API](https://stripe.com/docs/api/customers/update):

```php
$stripeCustomer = $user->updateStripeCustomer($options);
```

<a name="#balances"></a>

### Залишки

Stripe дозволяє кредитувати або дебетувати «залишок» клієнта. Пізніше цей залишок буде зараховано або списано за новими рахунками. Щоб перевірити загальний залишок клієнта, ви можете скористатися методом `balance`, який доступний у вашій платіжній моделі. Метод `balance` поверне відформатований рядок залишоку у валюті клієнта:

```php
$balance = $user->balance();
```

Щоб кредитувати залишки клієнта, ви можете надати значення методу `creditBalance`. За бажанням, ви також можете надати опис:

```php
$user->creditBalance(500, 'Premium customer top-up.');
```

Надання значення методу `debitBalance` призведе до дебетування балансу клієнта:

```php
$user->debitBalance(300, 'Bad usage penalty.');
```

Метод `applyBalance` створить нові транзакції залишка для клієнта. Ви можете отримати ці записи транзакцій за допомогою метода `balanceTransactions`, який може бути корисним для надання "log" кредитів і дебетів для перегляду клієнтом:

```php
// Отримати всі транзакції...
$transactions = $user->balanceTransactions();

foreach ($transactions as $transaction) {
    // Сума транзакції...
    $amount = $transaction->amount(); // $2.31

    // Отримати відповідний рахунок, коли він доступний...
    $invoice = $transaction->invoice();
}
```

<a name="tax-ids"></a>

#### Податкові IDs

Cashier пропонує простий спосіб керувати податковими IDs клієнтів. Наприклад, метод `taxIds` можна використовувати для отримання всіх [податкових IDs](https://stripe.com/docs/api/customer_tax_ids/object), які призначені клієнту як колекція:

```php
$taxIds = $user->taxIds();
```

Ви також можете отримати конкретний ID номер платника податків для клієнта за його ідентифікатором:

```php
$taxId = $user->findTaxId('txi_belgium');
```

Ви можете створити новий податковий ID, надавши дійсний [тип](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type) і значення методу `createTaxId`:

```php
$taxId = $user->createTaxId('eu_vat', 'BE0123456789');
```

Метод `createTaxId` негайно додасть ПДВ ID номер до облікового запису клієнта. [Перевірка ідентифікаційних номерів ПДВ також виконується Stripe](https://stripe.com/docs/invoicing/customer/tax-ids#validation); однак це асинхронний процес. Ви можете отримувати повідомлення про оновлення верифікації, підписавшись на подію веб-гачків `customer.tax_id.updated` і перевіривши [параметр перевірки ПДВ IDs](#handling-stripe-webhooks). Щоб дізнатися більше про опрацювання веб-гачків, зверніться до [документації щодо визначення виконавців веб-гачків](#handling-stripe-webhooks).

Ви можете видалити податковий номер за допомогою методу `deleteTaxId`:

```php
$user->deleteTaxId('txi_belgium');
```

<a name="syncing-customer-data-with-stripe"></a>

### Синхронізація даних клієнтів зі Stripe

Як правило, коли користувачі вашого додатка оновлюють своє ім’я, адресу електронної пошти чи іншу інформацію, яка також зберігається в Stripe, ви повинні повідомити Stripe про оновлення. Завдяки цьому, копія інформації Stripe буде синхронізована з копією вашого додатка.

Щоб автоматизувати це, ви можете визначити слухача подій у вашій оплачуваній моделі, яка реагує на подію `updated` моделі. Потім у вашому слухачі подій ви можете викликати метод `syncStripeCustomerDetails` у моделі:

```php
use function Illuminate\Events\queueable;

/**
 * "booted" метод моделі.
 *
 * @return void
 */
protected static function booted()
{
    static::updated(queueable(function ($customer) {
        if ($customer->hasStripeId()) {
            $customer->syncStripeCustomerDetails();
        }
    }));
}
```

Тепер щоразу, коли ваша модель клієнта оновлюється, інформація про неї синхронізуватиметься зі Stripe. Для зручності Cashier автоматично синхронізує інформацію вашого клієнта зі Stripe під час початкового створення клієнта.

Ви можете налаштувати стовпці, які використовуються для синхронізації інформації про клієнтів у Stripe, замінивши різні методи, надані Cashier. Наприклад, ви можете змінити метод `stripeName`, щоб налаштувати атрибут, який слід вважати «ім’ям» клієнта, коли Cashier синхронізує інформацію клієнта зі Stripe:

```php
/**
 * Отримайте ім’я клієнта, яке потрібно синхронізувати зі Stripe.
 *
 * @return string|null
 */
public function stripeName()
{
    return $this->company_name;
}
```

Так само ви можете замінити методи `stripeEmail`, `stripePhone`, `stripeAddress` і `stripePreferredLocales`. Ці методи синхронізуватимуть інформацію з відповідними параметрами клієнта під час [оновлення об’єкта клієнта Stripe](https://stripe.com/docs/api/customers/update). Якщо ви хочете отримати повний контроль над процесом синхронізації інформації про клієнта, ви можете перевизначити метод `syncStripeCustomerDetails`.

<a name="billing-portal"></a>

### Платіжний портал

Stripe пропонує [простий спосіб налаштувати платіжний портал](https://stripe.com/docs/customer-management), щоб ваш клієнт міг керувати своєю підпискою, способами оплати та переглядати свою історію платежів. Ви можете перенаправляти своїх користувачів на платіжний портал, викликавши метод `redirectToBillingPortal` платіжної моделі з контролера або маршруту:

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal();
});
```

За замовчуванням, коли користувач завершить керування своєю підпискою, він зможе повернутися до маршруту `home` вашого додатка за посиланням на платіжному порталі Stripe. Ви можете надати спеціальну URL-адресу, до якої користувач має повернутися, передавши URL-адресу як аргумент методу `redirectToBillingPortal`:

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('billing'));
});
```

Якщо ви хочете створити URL-адресу платіжного порталу без генерації відповіді перенаправлення HTTP, ви можете викликати метод `billingPortalUrl`:

```php
$url = $request->user()->billingPortalUrl(route('billing'));
```

<a name="payment-methods"></a>

## Cпособи оплати

<a name="storing-payment-methods"></a>

### Збереження способів оплати

Щоб створювати підписки або виконувати "одноразові" платежі за допомогою Stripe, вам потрібно буде зберегти спосіб оплати та отримати його ID із Stripe. Підхід, який використовується для цього, відрізняється залежно від того, який спосіб оплати ви плануєте використовувати, оплату підписки чи одноразову оплату, тому нижче ми розглянемо обидва.

<a name="payment-methods-for-subscriptions"></a>

#### Способи оплати для підписок

Зберігаючи інформацію про кредитну картку клієнта для подальшого використання в рамках підписки, слід використовувати API Stripe «Setup Intents» для безпечного збору даних про спосіб оплати клієнта. «Setup Intent» вказує Stripe на намір стягнути плату за допомогою спосіба оплати клієнта. Трейт `Billable` Cashier включає метод `createSetupIntent` для легкого створення нового «Setup Intents». Ви повинні викликати цей метод із маршруту або контролера, який відображатиме форму, яка збирає дані про спосіб оплати вашого клієнта:

```php
return view('update-payment-method', [
    'intent' => $user->createSetupIntent()
]);
```

Після того, як ви створили «Setup Intents» та передали його до візуалізації, ви повинні приєднати його секретний рядок до елемента, який збиратиме спосіб оплати. Наприклад, розглянемо цю форму «оновити спосіб оплати»:

```html
<input id="card-holder-name" type="text" />

<!-- Stripe Elements Placeholder -->
<div id="card-element"></div>

<button id="card-button" data-secret="{{ $intent->client_secret }}">
  Update Payment Method
</button>
```

Далі можна використовувати бібліотеку Stripe.js для приєднання [елемента Stripe](https://stripe.com/docs/payments/elements) до форми та безпечного збору платіжних даних клієнта:

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
  const stripe = Stripe("stripe-public-key");

  const elements = stripe.elements();
  const cardElement = elements.create("card");

  cardElement.mount("#card-element");
</script>
```

Далі можна перевірити картку та отримати безпечний «ідентифікатор спосіба оплати» зі Stripe за допомогою [метода `confirmCardSetup` Stripe](https://stripe.com/docs/js/setup_intents/confirm_card_setup):

```js
const cardHolderName = document.getElementById("card-holder-name");
const cardButton = document.getElementById("card-button");
const clientSecret = cardButton.dataset.secret;

cardButton.addEventListener("click", async (e) => {
  const { setupIntent, error } = await stripe.confirmCardSetup(clientSecret, {
    payment_method: {
      card: cardElement,
      billing_details: { name: cardHolderName.value },
    },
  });

  if (error) {
    // Показати "error.message" для користувача...
  } else {
    // Картка успішно перевірена...
  }
});
```

Після перевірки картки Stripe ви можете передати отриманий ідентифікатор `setupIntent.payment_method` в свій додаток Laravel, де його можна прикріпити до клієнта. Спосіб оплати можна [додати як новий спосіб оплати](#adding-payment-methods) або [використати для оновлення способа оплати за замовчуванням](#updating-the-default-payment-method). Ви також можете негайно використати ідентифікатор способу оплати, щоб [створити нову підписку](#creating-subscriptions).

> **Note**  
> Якщо вам потрібна додаткова інформація про «Setup Intents» та збір платіжних даних клієнтів, будь-ласка [перегляньте цей огляд, наданий Stripe](https://stripe.com/docs/payments/save-and-reuse#php).

<a name="payment-methods-for-single-charges"></a>

#### Способи оплати для одноразових платежів

Звичайно, під час одноразового стягнення коштів із способу оплати клієнта, нам знадобиться ідентифікатор способа оплати лише один раз. Через обмеження Stripe ви не можете використовувати збережений спосіб оплати клієнта за замовчуванням для одноразових платежів. Ви повинні дозволити клієнту ввести дані свого способу оплати за допомогою бібліотеки Stripe.js. Наприклад, розглянемо наступну форму:

```html
<input id="card-holder-name" type="text" />

<!-- Заповнювач елементів Stripe -->
<div id="card-element"></div>

<button id="card-button">Process Payment</button>
```

Після визначення такої форми бібліотеку Stripe.js можна використовувати для приєднання [елемента Stripe](https://stripe.com/docs/payments/elements) до форми та безпечного збору платіжних даних клієнта:

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
  const stripe = Stripe("stripe-public-key");

  const elements = stripe.elements();
  const cardElement = elements.create("card");

  cardElement.mount("#card-element");
</script>
```

Далі можна перевірити картку та отримати безпечний «ідентифікатор способа оплати» зі Stripe, за допомогою [метода `createPaymentMethod` Stripe](https://stripe.com/docs/js/payment_methods/create_payment_method):

```js
const cardHolderName = document.getElementById("card-holder-name");
const cardButton = document.getElementById("card-button");

cardButton.addEventListener("click", async (e) => {
  const { paymentMethod, error } = await stripe.createPaymentMethod(
    "card",
    cardElement,
    {
      billing_details: { name: cardHolderName.value },
    }
  );

  if (error) {
    // Показати "error.message" для користувача...
  } else {
    // Картка успішно перевірена...
  }
});
```

Якщо картку успішно підтверджено, ви можете передати `paymentMethod.id` у свій додаток Laravel та опрацювати [одноразову оплату](#simple-charge).

<a name="retrieving-payment-methods"></a>

### Отримання способів оплати

Метод `paymentMethods` екземпляра оплачуваної моделі повертає колекцію екземплярів `Laravel\Cashier\PaymentMethod`:

```php
$paymentMethods = $user->paymentMethods();
```

За замовчуванням цей метод повертає способи оплати типу `card`. Щоб отримати способи оплати іншого `type`, ви можете передати методу `type` аргументом:

```php
$paymentMethods = $user->paymentMethods('sepa_debit');
```

Щоб отримати спосіб оплати клієнта за замовчуванням, можна використати метод `defaultPaymentMethod`:

```php
$paymentMethod = $user->defaultPaymentMethod();
```

Ви можете отримати певний спосіб оплати, який додається до платіжної моделі за допомогою метода `findPaymentMethod`:

```php
$paymentMethod = $user->findPaymentMethod($paymentMethodId);
```

<a name="determining-if-a-user-has-a-payment-method"></a>

### Визначення того, чи має користувач спосіб оплати

Щоб визначити, чи платіжна модель має спосіб оплати за замовчуванням, приєднаний до їхнього облікового запису, викличте метод `hasDefaultPaymentMethod`:

```php
if ($user->hasDefaultPaymentMethod()) {
    //
}
```

Ви можете використовувати метод `hasPaymentMethod`, щоб визначити, чи платіжна модель має принаймні один спосіб оплати, приєднаний до їхнього облікового запису:

```php
if ($user->hasPaymentMethod()) {
    //
}
```

Цей метод визначає, чи платіжна модель має способи оплати типу `card`. Щоб визначити, чи існує спосіб оплати іншого типу для моделі, ви можете передати методу `type` аргументом:

```php
if ($user->hasPaymentMethod('sepa_debit')) {
    //
}
```

<a name="updating-the-default-payment-method"></a>

### Оновлення способа оплати за замовчуванням

Метод `updateDefaultPaymentMethod` можна використовувати для оновлення інформації про спосіб оплати клієнта за замовчуванням. Цей метод приймає ідентифікатор способа оплати Stripe, який буде призначино як новий спосіб оплати за замовчуванням:

```php
$user->updateDefaultPaymentMethod($paymentMethod);
```

Щоб синхронізувати інформацію про спосіб оплати за замовчуванням з інформацією про спосіб оплати за замовчуванням клієнта в Stripe, ви можете використати метод `updateDefaultPaymentMethodFromStripe`:

```php
$user->updateDefaultPaymentMethodFromStripe();
```

> **Warning**  
> Спосіб оплати клієнта за замовчуванням можна використовувати лише для виставлення рахунків і створення нових підписок. Через обмеження, які накладає Stripe, його не можна використовувати для одноразової оплати.

<a name="adding-payment-methods"></a>

### Додавання способів оплати

Щоб додати новий спосіб оплати, ви можете викликати метод `addPaymentMethod` у платіжній моделі, передаючи ідентифікатор способа оплати:

```php
$user->addPaymentMethod($paymentMethod);
```

> **Note**  
> Щоб дізнатися, як отримати ідентифікатори способа оплати, перегляньте [документацію щодо збереження способа оплати](storing-payment-methods).

<a name="deleting-payment-methods"></a>

### Видалення способів оплати

Щоб видалити спосіб оплати, ви можете викликати метод видалення в екземплярі `Laravel\Cashier\PaymentMethod`, який ви хочете видалити:

```php
$paymentMethod->delete();
```

Метод `deletePaymentMethod` видаляє певний спосіб оплати з платіжної моделі:

```php
$user->deletePaymentMethod('pm_visa');
```

Метод `deletePaymentMethods` видаляє всю інформацію про спосіб оплати для платіжної моделі:

```php
$user->deletePaymentMethods();
```

За замовчуванням цей метод видаляє способи оплати типу `card`. Щоб видалити способи оплати іншого типу, ви можете передати методу `type` аргументом:

```php
$user->deletePaymentMethods('sepa_debit');
```

> **Note**  
> Якщо користувач має активну підписку, ваш додаток не повинен дозволяти йому видаляти спосіб оплати за замовчуванням.

<a name="subscriptions"></a>

## Підписки

Підписки дають можливість налаштувати регулярні платежі для ваших клієнтів. Підписки на Stripe, якими керує Cashier, забезпечують підтримку різних цін на підписку, кількості підписок, пробних версій тощо.

<a name="creating-subscriptions"></a>

### Створення підписок

Щоб створити підписку, спочатку отримайте екземпляр вашої платної моделі, яка зазвичай буде екземпляром `App\Models\User`. Отримавши екземпляр моделі, ви можете використовувати метод `newSubscription` для створення підписки на модель:

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription(
        'default', 'price_monthly'
    )->create($request->paymentMethodId);

    // ...
});
```

Перший аргумент, переданий методу `newSubscription`, має бути внутрішнім ім’ям підписки. Якщо ваш додаток пропонує лише одну підписку, ви можете назвати її `default` або `primary`. Ця назва підписки призначена лише для внутрішнього використання додатком та не призначена для показу користувачам. Крім того, вона не повинна містити пробілів і її ніколи не можна змінювати після створення підписки. Другим аргументом має бути конкретна ціна, на яку користувач підписується. Це значення має відповідати ідентифікатору ціни в Stripe.

Метод `create`, який приймає [ідентифікатор способу оплати Stripe](#payment-methods) або об’єкт Stripe `PaymentMethod`, почне підписку, а також оновить вашу базу даних за допомогою ID клієнта Stripe платіжної моделі та іншою відповідною платіжною інформацією.

> **Warning**  
> Передача ідентифікатора способа оплати безпосередньо до метода створення підписки також автоматично додасть його до збережених способів оплати користувача.

<a name="collecting-recurring-payments-via-invoice-emails"></a>

#### Збір регулярних платежів через електронні листи з рахунками

Замість того, щоб автоматично стягувати регулярні платежі клієнта, ви можете доручити Stripe надсилати клієнту рахунок електронною поштою щоразу, коли настає термін платежу. Потім клієнт може вручну оплатити рахунок-фактуру, щойно він його отримає. Клієнту не потрібно вказувати спосіб оплати наперед під час отримання регулярних платежів через рахунки:

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice();
```

Час, протягом якого клієнт має сплатити рахунок-фактуру до скасування підписки, визначається параметром `days_until_due`. За замовчуванням це 30 днів; однак ви можете вказати конкретне значення для цього параметра, якщо хочете:

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
    'days_until_due' => 30
]);
```

<a name="quantities"></a>

#### Вартість підписки

Якщо ви хочете встановити конкретну [вартість]() для ціни під час створення підписки, вам слід викликати метод `quantity` в конструкторі підписки перед створенням підписки:

```php
$user->newSubscription('default', 'price_monthly')
     ->quantity(5)
     ->create($paymentMethod);
```

<a name="additional-Details"></a>

#### Додаткова інформація

Якщо ви хочете вказати додаткові параметри клієнта або підписки, які підтримує Stripe, ви можете зробити це, передавши їх як другий і третій аргументи методу `create`:

```php
$user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
    'email' => $email,
], [
    'metadata' => ['note' => 'Some extra information.'],
]);
```

<a name="coupons"></a>

#### Купони

Якщо ви хочете застосувати купон під час створення підписки, ви можете скористатися методом `withCoupon`:

```php
$user->newSubscription('default', 'price_monthly')
     ->withCoupon('code')
     ->create($paymentMethod);
```

Або, якщо ви хочете застосувати [промо-код Stripe](https://stripe.com/docs/billing/subscriptions/coupons), ви можете скористатися методом `withPromotionCode`:

```php
$user->newSubscription('default', 'price_monthly')
     ->withPromotionCode('promo_code_id')
     ->create($paymentMethod);
```

Наданий ID промо-код має бути Stripe API ID, призначеним промо-коду, а не клієнтського промо-коду, який призначений клієнту. Якщо вам потрібно знайти ID промо-код на основі клієнтського коду, ви можете скористатися методом `findPromotionCode`:

```php
// Знайдіть ID промо-код за власним кодом клієнта...
$promotionCode = $user->findPromotionCode('SUMMERSALE');

// Знайдіть активний ID промо-код за власним кодом клієнта...
$promotionCode = $user->findActivePromotionCode('SUMMERSALE');
```

У наведеному вище прикладі повернутий об’єкт `$promotionCode` є екземпляром `Laravel\Cashier\PromotionCode`. Цей клас розширюють базовий об’єкт `Stripe\PromotionCode`. Ви можете отримати купон, пов’язаний із промо-кодому, викликавши метод `coupon`:

```php
$coupon = $user->findPromotionCode('SUMMERSALE')->coupon();
```

Екземпляр купона дозволяє визначити суму знижки а також, чи є купон фіксованою знижкою чи знижкою на основі відсотка:

```php
if ($coupon->isPercentage()) {
    return $coupon->percentOff().'%'; // 21.5%
} else {
    return $coupon->amountOff(); // $5.99
}
```

Ви також можете отримати знижки, які зараз застосовуються до клієнта або підписки:

```php
$discount = $billable->discount();

$discount = $subscription->discount();
```

Повернені екземпляри `Laravel\Cashier\Discount` розширюють базовий екземпляр об’єкта `Stripe\Discount`. Ви можете отримати купон, пов’язаний з цією знижкою, викликавши метод `coupon`:

```php
$coupon = $subscription->discount()->coupon();
```

Якщо ви хочете застосувати новий купон або промо-код до клієнта або підписки, ви можете зробити це за допомогою методів `applyCoupon` або `applyPromotionCode`:

```php
$billable->applyCoupon('coupon_id');
$billable->applyPromotionCode('promotion_code_id');

$subscription->applyCoupon('coupon_id');
$subscription->applyPromotionCode('promotion_code_id');
```

Пам’ятайте, що вам слід використовувати Stripe API ID, призначений промо-коду, а не клієнтський промо-код. Одночасно до клієнта або підписки можна застосувати лише один купон або промо-код.

Для отримання додаткової інформації на цю тему зверніться до документації Stripe щодо [купонів](https://stripe.com/docs/billing/subscriptions/coupons) і [промо-кодів](https://stripe.com/docs/billing/subscriptions/coupons).

<a name="adding-subscriptions"></a>

#### Додавання підписок

Якщо ви хочете додати підписку для клієнта, який вже має спосіб оплати за замовчуванням, ви можете викликати метод `add` в конструкторі підписок:

```php
use App\Models\User;

$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->add();
```

<a name="creating-subscriptions-from-the-stripe-dashboard"></a>

#### Створення підписок із інформаційної панелі Stripe

Ви також можете створювати підписки на інформаційній панелі Stripe. При цьому Cashier синхронізує щойно додані підписки та призначить їм назву `default`. Щоб налаштувати назву підписки, яка призначається підпискам, створеним на інформаційній панелі, [розширте `WebhookController`](#defining-webhook-event-handlers) і перезапишіть метод `newSubscriptionName`.

Крім того, ви можете створити лише один тип підписки через інформаційну панель Stripe. Якщо ваш додаток пропонує декілька підписок з різними іменами, лише один тип підписки можна додати через інформаційну панель Stripe.

Нарешті, ви завжди повинні бути впевнені в тому, що додаєте лише одну активну підписку на тип підписки, яку пропонує ваш додаток. Якщо клієнт має дві підписки `default`, Cashier використовуватиме лише останню додану підписку, навіть якщо обидві будуть синхронізовані з базою даних вашого дадатка.

<a name="checking-subscription-status"></a>

### Перевірка статусу підписки

Після того як клієнт підписався на ваш додаток, ви можете легко перевірити його статус підписки за допомогою різноманітних зручних методів. По-перше, метод `subscribed` повертає `true`, якщо у клієнта є активна підписка, навіть якщо підписка наразі діє в межах пробного періоду. Метод `subscribed` приймає назву підписки першим аргументом:

```php
if ($user->subscribed('default')) {
    //
}
```

Метод `subscribed` також є чудовим кандидатом для [route middleware](/docs/{{version}}/middleware), дозволяючи вам фільтрувати доступ до маршрутів і контролерів на основі статусу підписки користувача:

```php
<?php

namespace App\Http\Middleware;

use Closure;

class EnsureUserIsSubscribed
{
    /**
     * Опрацьовувати вхідний запит.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if ($request->user() && !$request->user()->subscribed('default')) {
            // Цей користувач не платоспроможний клієнт...
            return redirect('billing');
        }

        return $next($request);
    }
}
```

Якщо ви бажаєте визначити, чи триває пробний період користувача, ви можете скористатися методом `onTrial`. Цей метод може бути корисним для визначення того, чи слід відображати попередження для користувача про те, що він все ще на пробному періоді:

```php
if ($user->subscription('default')->onTrial()) {
    //
}
```

Метод `subscribedToProduct` можна використовувати, щоб визначити, чи підписаний користувач на даний продукт на основі ідентифікатора даного продукту Stripe. У Stripe продукти є колекціями цін. У цьому прикладі ми визначимо, чи підписка користувача `default` активно підписується на «преміальний» продукт додатка. Даний ідентифікатор продукту Stripe має відповідати одному з ідентифікаторів вашого продукту на інформаційній панелі Stripe:

```php
if ($user->subscribedToProduct('prod_premium', 'default')) {
    //
}
```

Передаючи масив методу `subscribedToPlan`, ви можете визначити, чи підписка користувача `default` активно підписана на «базовий» або «преміальний» продукт додатка:

```php
if ($user->subscribedToProduct(['prod_basic', 'prod_premium'], 'default')) {
    //
}
```

Метод `subscribedToPrice` можна використовувати, щоб визначити, чи відповідає підписка клієнта заданому ідентифікатору ціни:

```php
if ($user->subscribedToPrice('price_basic_monthly', 'default')) {
    //
}
```

Mетод `recurring` може бути використаний, щоб визначити, чи користувач наразі підписаний і чи більше не діє його пробний період:

```php
if ($user->subscription('default')->recurring()) {
    //
}
```

> **Warning**  
> Якщо користувач має дві підписки з однаковою назвою, метод `subscription` завжди повертатиме останню підписку. Наприклад, користувач може мати два записи про підписку з іменем `default`; однак одна з підписок може бути старою підпискою, термін дії якої закінчився, тоді як інша є поточною, активною підпискою. Найновіша підписка завжди повертатиметься, тоді як старіші підписки зберігаються в базі даних для перегляду історії.

<a name="cancelled-subscription-status"></a>

#### Статус скасованої підписки

Щоб визначити, чи був користувач колись активним підписником, але скасував свою підписку, ви можете скористатися методом `cancelled`:

```php
if ($user->subscription('default')->canceled()) {
    //
}
```

Ви також можете визначити, чи користувач скасував свою підписку, але все ще діє «пільговий період» до повного закінчення терміну дії підписки. Наприклад, якщо користувач скасовує підписку 5 березня, термін дії якої спочатку було заплановано на 10 березня, у користувача діє «пільговий період» до 10 березня. Зауважте, що метод `subscribed` все ще повертатиме `true` протягом цього часу:

```php
if ($user->subscription('default')->onGracePeriod()) {
    //
}
```

Щоб визначити, чи користувач скасував свою підписку та більше не перебуває в межах свого «пільгового періоду», ви можете використати метод `ended`:

```php
if ($user->subscription('default')->ended()) {
    //
}
```

<a name="incomplete-and-past-due-status"></a>

#### Неповний і прострочений статус

Якщо підписка потребує вторинної оплати після створення, підписку буде позначено як `неповну`. Статуси підписки зберігаються в стовпці `stripe_status` таблиці бази даних `subscriptions` Cashier.

Подібним чином, якщо під час заміни цін необхідна додаткова платіжна дія, підписка буде позначена як `past_due`. Якщо ваша підписка знаходиться в одному з цих станів, вона не буде активною, доки клієнт не підтвердить свій платіж. Визначити, чи має підписка неповний платіж, можна здійснити за допомогою метода `hasIncompletePayment` платіжної моделі або екземпляра підписки:

```php
if ($user->hasIncompletePayment('default')) {
    //
}

if ($user->subscription('default')->hasIncompletePayment()) {
    //
}
```

Якщо підписка має неповний платіж, вам слід спрямувати користувача на сторінку підтвердження платежу Cashier, передавши ідентифікатор `latestPayment`. Ви можете використати метод `latestPayment`, який доступний в екземплярі підписки, щоб отримати цей ідентифікатор:

```html
<a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
  Please confirm your payment.
</a>
```

Якщо ви бажаєте, щоб підписка все ще вважалася активною, коли вона перебуває в стані `past_due`, ви можете скористатися методом `keepPastDueSubscriptionsActive`, наданим Cashier. Як правило, цей метод слід викликати в методі `register` вашого `App\Providers\AppServiceProvider`:

```php
use Laravel\Cashier\Cashier;

/**
 * Зареєструйте будь-які сервіси додатка.
 *
 * @return void
 */
public function register()
{
    Cashier::keepPastDueSubscriptionsActive();
}
```

> **Warning**  
> Якщо підписка знаходиться в `незавершеному` стані, її не можна змінити, доки платіж не буде підтверджено. Таким чином, методи `swap` і `updateQuantity` створять виняток, коли підписка перебуває в `незавершеному` стані.

<a name="subscription-scopes"></a>

#### Області застосування підписки

Більшість станів підписки також доступні як області запитів, щоб ви могли легко запитувати свою базу даних стосовно підписок, які перебувають у певному стані:

```php
// Отримати всі активні підписки...
$subscriptions = Subscription::query()->active()->get();

// Отримати всі скасовані підписки для користувача...
$subscriptions = $user->subscriptions()->canceled()->get();
```

Нижче наведено повний список доступних областей:

```php
Subscription::query()->active();
Subscription::query()->onTrial();
Subscription::query()->notOnTrial();
Subscription::query()->pastDue();
Subscription::query()->recurring();
Subscription::query()->ended();
Subscription::query()->paused();
Subscription::query()->notPaused();
Subscription::query()->onPausedGracePeriod();
Subscription::query()->notOnPausedGracePeriod();
Subscription::query()->cancelled();
Subscription::query()->notCancelled();
Subscription::query()->onGracePeriod();
Subscription::query()->notOnGracePeriod();
```

<a name="changing-prices"></a>

### Зміна цін

Після того, як клієнт підписався на ваш додаток, він час від часу може захотіти змінити ціну підписки на нову. Щоб змінити клієнта на нову ціну, передайте ідентифікатор ціни Stripe методу `swap`. Під час обміну цін передбачається, що користувач бажає повторно активувати свою підписку, якщо вона була раніше скасована. Даний ідентифікатор ціни має відповідати ідентифікатору ціни Stripe, доступному на інформаційній панелі Stripe:

```php
use App\Models\User;

$user = App\Models\User::find(1);

$user->subscription('default')->swap('price_yearly');
```

Якщо клієнт знаходиться на випробувальному терміні, випробувальний період буде збережено. Крім того, якщо для підписки існує «кількість», ця кількість також зберігатиметься.

Якщо ви бажаєте змінювати ціни та скасувати будь-який пробний період, який зараз використовує клієнт, ви можете викликати метод `skipTrial`:

```php
$user->subscription('default')
        ->skipTrial()
        ->swap('price_yearly');
```

Якщо ви бажаєте змінювати ціни та негайно виставити рахунок клієнту, а не чекати наступного платіжного циклу, ви можете скористатися методом `swapAndInvoice`:

```php
$user = User::find(1);

$user->subscription('default')->swapAndInvoice('price_yearly');
```

<a name="prorations"></a>

#### Пропорції

За замовчуванням Stripe пропорційно розподіляє витрати під час перемикання між цінами. Метод `noProrate` можна використовувати для оновлення ціни підписки без пропорційного розподілу плати:

```php
$user->subscription('default')->noProrate()->swap('price_yearly');
```

Для отримання додаткової інформації про розподіл підписки зверніться до [документації Stripe](https://stripe.com/docs/billing/subscriptions/prorations).

> **Warning**  
> Виконання метода `noProrate` перед методом `swapAndInvoice` не вплине на пропорцію. Рахунок-фактура завжди буде виданий.

<a name="subscription-quantity"></a>

### Вартість підписаки

Іноді на підписки впливає «Вартість». Наприклад, додаток для керування проектами може стягувати $10 на місяць за проект. Ви можете використовувати методи `incrementQuantity` і `decrementQuantity`, які дозволяють вам легко збільшити або зменшити вартість вашої підписки:

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->incrementQuantity();

// Додайте п'ять до поточної вартості підписки...
$user->subscription('default')->incrementQuantity(5);

$user->subscription('default')->decrementQuantity();

// Відніміть п'ять від поточної вартості підписки...
$user->subscription('default')->decrementQuantity(5);
```

Крім того, ви можете встановити певну вартість за допомогою метода `updateQuantity`:

```php
$user->subscription('default')->updateQuantity(10);
```

Метод `noProrate` можна використовувати для оновлення вартості підписки без пропорційного розподілу плати:

```php
$user->subscription('default')->noProrate()->updateQuantity(10);
```

Для отримання додаткової інформації про вартість підписок зверніться до [документації Stripe](https://stripe.com/docs/billing/subscriptions/quantities).

<a name="quantities-for-subscriptions-with-multiple-products"></a>

### Вартість для підписки, яка має декілька продуктів

Якщо ваша підписка має декілька продуктів, вам слід передати ID ціни, вартість якої ви хочете збільшити або зменшити, як другий аргумент для методів збільшення/зменшення:

```php
$user->subscription('default')->incrementQuantity(1, 'price_chat');
```

<a name="subscriptions-with-multiple-products"></a>

### Підписки з декількома продуктами

[Підписка з декількома продуктами](https://stripe.com/docs/billing/subscriptions/multiple-products) дозволяє вам призначити декілька платіжних продуктів для однієї підписки. Наприклад, уявіть, що ви створюєте додаток служби підтримки клієнтів із базовою ціною підписки $10 доларів на місяць, але пропонуєте додатковий продукт для живого чату за додаткові $15 доларів на місяць. Інформація для підписок на декілька продуктів зберігається в таблиці бази даних `subscription_items` Cashier.

Ви можете вказати декілька продуктів для певної підписки, передавши масив цін як другий аргумент методу `newSubscription`:

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', [
        'price_monthly',
        'price_chat',
    ])->create($request->paymentMethodId);

    // ...
});
```

У наведеному вище прикладі клієнт матиме дві ціни, додані до його підписки `default`. Обидві ціни стягуватимуться відповідно до розрахункових інтервалів. Якщо необхідно, ви можете використовувати метод `quantity`, щоб вказати конкретну кількість для кожної ціни:

```php
$user = User::find(1);

$user->newSubscription('default', ['price_monthly', 'price_chat'])
    ->quantity(5, 'price_chat')
    ->create($paymentMethod);
```

Якщо вам потрібно додати іншу ціну до існуючої підписки, ви можете викликати метод `addPrice` підписки:

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat');
```

Наведений вище приклад додасть нову ціну, яку буде виставлено клієнту в рахунку у наступному платіжному циклі. Якщо ви хочете негайно виставити рахунок клієнту, ви можете скористатися методом `addPriceAndInvoice`:

```php
$user->subscription('default')->addPriceAndInvoice('price_chat');
```

Якщо ви хочете додати ціну з певною кількістю, ви можете передати кількість другим аргументом методам `addPrice` або `addPriceAndInvoice`:

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat', 5);
```

Ви можете видалити ціни з підписок за допомогою метода `removePrice`:

```php
$user->subscription('default')->removePrice('price_chat');
```

> **Warning**  
> Ви не зможете видалити останню ціну підписки. Натомість вам слід просто скасувати підписку.

<a name="swapping-prices"></a>

#### Зміна цін підписки

Ви також можете змінити ціни, пов’язані з підпискою на декілька продуктів. Наприклад, уявіть, що клієнт має підписку `price_basic` із додатковим продуктом `price_chat`, і ви хочете оновити клієнта з `price_basic` до `price_pro price`:

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->swap(['price_pro', 'price_chat']);
```

Під час виконання наведеного вище прикладу основний елемент підписки з `price_basic` видаляється, а елемент з `price_chat` зберігається. Крім того, створюється новий елемент підписки для `price_pro`.

Ви також можете вказати параметри елемента підписки, передавши масив пар ключ/значення в метод `swap`. Наприклад, вам може знадобитися вказати кількість вартості підписки:

```php
$user = User::find(1);

$user->subscription('default')->swap([
    'price_pro' => ['quantity' => 5],
    'price_chat'
]);
```

Якщо ви хочете поміняти єдину ціну на підписку, ви можете зробити це за допомогою метода `swap` на самому елементі підписки. Цей підхід особливо корисний, якщо ви хочете зберегти всі наявні метадані про інші ціни підписки:

```php
$user = User::find(1);

$user->subscription('default')
        ->findItemOrFail('price_basic')
        ->swap('price_pro');
```

<a name="proration"></a>

#### Пропорція

За замовчуванням Stripe пропорційно розподіляє витрати під час додавання або видалення цін з підписки на кілька продуктів. Якщо ви хочете скоригувати ціну без пропорції, вам слід приєднати метод `noProrate` до вашої операції з ціною:

```php
$user->subscription('default')->noProrate()->removePrice('price_chat');
```

<a name="quantities"></a>

#### Вартості

Якщо ви бажаєте оновити вартість на окремих цінах підписки, ви можете зробити це за допомогою [існуючих методів вартості](#subscription-quantity), передавши методу назву ціни як додатковий аргумент:

> **Warning**  
> Якщо підписка має декілька цін, атрибути `stripe_price` і `quantity` в моделі `Subscription` будуть `null`. Щоб отримати доступ до окремих цінових атрибутів, ви повинні використовувати зв’язок `items`, доступний у моделі `підписки`.

<a name="subscription-items"></a>

#### Елементи підписки

Якщо підписка має декілька цін, у таблиці subscription_items вашої бази даних зберігається кілька «елементів» підписки. Якщо підписка має кілька цін, у таблиці subscription_items вашої бази даних зберігається кілька «елементів» підписки. Ви можете отримати доступ до них через зв’язок елементів у підписці:

```php
use App\Models\User;

$user = User::find(1);

$subscriptionItem = $user->subscription('default')->items->first();

// Отримати ціну та кількість Stripe для певного елементу......
$stripePrice = $subscriptionItem->stripe_price;
$quantity = $subscriptionItem->quantity;
```

Ви також можете отримати конкретну ціну за допомогою метода `findItemOrFail`:

```php
$user = User::find(1);

$subscriptionItem = $user->subscription('default')->findItemOrFail('price_chat');
```

<a name="metered-billing"></a>

### Плата за лічильником

[Плата за лічильником](https://stripe.com/docs/products-prices/pricing-models#usage-based-pricing) дає змогу стягувати плату з клієнтів на основі використання продукту протягом платіжного циклу. Наприклад, ви можете стягувати плату з клієнтів на основі кількості текстових повідомлень або електронних листів, які вони надсилають за місяць.

Щоб почати використовувати тарифікацію за лічильником, вам спочатку потрібно буде створити новий продукт на інформаційній панелі Stripe з ціною за лічильником. Потім використовуйте `meteredPrice`, щоб додати ID визначеної ціни до підписки клієнта:

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default')
        ->meteredPrice('price_metered')
        ->create($request->paymentMethodId);

    // ...
});
```

Ви також можете розпочати передплату з обмеженнями через [Stripe Checkout](#checkout):

```php
$checkout = Auth::user()
        ->newSubscription('default', [])
        ->meteredPrice('price_metered')
        ->checkout();

return view('your-checkout-view', [
    'checkout' => $checkout,
]);
```

<a name="reporting-usage"></a>

#### Звіт про використання

Коли ваш клієнт буде використовувати ваш додаток, ви повідомлятимете про його використання в Stripe, щоб їм могли виставляти точні рахунки. Щоб збільшити використання облікової підписки, ви можете використати метод `reportUsage`:

```php
$user = User::find(1);

$user->subscription('default')->reportUsage();
```

За замовчуванням до розрахункового періоду додається «кількість використання» 1. Крім того, ви можете передати певну суму «використання», щоб додати до використання клієнтом за розрахунковий період:

```php
$user = User::find(1);

$user->subscription('default')->reportUsage(15);
```

Якщо ваш додаток пропонує декілька цін на одну підписку, вам потрібно буде використати метод `reportUsageFor`, щоб вказати визначену ціну, використання якої ви хочете звітувати:

```php
$user = User::find(1);

$user->subscription('default')->reportUsageFor('price_metered', 15);
```

Іноді вам може знадобитися оновити використання, про яке ви повідомляли раніше. Щоб досягти цього, ви можете передати мітку часу або екземпляр `DateTimeInterface` другим параметром для `reportUsage`. При цьому Stripe оновить дані про використання, які повідомлялись на той момент. Ви можете продовжувати оновлювати попередні записи про використання, оскільки вказані дата й час все ще входять до поточного розрахункового періоду:

```php
$user = User::find(1);

$user->subscription('default')->reportUsage(5, $timestamp);
```

<a name="retrieving-usage-records"></a>

#### Отримання записів про використання

Щоб отримати дані про минуле використання клієнтом, ви можете використати метод `usageRecords` екземпляра підписки:

```php
$user = User::find(1);

$usageRecords = $user->subscription('default')->usageRecords();
```

Якщо ваш додаток пропонує декілька цін на одну підписку, ви можете використовувати метод `usageRecordsFor`, щоб вказати визначену ціну, для якої ви бажаєте отримати записи про використання:

```php
$user = User::find(1);

$usageRecords = $user->subscription('default')->usageRecordsFor('price_metered');
```

Методи `usageRecords` і `usageRecordsFor` повертають екземпляр колекції, яка містить асоціативний масив записів про використання. Ви можете переглянути цей масив, щоб відобразити загальне використання клієнта:

```php
@foreach ($usageRecords as $usageRecord)
    - Period Starting: {{ $usageRecord['period']['start'] }}
    - Period Ending: {{ $usageRecord['period']['end'] }}
    - Total Usage: {{ $usageRecord['total_usage'] }}
@endforeach
```

Для повного ознайомлення з усіма повернутими данними про використання та про те, як використовувати пагінацію на основі курсора Stripe, зверніться до [офіційної документації Stripe API](https://stripe.com/docs/api/usage_records/subscription_item_summary_list).

<a name="subscription-taxes"></a>

### Податки на підписку

> **Warning**  
> Замість того, щоб обчислювати податкові відсотки вручну, ви можете [автоматично розраховувати податки за допомогою Stripe Tax](#tax-configuration).

Щоб вказати податкові відсотки, які користувач сплачує за підписку, вам слід застосувати метод `taxRates` у вашій платіжній моделі та повернути масив, що містить ідентифікатори податкових ставок Stripe. Ви можете визначити ці податкові відсотки на [інформаційній панелі Stripe](https://dashboard.stripe.com/login?redirect=%2Ftest%2Ftax-rates):

```php
/**
 * Відсотки податку, які мають застосовуватися до підписок клієнта.
 *
 * @return array
 */
public function taxRates()
{
    return ['txr_id'];
}
```

Метод `taxRates` дає змогу застосовувати податкову ставку для кожного клієнта, що може бути корисним для бази користувачів, яка охоплює декілька країн і податкових ставок.

Якщо ви пропонуєте підписку з деількома продуктами, ви можете визначити різні відсотки податку для кожної ціни, застосувавши метод `priceTaxRates` у своїй платіжній моделі:

```php
/**
 * Відсотки податку, які мають застосовуватися до підписок клієнта.
 *
 * @return array
 */
public function priceTaxRates()
{
    return [
        'price_monthly' => ['txr_id'],
    ];
}
```

> **Warning**  
> Метод `taxRates` застосовується лише на оплату за підписку. Якщо ви використовуєте Cashier для одноразових платежів, вам потрібно буде вручну вказати відсоток податку на цей момент.

<a name="syncing-tax-rates"></a>

### Синхронізація податкових відсотків

Під час зміни жорстко закодованих IDs відсотку податку, отриманих методом `taxRates`, налаштування податку на будь-яких існуючих підписок для користувача залишаться незмінними. Якщо ви хочете оновити значення податку для існуючих підписок новими значеннями `taxRates`, вам слід викликати метод `syncTaxRates` в екземплярі підписки користувача:

```php
$user->subscription('default')->syncTaxRates();
```

Це також синхронізує будь-які відсотки податку на елементи для підписки за декількома продуктами. Якщо ваш додаток пропонує передплату за декількома продуктами, ви повинні переконатися, що ваша оплачувана модель реалізує [описаний вище](#subscription-taxes) метод `priceTaxRates`.

<a name="tax-exemption"></a>

### Звільнення від сплати податків

Cashier також пропонує методи `isNotTaxExempt`, `isTaxExempt` і `reverseChargeApplies`, щоб визначити, чи звільнений клієнт від податків. Ці методи викличуть API Stripe для визначення статусу звільнення клієнта від податків:

```php
use App\Models\User;

$user = User::find(1);

$user->isTaxExempt();
$user->isNotTaxExempt();
$user->reverseChargeApplies();
```

> **Warning**  
> Ці методи також доступні для будь-якого об’єкта `Laravel\Cashier\Invoice`. Однак, викликані для об’єкта `Invoice`, методи визначатимуть статус звільнення на момент створення рахунку-фактури.

<a name="subscription-anchor-date"></a>

### Прив’язана дата підписки

За замовчуванням прив’язкою платіжного циклу є дата створення підписки або, якщо використовується пробний період, дата закінчення пробного періоду. Якщо ви хочете змінити прив’язану дату виставлення рахунків, ви можете скористатися методом `anchorBillingCycleOn`:

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $anchor = Carbon::parse('first day of next month');

    $request->user()->newSubscription('default', 'price_monthly')
        ->anchorBillingCycleOn($anchor->startOfDay())
        ->create($request->paymentMethodId);

    // ...
});
```

Для отримання додаткової інформації про керування платіжними циклами підписки зверніться до [документації про платіжні цикли Stripe](https://stripe.com/docs/billing/subscriptions/billing-cycle).

<a name="cancelling-subscriptions"></a>

### Скасування підписок

Щоб скасувати підписку, викличте метод `cancel` у підписці користувача:

```php
$user->subscription('default')->cancel();
```

Коли підписка скасована, Cashier автоматично фіксує стовпець `ends_at` у вашій таблиці бази даних `subscriptions`. Цей стовпець використовується, щоб знати, коли `subscribed` метод повинен почати повертати `false`.

Наприклад, якщо клієнт скасовує підписку 1 березня, але термін дії підписки не був запланований до 5 березня, метод `subscribed` повертатиме значення `true` до 5 березня. Це робиться тому, що користувачеві зазвичай дозволяється продовжувати користуватися додатком до кінця свого платіжного циклу.

За допомогою метода `onGracePeriod` можна визначити, чи користувач скасував свою підписку, але все ще діє «пільговий період»:

```php
if ($user->subscription('default')->onGracePeriod()) {
    //
}
```

Якщо ви бажаєте негайно скасувати підписку та виставити рахунок-фактуру за будь-які невиставлені рахунки, за виміряним використанням або нові/очікувані пропорційні елементи рахунку-фактури, викличте метод `cancelNowAndInvoice` у підписці користувача:

```php
$user->subscription('default')->cancelNowAndInvoice();
```

Ви також можете скасувати підписку в певний момент часу:

```php
$user->subscription('default')->cancelAt(
    now()->addDays(10)
);
```

<a name="resuming-subscriptions"></a>

## Відновлення підписок

Якщо клієнт скасував свою підписку, і ви хочете відновити її, ви можете викликати метод `resume` для підписки. Щоб відновити підписку, клієнт все ще має перебувати в межах свого «пільгового періоду»:

```php
$user->subscription('default')->resume();
```

Якщо клієнт скасовує підписку, а потім відновлює цю підписку до того, як термін дії підписки повністю закінчиться, клієнту не буде виставлено рахунок негайно. Натомість їхню підписку буде повторно активовано, і їм буде виставлено рахунок згідно з початковим платіжним циклом.

<a name="subscription-trials"></a>

## Пробні версії підписки

<a name="with-payment-method-up-front"></a>

### З передоплатою

Якщо ви хочете запропонувати своїм клієнтам пробні періоди, водночас зберігаючи інформацію про спосіб оплати, вам слід використовувати метод `trialDays` під час створення підписок:

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', 'price_monthly')
        ->trialDays(10)
        ->create($request->paymentMethodId);

    // ...
});
```

Цей метод встановить дату закінчення пробного періоду в записі про підписку в базі даних, та проінформує Stripe не починати виставляти рахунок клієнту до цієї дати. У разі використання метода `trialDays` Cashier перезапише будь-який пробний період за замовчуванням, налаштований для ціни в Stripe.

> **Warning**  
> Якщо підписку клієнта не скасовано до дати закінчення пробного періоду, з нього буде стягнено плату, щойно закінчиться пробний період, тому ви повинні обов’язково повідомити своїх користувачів про дату закінчення пробного періоду.

Метод `trialUntil` дозволяє вам надати екземпляр `DateTime`, який визначає, коли має закінчитися пробний період:

```php
use Carbon\Carbon;

$user->newSubscription('default', 'price_monthly')
    ->trialUntil(Carbon::now()->addDays(10))
    ->create($paymentMethod);
```

Ви можете визначити, чи триває пробний період користувача, використовуючи метод `onTrial` екземпляра користувача або метод `onTrial` екземпляра підписки. Два наведені нижче приклади еквівалентні:

```php
if ($user->onTrial('default')) {
    //
}

if ($user->subscription('default')->onTrial()) {
    //
}
```

Ви можете використати метод `endTrial`, щоб негайно завершити пробну версію підписки:

```php
$user->subscription('default')->endTrial();
```

Щоб визначити, чи закінчився термін дії існуючої пробної версії, ви можете скористатися методами `hasExpiredTrial`:

```php
if ($user->hasExpiredTrial('default')) {
    //
}

if ($user->subscription('default')->hasExpiredTrial()) {
    //
}
```

<a name="defining-trial-days-in-stripe-cashier"></a>

#### Визначення пробних днів у Stripe / Cashier

Ви можете обирати, скільки пробних днів отримуватиме ваша ціна на інформаційній панелі Stripe або завжди передавати їх явно за допомогою Cashier. Якщо ви вирішите визначити пробні дні вашої ціни в Stripe, пам’ятайте, що нові підписки, включаючи нові підписки для клієнта, який мав підписку в минулому, завжди отримуватиме пробний період, якщо ви явно не викличете метод `skipTrial()`.

<a name="without-payment-method-up-front"></a>

### Без попереднього способу оплати

Якщо ви хочете запропонувати пробні періоди без попереднього збору інформації про спосіб оплати користувача, ви можете встановити в стовпці `trial_ends_at` запису користувача бажану дату закінчення пробного періоду. Зазвичай це робиться під час реєстрації користувача:

```php
use App\Models\User;

$user = User::create([
    // ...
    'trial_ends_at' => now()->addDays(10),
]);
```

> **Warning**  
> Обов’язково додайте [дату `casts`](https://laravel.com/docs/9.x/eloquent-mutators##date-casting) для атрибута `trial_ends_at` у визначенні класу вашої оплачуваної моделі.

Cashier називає цей тип пробної версії «загальною пробною версією», оскільки вона не пов’язана з жодною існуючою підпискою. Метод `onTrial` в екземплярі оплачуваної моделі поверне значення `true`, якщо поточна дата не перевищує значення `trial_ends_at`:

```php
if ($user->onTrial()) {
    // Випробувальний період користувача...
}
```

Коли ви будете готові створити фактичну підписку для користувача, ви можете використовувати метод `newSubscription` як зазвичай:

```php
$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->create($paymentMethod);
```

Щоб отримати дату закінчення пробної версії користувача, ви можете використати метод `trialEndsAt`. Цей метод поверне екземпляр дати Carbon, якщо користувач перебуває на пробній версії, або `null`, якщо це не так. Ви також можете передати додатковий параметр назви підписки, якщо хочете отримати дату закінчення пробного періоду для певної підписки, яка відрізняється від стандартної:

```php
if ($user->onTrial()) {
    $trialEndsAt = $user->trialEndsAt('main');
}
```

Ви також можете використовувати метод `onGenericTrial`, якщо хочете точно знати, що користувач перебуває в межах свого «загального» пробного періоду та ще не створив фактичну підписку:

```php
if ($user->onGenericTrial()) {
    // Користувач знаходиться в межах свого "загального" пробного періоду...
}
```

<a name="extending-trials"></a>

### Розширення випробувань

Метод `extendTrial` дозволяє продовжити пробний період підписки після того, як підписку було створено. Якщо термін дії пробного періоду вже закінчився, а клієнту вже виставлено рахунок за підписку, ви все одно можете запропонувати йому продовжений пробний період. Час, витрачений протягом пробного періоду, буде вираховано з наступного рахунку клієнта:

```php
use App\Models\User;

$subscription = User::find(1)->subscription('default');

// Закінчити пробну версію через 7 днів...
$subscription->extendTrial(
    now()->addDays(7)
);

// Додайте ще 5 днів до пробної версії...
$subscription->extendTrial(
    $subscription->trial_ends_at->addDays(5)
);
```

<a name="handling-stripe-webhooks"></a>

## Опрацювання веб-гачків Stripe

> **Note**  
> Ви можете використовувати [Stripe CLI](https://stripe.com/docs/stripe-cli), щоб допомогти тестувати веб-гачки під час локальної розробки.

Stripe може повідомляти ваш додаток про різноманітні події через веб-гачки. За замовчуванням маршрут, який вказує на контроллер веб-гачка Cashier, автоматично реєструється постачальником служб Cashier. Цей контролер опрацьовуватиме всі вхідні запити веб-гачока, які надходять.

За замовчуванням контролер веб-гачка Cashier автоматично опрацьовуватиме скасування підписок з занадто великою кількістю невдалих стягнень (як визначено налаштуваннями Stripe), оновлення клієнтів, видалення клієнтів, оновлення підписки та зміни способу оплати; однак, як ми незабаром дізнаємося, ви можете розширити цей контролер для опрацювання будь-якої події веб-гачка Stripe.

Щоб ваш додаток міг опрацьовувати веб-гачки Stripe, обов’язково налаштуйте URL-адресу веб-гачків на панелі керування Stripe. За замовчуванням контроллер веб-гачка Cashier відповідає на URL-шлях `/stripe/webhook`. Повний список всіх веб-гачків, які ви повинні ввімкнути на панелі керування Stripe:

- customer.subscription.created
- customer.subscription.updated
- customer.subscription.deleted
- customer.updated
- customer.deleted
- invoice.payment_action_required

Для зручності Cashier містить команду `cashier:webhook` Artisan. Ця команда створить веб-гачок у Stripe, який прослуховує всі події, необхідні Cashier:

```ini
php artisan cashier:webhook
```

За замовчуванням створений веб-гачок вказуватиме на URL-адресу, визначену змінною середовища `APP_URL` і маршрутом `cashier.webhook`, який включено до Cashier. Ви можете вказати параметр `--url` під час виклику команди, якщо ви хочете використовувати іншу URL-адресу:

```ini
php artisan cashier:webhook --url "https://example.com/stripe/webhook"
```

Створений веб-гачок використовуватиме версію Stripe API, з якою сумісна ваша версія Cashier. Якщо ви хочете використовувати іншу версію Stripe, ви можете вказати параметр `--api-version`:

```ini
php artisan cashier:webhook --api-version="2019-12-03"
```

Після створення, веб-гачок одразу стане активним. Якщо ви бажаєте створити веб-гачок, але вимкнути його, поки не будете готові, ви можете вказати параметр `--disabled` під час виклику команди:

```ini
php artisan cashier:webhook --disabled
```

> **Note**  
> Переконайтеся, що ви захищаєте вхідні запити веб-гачка Stripe за допомогою [посередника перевірки підпису веб-гачка Cashier](#verifying-webhook-signatures).

<a name="webhooks-&-csrf-protection"></a>

#### Веб-гачок та захист CSRF

Оскільки веб-гачкам Stripe потрібно обійти захист `CSRF Laravel`, переконайтеся, що URI вказано як виняток в посереднику `App\Http\Middleware\VerifyCsrfToken` вашого додатка або вкажіть маршрут за межами групи посередників `web`:

```php
protected $except = [
    'stripe/*',
];
```

<a name="defining-webhook-event-handlers"></a>

### Визначення виконавців подій веб-гачків

Cashier автоматично опрацьовує скасування підписки через невдалу оплату та інші поширені події веб-гачка Stripe. Однак, якщо у вас є додаткові події веб-гачка, які б ви хотіли опрацювати, ви можете зробити це, прослухавши такі події, які надсилає Cashier:

- Laravel\Cashier\Events\WebhookReceived
- Laravel\Cashier\Events\WebhookHandled

Обидві події містять повне корисне навантаження веб-гачка Stripe. Наприклад, якщо ви бажаєте опрацювати веб-гачок `invoice.payment_succeeded`, ви можете зареєструвати слухача, який опрацьовуватиме подію:

```php
<?php

namespace App\Listeners;

use Laravel\Cashier\Events\WebhookReceived;

class StripeEventListener
{
    /**
     * Опрацьовуватиме отримані вебхуки Stripe.
     *
     * @param  \Laravel\Cashier\Events\WebhookReceived  $event
     * @return void
     */
    public function handle(WebhookReceived $event)
    {
        if ($event->payload['type'] === 'invoice.payment_succeeded') {
            // Опрацювати вхідну подію...
        }
    }
}
```

Після того, як ваш слухач визначено, ви можете зареєструвати його в `EventServiceProvider` вашого додатка:

```php
<?php

namespace App\Providers;

use App\Listeners\StripeEventListener;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use Laravel\Cashier\Events\WebhookReceived;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        WebhookReceived::class => [
            StripeEventListener::class,
        ],
    ];
}
```

<a name="verifying-webhook-signatures"></a>

### Перевірка підписаних Веб-гачків

Щоб захистити свої веб-гачки, ви можете використовувати [підписані веб-гачки Stripe](https://stripe.com/docs/webhooks/signatures). Для зручності Cashier автоматично включає посередника, який перевіряє дійсність вхідного запита веб-гачка Stripe.

Щоб увімкнути перевірку веб-гачка, переконайтеся, що змінну середовища `STRIPE_WEBHOOK_SECRET` встановлено у файлі `.env` додатка. `secret` веб-гачка можна отримати з інформаційної панелі вашого облікового запису Stripe.

<a name="single-charges"></a>

## Одноразові платежі

<a name="simple-charge"></a>

### Одноразовий платіж

Якщо ви хочете одноразово стягнути плату з клієнта, ви можете використати метод `charge` на екземплярі оплачуваної моделі.
Вам потрібно буде [надати ідентифікатор спосіба оплати](#payment-methods-for-single-charges) другим аргументом методу `charge`:

```php
use Illuminate\Http\Request;

Route::post('/purchase', function (Request $request) {
    $stripeCharge = $request->user()->charge(
        100, $request->paymentMethodId
    );

    // ...
});
```

Метод `charge` приймає масив третім аргументом, дозволяючи вам передавати будь-які параметри, які ви бажаєте, для базового створення платежу Stripe. Докладніше про параметри, які доступні для вас під час створення платежів, можна знайти в [документації Stripe](https://stripe.com/docs/api/charges/create):

```php
$user->charge(100, $paymentMethod, [
    'custom_option' => $value,
]);
```

Ви також можете використовувати метод `charge` без основного клієнта чи користувача. Для цього викличте метод `charge` на новому екземплярі моделі оплати вашого додатка:

```php
use App\Models\User;

$stripeCharge = (new User)->charge(100, $paymentMethod);
```

Метод `charge` викличе виняток, якщо оплата буде невдалою. Якщо стягнення плати буде успішним, з метода буде відданий екземпляр `Laravel\Cashier\Payment`:

```php
try {
    $payment = $user->charge(100, $paymentMethod);
} catch (Exception $e) {
    //
}
```

> **Note**  
> Метод `charge` приймає суму платежу в найменшому знаменнику валюти, яка використовується вашим додатком. Наприклад, якщо клієнти платять у доларах США, суми слід вказувати в пенні.

<a name="charge-with-invoice"></a>

### Платежі за рахунком-фактурою

Іноді вам може знадобитися здійснити одноразову оплату та запропонувати клієнту квитанцію у форматі PDF. Метод `invoicePrice` дозволяє зробити саме це. Наприклад, давайте виставимо клієнту рахунок за п’ять нових сорочок:

```php
$user->invoicePrice('price_tshirt', 5);
```

Рахунок буде негайно стягнено за способом оплати користувача за замовчуванням. Метод `invoicePrice` також приймає масив третім аргументом. Цей масив містить параметри виставлення рахунків для позиції рахунку. Четвертий аргумент, прийнятий методом, також є масивом, який має містити параметри виставлення рахунків для самого рахунку:

```php
$user->invoicePrice('price_tshirt', 5, [
    'discounts' => [
        ['coupon' => 'SUMMER21SALE']
    ],
], [
    'default_tax_rates' => ['txr_id'],
]);
```

Подібно до `invoicePrice`, ви можете використовувати метод `tabPrice`, щоб створити одноразову плату за декілька позицій (до 250 позицій на рахунок-фактуру), додавши їх на «вкладку» клієнта, а потім виставивши клієнту рахунок. Наприклад, ми можемо виставити клієнту рахунок на п’ять сорочок і дві чашки:

```php
$user->tabPrice('price_tshirt', 5);
$user->tabPrice('price_mug', 2);
$user->invoice();
```

Крім того, ви можете використати метод `invoiceFor`, щоб здійснити «одноразову» оплату зі способу оплати клієнта за замовчуванням:

```php
$user->invoiceFor('One Time Fee', 500);
```

Хоча метод `invoiceFor` доступний для використання, всеж таки рекомендується використовувати методи `invoicePrice` і `tabPrice` із попередньо визначеними цінами. Зробивши це, ви отримаєте доступ до кращої аналітики та даних на інформаційній панелі Stripe щодо ваших продажів для кожного продукту.

> **Warning**  
> Методи `invoice`, `invoicePrice` і `invoiceFor` створять рахунок-фактуру Stripe, який повторюватиме невдалі спроби виставлення рахунка. Якщо ви не хочете, щоб рахунки повторювали невдалі платежі, вам потрібно буде закрити їх за допомогою Stripe API після першого невдалого стягнення.

<a name="creating-payment-intents"></a>

### Створення платіжних намірів

Ви можете створити новий платіжний намір Stripe, викликавши метод `pay` в екземплярі оплачуваної моделі. Виклик цього метода створить намір платежу, загорнутий в екземпляр `Laravel\Cashier\Payment`:

Після створення платіжного наміру ви можете повернути "секрет" клієнта до інтерфейсу додатка, щоб користувач міг завершити платіж у своєму браузері. Щоб дізнатися більше про створення повних потоків платежів за допомогою платіжних намірів Stripe, зверніться до [документації Stripe](https://stripe.com/docs/payments/accept-a-payment?platform=web).

Під час використання метода оплати клієнту будуть доступні способи оплати за замовчуванням, увімкнені на інформаційній панелі Stripe. Крім того, якщо ви хочете дозволити використовувати лише певні методи оплати, ви можете скористатися методом `payWith`:

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->payWith(
        $request->get('amount'), ['card', 'bancontact']
    );

    return $payment->client_secret;
});
```

> **Note**  
> Методи `pay` і `payWith` приймають суму платежу в найменшому знаменнику валюти, яка використовується вашим додатком. Наприклад, якщо клієнти платять у доларах США, суми слід вказувати в пенні.

<a name="refunding-charges"></a>

## Відшкодування платежів

Якщо вам потрібно відшкодувати плату за Stripe, ви можете скористатися методом `refund`. Цей метод приймає ID наміру оплати Stripe першим аргументом:

```php
$payment = $user->charge(100, $paymentMethodId);

$user->refund($payment->id);
```

<a name="invoices"></a>

## Рахунок-фактура

<a name="retrieving-invoices"></a>

### Отримання рахунків

Ви можете легко отримати масив рахунків платіжної моделі за допомогою метода `invoices`. Метод `invoices` повертає колекцію екземплярів `Laravel\Cashier\Invoice`:

```php
$invoices = $user->invoices();
```

Якщо ви хочете включити в результати рахунки,які очікують на розгляд, ви можете скористатися методом `invoicesIcludePending`:

```php
$invoices = $user->invoicesIncludingPending();
```

Ви можете використовувати метод `findInvoice`, щоб отримати певний рахунок за його ID:

```php
$invoice = $user->findInvoice($invoiceId);
```

<a name="displaying-invoice-information"></a>

#### Відображення інформації про рахунок

Перераховуючи рахунки для клієнта, ви можете використовувати методи `invoices` для відображення відповідної інформації про рахунок-фактуру. Наприклад, ви можете перерахувати кожен рахунок в таблиці, що дозволить користувачеві легко завантажити будь-який із них:

```php
<table>
    @foreach ($invoices as $invoice)
        <tr>
            <td>{{ $invoice->date()->toFormattedDateString() }}</td>
            <td>{{ $invoice->total() }}</td>
            <td><a href="/user/invoice/{{ $invoice->id }}">Download</a></td>
        </tr>
    @endforeach
</table>
```

<a name="upcoming-invoices"></a>

### Майбутні рахунки

Щоб отримати майбутній рахуноки для клієнта, ви можете використати метод `upcomingInvoice`:

```php
$invoice = $user->upcomingInvoice();
```

Подібним чином, якщо клієнт має декілька підписок, ви також можете отримати майбутній рахунок для певної підписки:

```php
$invoice = $user->subscription('default')->upcomingInvoice();
```

<a name="previewing-subscription-invoices"></a>

### Попередній перегляд рахунків за підписку

Використовуючи метод `previewInvoice`, ви можете попередньо переглянути рахуноки, перш ніж вносити зміни в ціну. Це дозволить вам визначити, як виглядатиме рахунок вашого клієнта після зміни певної ціни:

```php
$invoice = $user->subscription('default')->previewInvoice('price_yearly');
```

Ви можете передати масив цін у метод `previewInvoice`, для попереднього перегляду рахунків з декількома новими цінами:

<a name="generating-invoice-pdfs"></a>

### Створення рахунків у PDF-файлах

Перш ніж створювати PDF-файли рахунків-фактур, вам слід за допомогою Composer встановити бібліотеку Dompdf, яка є засобом обробки рахунків за замовчуванням для Cashier:

```ini
composer require dompdf/dompdf
```

З маршруту або контролера ви можете використовувати метод `downloadInvoice`, щоб створити PDF-файл для завантаження даного рахунка-фактури. Цей метод автоматично створить належну відповідь HTTP, необхідну для завантаження рахунку:

```php
use Illuminate\Http\Request;

Route::get('/user/invoice/{invoice}', function (Request $request, $invoiceId) {
    return $request->user()->downloadInvoice($invoiceId);
});
```

За замовчуванням всі дані в рахунку походять із даних клієнта та рахунку-фактури, які зберігаються в Stripe. Назва файлу базується на значенні конфігурації `app.name`. Однак ви можете налаштувати деякі з цих даних, надавши масив як другий аргумент методу `downloadInvoice`. Цей масив дозволяє налаштувати таку інформацію, як інформація про вашу компанію та продукт:

```php
return $request->user()->downloadInvoice($invoiceId, [
    'vendor' => 'Your Company',
    'product' => 'Your Product',
    'street' => 'Main Str. 1',
    'location' => '2000 Antwerp, Belgium',
    'phone' => '+32 499 00 00 00',
    'email' => 'info@example.com',
    'url' => 'https://example.com',
    'vendorVat' => 'BE123456789',
]);
```

Метод `downloadInvoice` також дозволяє визначити користувацьке ім’я файлу за допомогою третього аргументу. До цієї назви файлу буде автоматично додано суфікс .pdf:

```php
return $request->user()->downloadInvoice($invoiceId, [], 'my-invoice');
```

<a name="custom-invoice-renderer"></a>

#### Індивідуальний засіб рендерінгу рахунків

Cashier також дає можливість використовувати спеціальний рендерер рахунків. За замовчуванням Cashier використовує реалізацію `DompdfInvoiceRenderer`, яка використовує PHP-бібліотеку [dompdf](https://github.com/dompdf/dompdf) для створення рахунків Cashier. Однак ви можете використовувати будь-який рендерер, реалізувавши інтерфейс `Laravel\Cashier\Contracts\InvoiceRenderer`. Наприклад, ви можете відобразити PDF-файл рахунка за допомогою виклику API сторонньої служби відтворення PDF-файлів:

```php
use Illuminate\Support\Facades\Http;
use Laravel\Cashier\Contracts\InvoiceRenderer;
use Laravel\Cashier\Invoice;

class ApiInvoiceRenderer implements InvoiceRenderer
{
    /**
     * Візуалізуйте вказаний рахунок-фактуру та поверніть необроблені байти PDF.
     *
     * @param  \Laravel\Cashier\Invoice. $invoice
     * @param  array  $data
     * @param  array  $options
     * @return string
     */
    public function render(Invoice $invoice, array $data = [], array $options = []): string
    {
        $html = $invoice->view($data)->render();

        return Http::get('https://example.com/html-to-pdf', ['html' => $html])->get()->body();
    }
}
```

Після того, як ви реалізували договір рендерінгу рахунків, вам слід оновити значення конфігурації `cashier.invoices.renderer` у конфігураційному файлі додатка `config/cashier.php`. Це значення конфігурації має бути встановлено на ім’я класу вашої власної реалізації рендера.

<a name="checkout"></a>

## Перевірка

Cashier Stripe також підтримує [Stripe Checkout](https://stripe.com/payments/checkout). Stripe Checkout позбавляє від проблем реалізації власних сторінок для прийому платежів, надаючи попередньо створену сторінку платежів.

Наступна документація містить інформацію про те, як почати використовувати Stripe Checkout із Cashier. Щоб дізнатися більше про Stripe Checkout, вам слід також переглянути власну документацію Stripe щодо Checkout.

<a name="product-checkouts"></a>

### Перевірка товару

Ви можете здійснити перевірку існуючого продукту, який було створено на вашій інформаційній панелі Stripe, використовуючи метод `checkout` на платній моделі. Метод `checkout` ініціює новий сеанс перевірки Stripe. За замовчуванням вам потрібно передати Stripe Price ID:

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout('price_tshirt');
});
```

При необхідності ви також можете вказати кількість товару:

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 15]);
});
```

Коли клієнт відвідує цей маршрут, він буде перенаправлений на сторінку оформлення замовлення Stripe. За замовчуванням, коли користувач успішно завершує або скасовує покупку, він буде перенаправлений до вашого маршруту `home`, але ви можете вказати власні URL-адреси зворотного виклику за допомогою параметрів `success_url` і `cancel_url`:

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

Визначаючи параметр перевірки для `success_url`, ви можете вказати Stripe додати перевірку ID сесії як параметр рядка запиту під час виклику вашої URL-адреси. Для цього додайте літеральний рядок `{CHECKOUT_SESSION_ID}` до свого рядка запита `success_url`. Stripe замінить цей заповнювач на фактичну перевірку ID сесії:

```php
use Illuminate\Http\Request;
use Stripe\Checkout\Session;
use Stripe\Customer;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('checkout-success').'?session_id={CHECKOUT_SESSION_ID}',
        'cancel_url' => route('checkout-cancel'),
    ]);
});

Route::get('/checkout-success', function (Request $request) {
    $checkoutSession = $request->user()->stripe()->checkout->sessions->retrieve($request->get('session_id'));

    return view('checkout.success', ['checkoutSession' => $checkoutSession]);
})->name('checkout-success');
```

<a name="promotion-codes"></a>

#### Промо-коди

За замовчуванням Stripe Checkout не дозволяє [користувачам активувати промо-коди](https://stripe.com/docs/billing/subscriptions/coupons). На щастя, є простий спосіб увімкнути їх для вашої сторінки Checkout. Для цього ви можете викликати метод `allowPromotionCodes`:

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()
        ->allowPromotionCodes()
        ->checkout('price_tshirt');
});
```

<a name="single-charge-checkouts"></a>

#### Перевірка одноразовх платежів

Ви також можете здійснити просте стягнення оплати за спеціальний продукт, який не було створено на інформаційній панелі Stripe. Для цього ви можете використати метод `checkoutCharge` на платіжній моделі та передати йому суму, яка підлягає оплаті, назву продукту та необов’язкову кількість. Коли клієнт відвідує цей маршрут, він буде перенаправлений на сторінку оформлення замовлення Stripe:

```php
use Illuminate\Http\Request;

Route::get('/charge-checkout', function (Request $request) {
    return $request->user()->checkoutCharge(1200, 'T-Shirt', 5);
});
```

Під час використання методу `checkoutCharge` Stripe завжди створюватиме новий продукт і ціну на панелі інструментів Stripe. Тому ми рекомендуємо створювати продукти на інформаційній панелі Stripe і замість цього використовувати метод `checkout` замовлення.

<a name="testing"></a>

## Testing

For automated tests, including those executed within a CI environment, you may use [Laravel's HTTP Client](/docs/{{version}}/http-client#testing) to fake HTTP calls made to Paddle. Although this does not test the actual responses from Paddle, it does provide a way to test your application without actually calling Paddle's API.
