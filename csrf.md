git 1ca38aade2baea9b663c3524846f6f5b7d1e27bf

---

# Предотвращение атак CSRF

- [Введение](#csrf-introduction)
- [Предотвращение запросов CSRF](#preventing-csrf-requests)
    - [Исключение URI из защиты от CSRF](#csrf-excluding-uris)
- [Токен X-CSRF](#csrf-x-csrf-token)
- [Токен X-XSRF](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## Введение

Межсайтовая подделка запроса – это разновидность вредоносного эксплойта, при котором неавторизованные команды выполняются от имени аутентифицированного пользователя. К счастью, Laravel позволяет легко защитить ваше приложение от [Межсайтовой подделки запроса](https://ru.wikipedia.org/wiki/Межсайтовая_подделка_запроса) ([Cross Site Request Forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery) – CSRF).

<a name="csrf-explanation"></a>
#### Объяснение уязвимости

Если вы не знакомы с Межсайтовой подделкой запросов, то давайте обсудим пример того, как можно использовать эту уязвимость. Представьте, что ваше приложение имеет маршрут `/user/email`, который принимает `POST`-запрос для изменения адреса электронной почты аутентифицированного пользователя. Скорее всего, этот маршрут ожидает, что поле ввода `email` будет содержать адрес электронной почты, который пользователь хотел бы начать использовать.

Без защиты от CSRF вредоносный веб-сайт может создать HTML-форму, которая указывает на маршрут вашего приложения `/user/email` и отправляет собственный адрес электронной почты злоумышленника:

    <form action="https://your-application.com/user/email" method="POST">
        <input type="email" value="malicious-email@example.com">
    </form>

    <script>
        document.forms[0].submit();
    </script>

Если вредоносный веб-сайт автоматически отправляет форму при загрузке страницы, злоумышленнику нужно только подтолкнуть ничего не подозревающего пользователя вашего приложения посетить свой веб-сайт, и его адрес электронной почты будет изменен в вашем приложении.

Чтобы предотвратить эту уязвимость, нам необходимо проверять каждый входящий запрос `POST`, `PUT`, `PATCH`, или `DELETE` на секретное значение сессии, к которому вредоносное приложение не может получить доступ.

<a name="preventing-csrf-requests"></a>
## Предотвращение запросов CSRF

Laravel автоматически генерирует «токен» CSRF для каждой активной [пользовательской сессии](/docs/{{version}}/session), управляемой приложением. Этот токен используется для проверки того, что аутентифицированный пользователь действительно является лицом, выполняющим запросы к приложению. Поскольку этот токен хранится в сессии пользователя и изменяется каждый раз при повторном создании сессии, вредоносное приложение не может получить к нему доступ.

К CSRF-токену текущей сессии можно получить доступ через сессию запроса или с помощью глобального помощника `csrf_token`:

    use Illuminate\Http\Request;

    Route::get('/token', function (Request $request) {
        $token = $request->session()->token();

        $token = csrf_token();

        // ...
    });

Каждый раз, когда вы создаете HTML-форму «POST», «PUT», «PATCH» или «DELETE» в своем приложении, вы должны включать в форму скрытое поле `_token` CSRF, чтобы посредник CSRF мог проверить запрос. Для удобства вы можете использовать директиву Blade `@csrf` для создания скрытого поля ввода, содержащего токен:

    <form method="POST" action="/profile">
        @csrf

        <!-- Эквивалентно ... -->
        <input type="hidden" name="_token" value="{{ csrf_token() }}" />
    </form>

[Посредник](/docs/{{version}}/middleware) `App\Http\Middleware\VerifyCsrfToken`, который по умолчанию стоит в группе посредников `web`, автоматически проверяет соответствие токена во входном запросе и токен, хранящийся в сессии. Когда эти два токена совпадают, мы знаем, что запрос инициирует аутентифицированный пользователь.

<a name="csrf-tokens-and-spas"></a>
### CSRF-токены и SPA-приложения

Если вы создаете SPA, который использует Laravel в качестве серверной части API, вам следует обратиться к [документации Laravel Sanctum](/docs/{{version}}/sanctum) для получения информации об аутентификации с помощью вашего API и защите от уязвимостей CSRF.

<a name="csrf-excluding-uris"></a>
### Исключение URI из защиты от CSRF

По желанию можно исключить набор URI из защиты от CSRF. Например, если вы используете [Stripe](https://stripe.com) для обработки платежей и используете их систему веб-хуков, вам нужно будет исключить маршрут обработчика веб-хуков Stripe из защиты от CSRF, поскольку Stripe не будет знать, какой токен CSRF отправить вашим маршрутам.

Как правило, вы должны размещать эти виды маршрутов вне группы посредников `web`, которую `App\Providers\RouteServiceProvider` применяет ко всем маршрутам в файле `routes/web.php`. Однако, вы также можете исключить маршруты, добавив их URI в свойство `$except` посредника `VerifyCsrfToken`:

    <?php

    namespace App\Http\Middleware;

    use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken as Middleware;

    class VerifyCsrfToken extends Middleware
    {
        /**
         * URI, которые следует исключить из проверки CSRF.
         *
         * @var array
         */
        protected $except = [
            'stripe/*',
            'http://example.com/foo/bar',
            'http://example.com/foo/*',
        ];
    }

> {tip} Для удобства посредник CSRF автоматически отключается для всех маршрутов при [выполнении тестов](/docs/{{version}}/testing).

<a name="csrf-x-csrf-token"></a>
## Токен X-CSRF

В дополнение к проверке токена CSRF в качестве параметра POST-запроса посредник `App\Http\Middleware\VerifyCsrfToken` также проверяет заголовок запроса `X-CSRF-TOKEN`. Вы можете, например, сохранить токен в HTML-теге `meta`:

    <meta name="csrf-token" content="{{ csrf_token() }}">

Затем, вы можете указать библиотеке, такой как jQuery, автоматически добавлять токен во все заголовки запросов. Это обеспечивает простую и удобную защиту от CSRF для ваших приложений с использованием устаревшей технологии JavaScript на основе AJAX:

    $.ajaxSetup({
        headers: {
            'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
        }
    });

<a name="csrf-x-xsrf-token"></a>
## Токен X-XSRF

Laravel хранит текущий токен CSRF в зашифрованном файле Cookies `XSRF-TOKEN`, который содержится в каждом ответе, генерируемым фреймворком. Вы можете использовать значение Cookies для установки заголовка запроса `X-XSRF-TOKEN`.

Этот файл Cookies, в первую очередь, отправляется для удобства разработчика, поскольку некоторые фреймворки и библиотеки JavaScript, такие, как Angular и Axios, автоматически помещают его значение в заголовок `X-XSRF-TOKEN` в запросах с одним и тем же источником.

> {tip} По умолчанию файл `resources/js/bootstrap.js` включает HTTP-библиотеку Axios, которая автоматически отправляет заголовок `X-XSRF-TOKEN`.