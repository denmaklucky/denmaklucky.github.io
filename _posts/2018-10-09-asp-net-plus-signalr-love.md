---
layout: post
title: ASP.NET + SignalR = ❤
comments: true
---

На конец-то я закончил писать пост про [ASP.NET](https://www.asp.net/). Всё никак не могу выбрать тему для написания на данной технологии, но после долгих раздумей нашел наконец-то то, что искал. И это [SignalR](https://www.asp.net/signalr). 

И так по порядку. 

Во-первых, что такое SignalR. 

SignalR - это абстракция над абстракцией, которая позволяет создавать динамический контент с использованием веб-технологий (и не только). Данная технология позволяет удаленно вызывать JS кода на стороне клиента. До того, как я узнал про SignalR я использовал для этой задачи WebSocket'ы. Так в чем главное отличие SignalR'а от Web Socket'ов?

А отличие состоит в том, что Signal может использовать WebSocket'ы как транспорт.
А также SignalR может использовать в качества транспорта:
* WebSocket;
* EventSource;
* Forever Frame;
* Ajax long polling.

Да, можно на прямую создавать приложения с ипользованием WebSocket, но ... зачем? Если можно перейти на ещё один уровень абстрации и избавиться от всех лишних действий.

Для того, чтобы показать как работает SignalR я создам Чат ~~ha, classic~~.

Для этого нужно создать проект, как на картинке

![asp-net-signalr-love]({{ site.url }}/images/p6/1.png)

Сделаем наш чат немного по серьезней и добавим туда базу для пользователей. Для этого я буду использовать [ASP.NET Indentity](https://docs.microsoft.com/ru-ru/aspnet/identity/overview/getting-started/introduction-to-aspnet-identity). Скажу честно не с первого раз у меня получилось "стартануть" Indentity.

Первым делом нужно добавить в проект следующее сборки:
* Microsoft.AspNet.Identity.EntityFramework;
* Microsoft.AspNet.Identity.OWIN;
* Microsoft.Owin.Host.SystemWeb.

Далее нужно обновить *Web.config*, добавим туда следующее строчку:

```xml
<connectionStrings>
    <add name="DefaultConnection" connectionString="Data Source=(LocalDb)\MSSQLLocalDB;AttachDbFilename=|DataDirectory|\aspnet-WebApplication6-20180724080537.mdf;Initial Catalog=aspnet-WebApplication6-20180724080537;Integrated Security=True" providerName="System.Data.SqlClient" />
  </connectionStrings>
```

P.S. Названия для подключения я выбрал весьма оригинальное.

База будет хранить наших пользоватлей, который были зарегистрированы.

Далее нужно создать наследника для *IdentityUser*, я назвал его ApplicationUser: 

```csharp
public class ApplicationUser : IdentityUser
{
    public async Task<ClaimsIdentity> GenerateUserIdentityAsync(UserManager<ApplicationUser> manager)
    {
        var userIdentity = await manager.CreateIdentityAsync(this, DefaultAuthenticationTypes.ApplicationCookie);
        return userIdentity;
    }
}
```
Далее нужно добавить пользовательский контекс:

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext()
        : base("DefaultConnection", throwIfV1Schema: false)
    {

    }

    public static ApplicationDbContext Create()
    {
        return new ApplicationDbContext();
    }
}
```

Теперь нужно создать две сущестности: первая будет отвечать за авторизацию, а вторая за регистрацию:

```csharp
public class LoginModel
{
    public string Email { get; set; }

    [Required(ErrorMessage ="Поле должно быть задано")]
    public string Password { get; set; }
}
```

Первая сущность будет называться **LoginModel**. Данный класс будет содержать всего два свойства. При этом *Password* должен быть обязательным (как-будто без email'a кто-нибудь сможет зарегистрироваться ^_^).

Вторая сущность будет называться **RegisterModel**, а этот класс будет содержать уже целых три поля:

```csharp
public class RegisterModel
{
    public string Email { get; set; }

        public string Password { get; set; }

    [Compare("Password", ErrorMessage ="Пароли не совпадают")]
    public string ConfirmPassword { get; set; }
}
```

Для того, чтобы не надо было проверять коректность паролей в ручную я буду использовать атрибут *Compare*, тем самым делегирую часть работы самой платформе.

Теперь осталось создать два специализированных класса. Первый класс будет отвечать за авторизацию и регистрацию, а второй будет отвечать за антентификацию.

Для этого в папке *App_Start* создадим класс с именем *IndentityConfig.cs*

![asp-net-signalr-love]({{ site.url }}/images/p6/2.PNG)

В нем создадим два класса (да-да, я знаю что в одном файле не рекомендуют создавать по несколько классов, но для примера можно и создать). 

Первый класс будет называться *ApplicationUserManager*:

```csharp
public class ApplicationUserManager : UserManager<ApplicationUser>
{
    public ApplicationUserManager(IUserStore<ApplicationUser> store)
        : base(store)
    {

    }

    public static ApplicationUserManager Create(IdentityFactoryOptions<ApplicationUserManager> options, IOwinContext context)
    {
        var manager = new ApplicationUserManager(new UserStore<ApplicationUser>(context.Get<ApplicationDbContext>()));
        // Настройка логики проверки имен пользователей
        manager.UserValidator = new UserValidator<ApplicationUser>(manager)
        {
            AllowOnlyAlphanumericUserNames = false,
            RequireUniqueEmail = true
        };

        // Настройка логики проверки паролей
        manager.PasswordValidator = new PasswordValidator
        {
            RequiredLength = 6
        };

        // Настройка параметров блокировки по умолчанию
        manager.UserLockoutEnabledByDefault = true;
        manager.DefaultAccountLockoutTimeSpan = TimeSpan.FromMinutes(5);
        manager.MaxFailedAccessAttemptsBeforeLockout = 5;

        return manager;
    }
}
```

Ну здесь вроде все понятно, а если не понятно то: создаем менеджер пользователей, задаем валидатор для проверки пользователей, задаем валидатор для проверки паролей и задаем найстроки блокировки по умолчанию. У *UserValidator* и *PasswordValidator* есть множество настройек, с которыми можно "поиграться" для получения нужного эффекта.

Далее создаем *ApplicationSignManager*:

```csharp
public class ApplicationSignManager : SignInManager<ApplicationUser, string>
{
    public ApplicationSignManager(ApplicationUserManager userManager, IAuthenticationManager authenticationManager)
        :base(userManager, authenticationManager)
    {

    }

    public static ApplicationSignManager Create(IdentityFactoryOptions<ApplicationSignManager> options, IOwinContext context)
    {
        return new ApplicationSignManager(context.GetUserManager<ApplicationUserManager>(), context.Authentication);
    }
}
```

Не буду вдоваться в подробности, скажу что этот класс нужен для входа в наш будующий чат.

Следующем шагом будем все это дело зарегистрировать, чтобы система зналана с чем она будет работать.

Так как мы использовали **OWIN** то заходим в *Startup.cs* редактируем его следующим образом:

```csharp
public partial class Startup
{
    public void Configuration(IAppBuilder app)
    {
        app.MapSignalR();
        ConfigureAuth(app);
    }
}
```

Во-первых, я его сделал **partial**. Это я сделал, для того, чтобы вынесни логику для настройки классов созданных выше. Логика настройки: 

```csharp
public partial class Startup
{
    public void ConfigureAuth(IAppBuilder app)
    {
        app.CreatePerOwinContext<ApplicationDbContext>(ApplicationDbContext.Create);
        app.CreatePerOwinContext<ApplicationUserManager>(ApplicationUserManager.Create);
        app.CreatePerOwinContext<ApplicationSignManager>(ApplicationSignManager.Create);

        app.UseCookieAuthentication(new CookieAuthenticationOptions
        {
            AuthenticationType = DefaultAuthenticationTypes.ApplicationCookie,
            LoginPath = new PathString("/Account/Login"),
            Provider = new CookieAuthenticationProvider
            {
                // Позволяет приложению проверять метку безопасности при входе пользователя. 
                OnValidateIdentity = SecurityStampValidator.OnValidateIdentity<ApplicationUserManager, ApplicationUser>(
                   validateInterval: TimeSpan.FromMinutes(30),
                   regenerateIdentity: (manager, user) => user.GenerateUserIdentityAsync(manager))
            }
        });
        app.UseExternalSignInCookie(DefaultAuthenticationTypes.ExternalCookie);
    }
}
```

Предворительная работа была сделана. Теперь можно созадть и контроллеры. Я создам два контроллера: HomeController, AccountController. 

HomeController будет отвечать за отображения главной страницы чата: добавления сообщений и новых пользователей. 

А вот работа с AccountController будет по интересней. Для начало создадим поля, которые будут отвечать за управления пользователями: входом и аутентификацией.

Поля:

```csharp
private ApplicationUserManager _userManager;
private ApplicationSignManager _signManager;
private IAuthenticationManager _authenticationManager;
```

Далее создадом действия *Login*. На Login будет открываться страница авторизация. 

Само действия: 

```csharp
[HttpGet]
public ActionResult Login()
{
    if (User.Identity.IsAuthenticated)
    {
        return Redirect("/Home/Index");
    }
    else
    {
        return View();
    }
}
```

Я всегда добавляю фильрт на запрос. Мне кажется так, более понятней за что отвечает данное действия, во всяком случаи что делаеть действия. И у меня просто будет два действия с одним именем и разными фильтрами.

Теперь время пришло, для того, чтобы добавить View-ку.

Здесь ничего особенного. Я буду использовать стандартные Boostrap-компоненты.

Исходный код View *Login*:

```html
@model SignalChat.Models.Account.LoginModel

@{
    ViewBag.Title = "Login";
}

@Styles.Render("~/Content/Account/")

<img id="logo" src="~/Content/Images/ToxLogo.png" />

<br />

@using (Html.BeginForm("Login", "Account", FormMethod.Post, new { @class = "login-form", role = "form" }))
{
    @Html.AntiForgeryToken()
    <br/>
    <h4 style="text-align:center;">Войдите</h4>
    <br />
    <div class="form-group">
        @Html.LabelFor(m => m.Email, "Email", new { @class = "col-md-2 control-label" })
        <div>
            @Html.TextBoxFor(m => m.Email, new { @class = "form-control", style="width:350px;margin-left:13px;"})
        </div>
    </div>

    <br />
    <div class="form-group">
        @Html.LabelFor(m => m.Password, "Пароль", new { @class = "col-md-2 control-label" })
        <div>
            @Html.PasswordFor(m => m.Password, new { @class = "form-control", style = "width:350px;margin-left:13px;" })
        </div>
    </div>

    <div class="row">
        <input type="submit" class="col-md-3 btn btn-success" style="margin-left:28px;" value="Войти" />
        @Html.ActionLink("Регистрация", "Register", "Account", new { @class = "col-md-3 act-link", style= "float:right;margin-right:38px; margin-top:10px;" })
    </div>
}
```

Давайте разберемся, что я здесь написал. Для начало я указал в качестве модели для View-ки класс *LoginModel*(y нас почти все View-ки будут строго типизированные ^_^). Далее создаю форму, которая шлет запрос на контроллер (тот самый POST запрос, для которого я ещё не неписал действия).

Интересным момент здесь является эта строчка:

```csharp
@Html.ActionLink("Регистрация", "Register", "Account", new { @class = "col-md-3 act-link", style= "float:right;margin-right:38px; margin-top:10px;" })
```

Здесь я делаю редирект на другое действия, которое отвечает за регистрацию.

Ну и вот что у нас получилось в итоге:

![asp-net-signalr-love]({{ site.url }}/images/p6/3.png)

Теперь давайте посмотрим на действия контроллера, которое проверяет валидность пользователя. Тот самый *Login* c фильтром POST:

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<ActionResult> Login(LoginModel account)
{
    if (!ModelState.IsValid)
    {
        return View(account);
    }

    Inint();

    var result = await _signManager.PasswordSignInAsync(account.Email, account.Password, false, false);

    switch (result)
    {
        case SignInStatus.Success:
            return Redirect("/Home/Index");
        default:
            ModelState.AddModelError("", "Не удачная попытка входа");
            return View();
    }
}
```

Что здесь происходит? Я проверяю валидность модели, инцилизурую служубные классы (метод *Init*) и пытауюсь зайти под пользователем.

Код метода *Init*:

```csharp
private void Inint()
{
    _userManager = HttpContext.GetOwinContext().GetUserManager<ApplicationUserManager>();
    _signManager = HttpContext.GetOwinContext().Get<ApplicationSignManager>();
    _authenticationManager = HttpContext.GetOwinContext().Authentication;
}
```

В зависимости от результата я перенаправляю пользователя или вывожу ошибку.

Так же нужно сделать View-ку для регистрации пользоватлей. Ну я уже её сделал:

```html
@model SignalChat.Models.Account.RegisterModel
@{
    ViewBag.Title = "Регистрация";
}

@Styles.Render("~/Content/Account/")

<img id="logo" src="~/Content/Images/ToxLogo.png" />

<br />

@using (Html.BeginForm("Register", "Account", FormMethod.Post, new { @class = "register-form", role = "form" }))
{
    @Html.AntiForgeryToken()
    <br />
    <h4 style="text-align:center;">Регистрация</h4>
    <br />
    <div class="form-group">
        @Html.LabelFor(m => m.Email, "Email", new { @class = "col-md-2 control-label" })
        <div>
            @Html.TextBoxFor(m => m.Email, new { @class = "form-control", style = "width:350px;margin-left:13px;" })
        </div>
    </div>
    <br />
    <div class="form-group">
        @Html.LabelFor(m => m.Password, "Пароль", new { @class = "col-md-2 control-label" })
        <div>
            @Html.PasswordFor(m => m.Password, new { @class = "form-control", style = "width:350px;margin-left:13px;" })
        </div>
    </div>
    <br />
    <div class="form-group">
        @Html.LabelFor(m => m.Password, "Повторите Пароль", new { @class = "col-md-2 control-label" })
        <div>
            @Html.PasswordFor(m => m.ConfirmPassword, new { @class = "form-control", style = "width:350px;margin-left:13px;" })
        </div>
    </div>
    <div class="form-group col-md-10">
        <input type="submit" class="col-md-3 btn btn-success" style="width:170px;" value="Зарегистрироваться" />
        @Html.ActionLink("Войти", "Login", "Account", new { @class = "col-md-3 act-link", style = "float:right;margin-right:-65px;margin-top:10px;" })
    </div>

}
```

По большей части это такая же View-ка, что и для входа, но только в качестве модели используется *RegisterModel*:

![asp-net-signalr-love]({{ site.url }}/images/p6/4.png)

По большому счету я все сделал. Осталось только реализовать сам чат.

И так вот и View-ка чата:

```html
@{
    ViewBag.Title = "Chat";
}

@Scripts.Render("~/bundles/home/")
@Styles.Render("~/Content/Home/")
<script src="~/signalr/hubs"></script>

<section class="stn-root">
    <div class="row" style="background:#6bc260;height:5%;width:1000px;margin-left:0px;">
        <div class="col-md-offset-10">
            <img src="https://mem.gfx.ms/me/MeControl/9.18199.0/msa_enabled.png" style="height: 30px; width: 30px; display:inline-block;">
            <span id="span-user" style="color:white;display:inline-block;margin-top:8px;margin-left:10px;">@ViewBag.User</span>
        </div>
    </div>
    <div class="row" style="height:100%;width:1000px;margin-left:0px;background-color:#FCFCFC;">
        <div id="div-users" class="col-sm-4" style="height:100%;overflow:auto;border-right:1px solid;">
        </div>
        <div class="col-sm-8" style="height:100%;">
            <div id="div-messages" style="height:87%;overflow:auto;">
            </div>
            <div class="row" style="height:7%;">
                <div class="col-md-10">
                    <input id="input-message" type="text" class="form-control" placeholder="Введите сообщения" />
                </div>
                <div class="col-md-1" style="margin-left:-14px;">
                    <button class="btn btn-success" onclick="onSendMessage()">Отправить</button>
                </div>
            </div>
        </div>
    </div>

</section>
```

Вот что получилось:

![asp-net-signalr-love]({{ site.url }}/images/p6/5.png)

И так что здесь интересного? Во-первых, это [Bundles](https://docs.microsoft.com/en-us/aspnet/mvc/overview/performance/bundling-and-minification). Вот код для bundles: 

```csharp
public static void RegisterBundles(BundleCollection bundles)
{
    bundles.Add(new ScriptBundle("~/bundles/home/").Include(
        "~/Scripts/jquery-{version}.js",
        "~/Scripts/Home/index.js",
        "~/Scripts/jquery.signalR-2.3.0.js"
    ));

    bundles.Add(new StyleBundle("~/Content/Account/").Include(
        "~/Content/bootstrap.css",
        "~/Content/Account/Account.css"));

    bundles.Add(new StyleBundle("~/Content/Home/").Include(
        "~/Content/bootstrap.css",
        "~/Content/Home/Index.css"));
}
```

Во-вторых, логика для добавления новых сообщений будет осуществлятся на клиенте. 

Для начала создам js-скрип в нем создам метод, который будет подписываться на hub'ы:

```javascript
function workWithHub() {
    var chat = $.connection.chatHub;
    chat.client.sendMessages = function (userName, text) {
        addMessage(userName, text);
    };
    chat.client.addUser = function (userName) {
        addNewUser(userName);
    };
    $.connection.hub.start().done(function () {
        var userName = document.getElementById('span-user').textContent;
        chat.server.login(userName);
    });
}
```

Берем наш hub, и подписываеся на методы, которые достпуны для клиента. В данном случаии это добавления новых сообщений и добавления новых пользоватлей.

Функции *addMessage()* и *addNewUser()*:

```javascript
function addMessage(userName, text) {
    var newMessage = getOtherMessage(text, userName);
    $('#div-messages').append(newMessage);
    document.getElementById('input-message').value = '';
}
```

```javascript
function addNewUser(user) {
    var newUser = getUser(user);
    $('#div-users').append(newUser);
}
```

Ну и вспомогательные методы для создания html разметки:

```javascript
function getOtherMessage(text, name) {
    return '<div style="background:#f3f3f3;display:block; border-radius:5px;margin-top:15px;margin-right:10px;">'
        + '<span style = "font-size:12px; margin-left:5px;">' + name + '</span>'
        + '<br /><span style="margin-left:5px;text-wrap:normal">' + text + '</span></div>';
}

function getUser(userName) {
    return '<div style="border-bottom:1px solid;border-top:1px solid; margin-top:5px;">'
        + '<img src = "https://mem.gfx.ms/me/MeControl/9.18199.0/msa_enabled.png" style = "height: 30px; width: 30px; display:inline-block;">'
        + '<span style="display:inline-block;margin-top:8px;margin-left:10px;">' + userName + '</span></div>';
}
```

Теперь нам надо реализовать функцию при нажатие на кнопку "Отправить":

```javascript
function onSendMessage() {
    var userName = document.getElementById('span-user').textContent;
    var text = document.getElementById('input-message').value;
    var chat = $.connection.chatHub;
    chat.server.send(userName, text);
    var newMessage = getMyMessage(text, userName);
    $('#div-messages').append(newMessage);
    document.getElementById('input-message').value = '';
}
```

И вспомогательная функция для отправки своих сообщений:

```javascript
function getMyMessage(text, name) {
    return '<div style="background:#E6F0FA;display:block; border-radius:5px;margin-top:15px;margin-right:10px;">'
        + '<span style = "font-size:12px; margin-left:5px;">' + name + '</span>'
        + '<br /><span style="margin-left:5px;">' + text + '</span></div>';
}
```

Побольшому счету вот и всё.

Посмотрим, что у нас получилось.

![asp-net-signalr-love]({{ site.url }}/images/p6/6.png)

Вот в приципе и всё. Получилось много буков, но всё же давольно интересная тема для изучения.