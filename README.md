## Документація Laravel 9.x

Вільний переклад документації репозиторія [**laravel/docs**](https://github.com/laravel/docs/tree/9.x) гілки 9.x на українську мову. Орфографічні, пунктуаційні та суворі смислові помилки виправляються по мірі їх виявлення.

<a name="navigation"></a>

### Зміст документації

> Перед читанням документації ознайомтесь з [угодою перекладу]().

Розділи, які помічені галочкою, вже мають повні актуалізовані **переклади на українську мову**.

- #### Передмова
  - [ ] [Примітки по релізу](./docs/releases.md)
  - [ ] [Керівництво по оновленню](./docs/upgrade.md)
  - [ ] [Рекомендації по участі](./docs/contributions.md)
- #### Початок
  - [ ] [Встановлення](./docs/installation.md)
  - [ ] [Конфігурування](./docs/configuration.md)
  - [ ] [Структура каталогів](./docs/structure.md)
  - [ ] [Стартові комплекти](./docs/starter-kits.md)
  - [ ] [Розгортання](./docs/deployment.md)
- #### Архітектурні концепції
  - [x] [Життевий цикл запитів](./docs/lifecycle.md)
  - [x] [Контейнер служб](./docs/container.md)
  - [x] [Постачальник служб](./docs/providers.md)
  - [ ] [Фасади](./docs/facades.md)
- #### Основи
  - [x] [Маршрутизація](./docs/routing.md)
  - [x] [Посередники](./docs/middleware.md)
  - [ ] [Попередження атак CSRF](./docs/csrf.md)
  - [x] [Контроллери](./docs/controllers.md)
  - [x] [HTTP-запити](./docs/requests.md)
  - [x] [HTTP-відповіді](./docs/responses.md)
  - [ ] [HTML-шаблони](./docs/views.md)
  - [ ] [Шаблонізатор Blade](./docs/blade.md)
  - [x] [Генерація URL-адрес](./docs/urls.md)
  - [x] [Сесія HTTP](./docs/session.md)
  - [x] [Валідація](./docs/validation.md)
  - [ ] [Обробка помилок](./docs/errors.md)
  - [ ] [Логування](./docs/logging.md)
- #### Поглиблення
  - [x] [Консоль Artisan](./docs/artisan.md)
  - [x] [Транслювання](./docs/broadcasting.md)
  - [x] [Кеш додатка](./docs/cache.md)
  - [ ] [Колекції](./docs/collections.md)
  - [ ] [Компіляція веб-активів за допомогою Mix](./docs/mix.md)
  - [ ] [Кнтракти](./docs/contracts.md)
  - [x] [Події](./docs/events.md)
  - [x] [Файлове сховище](./docs/filesystem.md)
  - [ ] [Глобальні помічники](./docs/helpers.md)
  - [x] [HTTP-клієнт](./docs/http-client.md)
  - [x] [Локалізація інтерфейса](./docs/localization.md)
  - [x] [Поштові відправлення](./docs/mail.md)
  - [x] [Повідомлення](./docs/notifications.md)
  - [ ] [Розробка пакетів](./docs/packages.md)
  - [x] [Черги](./docs/queues.md)
  - [ ] [Обмеження частоти](./docs/rate-limiting.md)
  - [x] [Планування завдань](./docs/scheduling.md)
- #### Безпека
  - [x] [Автентифікація](./docs/authentication.md)
  - [x] [Авторизація](./docs/authorization.md)
  - [x] [Підтвердження електронної пошти](./docs/verification.md)
  - [ ] [Шифрування](./docs/encryption.md)
  - [ ] [Хешування](./docs/hashing.md)
  - [x] [Скидання пароля](./docs/passwords.md)
- #### База данных
  - [x] [Початок роботи](./docs/database.md)
  - [x] [Конструктор запитів](./docs/queries.md)
  - [ ] [Пагінація](./docs/pagination.md)
  - [ ] [Міграції](./docs/migrations.md)
  - [ ] [Наповнення фіктивними даними](./docs/seeding.md)
  - [ ] [Використання Redis](./docs/redis.md)
- #### Eloquent ORM
  - [x] [Початок роботи](./docs/eloquent.md)
  - [x] [Відношення](./docs/eloquent-relationships.md)
  - [ ] [Колекції](./docs/eloquent-collections.md)
  - [ ] [Мутатори та типізатори](./docs/eloquent-mutators.md)
  - [x] [Ресурси API](./docs/eloquent-resources.md)
  - [ ] [Сериалізація](./docs/eloquent-serialization.md)
- #### Тестування
  - [ ] [Початок роботи](./docs/testing.md)
  - [ ] [Тести HTTP](./docs/http-tests.md)
  - [ ] [Тести консольних команд](./docs/console-tests.md)
  - [ ] [Браузерні тести](./docs/dusk.md)
  - [ ] [База даних](./docs/database-testing.md)
  - [ ] [Імітація](./docs/mocking.md)
- #### Пакети
  - [x] [_Breeze_](./docs/starter-kits.md#laravel-breeze) – легка реалізація автентифікації Laravel для ознайомлення з функціоналом. Включає в себе прості шаблони Blade, стилізовані за допомогою Tailwind CSS. Містить маршрути для публікації.
  - [x] [_Sanctum_](./docs/sanctum.md) – легка система аутентифікації для SPA (односторінкових додатків), мобільних додатків і простих API на основі токенів. Управління токенами API, автентифікація сесії. Не містить жодних шаблонів. Використовується в Laravel Jetstream.
  - [x] [_Passport_](./docs/passport.md) – реалізація сервера OAuth2 для вашого додатка Laravel на основі [League OAuth2](https://github.com/thephpleague/oauth2-server).
  - [x] [_Cashier (Stripe)_](./docs/cashier-stripe.md) – система оплати, яка працює зі службою (Stripe). Вона надає виразний та гнучкий інтерфейс, для послуги виставлення рахунків за підписку [Stripe](https://stripe.com/). Також опрацьовує майже весь стандартний платіжний код передплати, який вам страшно писати. На додаток до базового керування підписок, Cashier може керувати купонами, обмінювати підписки, «вартість» підписок, пільгові періоди скасування та навіть створювати PDF-файли рахунків.

<a name="license"></a>

### Ліцензія

Посилання на офіційну [ліцензію](https://github.com/laravel/docs/blob/9.x/license.md) документації **laravel/docs**.

Поточний переклад документації розповсюджується по ліцензії [MIT](LICENSE).
