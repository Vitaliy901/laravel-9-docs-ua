# Laravel 9 · Валідація

- [Вступ](#introduction)
- [Швидкий старт](#validation-quickstart)
    - [Визначення маршрутів](#quick-defining-the-routes)
    - [Створення контроллера](#quick-creating-the-controller)
    - [Написання логіки валідації](#quick-writing-the-validation-logic)
    - [Відображення помилок валідації](#quick-displaying-the-validation-errors)
    - [Повторне заповнення форми](#repopulating-forms)
    - [Примітка не обов'язкових полів](#a-note-on-optional-fields)
    - [Формат відповіді валідаційних помилок](#validation-error-response-format)
- [Валідація запита форми](#form-request-validation)
    - [Створення запита форми](#creating-form-requests)
    - [Авторизація запитів](#authorizing-form-requests)
    - [Корегування повідомлень про помилки](#customizing-the-error-messages)
    - [Підготовка вхідних данних для валідації](#preparing-input-for-validation)
- [Створення валідатора на вимогу](#manually-creating-validators)
    - [Автоматичне перенаправлення](#automatic-redirection)
    - [Іменування колекції помилок](#named-error-bags)
    - [Корегування повідомлень про помилки](#manual-customizing-the-error-messages)
    - [Хук валидатора After](#after-validation-hook)
- [Робота з провалідованими вхідними даними](#working-with-validated-input)
- [Робота з повідомленнями про помилки](#working-with-error-messages)
    - [Визначення користувацьких повідомлень в мовних файлах](#specifying-custom-messages-in-language-files)
    - [Визначення атрибутів в мовних файлах](#specifying-attribute-in-language-files)
    - [Визначення користувацьких імен для атрибутів в мовних файлах](#specifying-values-in-language-files)
- [Доступні правила валідації](#available-validation-rules)
- [Умовне додавання правил](#conditionally-adding-rules)
- [Валідація масивів](#validating-arrays)
    - [Валідація вкладених масивів](#validating-nested-array-input)
    - [Індекси та позиції повідомлень про помилки](#error-message-indexes-and-positions)
- [Валідація паролей](#validating-passwords)
- [Користувацькі правила валідації](#custom-validation-rules)
    - [Використання класа Rule](#using-rule-objects)
    - [Використання замикання](#using-closures)
    - [Неявні правила](#implicit-rules)

<a name="introduction"></a>
## Вступ

Laravel пропонує декілька підходів для перевірки вхідних даних вашого додатка. Наприклад, метод `validate`, доступний для всіх вхідних HTTP-запитів. Однак ми обговоримо й інші підходи до валідації.
Laravel містить корисні правила валідації, які застосовуються до даних, включаючи валідацію на унікальність значення в конкретній таблиці бази даних. Ми детально розглянемо кожне з цих правил валідації, щоб ви були ознайомленні з усіма особливостями валідації в Laravel.

<a name="validation-quickstart"></a>
## Швидкий старт

Щоб дізнатися про потужні функції валідації Laravel, давайте розглянемо повний приклад валідації форми і відображень повідомлень про помилки кінцевому користувачеві. Прочитавши цей загальний огляд, ви зможите отримати уявлення про те, як перевіряти дані вхідного запита за допомогою Laravel:

<a name="quick-defining-the-routes"></a>
### Визначення маршрутів

По-перше, припустимо, що в нашому файлі `routes/web.php` визначені наступні маршрути:
```php
    use App\Http\Controllers\PostController;

    Route::get('/post/create', [PostController::class, 'create']);
    Route::post('/post', [PostController::class, 'store']);
```
Маршрут `GET` покаже користувачеві форму для створення нового повідомлення в блозі, а маршрут `POST` збереже нове повідомлення в базі даних.

<a name="quick-creating-the-controller"></a>
### Створення контроллера

Тепер, давайте розглянемо звичайний контроллер, який обробляє вхідні запити на ці маршрути. Поки що залишимо метод `store` порожнім:
```php
    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;

    class PostController extends Controller
    {
        /** 
         * Показати форму для створення нового повідомлення в блозі
         *
         * @return \Illuminate\View\View
         */
        public function create()
        {
            return view('post.create');
        }

        /** 
         * Зберегти новий запис в блозі
         *
         * @param  \Illuminate\Http\Request  $request
         * @return \Illuminate\Http\Response
         */
        public function store(Request $request)
        {	
            // Виконати валідацію та зберегти повідомлення в блозі ...
        }
    }
```
<a name="quick-writing-the-validation-logic"></a>
### Написання логіки валідації

Тепер ми готові заповнити наш метод `store` логікою для валідації нового повідомлення в блозі. Для цього ми будемо вживати базовий метод `validate`, який надається об'єктом `Illuminate\Http\Request`. Якщо правила валідації будуть пройдені, то ваш код продовжить нормальне виконання; однак, якщо перевірка не пройдена, то буде викинуто виняток `Illuminate\Validation\ValidationException`, і автоматично буде додана відповідна відповідь про помилку до змінної `$errors` яка є екземпляром `Illuminate\Support\MessageBag`, яку можно показати відправивши назад користувачеві через шаблон.

Також якщо валідація не пройдена під час традиційного HTTP-запита, то буде сгенеровано відповідь-перенаправлення на попередню URL-адресу. Якщо вхідний запит є XMLHttpRequest-запитом, то буде повернуто [JSON-відповідь, яка містить повідомлення про помилки валідації](#validation-error-response-format).

Щоб краще зрозуміти метод `validate`, давате повернемося до метода `store`:
```php
    /**
     * Зберегти новий запис в блозі.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request)
    {
        $validatedData = $request->validate([
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ]);

        // Запис блога корректний ...
    }
```
Як бачите, правила валідації передаються в метод `validate`. Не хвилюйтесь - всі доступні правила валідації [задокументовані](#available-validation-rules). Знов-таки, якщо перевірка не пройдена, то буде автоматично згенеровано корректну відповідь. Якщо перевірку пройдено успішно, то наш контроллер продовжить нормальну роботу, а метод `validate` поверне масив провалідованих даних.

В якості альтернативи правила валідації можуть бути вказані як масиви правил замість одного рядка з дільником `|`:
```php
    $validatedData = $request->validate([
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);
```
Крім того, ви можете використовувати метод `validateWithBag` для валідації запита зі збереженням будь-яких повідомлень про помилки у [іменовану колекцію помилок](#named-error-bags), вказавши першим аргументом назву цій колекції:
```php
    $validatedData = $request->validateWithBag('post', [
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);
```
<a name="stopping-on-first-validation-failure"></a>
#### Припинення валідації при виникненні першої помилки

За бажанням можна припинити виконання правил валідації для атрибута (атрибутом називається ім'я поля форми) після першої помилки. Для цього потрібно встановити атрибуту правило `bail`:
```php
    $request->validate([
        'title' => 'bail|required|unique:posts|max:255',
        'body' => 'required',
    ]);
```
В цьому прикладі, якщо правило `unique` для атрибута `title` не буде пройдено, то наступне правило `max` не виконається. Правила перевірятимуться по порядку їх призначення.

<a name="a-note-on-nested-attributes"></a>
#### Примітка про вкладені атрибути

Якщо вхідний HTTP-запит містить дані «вкладених» полей `<input type="user" name="user[name]">`, то ви можете вказати ці поля в своїх правилах валідації, використовуючи «крапкову нотацію»:
```php
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'author.name' => 'required',
        'author.description' => 'required',
    ]);
```
З іншого боку, якщо ім'я вашого поля буквально містить крапку, то ви можете явно заборонити її інтерпритацію, як частину «крапкової нотації», екранувавши крапку за допомогою зворотної косої риски `\` <span style="color:red">(чомусь не працює!).</span>:
```php
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'v1\.0' => 'required',
    ]);
```
<a name="quick-displaying-the-validation-errors"></a>
### Відображення помилок валідації

А що, якщо поля вхідних запитів не проходять зазначені правила валідації? Як згадувалось раніше, Laravel автоматично пренаправить користувача назад у вихідне положення. Крім того, всі помилки валідації разом з [вхідними даними запита](requests.md#retrieving-old-input) будуть автоматично [записані в сессію](session.md#flash-data).

Змінна `$errors` використовується у всіх шаблонах вашого додатка завдяки посереднику `Illuminate\View\Middleware\ShareErrorsFromSession`, який занесений до групи посередників `web` у класі `Kernel.php`. Поки застосовується цей посередник, у ваших шаблонах завжи буде доступна така змінна як `$errors`, яка дозволить вам припускати, що змінна `$errors` завжи визначена і може безпечно використовуватися. Змінна `$errors` як зазначалося вище є екземпляром `Illuminate\Support\MessageBag`. Для отримання додаткової інформації по роботі з цим об'єктом [ознайомтесь з його документацією](#working-with-error-messages).

Отже, в нашому прикладі користувач буде перенаправлений на метод нашого контроллера `create`, у випадко, якщо валідація завершиться невдало, що дозволяє нам відобразити повідомлення про помилку в шаблоні:
```blade
<!-- /resources/views/post/create.blade.php -->

<h1>Створення поста в блозі</h1>

@if ($errors->any())
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif

<!-- Форма для створення поста в блозі -->
```
<a name="quick-customizing-the-error-messages"></a>
#### Корегування повідомлень про помилки

Кожне вмонтоване правило валідації Laravel містить повідомлення про помилку, яке знаходиться у файлі `lang/en/validation.php` вашого додатка. В цьому файлі ви знайдете записи з перекладом для кожного правила вілідації. Також ви можете зміюват або модифікувати ці повідомлення в залежності від потреб вашого додатка.

Крім того, ви можете скопіювати цей файл до каталогу перекладу іншої мови, щоб перекласти повідомлення на мову вашого додатка. Щоб дізнатися більше про локалізацію Laravel, ознайомтеся з повною [документацією про локалізацію](localization.md).

<a name="quick-xhr-requests-and-validation"></a>
#### XHR-запити і валідація

У наведених вище прикладах ми використовували форму для відправки даних в додаток. Однак, багато додатків отримують запити XHR з фронтенду з використанням JavaScript. При використанні метода `validate`, під час виконання XHR-запита, Laravel не буде генерувати відповідь-перенаправлення. Замість цього Laravel генерує [JSON-відповідь, яка міститиме всі помилки валідації](#validation-error-response-format). Ця JSON відповідь буде відправлена з кодом 422 HTTP статуса.

<a name="the-at-error-directive"></a>
#### Директива `@error`

Ви можете використовувати директиву [`@error` Blade](blade.md#validation-errors), щоб швидко виявити, чи існують повідомлення про помилки валідації для окремо взятого атрибута, включаючи повідомлення про помилки в іменованій колекції помилок. В директиві `@error` ви можите вивести вміст змінної `$message` для відображення повідомлення про помилку:
```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input id="title"
    type="text"
    name="title"
    class="@error('title') is-invalid @enderror">

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```
Якщо ви використовуєте [іменовані колекції помилок](#named-error-bags), то ви забов'язані передати ім'я колекції в якості другого аргумента директиви `@error`:
```blade
<input ... class="@error('title', 'post') {{ $message }} @enderror">
```
<a name="repopulating-forms"></a>
### Повторне заповнення полей форми

Коли Laravel генерує відповідь-перенаправлення через помилку валідації, фреймворк автоматично [одноразово записує всі віходні дані запита до сессії](session.md#flash-data). Це зроблено для того, щоб ви мали змогу зручно отримати доступ до вхідних даних під час наступного запита і повторно заповнити поля форми, яку користувачь намагався відправити.

Щоб отримати вхідні дані попереднього запита, викличте метод `old` екземпляра `Illuminate\Http\Request`. Метод `old` дістане попередньо записані вхідні данні з [сессії](session.md):
```php
	// поверне масив з усіма даними попередньо отриманого запита
    $title = $request->old();
	// поверне вміст окремо взятого поля форми
    $title = $request->old('title');
```
Laravel також містить гломального помічника `old`. Якщо вам потрібно показати вхідні дані минулого запита в [шаблоні Blade](blade.md), то варто використовувати моічник `old` для повторного заповнення полів форми. Якщо для якогось поля не були надані дані у минулому запиті, то буде повернуто `null`:
```blade
<input type="text" name="title" value="{{ old('title') }}">
```
<a name="a-note-on-optional-fields"></a>
### Примітка про необов'язкові поля

По дефолту Laravel містить посередників `App\Http\Middleware\TrimStrings` і `App\Http\Middleware\ConvertEmptyStringsToNull`
в глобальному стеку посередників вашого додатка. Ці посередники перераховані в класі `App\Http\Kernel`. Перший буде автоматично обрізати пробіли всіх вхідних рядкових полів запита на початку та в кінці рядка, а другий - конвертувати будь-які порожні рядкові поля в `null`. Через це вам необхідно буде позначати ваші «необов'язкові» поля запита як `nullable`, якщо ви не хочите, щоб валідатор не вважав такі поля недійсними. Наприклад:
```php
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);
```
В цьому прикладі ми вказали, що поле `publish_at` може бути `null` або допустимим поданням дати. Якщо модифікатор `nullable` не додано до визначення правил, валідатор вважатиме `null` неприпустимою датою.

<a name="validation-error-response-format"></a>
### Формат відповіді помилок валідації

Коли ваш додаток викидає виняток `Illuminate\Validation\ValidationException`, і на вхідний HTTP-запит очікується відповідь JSON, то Laravel автоматично відформатує для вас повідомлення про помилки і поверне HTTP-відповідь `422 Unprocessable Entity`.

Нижче ви можите переглянути приклад формата ответа JSON для помилок валідації. зверніть увагу, що вкладені ключі помилок зведені до «крапкової нотації»:
```json
{
    "message": "The team name must be a string. (and 4 more errors)",
    "errors": {
        "team_name": [
            "The team name must be a string.",
            "The team name must be at least 1 characters."
        ],
        "authorization.role": [
            "The selected authorization.role is invalid."
        ],
        "users.0.email": [
            "The users.0.email field is required."
        ],
        "users.2.email": [
            "The users.2.email must be a valid email address."
        ]
    }
}
```
<a name="form-request-validation"></a>
## Валідація запита форми (FormRequest)

<a name="creating-form-requests"></a>
### Створення запитів фоми

Для більш складних сценаріїв валідації ви можете створити «form request». Form request - це ваш клас запита, який інкапсулює свою власну логіку валідації та авторизації. Щоб сгенерувати новий запит форми, використовуйте команду `make:request` [Artisan](artisan.md):

```shell
php artisan make:request StorePostRequest
```
Ця команда помістить новий клас запита форми в каталог `app/Http/Requests` вашого додатка. Якщо такий каталог відсутній у вашому додатку, то laravel попередньо створить його, після того як ви запустите команду `make:request`. Кожний запит форми,створений Laravel, матиме два методи: `authorize` і `rules`.

Як ви могли здогадатися, метод `authorize` відповідає за визначення того, чи може поточний аутентифікований користувач виконувати дію, представлену запитом, а метод `rules` повертає правила валідації, які повинні застосуватися до даних запита.
```php
    /**
     * Отримати масив правил валідації, які будуть накладатися на запит
     *
     * @return array
     */
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ];
    }
```

> {tip} Ви можите оголосити будь-які залежності, які вам потрібні, в сигнатурі метода `rules`. Вони будуть автоматично отримані за допомогою [контейнера служб](container.md) Laravel.

Отже як аналізуються правила валідації? Все, що вам потрібно зробити - це оголосити залежність від створеного класа «form request» у методі вашого контроллера. Вхідний запит форми перевірятиметься до виклику метода контроллера - це означає, що вам не доведеться переповнювати ваш контроллер будь-якою логікою валідації. Метод `safe` доступний лише для класа «form request»:
```php
    /**
     * Зберегти новий запис в блозі
     *
     * @param  \App\Http\Requests\StorePostRequest  $request
     * @return Illuminate\Http\Response
     */
    public function store(StorePostRequest $request)
    {
        // Вхідний запит пройшов валідацію ...

        // Отримати провалідовані вхідні дані ...
        $validated = $request->validated(): array;
        // Отримати вміст окремо взятого провалідованого поля ...
        $validated = $request->validated('email');
		// Отримати частину підтверджених вхідних даних...
        $validated = $request->safe()->only(['name', 'email']);
        $validated = $request->safe()->except(['name', 'email']);
    }
```
При невдалій валідації також буде сгенеровано відповідь-перенаправлення, щоб повернути користувача в його вихідне положення. Помилки також будуть одноразово додані до сессії, щоб вони були доступні для відображення. Якщо запит був XHR, то користувачеві буде повернуто HTTP-відповідь з кодом статуса 422, включаючи [JSON-подання помилок валідації](#validation-error-response-format).

<a name="adding-after-hooks-to-form-requests"></a>
#### Додавання гачків `after` для запитів форми

Якщо ви бажаєте додати гачок валідації для запита форми, то ви можете використовувати метод `withValidator`. Цей метод отримує повністю ініційований валідатор, що дозволяє вам викликати будь-який з його методів до того як, правила валідації будуть фактично проаналазовані:
```php
    /**
     * Надстройка экземпляра валидатора.
     *
     * @param  \Illuminate\Validation\Validator  $validator
     * @return void
     */
    public function withValidator($validator)
    {
        $validator->after(function ($validator) {
            if ($this->somethingElseIsInvalid()) {
                $validator->errors()->add('field', 'Щось не так з цим полем!');
            }
        });
    }
```
<a name="request-stopping-on-first-validation-rule-failure"></a>

#### Припинення валідації після першої невдалої перевірки

Додавши властивість `$stopOnFirstFailure` до вашого класу запитів, ви можете повідомити валідатору, що він повинен припинити валідацію всіх атрибутів після виникнення першої помилки валідації:
```php
    /**
     * Припинення валідації після першої невдалої перевірки.
     *
     * @var bool
     */
    protected $stopOnFirstFailure = true;
```
<a name="customizing-the-redirect-location"></a>

#### Зміна адреси вдповіді-перенаправлення

Як обговорювалось раніше, буде згенерована відповідь-перенаправлення, щоб повернути користувача в його вихідне положення, якщо валідація запита форми невдала. Однак, ви можите налаштувати цю поведінку. Для цього визначте властивість `$redirect` у вашому запиті форми:
```php
    /**
     * URI перенаправлення користувачів у випадку невдалої валідації.
     *
     * @var string
     */
    protected $redirect = '/dashboard';
```
Або, якщо ви бажаєте перенаправити користувачів на іемнований маршрут, то вам потрібно замість `$redirect` визначити властивість `$redirectRoute`:
```php
    /**
     * Маршрут перенаправлення користувачів у випадку невдалої валідації.
     *
     * @var string
     */
    protected $redirectRoute = 'dashboard';
```
<a name="authorizing-form-requests"></a>

### Авторизація запитів

Клас запита форми також містить метод `authorize`. В рамках цього метода ви можете визначити, чи дійсно аутентифікований користувач має право на зміну поточного ресурса. Наприклад, ви можете визначити, чи дійсно користувач володіє коментарем у блозі, який він намагається оновити. Швидше за все, ви взаємодіятимете зі своїми [шлюзами та політиками авторизації](authorization.md) за допомогою цього методу:

```php
    use App\Models\Comment;

    /**
     * Визначте, чи має користувач право робити цей запит.
     *
     * @return bool
     */
    public function authorize()
    {
        $comment = Comment::find($this->route('comment'));

        return $comment && $this->user()->can('update', $comment);
    }
```
Оскільки всі запити форми розширюють базовий клас запитів Laravel, ми можемо використовувати метод `user` для доступу до поточного аутентифікованого користувача. Також зверніть увагу на виклик метода `route` в наведеному вище прикладі. Цей метод надає вам доступ до параметрів URL, які визначені для викликаємого маршрута, такі як `{comment}` у наведеному нижче прикладі:

```php
    Route::post('/comment/{comment}');
```
Таким чином, якщо ваша програма використовує переваги, [зв’язування моделі маршруту](routing.md#route-model-binding), то ваш код може бути ще більш стислим за допомогою доступу до витягнутої моделі як властивості класа запиту форми:

```php
    return $this->user()->can('update', $this->comment);
```
Якщо метод `authorize` повертає `false`, то буде автоматично повернуто HTTP-відповідь з кодом статуса 403, і метод вашего контроллера не виконається.

Якщо ви плануєте обробляти логіку авторизації для запита в іншій частині вашого додатка, то ви можете просто повернути `true` з метода `authorize`:

```php
    /**
     * Визначте, чи має користувач право робити цей запит.
     *
     * @return bool
     */
    public function authorize()
    {
        return true;
    }
```

> {tip} Ви можете оголосити будь-які залежності які вам потрібні в сигнатурі методу `authorize`. Вони будуть автоматично витягнуті з [контейнера служб](container.md) Laravel.

<a name="customizing-the-error-messages"></a>
### Корегування повідомлень про помилки

Ви можете змінювати повідомлення про помилки використовуючи запити форм, перевизначивши метод `messages`. Цей метод повинен повертати масив пар атрибут / правило і відповідні їм повідомлення про помилки. Якщо ключем буде лише правило, то повідомлення про помилку буде росповсюджуватися тільки на правило. Можна визначити пару атрибут / правило через «крапкову нотацію» тоді, повідомлення про помилку буде втановлено для конкретного атрибута з правилом:
```php
    /**
     * Отримати повідомлення про помилку для визначених правил перевірки.
     *
     * @return array
     */
    public function messages()
    {
        return [
            'title.required' => 'A title is required',
			'title.min' => 'A title min lenght is ...',
            'body.required' => 'A message is required',
        ];
    }
```
<a name="customizing-the-validation-attributes"></a>

#### Коригування атрибутів валідації

Багато повідомлень про помилку вмонтованого правила валідації Laravel містять заповнювач :attribute. Якщо ви бажаєте, щоб заповнювач `:attribute` вашого повідомлення про помилку валідації був замінений іншим іменем атрибута, то ви можите вказати власні імена перевизначивши метод `attributes`. Цей метод повинен повертати масив пар атрибут (ім'я інпута) / нове ім'я:
```php
    /**
     * Отримайти власні атрибути для помилок валідатора.
     *
     * @return array
     */
    public function attributes()
    {
        return [
            'email' => 'email address',
        ];
    }
```
<a name="preparing-input-for-validation"></a>
### Підготовка вхідних данних для валідації

Якщо вам необхідно підготувати або обробити будь-які дані із запиту перш ніж застосувати правила валідації, то вам варто використати метод `prepareForValidation`:

```php
    use Illuminate\Support\Str;

    /**
     * Підготовка даних для валідації.
     *
     * @return void
     */
    protected function prepareForValidation()
    {
        $this->merge([
            'slug' => Str::slug($this->slug),
        ]);
    }
```

<a name="manually-creating-validators"></a>
## Створення валідатора на замовлення

Якщо ви не бажаєте використовувати метод `validate` запита, ви можете створити екземпляр валідатора вручну, використовуючи [фасад](facades.md) `Validator`. Метод `make` фасада генерує новий екземпляр валідатора:
```php
    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Validator;

    class PostController extends Controller
    {
        /**
         * Зберегти новий запис в блозі.
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            $validator = Validator::make($request->all(), [
                'title' => 'required|unique:posts|max:255',
                'body' => 'required',
            ]);

            if ($validator->fails()) {
                return redirect('post/create')
                            ->withErrors($validator)
                            ->withInput();
            }

            // Отримати масив з провалідованими вхідними даними ...
            $validated = $validator->validated();

            // Отримати вміст окремо взятого провалідованого поля ...
            $validated = $request->validated('email');

            // Зберегти повідомлення в блозі ...
        }
    }
```
Перший аргумент, який передається методу `make`, — це дані, які перевіряються. Другий аргумент - це масив правил валідації, які слід застосовувати до даних.

Після визначення того, що запит не пройшов валідацію за допомогою метода `fails`, ви можете здійснити перенаправленя користувача у вихідне положення, використавши глобального помічника `redirect`, а за допомогою метода `withErrors` можна одноразово занести повідомлення про помилки в сесію. При використанні цього метода змінна `$errors` буде автоматично передана до ваших шаблонів після перенаправлення, що дозволить вам з легкістю відобразити їх користувачеві. Метод `withErrors` приймає екземпляр валідатора, екземпляр `MessageBag` або звичайний масив PHP. Метод `withInput` одноразово заносить дані полів форми до сесії.

#### Припинення валідації пілся першої невдалої перевірки

Методо `stopOnFirstFailure` проінформує валідотор про те, що він має зупинити валідацію всіх атрибутів після виникнення першої помилки валідації.
```php
    if ($validator->stopOnFirstFailure()->fails()) {
        // ...
    }
```
<a name="automatic-redirection"></a>
### Автоматичне перенаправлення

Якщо ви бажаєте створити екземпляр валідатора в ручну, але як і раніше скористатися перевагами автоматичного перенаправлення з занесенням помилок та даних полів форми, пропонованого методом `validate` HTTP-запита, ви можете викликати метод `validate` створеного екземпляра валідатора. Якщо валідація пройдена успішно, то метод `validate` вашого екземпляра валідатора поверне масив провалідованих даних. Користувач буде автоматично перенаправлений або у випадку запита XHR, [буде повернуто відповідь JSON](#validation-error-response-format), якщо валідація невдала:
```php
    $validatedData = Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validate();
```
Ви також можете використовувати метод `validateWithBag` для збереження повідомлень про помилки до [іменованої колекції помилок](#named-error-bags), якщо валідація буде невдалою:
```php
    $validatedData = Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validateWithBag('post');
```
<a name="named-error-bags"></a>
### Іменовані колекції помилок

Якщо ви маєте декілька форм на одній сторінці, то вам варто встановити ім'я екземпляру `MessageBag`, який містить помилки валідації, що дозволить вам отримати повідомлення про помилки для окремо взятої форми. Щоб досягти цього, передайте ім'я у якості другого аргумента метода `withErrors`:
```php
    return redirect('register')->withErrors($validator, 'login');
```
Потім, ви можете отримати доступ до іменованого екземпляра MessageBag зі змінної $errors через властивість:

```blade
{{ $errors->login->first('email') }}
```

<a name="manual-customizing-the-error-messages"></a>
### Коригування повідомлень про помилки

За необхідністю ви також маєте можливість визначати власні повідомлення про помилки, які повинен використовувати екземпляр валідатора замість повідомлень про помилки за замовчуванням, наданими Laravel. Є декілька способів вказати власні повідомлення. По-перше, ви можете передати власні повідомлення в якості третього аргумента метода `Validator::make`:
```php
    $validator = Validator::make($input, $rules, $messages = [
        'required' => 'The :attribute field is required.',
    ]);
```
В цьому прикладі заповнювач `:attribute` буде замінений фактичним ім'ям поля, яке перевіряється. Ви також можете використовувати інші заповнювачі у повідомленнях валідатора. Наприклад
```php
    $messages = [
        'same' => 'The :attribute and :other must match.',
        'size' => 'The :attribute must be exactly :size.',
        'between' => 'The :attribute value :input is not between :min - :max.',
        'in' => 'The :attribute must be one of the following types: :values',
    ];
```
<a name="specifying-a-custom-message-for-a-given-attribute"></a>

#### Визначення користувацьких повідомленнь для заданого атрибута
За бажанням можна визначити власне повідомлення про помилки тільки для окремого атрибута. Ви можете це зробити використовуючи «крапкову нотацію». Спочатку вкажіть ім'я атрибута, а потім правило:
```php
    $messages = [
        'email.required' => 'We need to know your email address!',
    ];
```
<a name="specifying-custom-attribute-values"></a>

#### Визначення користувацьких імен для атрибутів

Багато повідомлень про помилку вмонтованого правила валідації Laravel містять заповнювач :attribute, який замінюється іменем перевіряємого поля або атрибута. Щоб визначити власні значення, які використовуються для заміни цих заповнювачів для конкретних полів, ви можете передати масив ваших атрибутів у якості четвертого аргумента методу `Validator::make`:
```php
    $validator = Validator::make($input, $rules, $messages, [
        'email' => 'email address',
    ]);
```
<a name="after-validation-hook"></a>
### Гачок валідатора After

Ви також маєте можливість визначити замикання, які будуть запускатися після завершення валідації. Це дозволяє легко виконувати подальшу валідацію і навіть додавати повідомлення про помилки до колекції помилок. Викличте метод `after` екземпляра валідатора:
```php
    $validator = Validator::make(/* ... */);

    $validator->after(function ($validator) {
        if ($this->somethingElseIsInvalid()) {
            $validator->errors()->add(
                'field', 'Something is wrong with this field!'
            );
        }
    });

    if ($validator->fails()) {
        //
    }
```
<a name="working-with-validated-input"></a>
## Робота з провалідованими вхідними даними

Після валідації даних вхідного запиту за допомогою запита форми (FormRequest), ви можете отримати дані вхідного запита, які дійсно пройшли валідацію. Це робиться кількома способами. По перше, ви можете викликати метод `validated` запита форми. Цей метод поверне масив даних, які були провалідовані:

```php
    $validated = $request->validated(): array;
```

У якості альтернативи ви моете викликати метод `safe` запита форми. Цей метод повертає екземпляр `Illuminate\Support\ValidatedInput`. Цей об'єкт надає методи `only`, `except` та `all` для отримання вибіркових даних або цілого масива провалідованих даних:

```php
    $validated = $request->safe()->only(['name', 'email']): array;

    $validated = $request->safe()->except(['name', 'email']): array;

    $validated = $request->safe()->all(): array;
```

Крім того, екземпляр `Illuminate\Support\ValidatedInput` може бути ітерованим та доступним, як масив:

```php
    // Провалідовані дані можуть бути ітеровані ...
    foreach ($request->safe() as $key => $value) {
        //
    }
    // До провалідованих даних можна отримати доступ, як до звичайного масива ...
    $validated = $request->safe();

    $email = $validated['email'];
```

Якщо є необхідність додати додаткові поля до провалідованих даних, то ви можете викликати додатковий метод `merge`:

```php
    $validated = $request->safe()->merge(['name' => 'Taylor Otwell']);
```

Якщо ви хочите отримати провалідовані дані, як екземпляр [колекції](collections.md), то ви можите викликати метод `collect`:

```php
    $collection = $request->safe()->collect();
```

<a name="working-with-error-messages"></a>
## Робота з повідомленнями про помилки

Після виклику метода `errors` екземпляра `Validator`, ви отримаєте екземпляр `Illuminate\Support\MessageBag`, який містить безліч зручних методів для роботи з повідомленнями про помилки. Змінна `$errors`, яка автоматично стає доступною для всіх шаблонів, також є екземпляром класу `MessageBag`.

<a name="retrieving-the-first-error-message-for-a-field"></a>
#### Отримання одного повідомлення про помилку для поля

Щоб отримати одне повідомлення про помилку для вказаного поля, використовуйте метод `first`:

```php
    $errors = $validator->errors();

    echo $errors->first('email'): string;
    echo $errors->first('user.name'): string;
```

<a name="retrieving-all-error-messages-for-a-field"></a>
#### Отримання всіх повідомлень про помилки для поля

Якщо вам потрібно отримати масив всіх повідомлень для певного поля, використайте метод `get`:

```php
    foreach ($errors->get('email') as $message) {
        //
    }
```

Якщо ви перевіряєте масив полей форми, то ви можете отримати всі повідомлення для кожного з елементів масива, використавши символ `*`:

```php
    foreach ($errors->get('attachments.*') as $message) {
        //
    }
```

<a name="retrieving-all-error-messages-for-all-fields"></a>
#### Отримання всіх повідомлень про помилки для всіх полів

Щоб отримати масив всіх повідомлень для всіх полів, використовутей метод `all`:

```php
    foreach ($errors->all() as $message) {
        //
    }
```

<a name="determining-if-messages-exist-for-a-field"></a>

#### Визначення наявності повідомлень для поля

Метод `has` використовується для визначення наявності повідомлень про помилки для вказаного поля:

```php
    if ($errors->has('email')) {
        //
    }
```

<a name="specifying-custom-messages-in-language-files"></a>

### Визначення користувацьких повідомлень в мовних файлах валідації

Кожне вмонтоване правило валідації Laravel містить повідомлення про помилку, яке знаходиться у файлі `lang/en/validation.php` вашого додатка. В цьому файлі ви знайдете запис з перекладом для кожного правила валідації. Ви можете змінювати або модифікувати ці повідомлення в залежності від потреб вашого додатка.

Крім того, ви можете зробити копію цього файла і додати його до каталогу з іншою мовою, щоб перекласти повідомлення на іншу мову вашого додатка. Щоб дізнатися більше про локалізацію Laravel, ознайомтеся з повною [документацією про локалізацію](localization.md).

<a name="custom-messages-for-specific-attributes"></a>

#### Визначення користувацького повідомлення для певного атрибута

Ви можете налаштувати повідомлення про помилки, які використовуються для вказаних комбінацій атрибутів і правил у файлах мови перевірки вашого додатка. Для цього додайте повідомлення в масив `custom` мовного файла `lang/xx/validation.php` вашого додатка.

```php
    'custom' => [
        'email' => [
            'required' => 'We need to know your email address!',
            'max' => 'Your email address is too long!'
        ],
    ],
```

<a name="specifying-attribute-in-language-files"></a>

### Визначення атрибутів в мовних файлах валідації

Багато повідомлень про помилку вмонтованого правила валідації Laravel містять заповнювач :attribute, який замінюється іменем перевіряємого поля або атрибута. Якщо ви бажаєте, щоб частина `:attribute` вашого повідомлення валідації була змінена власним значенням, то вам варто визначити ім'я атрибута в масиві `attributes` вашого мовного файла `lang/xx/validation.php`:

```php
    'attributes' => [
        'email' => 'email address',
    ],
```
<a name="specifying-values-in-language-files"></a>

### Визначення користувацьких імен для атрибутів в мовних файлах валідації

Деякі повідомлення про помилки вмонтованих правил валідації Laravel містять заповнювач `:value`, який замінюється поточним значенням атрибута запиту. Іноді потрібно замінити частину `:value` вашого повідомлення валідації на власне значення. Наприклад, розглянемо наступне правило, яке вказує, що номер кредитної карти вимагається обов'язково, якщо для параметра `payment_type` встановленно значення `cc`:
```php
    Validator::make($request->all(), [
        'credit_card_number' => 'required_if:payment_type,cc'
    ]);
```
Якщо дане правило не пройде перевірки, то буде видано наступне повідомлення про помилку:

```none
The credit card number field is required when payment type is cc.
```
Замість того, щоб відобразити `cc` в якості значення типа платежа, ви можете вказати більш зручне представлення значення для користувача у вашому язиковому файлі`lang/xx/validation.php`, визначив масив `values`:
```php
    'values' => [
        'payment_type' => [
            'cc' => 'credit card'
        ],
    ],
```
Після визначення цього значення правило валідації видасть наступне повідомлення про помилку:

```none
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## Доступні правила валідації

Нижче наведено список всіх доступних правил валідації та опис їх функцій:

- [Accepted](#rule-accepted)
- [Accepted If](#rule-accepted-if)
- [Active URL](#rule-active-url)
- [After (Date)](#rule-after)
- [After Or Equal (Date)](#rule-after-or-equal)
- [Alpha](#rule-alpha)
- [Alpha Dash](#rule-alpha-dash)
- [Alpha Numeric](#rule-alpha-num)
- [Array](#rule-array)
- [Bail](#rule-bail)
- [Before (Date)](#rule-before)
- [Before Or Equal (Date)](#rule-before-or-equal)
- [Between](#rule-between)
- [Boolean](#rule-boolean)
- [Confirmed](#rule-confirmed)
- [Current Password](#rule-current-password)
- [Date](#rule-date)
- [Date Equals](#rule-date-equals)
- [Date Format](#rule-date-format)
- [Declined](#rule-declined)
- [Declined If](#rule-declined-if)
- [Different](#rule-different)
- [Digits](#rule-digits)
- [Digits Between](#rule-digits-between)
- [Dimensions (Image Files)](#rule-dimensions)
- [Distinct](#rule-distinct)
- [Email](#rule-email)
- [Ends With](#rule-ends-with)
- [Enum](#rule-enum)
- [Exclude](#rule-exclude)
- [Exclude If](#rule-exclude-if)
- [Exclude Unless](#rule-exclude-unless)
- [Exclude With](#rule-exclude-with)
- [Exclude Without](#rule-exclude-without)
- [Exists (Database)](#rule-exists)
- [File](#rule-file)
- [Filled](#rule-filled)
- [Greater Than](#rule-gt)
- [Greater Than Or Equal](#rule-gte)
- [Image (File)](#rule-image)
- [In](#rule-in)
- [In Array](#rule-in-array)
- [Integer](#rule-integer)
- [IP Address](#rule-ip)
- [JSON](#rule-json)
- [Less Than](#rule-lt)
- [Less Than Or Equal](#rule-lte)
- [MAC Address](#rule-mac)
- [Max](#rule-max)
- [MIME Types](#rule-mimetypes)
- [MIME Type By File Extension](#rule-mimes)
- [Min](#rule-min)
- [Multiple Of](#multiple-of)
- [Not In](#rule-not-in)
- [Not Regex](#rule-not-regex)
- [Nullable](#rule-nullable)
- [Numeric](#rule-numeric)
- [Password](#rule-password)
- [Present](#rule-present)
- [Prohibited](#rule-prohibited)
- [Prohibited If](#rule-prohibited-if)
- [Prohibited Unless](#rule-prohibited-unless)
- [Prohibits](#rule-prohibits)
- [Regex (regular expression)](#rule-regex)
- [Required](#rule-required)
- [Required If](#rule-required-if)
- [Required Unless](#rule-required-unless)
- [Required With](#rule-required-with)
- [Required With All](#rule-required-with-all)
- [Required Without](#rule-required-without)
- [Required Without All](#rule-required-without-all)
- [Required Array Keys](#rule-required-array-keys)
- [Same](#rule-same)
- [Size](#rule-size)
- [Sometimes](#validating-when-present)
- [Starts With](#rule-starts-with)
- [String](#rule-string)
- [Timezone](#rule-timezone)
- [Unique (Database)](#rule-unique)
- [URL](#rule-url)
- [UUID](#rule-uuid)

<!-- </div> -->

<a name="rule-accepted"></a>
#### accepted
Поле, яке перевіряється, повинно мати значення `"yes"`, `"on"`, `1` или `true`. Це корисно для підтвердження прийняття «Умов використання» або подібних полів.

<a name="rule-accepted-if"></a>
#### accepted_if:anotherfield,value,...

Поле, яке перевіряється, повинно мати значення `"yes"`, `"on"`, `1` или `true` якщо інше поле, яке перевіряється, дорівнює вказаному значенню. Це корисно для підтвердження прийняття «Умов використання» або подібних полів.

<a name="rule-active-url"></a>
#### active_url

Поле, яке перевіряється, повинно містити дійсний запис A або AAAA відповідно до функції `dns_get_record` PHP. Ім’я хоста зазначеного URL витягується за допомогою функції PHP parse_url перед передачею в dns_get_record.

<a name="rule-after"></a>
#### after:_date_

Поле, яке перевіряється, повинно мати значення після зазначеної дати. Дати будуть передані до функції `strtotime` PHP для перетворення в дийсний екземпляр `DateTime`:

```php
    'start_date' => 'required|date|after:tomorrow'
```
Замість передачі рядка дати, яка буде проаналізована за допомогою `strtotime`, ви можете вказати інше поле для порівняння з датою.

```php
    'finish_date' => 'required|date|after:start_date'
```
<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_


Поле, яке перевіряється, повинно мати значення після зазаначеної дати або дорівнювати їй. Для дотримання додатково інформації див. правило [after](#rule-after).

<a name="rule-alpha"></a>
#### alpha

Поле, яке перевіряється, повинно складатися лише з літер.

<a name="rule-alpha-dash"></a>
#### alpha_dash

Поле, яке перевіряється, може містити в собі символи чисел та літер, а також дефіси і підкреслення.

<a name="rule-alpha-num"></a>
#### alpha_num

Поле, яке перевіряється повинно складатися лише з літер та чисел.

<a name="rule-array"></a>
#### array

Поле, яке перевіряється повинно бути масивом PHP.

Коли для правила `array` надаються додаткові значення, кожний ключ у вхідному масиві повинен бути присутній у списку значень, які надані правилу. В наступному прикладі ключ `admin` у вхідному масиві буде недійсним тому, що він не присутній у списку значень, які надані правилу `array`. Виникне помилка з наступним повідомленням 'The user must be an array.':

```php
    use Illuminate\Support\Facades\Validator;

    $input = [
        'user' => [
            'name' => 'Taylor Otwell',
            'username' => 'taylorotwell',
            'admin' => true,
        ],
    ];

    Validator::make($input, [
        'user' => 'array:username,name,locale',
    ]);
```

Взагалі, ви завжди маєте вказувати ключі масива, які можуть бути присутні у вашому масиві.

<a name="rule-bail"></a>
#### bail

Зупунити подальше застосування правил валідації атрибута пілся першої невдалої перевірки.

Порівняно з правилом `bail`, яке зупиняє подальшу валідацію тільки окремо взятого поля, метод `stopOnFirstFailure` повідомить валідатору, що він повинен припинити подальшу валідацію всіх атрибутів при виникненні першої помилки:

```php
    if ($validator->stopOnFirstFailure()->fails()) {
        // ...
    }
```

<a name="rule-before"></a>
#### before:_date_

Поле, яке перевіряється, має бути значенням, що передує вказаній даті. Дати будуть надані функції PHP `strtotime` для перетворення в дійсний екземпляр `DateTime`. Крім того, як і в правилі [`after`](#rule-after), ім'я іншого поля, яке перевіряється, може бути вказане у якості значення `date`.

<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

Поле, що перевіряється, має бути значенням, що передує вказаній даті або дорівнювати їй. Дати будуть надані функції PHP `strtotime` для перетворення в дійсний екземпляр `DateTime`. Крім того, як і в правилі [`after`](#rule-after), ім'я іншого поля, яке перевіряється, може бути вказане у якості значення `date`.

<a name="rule-between"></a>
#### between:_min_,_max_

Розмір поля, що перевіряється, має бути між вказаним _min_ і _max_. Рядки, числа, масиви та файли обчислюються так само, як і правило [`size`](#rule-size)..

<a name="rule-boolean"></a>
#### boolean

Поле, яке перевіряється, має бути приведено до логічного значення. Поля, які прийматимуться, повинні мати значення: `true`, `false`, `1`, `0`, `"1"` та `"0"`.

<a name="rule-confirmed"></a>
#### confirmed

Поле, яке перевіряється, повинно бути однаковим з полем `{field}_confirmation`. Наприклад, якщо поле, яке перевіряється – `password`, то обов'язково у віхідних даних повинно бути присутне поле `password_confirmation`.

<a name="rule-current-password"></a>
#### current_password

Поле підтвердження має бути однаковим з паролем автентифікованого користувача. Також ви можете вказати [захист автентифікації](authentication.md), за допомогою першого параметра правила:

```php
    'password' => 'current_password:api'
```

<a name="rule-date"></a>
#### date

Поле, яке перевіряється, має відповідати формату функції `strtotime` PHP, який вона приймає. Наприклад: 2021-07-09/09-07-2021 13:30:00

<a name="rule-date-equals"></a>
#### date_equals:_date_

Поле, яке перевіряється, повинно дорівнювати зазначеній даті. Дати будуть надані функції `strtotime` PHP для перетворення в дійсний екземпляр `DateTime`.

<a name="rule-date-format"></a>
#### date_format:_format_

Поле, яке перевіряється, повинно відповідати переданому _format_. При валідації поля слід використовувати `date` або `date_format`, але не обидва одразу. Це правило валідації підтримує всі формати, які підтримує клас [`DateTime`](https://www.php.net/manual/ru/class.datetime.php) PHP.

<a name="rule-declined"></a>
#### declined

Поле, яке перевіряється повинно містити значення `"no"`, `"off"`, `0` або `false`.

<a name="rule-declined-if"></a>
#### declined_if:anotherfield,value,...

Поле, яке перевіряється, повинно містити значення `"no"`, `"off"`, `0` або `false`, якщо інше поле, яке перевіряється, дорівнює зазначеному.

<a name="rule-different"></a>
#### different:_field_

Поле, що перевіряється, має відрізнятися від зазначеного _field_.

<a name="rule-digits"></a>
#### digits:_value_

Поле, яке перевіряється, має бути цілим і повинно мати зазначену довжину _value_.

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

Поле, яке перевіряється, має бути цілим і повинно мати довжину в зазначеному діапазоні _min_ и _max_.

<a name="rule-dimensions"></a>
#### dimensions

Файл, який перевіряється, має бути зображенням з зазначиними розмірами у параметрах:

```php
    'avatar' => 'dimensions:min_width=100,min_height=200'
```

Доступні обмеження:_min\_width_, _max\_width_, _min\_height_, _max\_height_, _width_, _height_, _ratio_.

Обмеження _ratio_ має бути представлене як ширина, поділена на висоту. Це можна вказати дробом, наприклад 3/2, або числом з плаваючою точкою, наприклад 1,5:

```php
    'avatar' => 'dimensions:ratio=3/2'
```
Оскільки це правило потребує декількох аргументів, ви можете використати метод `Rule::dimensions` для довільної побудови правила:

```php
    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'avatar' => [
            'required',
            Rule::dimensions()->maxWidth(1000)->maxHeight(500)->ratio(3 / 2),
        ],
    ]);
```

<a name="rule-distinct"></a>
#### distinct

Під час перевірки масивів поле, яке перевіряється, не повинно мати дубльованих значень:

```php
    'foo.*.id' => 'distinct'

    $validator = Validator::make([
		'first' => ['id'=> 1],
		'second' => ['id' => 1]
		], [
			'*.id' => 'distinct',
		]);
```

Distinct за замовчуванням використовує не суворе порівняння змінних. Щоб використовувати суворі порівняння, ви можете додати параметр strict до правила перевірки:


```php
    'foo.*.id' => 'distinct:strict'
```
Ви можете додати ignore_case до аргументів правила перевірки, щоб правило ігнорувало відмінності у використанні великих літер:

```php
    'foo.*.id' => 'distinct:ignore_case'
```

<a name="rule-email"></a>
#### email

Поле, яке перевіряється, повинно мати формат адреси електронної пошти. Це правило валідації використоює пакет [`egulias/email-validator`](https://github.com/egulias/EmailValidator) для перевірки адреси електронної пошти. За замовченням застосовується валідатор `RFCValidation`, але ви можете застосовувати інші стилі валідації:

```php
    'email' => 'email:rfc,dns'
```
У наведеному вище прикладі будуть застосовані перевірки `RFCValidation` та `DNSCheckValidation`. Ось повний перелік стилів перевірки, які ви можете застосовувати:

<!-- <div class="content-list" markdown="1"> -->

- `rfc`: `RFCValidation`
- `strict`: `NoRFCWarningsValidation`
- `dns`: `DNSCheckValidation`
- `spoof`: `SpoofCheckValidation`
- `filter`: `FilterEmailValidation`

<!-- </div> -->

Валідатор `filter`, який використовує функцію `filter_var` PHP, постачається з Laravel і застосовувався за замовченням до Laravel версії 5.8.

> {note} Валідатори `dns` і `spoof` потребують розширення `intl` PHP.

<a name="rule-ends-with"></a>
#### ends_with:_foo_,_bar_,...

Поле, яке перевіряється, повинно закінчуватися на одне з вказаних значень.

<a name="rule-enum"></a>
#### enum

Правило `Enum` — це правило на основі класу, яке перевіряє, чи містить поле, що перевіряється, дійсне значення enum. Правило `Enum` приймає назву `Enum` як єдиний аргумент конструктора:

```php
    use App\Enums\ServerStatus;
    use Illuminate\Validation\Rules\Enum;

    $request->validate([
        'status' => [new Enum(ServerStatus::class)],
    ]);
```

> {note} Перераховані типи доступні тільки в [PHP 8.1+](https://www.php.net/manual/ru/language.enumerations.php).

<a name="rule-exclude"></a>
#### exclude

Поле, яке перевіряється буде виключене з даних запита, яке повертається методами `validate` і `validated`.

<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

Поле, яке перевіряється, буде виключене з даних запита, яке повертається методами `validate` і `validated`, якщо поле _anotherfield_ дорівнює _value_.

Якщо потрібна складна логіка умовного виключення, ви можете використати метод `Rule::excludeIf`. Цей метод приймає логічне значення або замикання. Якщо надати замикання, то воно має повертати `true` або `false`, щоб вказати, чи потрібно виключити поле, що перевіряється:

```php
    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::excludeIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::excludeIf(fn () => $request->user()->is_admin),
    ]);
```

<a name="rule-exclude-unless"></a>
#### exclude_unless:_anotherfield_,_value_

Поле, яке перевіряється, буде виключене з даних запита, яке повертається методами `validate` і `validated`, якщо поле _anotherfield_ не дорівнює _value_. Якщо _value_ дорівнюватиме `null` (тобто `exclude_unless:name,null`), то поле, яке перевіряється, буде виключено, якщо поле порівняння не має значення `null` або, якщо поле для порівняння взагалі відсутне.
