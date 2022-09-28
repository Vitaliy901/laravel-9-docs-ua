# Транслювання

- [Вступ](#introduction)
- [Встановлення на боці сервера](#server-side-installation)
  - [Налаштування](#configuration)
  - [Pusher Channels](#pusher-channels)
  - [Ably](#ably)
  - [Альтернативи з відкритим кодом](#open-source-alternatives)
- [Встановлення на боці клієнта](#client-side-installation)
  - [Pusher Channels](#client-pusher-channels)
  - [Ably](#client-ably)
- [Огляд концепції](#concept-overview)
  - [Приклад використання](#using-example-application)
- [Визначення транслюючих подій](#defining-broadcast-events)
  - [Трансляція імені](#broadcast-name)
  - [Трансляція даних](#broadcast-data)
  - [Трансляція черги](#broadcast-queue)
  - [Трансляція через умову](#broadcast-conditions)
  - [Транслювання і транзакція баз даних](#broadcasting-and-database-transactions)
- [Авторизація каналів](#authorizing-channels)
  - [Визначення маршрутів аторизації](#defining-authorization-routes)
  - [Визначення авторизації канала](#defining-authorization-callbacks)
  - [Визначення каласа канала](#defining-channel-classes)
- [Транслювання подій](#broadcasting-events)
  - [Трансляція подій лише іншим користувачам](#only-to-others)
  - [Налаштування підключення](#customizing-the-connection)
- [Отримання трансляцій](#receiving-broadcasts)
  - [Прослуховування подій](#listening-for-events)
  - [Залишення канала](#leaving-a-channel)
  - [Простір імен](#namespaces)
- [Канали присутності](#presence-channels)
  - [Авторизація каналів присутності](#authorizing-presence-channels)
  - [Об'єднання каналів присутності](#joining-presence-channels)
  - [Трансляція на канали присутності](#broadcasting-to-presence-channels)
- [Трансляція подій моделі](#model-broadcasting)
  - [Домовленості транслювання подій моделі](#model-broadcasting-conventions)
  - [Прослуховування странслюючих подій моделі](#listening-for-model-broadcasts)
- [Клієнтські події](#client-events)
- [Повідомлення](#notifications)

<a name="introduction"></a>

## Вступ

У багатьох сучасних WEB додатках, WebSockets використовується для реалізації оновлення користувацьких інтерфейсів в реальному часі. Коли якісь дані оновлюються на сервері, зазвичай буде відправленно повідомлення через підключення WebSocket, яке опрацюється клієнтом. WebSockets надають більш ефективну альтернативу постійному опитуванню сервера вашого додатка, на наявності змін в даних, які мають відображатися у вашому інтерфейсі користувача.

Наприклад, ваш додаток здатен експортувати дані користувача у CSV файл та відправити цей файл йому на електронну пошту. Однак, створення цього CSV файла займає декілька хвилин, тож ви оберете створення та відправлення файла CSV на електронну пошту розташувавши [завдання в черзі](queues.md). Щойно буде створено файл CSV, та відправленно його користувачеві, ми можемо скористатись транслюванням події, щоб відправити подію `App\Events\UserDataExported`, яку отримає JavaScript нашого додатка. Як тільки подію беде отримано, ми можемо відобразити повідомлення для користувача про те, що його файл CSV був надісланий йому на електронну пошту, без небхідності оновлювати сторінки.

Щоб допомогти вам у створенні такого функціоналу, Laravel дозволяє легко "трансляцювати" на боці вашого сервера [події](events.md) Laravel за допомогою WebSocket підключення. Транслювання подій Laravel, дозволяє вам спільно використовувати однакові імена подій і даних між додатком Laravel серверної частини, та на боці JavaScript вашого додатка.

Основні концепції транслювання прості: клієнти підключаються до іменованих каналів на боці фронтенду, тоді як ваш додаток Laravel транслює події на ці канали на боці бекенду. Ці події можуть містити будь-які додаткові дані, які б ви хотіли зробити доступними для фронтенду.

<a name="supported-drivers"></a>

#### Підтримувані драйвери

За замовчуванням, Laravel включає два серверних драйвера транслювання на ваш вибір: [Pusher Channels](https://pusher.com/channels) та [Ably](https://ably.io). Однак пакунки спільноти, такі як [laravel-websockets](https://beyondco.de/docs/laravel-websockets/getting-started/introduction) і [soketi](https://docs.soketi.app/), надають додаткові драйвери транслювання, які не вимагають комерційних постачальників транслювання.

> **Note**  
> Перед ніж перейти до транслювання подій, переконайтесь, що ви прочитали документацію Laravel про [події та слухачів](events.md).

<a name="server-side-installation"></a>

## Встановлення на боці сервера

Щоб розпочати використання подій транслювання Laravel, нам потрібно налаштувати додаток Laravel, а також встановити декілька пакетів.

Транслювання подій здійснюється за допомогою драйвера транслювання на боці сервера, який транслює ваші події Laravel, щоб Laravel Echo (бібліотека JavaScript) могла отримати їх в клієнті браузера. Не хвилюйтесь - ми крок за кроком розглянемо кожну частину процесу встановлення.

<a name="configuration"></a>

### Налаштування

Вся конфігурація транслювання подій вашого додатка, зберігається в конфігураційному файлі `config/broadcasting.php`. Laravel з коробки піддримує декілька драйверів транслювання: [канали Pusher](https://pusher.com/channels), [Redis](redis.md), та драйвер `log`, який використовується для локальної розробки та налагодження. Крім того, підтримується драйвер `null`, який дозволяє вам повністю відключити транслювання під час тестування. Конфігураційний приклад присутній для кожного з цих драйверів знаходиться в конфігураційному файлі `config/broadcasting.php` .

<a name="broadcast-service-provider"></a>

#### Постачальник служб транслування

Перед транслюванням будь-яких події, спершу вам потрібно буде зареєструвати `App\Providers\BroadcastServiceProvider`. В новому додатку Laravel, вам потрібно лише розкоментувати цей постачальник служб в масиві `providers` вашого конфігураційного файла `config/app.php`. Цей `BroadcastServiceProvider` містить необхідний код, який реєструє маршрути авторизації транслювання та замикання.

<a name="queue-configuration"></a>

#### Налаштування черги

Вам також потрібно буде налаштувати та запустити [виконавця черги](queues.md). Всі трансляції подій виконуються через завдання в черзі, тож час відповіді вашого додатка не дуже залежить від транслюючих подій.

<a name="pusher-channels"></a>

### Канали Pusher

Якщо ви плануєте транслювати ваші події використовуючи [канали Pusher](https://pusher.com/channels), вам варто встановити канали Pusher PHP SDK використовуючи менеджер пакетів Composer:

```shell
composer require pusher/pusher-php-server
```

Далі, вам потрібно буде налаштувати ваші облікові дані каналів Pusher в конфігураційному файлі `config/broadcasting.php`. Приклад, конфігурації каналів Pusher вже присутній в файлі, що дозволяє вам швидко визначити ваш ключ, таємний ключ, та ID додатка. Як правило, ці значення варто визначати за допомогою [змінних середовища](configuration.md#environment-configuration) `PUSHER_APP_KEY`, `PUSHER_APP_SECRET`, і `PUSHER_APP_ID`

```ini
PUSHER_APP_ID=your-pusher-app-id
PUSHER_APP_KEY=your-pusher-key
PUSHER_APP_SECRET=your-pusher-secret
PUSHER_APP_CLUSTER=mt1
```

Конфігурація `pusher` файла `config/broadcasting.php`, також дозволяє вам визначити додаткові `options`, які підтримуються Каналами, такими як кластер.

Далі, вам потріно змінити ваш драйвер трансляції на `pusher`, у вашому файлі `.env`:

```ini
BROADCAST_DRIVER=pusher
```

Нарешті, ви готові встановити та налаштувати [Laravel Echo](#client-side-installation), який отримуватиме трансляцію подій на боці клієнта.

<a name="pusher-compatible-open-source-alternatives"></a>

#### Альтернативи Pusher з відкритим кодом

Пакунки [laravel-websockets](https://github.com/beyondcode/laravel-websockets) та [soketi](https://docs.soketi.app/) забезпечують сумісні з Pusher сервери WebSocket для Laravel. Ці пакунки дозволяють вам відчути всю силу Laravel транслювання без використання комерційних постачальників WebSocket. Щоб дізнатися більше про встановлення та використання цих пакетів, зверніться до нашої документації [альтернативи відкритого коду](#open-source-alternatives).

<a name="ably"></a>

### Ably

Якщо ви плануєте транслювати ваші події використовуючи [Ably](https://ably.io), вам варто встановити Ably PHP SDK за допомогою менеджера пакунків Composer:

```shell
composer require ably/ably-php
```

Далі, вам варто налаштувати ваші облікові дані Ably в конфігураційному файлі `config/broadcasting.php`. Приклад, конфігурації Ably вже присутній в файлі, що дозволяє вам швидко визначити ваш ключ. Як правило, це значення має бути визначене за допомогою [змінної середовища](configuration.md#environment-configuration) `ABLY_KEY`:

```ini
ABLY_KEY=your-ably-key
```

Далі вам потрібно буде змінити ваш драйвер транслювання на `ably` у вашому файлі `.env`:

```ini
BROADCAST_DRIVER=ably
```

Нарешті, ви готові встановити та налаштувати [Laravel Echo](#client-side-installation), який отримуватиме трансляцію подій на боці клієнта.

<a name="open-source-alternatives"></a>

### Альтернативи відкритого коду

<a name="open-source-alternatives-php"></a>

#### PHP

[Laravel-websockets](https://github.com/beyondcode/laravel-websockets) - це пакунок WebSocket для Laravel на чистому PHP, сумісний з Pusher. Цей пакунок дозволяють вам відчути всю силу Laravel транслювання без використання комерційних постачальників WebSocket. Щоб дізнатися більше про встановлення та використання цих пакунків, зверніться до нашої [офіційної документації](https://beyondco.de/docs/laravel-websockets).

<a name="open-source-alternatives-node"></a>

#### Node

[Soketi](https://github.com/soketi/soketi) - це сервер WebSocket для Laravel, сумісний з Pusher на основі Node. Під капотом Soketi
використовує µWebSockets.js для надзвичайної маштабованості та швидкості. Цей пакунок дозволяють вам відчути всю силу Laravel транслювання без використання комерційних постачальників WebSocket. Щоб дізнатися більше про встановлення та використання цих пакунків, зверніться до нашої [офіційної документації](https://docs.soketi.app/).

<a name="client-side-installation"></a>

## Встановлення на боці клієнта

<a name="client-pusher-channels"></a>

### Канали Pusher

[Laravel Echo](https://github.com/laravel/echo) - це бібліотека JavaScript, яка дозволяє безболісно підписатися на канали та прослуховувати події, які транслюються вашим драйвером транслювання на боці сервера. Ви можете встановити Echo за допомогою менеджера пакунків NPM. В даному прикладі, ми також встановимо пакунок `pusher-js`, оскільки ми будемо використовувати транслятор каналів Pusher:

```shell
npm install --save-dev laravel-echo pusher-js
```

Щойно Echo буде встановлено, ви одразу зможете створити свіжий екземпляр Echo у вашому додатку JavaScript. Чудове місце для цього – у кінцевій частині файлу resources/js/bootstrap.js, який входить до складу Laravel framework. За замовчування, приклад конфігурації Echo вже додано до цього файла, вам лише потріно розкоментувати його:

```js
import Echo from "laravel-echo";
import Pusher from "pusher-js";

window.Pusher = Pusher;

window.Echo = new Echo({
  broadcaster: "pusher",
  key: import.meta.env.VITE_PUSHER_APP_KEY,
  cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
  forceTLS: true,
});
```

Щойно ви розкоментували та налаштували конфігурацію Echo відповідно до ваших потреб, ви можете скомпілювати ресурси вашого додатка:

```shell
npm run dev
```

або

```shell
npm run build
```

> **Note**  
> Щоб дізнатися більше про компіляцію ресурсів вашого JavaScript додатка, будьласка ознайомтесь з документацією [Vite](vite.md).

<a name="using-an-existing-client-instance"></a>

#### Використання існуючого екземпляра клієнта

Якщо ви вже маєте попередньо налаштований екземпляр клієнта каналів Pusher, який ви хочите використовувати для Echo, ви можете передати його в Echo за допомогою параметра конфігурації `client`:

```js
import Echo from "laravel-echo";
import Pusher from "pusher-js";

const options = {
  broadcaster: "pusher",
  key: "your-pusher-channels-key",
};

window.Echo = new Echo({
  ...options,
  client: new Pusher(options.key, options),
});
```

<a name="client-ably"></a>

### Ably

[Laravel Echo](https://github.com/laravel/echo) - це бібліотека JavaScript, яка дозволяє безболісно підписатися на канали та прослуховувати події, які транслюються вашим драйвером транслювання на боці сервера. Ви можете встановити Echo за допомогою менеджера пакунків NPM. В даному прикладі, ми також встановимо пакунок `pusher-js`, оскільки ми будемо використовувати транслятор каналів Pusher:

Ви можете задатися питанням, чому ми встановлюємо бібліотеку JavaScript pusher-js, хоча ми використовуємо Ably для трансляції наших подій. На шастя, Ably містить режим сумісності Pusher, який дозволяє нам використовувати протокол Pusher під час прослуховування подій на боці клієнта нашого додатка:

```shell
npm install --save-dev laravel-echo pusher-js
```

**Перш ніж продовжити, вам слід увімкнути підтримку протоколу Pusher у налаштуваннях вашого додатка Ably. Ви можете ввімкнути цей функціонал в розділі "Protocol Adapter Settings" на інформаційній панелі налаштувань вашого додатка Ably.**

Щойно Echo буде встановлено, ви одразу зможете створити свіжий екземпляр Echo у вашому додатку JavaScript. Чудове місце для цього – у кінцевій частині файлу resources/js/bootstrap.js, який входить до складу Laravel framework. За замовчування, приклад конфігурації Echo вже додано до цього файла, вам лише потріно розкоментувати його; однак конфігурація за замовчуванням у файлі bootstrap.js призначена для Pusher. Ви можете скопіювати наведену нижче конфігурацію, щоб перенести свою конфігурацію на Ably:

```js
import Echo from "laravel-echo";
import Pusher from "pusher-js";

window.Pusher = Pusher;

window.Echo = new Echo({
  broadcaster: "pusher",
  key: import.meta.env.VITE_ABLY_PUBLIC_KEY,
  wsHost: "realtime-pusher.ably.io",
  wsPort: 443,
  disableStats: true,
  encrypted: true,
});
```

Зверніть увагу, що наша конфігурація Ably Echo посилається на змінну середовища `VITE_ABLY_PUBLIC_KEY`. Значенням цієї змінної має бути ваш публічний ключем Ably. Ваш публічний ключ – це частина вашого ключа Ably, яка знаходиться перед символом `:`.

Після того, як ви розкоментували та відредагували конфігурацію Echo відповідно до своїх потреб, ви можете скомпілювати ресурси свого додатка:

```shell
npm run dev
```

або

```shell
npm run build
```

> **Note**  
> Щоб дізнатися більше про компіляцію ресурсів вашого JavaScript додатка, будьласка ознайомтесь з документацією [Vite](vite.md).

<a name="concept-overview"></a>

## Огляд концепції

Трансляція подій Laravel дозволяє вам транслювати ваші серверні події Laravel у ваш клієнський додаток JavaScript, використовуючи підхід до WebSockets на основі драйверів. Наразі Laravel постачається з драйверами [Каналів Pusher](https://pusher.com/channels) і [Ably](https://ably.io). Події можна легко використовувати на боці клієнта за допомогою пакета [Laravel Echo](#client-side-installation) JavaScript.

Події транслюються через «канали», які можуть бути загальнодоступними або приватними. Будь-який відвідувач вашого додатка може підписатися на публічний канал без будь-якої автентифікації чи авторизації; однак, щоб підписатися на приватний канал, користувач має пройти автентифікацію та авторизуватися для прослуховування на цьому каналі.

> **Note**  
> Якщо ви бажаєте розглянути альтернативи Pusher з відкритим кодом, перегляньте [альтернативи відкритого коду](#open-source-alternatives).

<a name="using-example-application"></a>

### Приклад використання

Перш ніж заглибитися в кожен компонент трансляції подій, давайте зробимо короткий огляд на прикладі магазина електронної комерції.

Припустимо, що в нашому додатку є сторінка, яка дозволяє користувачам переглядати статус доставки своїх замовлень. Також припустимо, що подія `OrderShipmentStatusUpdated` викликається, коли додаток опрацьовує оновлення статуса доставки:

```php
  use App\Events\OrderShipmentStatusUpdated;

  OrderShipmentStatusUpdated::dispatch($order);
```

<a name="the-shouldbroadcast-interface"></a>

#### Інтерфейс `ShouldBroadcast`

Коли користувач переглядає одне зі своїх замовлень, ми не хочемо, щоб він оновлював сторінку, щоб переглянути оновлення статуса. Натомість, ми хочемо транслювати оновлення в додаток по мірі їх створення. Отже, нам потрібно позначити подію `OrderShipmentStatusUpdated` інтерфейсом `ShouldBroadcast`. Цей інтерфейс змусить Laravel транслювати подію, коли вона викликається:

```php
  <?php

  namespace App\Events;

  use App\Models\Order;
  use Illuminate\Broadcasting\Channel;
  use Illuminate\Broadcasting\InteractsWithSockets;
  use Illuminate\Broadcasting\PresenceChannel;
  use Illuminate\Broadcasting\PrivateChannel;
  use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
  use Illuminate\Queue\SerializesModels;

  class OrderShipmentStatusUpdated implements ShouldBroadcast
  {
      /**
       * Екземпляр замовлення.
       *
       * @var \App\Order
       */
      public $order;
  }
```

Інтерфейс `ShouldBroadcast` вимагає, щоб наша подія визначала метод `broadcastOn`. Цей метод відповідає за повернення каналів, на яких має транслюватися подія. Порожній приклад цього метода вже визначений в згенерованих класах подій, тому нам потрібно лише заповнити її деталі. Ми хочемо, щоб лише автор замовлення міг переглядати оновлення статуса, тому ми транслюватимемо подію на приватному каналі, який прив’язаний до замовлення:

```php
  /**
   * Отримати канали, на яких має транслюватися подія.
   *
   * @return \Illuminate\Broadcasting\PrivateChannel
   */
  public function broadcastOn()
  {
      return new PrivateChannel('orders.' . $this->order->id);
  }
```

<a name="example-application-authorizing-channels"></a>

#### Авторизація каналів

Пам’ятайте, що користувачі повинні мати дозвіл на те, щоб прослуховувати приватні канали. Ми можемо визначити наші правила авторизації каналів у файлі `routes/channels.php` нашого додатка. У цьому прикладі нам потрібно переконатися у тому, що будь-який користувач, який намагається прослухати приватний канал `orders.1`, насправді є автором замовлення:

```php
use App\Models\Order;

Broadcast::channel('orders.{orderId}', function ($user, $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

Метод `channel` приймає два аргументи: назву канала та зворотний виклик, який повертає `true` або `false`, вказуючи, чи має користувач право слухати цей канал.

Всі зворотні виклики авторизації отримують поточного автентифікованого користувача першим аргументом і будь-які додаткові параметри в якості своїх наступних аргументів. В цьому прикладі, ми використовуємо заповнювач `{orderId}`, щоб вказати, що частина «ID» назви каналу є параметром.

<a name="listening-for-event-broadcasts"></a>

#### Прослуховування трансляцій подій

Далі залишається лише прослухати подію в нашому JavaScript додатку. Ми можемо зробити це за допомогою [Laravel Echo](#client-side-installation). По-перше, ми використаємо `private` метод, щоб підписатися на приватний канал. Тоді ми можемо використати метод `listen` для прослуховування події `OrderShipmentStatusUpdated`. За замовчуванням всі публічні властивості події будуть додані до трансляції подій:

```js
Echo.private(`orders.${orderId}`).listen("OrderShipmentStatusUpdated", (e) => {
  console.log(e.order);
});
```

<a name="defining-broadcast-events"></a>

## Визначення транслюючих подій

Щоб повідомити Laravel про те, що певну подію слід транслювати, ви повинні реалізувати інтерфейс `Illuminate\Contracts\Broadcasting\ShouldBroadcast` в класі події. Цей інтерфейс вже імпортовано у всі класи подій, згенеровані фреймворком, тому ви можете легко додати його до будь-якої зі своїх подій.

Інтерфейс `ShouldBroadcast` вимагає реалізації єдиного метода: `broadcastOn`. Метод `broadcastOn` має повертати канал або масив каналів, на яких має транслюватися подія. Канали мають бути екземплярами `Channel`, `PrivateChannel` або `PresenceChannel`. Екземпляри `Channel` представляють публічні канали, на які може підписатися будь-який користувач, тоді як `PrivateChannels` і `PresenceChannels` представляють приватні канали, які вимагають [авторизації канала](#authorizing-channels):

```php
<?php

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class ServerCreated implements ShouldBroadcast
{
    use SerializesModels;

    /**
     * Користувач, який створив сервер.
     *
     * @var \App\Models\User
     */
    public $user;

    /**
     * Створіть новий екземпляр події.
     *
     * @param  \App\Models\User  $user
     * @return void
     */
    public function __construct(User $user)
    {
        $this->user = $user;
    }

    /**
     * Отримати канали, на яких має транслюватися подія.
     *
     * @return Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('user.' . $this->user->id);
    }
}
```

Після реалізації інтерфейса `ShouldBroadcast` вам потрібно лише [викликати подію](events.md), як зазвичай. Після виклику події, [завдання в черзі](queues.md) автоматично транслюватиме подію за допомогою вказаного вами драйвера трансляції.

<a name="broadcast-name"></a>

### Трансляція імені

За замовчування, Laravel транслюватиме подію, використовуючи назву класа події. Однак, ви можете налаштувати ім'я трансляції визначивши додатковий метода `broadcastAs` в класі події:

```php
  /**
   * Ім'я рансляції події.
   *
   * @return string
   */
  public function broadcastAs()
  {
      return 'server.created';
  }
```

Якщо ви визначеєте власну назву для транслюючої події за допомогою метод `broadcastAs`, вам слід переконатися, що ваш слухач має префікс `.`. Це вкаже Echo не додавати простір імен додатка до події:

```js
.listen('.server.created', function (e) {
    ....
});
```

<a name="broadcast-data"></a>

### Трансляція даних

Коли подія транслюється, всі її `public` властивості автоматично серіалізуються та транслюються як корисне навантаження події, що дозволяє вам отримати доступ до будь-яких загальнодоступних даних із додатка JavaScript. Наприклад, якщо ваша подія має одну публичну властивість `$user`, яке містить модель Eloquent, то корисне навантаження трансляції події буде таким:

```json
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```

Однак, якщо ви бажаєте мати більш точний контроль над корисним навантаженням трансляції, ви можете додати метод `broadcastWith` до своєї події. Цей метод має повертати масив даних, які ви бажаєте транслювати як корисне навантаження події:

```php
/**
 * Get the data to broadcast.
 *
 * @return array
 */
public function broadcastWith()
{
    return ['id' => $this->user->id];
}
```

<a name="broadcast-queue"></a>

### Трансляція черги

За замовчуванням кожна транслююча подія буде занесена до черги та підключення, які визначені у вашому конфігураційному файлі `queue.php`. Ви можете налаштувати власні підключення черги та назву черги, які використовує трансляція, прямо у вашому класі подій, визначивши властивості `connection` та `queue`:

```php
/**
 * Ім'я підключення черги, яке буде використовуватися під час трансляції події.
 *
 * @var string
 */
public $connection = 'redis';

/**
 * Ім'я черги, до якої потрібно розмістити завдання трансляції.
 *
 * @var string
 */
public $queue = 'default';
```

Крім того, ви можете налаштувати власне им'я черги, визначивши метод `broadcastQueue` у вашій події:

```php
/**
 * Ім'я черги, до якої потрібно розмістити завдання трансляції.
 *
 * @return string
 */
public function broadcastQueue()
{
    return 'default';
}
```

Якщо ви хочете транслювати свою подію за допомогою черги `sync` замість драйвера черги за замовчуванням, ви можете застосувати інтерфейс `ShouldBroadcastNow` замість `ShouldBroadcast`:

```php
<?php

use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;

class OrderShipmentStatusUpdated implements ShouldBroadcastNow
{
    //
}
```

<a name="broadcast-conditions"></a>

### Трансляція через умову

Іноді вам потрібно транслювати вашу поді тільки, якщо визначена умова повертає `true`. Ви можете виначити ці умови, додавши метод `broadcastWhen` до вашого класу події:

```php
/**
 * Визначте, чи слід транслювати цю подію.
 *
 * @return bool
 */
public function broadcastWhen()
{
    return $this->order->value > 100;
}
```

<a name="broadcasting-and-database-transactions"></a>

#### Транслювання & транзакція баз даних

Коли транлюючі події надсилаються в рамках транзакцій бази даних, вони можуть бути опрацьовані чергою до того, як транзакція бази даних буде зафіксована. Коли це станеться, будь-які оновлення, які ви внесли в моделі або записи бази даних під час транзакції бази даних, можуть ще не відображатися в базі даних. Крім того, будь-які моделі або записи бази даних, створені в рамках транзакції, можуть не існувати в базі даних. Якщо ваша подія залежить від цих моделей, під час опрацювання завдання, яке транслює подію, можуть виникнути неочікувані помилки.

Якщо для параметра конфігурації `after_commit` підключення черги, встановлено значення `false`, ви все ще можете вказати, що конкретна транслююча подія має бути відправлена ​​після того, як всі транзакції відкритої бази даних були зафіксовані, визначивши властивість `$afterCommit` в класі події:

```php
<?php

namespace App\Events;

use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class ServerCreated implements ShouldBroadcast
{
    use SerializesModels;

    public $afterCommit = true;
}
```

> **Note**  
> Щоб дізнатися більше про вирішення цих проблем, перегляньте документацію щодо [завдань в черзі та транзакцій бази даних](queues.md#jobs-and-database-transactions).

<a name="authorizing-channels"></a>

## Авторизація каналів

Приватні канали вимагають від вас авторизації, щоб поточний автентифікований користувач міг фактично слухати канал. Це досягається шляхом надсилання HTTP-запита вашому додатку Laravel за назвою канала, що дозволить вашому додатку визначити, чи може користувач слухати цей канал. Під час використання [Laravel Echo](#client-side-installation), HTTP-запит для авторизації підписки на приватні канали буде зроблено автоматично; однак, вам потрібно визначити відповідні маршрути для відповіді на ці запити.

<a name="defining-authorization-routes"></a>

### Визначення маршрутів аторизації

Laravel дозволяє легко визначати маршрути для відповіді на запити авторизації канала. У вашому додатку Laravel включений `App\Providers\BroadcastServiceProvider`, в якому ви побачите звернення до метода `Broadcast::routes`. Цей метод реєструє маршрут `/broadcasting/auth` для опрацювання запита на авторизацю:

```php
Broadcast::routes();
```

Метод `Broadcast::routes` автоматично розмістить свої маршрути в групі посередника `web`; однак, ви можете передати масив атрибутів маршрута до метода, якщо бажаєте змінити їхні призначені атрибути:

```php
Broadcast::routes($attributes);
```

<a name="customizing-the-authorization-endpoint"></a>

#### Налаштування кінцевої точки авторизації

За замовчуванням Echo використовуватиме кінцеву точку `/broadcasting/auth` для авторизації доступу до канала. Однак ви можете вказати власну кінцеву точку авторизації, передавши параметр конфігурації `authEndpoint` своєму екземпляру Echo:

```js
window.Echo = new Echo({
  broadcaster: "pusher",
  // ...
  authEndpoint: "/custom/endpoint/auth",
});
```

<a name="customizing-the-authorization-request"></a>

#### Налаштування запита вторизації

Ви можете налаштувати, як Laravel Echo виконує запити авторизації, надавши спеціальний авторизатор під час ініціалізації Echo:

```js
window.Echo = new Echo({
  // ...
  authorizer: (channel, options) => {
    return {
      authorize: (socketId, callback) => {
        axios
          .post("/api/broadcasting/auth", {
            socket_id: socketId,
            channel_name: channel.name,
          })
          .then((response) => {
            callback(null, response.data);
          })
          .catch((error) => {
            callback(error);
          });
      },
    };
  },
});
```

<a name="defining-authorization-callbacks"></a>

### Визначення авторизації канала

Далі нам потрібно визначити логіку, яка фактично визначатиме, чи може поточний автентифікований користувач слухати певний канал. Це вже зроблено у файлі `routes/channels.php`, який вже присутній у вашому додатку. В цьому файлі, ви можете використати метод `Broadcast::channel`, для реєестрації замикань авторизації каналів:

```php
Broadcast::channel('orders.{orderId}', function ($user, $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

Метод `channel` приймє два аргумента: назву канала, та замикання, яке повертає `true` або `false` вказуючи, чи має право користувач слухати цей канал.

Всі авторизаційні замикання отримують поточного автентифікаційного користувача своїм першим аргументом, та будь-які додаткові параметри як свої наступні аргументи. В цьому прикладі, ми використовуємо заповнювач `{orderId}` щоб позначити, що частина "ID" імені канала є параметром

<a name="authorization-callback-model-binding"></a>

#### Прив'язка моделі до авторизації

Подібно до маршрутів HTTP, маршрути каналів також можуть використовувати переваги неявного та явного [зв’язування моделі маршрута](routing.md#route-model-binding). Наприклад, замість отримання рядка або числового ідентифікатора замовлення ви можете запросити фактичний екземпляр моделі `Order`:

```php
use App\Models\Order;

Broadcast::channel('orders.{order}', function ($user, Order $order) {
    return $user->id === $order->user_id;
});
```

> **Warning**  
> На відміну від прив'язки моделі до HTTP-маршруту, прив'язка моделі до канала не підтримує [обмеження неявної прив'язки моделі](routing.md#implicit-model-binding-scoping). Однак це рідко є проблемою, тому що більшість каналів можна обмежити на основі унікального первинного ключа однієї моделі.

<a name="authorization-callback-authentication"></a>

#### Попередня авторизація автентифікація замикання

Приватні канали та канали присутності аутентифікують поточного користувача через стандартного охоронця автентифікації вашого додатка. Якщо користувач не автентифікований, авторизація канала автоматично відхиляється, а замикання авторизації ніколи не виконується. Однак, ви можете призначити декілька охоронців, які мають автентифікувати вхідний запит, якщо необхідно:

```php
Broadcast::channel('channel', function () {
    // ...
}, ['guards' => ['web', 'admin']]);
```

<a name="defining-channel-classes"></a>

### Визначення каласа канала

Якщо ваш додаток використовує багато різних каналів, ваш файл `routes/channels.php` може стати дуже об’ємним. Отже, замість використання замикань для авторизації каналів, ви можете використовувати класи каналів. Щоб створити клас канала, скористайтеся командою Artisan `make:channel`. Ця команда розмістить новий клас канала в каталозі `App/Broadcasting`.

```shell
php artisan make:channel OrderChannel
```

Далі, зареєструйте ваш канал у файлі `routes/channels.php`:

```php
use App\Broadcasting\OrderChannel;

Broadcast::channel('orders.{order}', OrderChannel::class);
```

Нарешті, ви можете розташувати авторизаційну логіку для вашого канала в методі `join` класа канала. Цей метод `join`, міститеме однакову логіку, яку ви зазвичай розташовували у вашому замиканні канала авторизації. Ви також можете скористатися прив’язкою моделі до канала:

```php
<?php

namespace App\Broadcasting;

use App\Models\Order;
use App\Models\User;

class OrderChannel
{
    /**
     * Створити новий екземпляр канала.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Підтвердити доступ користувача до каналу.
     *
     * @param  \App\Models\User  $user
     * @param  \App\Models\Order  $order
     * @return array|bool
     */
    public function join(User $user, Order $order)
    {
        return $user->id === $order->user_id;
    }
}
```

> **Note**  
> Як і більшість інших класів в Laravel, класи каналів автоматично впроваджуються [контейнером служби](container.md). Таким чином, в конструкторі, ви можете оголосити будь-які типізації залежності, необхідні вашому каналу.

<a name="broadcasting-events"></a>

## Транслювання подій

Після того, як ви визначили подію та позначили її за допомогою інтерфейса `ShouldBroadcast`, вам потрібно лише викликати подію за допомогою метода `dispatch` події. Виконавець подій помітить, що подія позначена інтерфейсом `ShouldBroadcast`, і поставить подію в чергу для її подальшої трансляції:

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```

<a name="only-to-others"></a>

### Трансляція подій лише іншим користувачам

Під час створення додатка, який використовує трансляцію подій, вам час від часу може знадобитись транслювати подію всім підписникам певного канала, за винятком поточного користувача. Ви можете зробити це за допомогою помічника `broadcast` та метода `toOthers`:

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->toOthers();
```

Щоб краще зрозуміти, коли ви можете використовувати метод `toOthers`, давайте уявімо додаток зі списком завдань, де користувач може створити нове завдання, ввівши назву завдання. Щоб створити завдання, ваш додаток може зробити запит до URL-адреси `/task`, яка транслює створення завдання та повертає JSON-представлення нового завдання. Коли ваш додаток JavaScript отримує відповідь від кінцевої точки, вона може безпосередньо вставити нове завдання у свій список завдань таким чином:

```js
axios.post("/task", task).then((response) => {
  this.tasks.push(response.data);
});
```

Однак пам’ятайте, що ми також транслюємо створення завдання. Якщо ваш додаток JavaScript також прослуховує цю подію, щоб додати завдання до списку завдань, у вашому списку будуть однакові завдання: один від кінцевої точки і один від трансляції. Ви можете вирішити це за допомогою метода toOthers, щоб вказати транслятору не транслювати подію поточному користувачеві.

> **Warning**  
> Щоб викликати метод `toOthers`, ваша подія має використовувати трейт `Illuminate\Broadcasting\InteractsWithSockets`.

<a name="only-to-others-configuration"></a>

#### Конфігурація при використанні метода `toOthers`

Коли ви ініціалізуєте екземпляр Laravel Echo, підключенню призначається ID сокета. Якщо ви використовуєте глобальний екземпляр [Axios](https://github.com/mzabriskie/axios) для відправлення HTTP-запитів із додатка JavaScript, ID сокета автоматично додаватиметься до кожного вихідного запита як заголовок `X-Socket-ID`. Потім, коли ви викликатимете метод `toOthers`, Laravel вилучить ID сокета з заголовка та вкаже транслятору не транслювати жодні підключення з цим ID сокета.

Якщо ви не використовуєте глобальний екземпляр Axios, вам потрібно буде вручну налаштувати додаток JavaScript для відправлення заголовка `X-Socket-ID` з усіма вихідними запитами. Ви можете отримати ID сокета за допомогою метода `Echo.socketId`:

```js
var socketId = Echo.socketId();
```

<a name="customizing-the-connection"></a>

### Налаштування підключення

Якщо ваш додаток взаємодіє з декількома транслюючими підключеннями, і ви хочете транслювати подію за допомогою транслятора, який відрізняється від стандартного, ви можете вказати, до якого підключення відправляти подію за допомогою метод `via`:

```php
  use App\Events\OrderShipmentStatusUpdated;

  broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');
```

Крім того, ви можете вказати трансляцію підключення події, викликавши метод `broadcastVia` в конструкторі події. Однак перед тим, як це зробити, ви повинні переконатися, що клас події використовує трейт `InteractsWithBroadcasting`:

```php
  <?php

  namespace App\Events;

  use Illuminate\Broadcasting\Channel;
  use Illuminate\Broadcasting\InteractsWithBroadcasting;
  use Illuminate\Broadcasting\InteractsWithSockets;
  use Illuminate\Broadcasting\PresenceChannel;
  use Illuminate\Broadcasting\PrivateChannel;
  use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
  use Illuminate\Queue\SerializesModels;

  class OrderShipmentStatusUpdated implements ShouldBroadcast
  {
      use InteractsWithBroadcasting;

      /**
       * Створити новий екземпляр події.
       *
       * @return void
       */
      public function __construct()
      {
          $this->broadcastVia('pusher');
      }
  }
```

<a name="receiving-broadcasts"></a>

## Отримання трансляцій

<a name="listening-for-events"></a>

### Прослуховування подій

Після того як ви [встановили та створили екземпляр Laravel Echo](#client-side-installation), ви готові почати прослуховування подій, які транслюються з вашого додатка Laravel. Спочатку використовуйте метод `channel`, щоб отримати екземпляр канала, а потім викличте метод `listen`, щоб прослухати зазначену подію:

```js
Echo.channel(`orders.${this.order.id}`).listen(
  "OrderShipmentStatusUpdated",
  (e) => {
    console.log(e.order.name);
  }
);
```

Якщо ви бажаєте прослуховувати події на приватному каналі, натомість використовуйте метод `private`. Ви можете продовжувати об’єднувати виклики метода `listen` для прослуховування декількох подій на одному каналі:

```js
Echo.private(`orders.${this.order.id}`)
  .listen(/* ... */)
  .listen(/* ... */)
  .listen(/* ... */);
```

<a name="stop-listening-for-events"></a>

#### Припинення прослуховування подій

Якщо ви хочете припинити прослуховування певної події, [не залишаючи канал](#leaving-a-channel), ви можете скористатися методом `stopListening`:

```js
Echo.private(`orders.${this.order.id}`).stopListening(
  "OrderShipmentStatusUpdated"
);
```

<a name="leaving-a-channel"></a>

### Залишення канала

Щоб залишити канал, ви можете викликати метод `leaveChannel` вашого екземпляра:

```js
Echo.leaveChannel(`orders.${this.order.id}`);
```

Якщо ви хочете залишити канал, а також пов’язані з ним приватні канали та канали присутності, ви можете викликати метод `leave`:

```js
Echo.leave(`orders.${this.order.id}`);
```

<a name="namespaces"></a>

### Простір імен

Можливо, ви помітили в наведених вище прикладах, що ми не вказали повний простір імен `App\Events` для класів подій. Це пояснюється тим, що Echo автоматично вважатиме, що події розташовані в просторі імен `App\Events`. Однак ви можете налаштувати кореневий простір імен під час створення екземпляра Echo, передавши параметр конфігурації `namespace`:

```js
window.Echo = new Echo({
  broadcaster: "pusher",
  // ...
  namespace: "App.Other.Namespace",
});
```

Крім того, ви можете додати до класів подій префікс `.` під час підписки на них за допомогою Echo. Це дозволить вам завжди вказувати повне ім’я класа:

```js
Echo.channel("orders").listen(".Namespace\\Event\\Class", (e) => {
  //
});
```

<a name="presence-channels"></a>

## Канали присутності

Канали присутності базуються на безпеці приватних каналів, водночас надаючи додаткову функцію усвідомлення того, хто підписаний на канал. Це спрощує створення потужних функцій додатка для спільної роботи, таких як сповіщення користувачів, коли інший користувач переглядає ту саму сторінку або список мешканців кімнати чата.

<a name="authorizing-presence-channels"></a>

### Авторизація каналів присутності

Всі канали присутності також є приватними; тому користувачі повинні [мати доступ до них](#authorizing-channels). Однак, під час визначення зворотних викликів авторизації для каналів присутності ви неповинні повертати значення `true`, якщо користувач авторизований для приєднання до канала. Замість цього ви повинні повернути масив даних про користувача.

Дані, повернуті зворотним викликом авторизації, будуть доступні для прослуховування подій канала присутності у вашому додатку JavaScript. Якщо користувач не авторизований для приєднання до канала присутності, ви повинні повернути `false` або `null`:

```php
Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```

<a name="joining-presence-channels"></a>

### Об'єднання каналів присутності

Щоб приєднатися до канала присутності, ви можете скористатися методом `join` Echo. Метод `join` поверне реалізацію `PresenceChannel`, яка разом з замиканням метода `listen` дозволяє вам підписуватися на події `here`, `joining`, та `leaving`.

```js
Echo.join(`chat.${roomId}`)
  .here((users) => {
    //
  })
  .joining((user) => {
    console.log(user.name);
  })
  .leaving((user) => {
    console.log(user.name);
  })
  .error((error) => {
    console.error(error);
  });
```

Замикання `here` буде виконано негайно після успішного приєднання до канала, також отримає масив, який містить інформацію про всіх інших користувачів, які зараз підписані на канал. Метод `joining` буде виконано, коли новий користувач приєднається до канала, тоді як метод `leaving` буде виконано, коли користувач покине канал. Метод `error` буде виконано, коли кінцева точка автентифікації поверне код статуса HTTP, який відрізняється від 200, або якщо виникне проблема з синтаксичним аналізом повернутого JSON.

<a name="broadcasting-to-presence-channels"></a>

### Трансляція на канали присутності

Канали присутності можуть отримувати події так само, як публічні чи приватні канали. Використовуючи приклад чату, ми можемо захотіти транслювати події `NewMessage` на канал присутності кімнати. Для цього ми повернемо екземпляр `PresenceChannel` з методf `broadcastOn` події:

```php
/**
 * Отримати канали, на яких має транслюватися подія.
 *
 * @return Channel|array
 */
public function broadcastOn()
{
    return new PresenceChannel('room.'.$this->message->room_id);
}
```

Як і з іншими подіями, ви можете використовувати помічник `broadcast` та метод `toOthers`, щоб виключити поточного користувача з отримання трансляції:

```php
broadcast(new NewMessage($message));

broadcast(new NewMessage($message))->toOthers();
```

Як і для інших типів подій, ви можете прослуховувати події, надіслані на канали присутності, використовуючи метод `listen` Echo:

```js
Echo.join(`chat.${roomId}`)
  .here(/* ... */)
  .joining(/* ... */)
  .leaving(/* ... */)
  .listen("NewMessage", (e) => {
    //
  });
```

<a name="model-broadcasting"></a>

## Трансляція подій моделі

> **Warning**  
> Перш ніж читати наведену нижче документацію про трансляцію подій моделі, ми рекомендуємо вам ознайомитись з загальними поняттями служб трансляції подій моделі Laravel, а також із тим, як вручну створювати та прослуховувати події трансляції.

Зазвичай події транслються, коли [моделі Eloquent](eloquent.md) вашого додатка створюються, оновлюються або видаляються. Звичайно, це можна легко зробити, за допомогою ручного [визначення власної події для змін стану моделі Eloquent](eloquent.md#events) і позначивши ці події за допомогою інтерфейса `ShouldBroadcast`.

Однак, якщо ви не використовуєте ці події для будь-яких інших цілей у своєму додатку, створення класів подій з єдиною метою їх трансляції може стати обтяжливим. Щоб виправити це, Laravel дозволяє вам вказати, що модель Eloquent повинна автоматично транслювати зміни свого стану.

Для початку, ваша модель Eloquent має використовувати трейт `Illuminate\Database\Eloquent\BroadcastsEvents`. Крім того, модель повинна визначити метод `broadcastOn`, який повертатиме масив каналів, на яких мають транслюватися події моделі:

```php
<?php

namespace App\Models;

use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Database\Eloquent\BroadcastsEvents;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use BroadcastsEvents, HasFactory;

    /**
     * Отримати автора посту.
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Отримайте канали, на яких мають транслюватися події моделі.
     *
     * @param  string  $event
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn($event)
    {
        return [$this, $this->user];
    }
}
```

Щойно ваша модель включає цей трейт і визначає канали трансляції, вона почне автоматично транслювати події, коли екземпляр моделі створюється, оновлюється, видаляється, переміщується в кошик або відновлюється.

Крім того, ви могли помітити, що метод `broadcastOn` отримує рядковий аргумент `$event`. Цей аргумент містить тип події, яка сталася з моделлю, і матиме значення `created`, `updated`, `deleted`, `trashed`, або `restored`. Перевіряючи значення цієї змінної, ви можете визначити, на які канали (якщо такі є) модель має транслювати конкретну подію:

```php
/**
 * Отримайте канали, на яких мають транслюватися події моделі.
 *
 * @param  string  $event
 * @return \Illuminate\Broadcasting\Channel|array
 */
public function broadcastOn($event)
{
    return match ($event) {
        'deleted' => [],
        default => [$this, $this->user],
    };
}
```

<a name="customizing-model-broadcasting-event-creation"></a>

#### Налаштування створення транслюючої події моделі

Іноді ви можете захотіти налаштувати те, як Laravel створює базову подію трансляції моделі. Ви можете досягти цього, визначивши метод `newBroadcastableEvent` у вашій моделі Eloquent. Цей метод має повертати екземпляр `Illuminate\Database\Eloquent\BroadcastableModelEventOccurred`:

```php
use Illuminate\Database\Eloquent\BroadcastableModelEventOccurred;

/**
 * Створити нову транслюючу подію моделі.
 *
 * @param  string  $event
 * @return \Illuminate\Database\Eloquent\BroadcastableModelEventOccurred
 */
protected function newBroadcastableEvent($event)
{
    return (new BroadcastableModelEventOccurred(
        $this, $event
    ))->dontBroadcastToCurrentUser();
}
```

<a name="model-broadcasting-conventions"></a>

### Домовленості транслювання подій моделі

<a name="model-broadcasting-channel-conventions"></a>

#### Домовленості про канали

Як ви могли помітити, метод `broadcastOn` у наведеному вище прикладі моделі не повернув екземпляри Channel. Натомість, безпосередньо були повернуті моделі Eloquent. Якщо екземпляр моделі Eloquent повертається з метода `broadcastOn` вашої моделі (або міститься в масиві, повернутому цим методом), Laravel автоматично створить екземпляр приватного канала для моделі, використовуючи назву класа моделі та ідентифікатор первинного ключа як назву канала.

Отже, модель `App\Models\User` з `id` - `1` буде перетворено в екземпляр `Illuminate\Broadcasting\PrivateChannel` з назвою `App.Models.User.1`. Звичайно, окрім повернення екземплярів моделі Eloquent з метода `broadcastOn` вашої моделі, ви можете повернути готові екземпляри `Channel`, щоб мати повний контроль над іменами каналів моделі:

```php
use Illuminate\Broadcasting\PrivateChannel;

/**
 * Отримайте канали, на яких мають транслюватися події моделі.
 *
 * @param  string  $event
 * @return \Illuminate\Broadcasting\Channel|array
 */
public function broadcastOn($event)
{
    return [new PrivateChannel('user.' . $this->id)];
}
```

Якщо ви плануєте явно повернути екземпляр канала з метода `broadcastOn` вашої моделі, ви можете передати екземпляр моделі Eloquent конструктору канала. Зробивши це, Laravel використовуватиме угоди про канали моделі, про які говорилося вище, щоб конвертувати модель Eloquent у рядок назви канала:

```php
return [new Channel($this->user)];
```

Якщо вам потрібно визначити назву канала моделі, ви можете викликати метод `broadcastChannel` у будь-якому екземплярі моделі.
Наприклад, цей метод повертає рядок `App.Models.User.1` для моделі `App\Models\User` з `id` - `1`:

```php
$user->broadcastChannel()
```

<a name="model-broadcasting-event-conventions"></a>

#### Домовленості подій

Оскільки трансляція подій моделі не пов’язана з «фактичною» подією в каталозі `App\Events` вашого додатка, їм призначається ім’я та корисне навантаження на основі домовленосте. Домовленість Laravel передбачає трансляцію події за допомогою назви класа моделі (не включаючи простір імен) і назви події моделі, яка ініціювала трансляцію.

Так, наприклад, оновлення моделі `App\Models\Post` транслюватиме подію до вашого клієнтського додаткаяка як `PostUpdated` з наступним корисним навантаженням:

```json
{
    "model": {
        "id": 1,
        "title": "My first post"
        ...
    },
    ...
    "socket": "someSocketId",
}
```

Видалення моделі `App\Models\User` передасть подію під назвою `UserDeleted`.

Якщо ви бажаєте, ви можете визначити спеціальну назву трансляції та корисне навантаження, додавши до своєї моделі методи `broadcastAs` і `broadcastWith`. Ці методи отримують назву події / операції моделі, яка відбувається, що дозволяє налаштувати ім’я події та корисне навантаження для кожної операції моделі. Якщо метод `broadcastAs` повертає значення `null`, під час трансляції події, Laravel використовуватиме домовленості про назви подій трансляції моделі, розглянуті вище:

```php
/**
 * Отримати ім'я транслюючої події моделі.
 *
 * @param  string  $event
 * @return string|null
 */
public function broadcastAs($event)
{
    return match ($event) {
        'created' => 'post.created',
        default => null,
    };
}

/**
 * Отримати дані транслюючої події моделі.
 *
 * @param  string  $event
 * @return array
 */
public function broadcastWith($event)
{
    return match ($event) {
        'created' => ['title' => $this->title],
        default => ['model' => $this],
    };
}
```

<a name="listening-for-model-broadcasts"></a>

### Прослуховування транслюючих подій моделі

Після того, як ви додали трейт `BroadcastsEvents` до своєї моделі та визначили метод `broadcastOn` вашої моделі, ви готові розпочати прослуховування транслюючих подій моделі у своєму додатку на боці клієнта. Перш ніж почати, ви можете ознайомитися з повною документацією про [прослуховування подій](#listening-for-events).

Спочатку використовуйте `private` метод, щоб отримати екземпляр канала, а потім викличте метод `listen`, щоб прослухати визначену подію. Як правило, ім’я канала, надане `private` методом, має відповідати стандартам [домовленостей про трансляцію подій моделі](#model-broadcasting-conventions) Laravel.

Отримавши екземпляр канала, ви можете використовувати метод `listen` для прослуховування певної події. Оскільки моделі транслюючих подій не пов’язані з «фактичною» подією в каталозі `App\Events` вашого додатка, [ім’я події](#model-broadcasting-event-conventions) повинно мати префікс `.` щоб вказати, що він не належить до певного простору імен. Кожна транслююча подія моделі має властивість `model`, яке містить всі транслювані властивості моделі:

```js
Echo.private(`App.Models.User.${this.user.id}`).listen(".PostUpdated", (e) => {
  console.log(e.model);
});
```

<a name="client-events"></a>

## Клієнтські події

> **Note**  
> Використовуючи [канали Pusher](https://pusher.com/channels), ви повинні увімкнути опцію "Client Events" в розділі "App Settings" [на інформаційній панелі додатка](https://dashboard.pusher.com/), щоб надсилати клієнтські події.

Іноді вам може знадобитися транслювати подію іншим підключеним клієнтам, не зачіпаючи додаток Laravel взагалі. Це може бути особливо корисним для таких речей, як «введення» повідомлень, коли ви хочете попередити користувачів вашого додатка про те, що інший користувач вводить повідомлення на певному екрані.

Щоб транслювати події клієнта, ви можете використовувати метод `whisper` Echo:

```js
Echo.private(`chat.${roomId}`).whisper("typing", {
  name: this.user.name,
});
```

Щоб прослухати клієнтські події, ви можете використовувати метод `listenForWhisper`:

```js
Echo.private(`chat.${roomId}`).listenForWhisper("typing", (e) => {
  console.log(e.name);
});
```

<a name="notifications"></a>

## Повідомлення

Якщо поєднати транслюючу події з [повідомленням](notifications.md), ваш додаток JavaScript може отримувати нові повідомлення, коли вони виникають, без необхідності оновлювати сторінку. Перш ніж почати, обов’язково прочитайте документацію щодо використання [каналів транслюючих повідомлень](notifications.md#broadcast-notifications).

Після того, як ви налаштували повідомлення для використання транслюючого канала, ви можете прослухати трансляцію подій за допомогою метода `notification` Echo. Пам’ятайте, що ім’я канала має відповідати імені класа об’єкта, який отримує повідомлення:

```js
Echo.private(`App.Models.User.${userId}`).notification((notification) => {
  console.log(notification.type);
});
```

У цьому прикладі всі повідомлення, відправлені екземплярам `App\Models\User` через `broadcast канал, будуть отримані в замиканні. Авторизацію канала `App.Models.User.{id}`вже включено в`BroadcastServiceProvider` за замовчуванням, який постачається з фреймворком Laravel.
