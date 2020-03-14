---
layout: post
title: Рецепт идеального приложения
comments: true
---

Blazor... Сколько шума наделала эта технология. Все хотели посмотреть на вторую попытку Microsoft привнести C# в создания интерфейсов. Не уже ли у них это получилось? Давай разберемся.

Как всегда, я создам приложения, но на этот раз не буду создавать приложения по шаблону **Blazor App**. На этот раз я создам приложения по типу ASP.NET Core MVC и добавлю в него Blazor Component.

Моё приложения будет эмитирует онлайн кинотеатр. 

Первое что надо сделать - это создать новый проект по типу Empty.

![Empty template]({{ site.url }}/images/p10/1.png) 

`При создании проект обратите особое внимания на версию .Net Core. Версия должна быть >= 3.1.x`

Далее нужно настроить приложения для работы как с инфраструктурой MVC и Blazor.

Для этого добавим в класс *Startup.cs* следующее:

{% highlight c# %}
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews();
        services.AddServerSideBlazor();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseStaticFiles();
        app.UseRouting();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllerRoute(
                name: "default",
                pattern: "{Controller=Home}/{Action=Index}"
            );
            endpoints.MapBlazorHub();
        });
    }
 }
{% endhighlight %}

Первое, что бросается в глаза (ну лично для меня) так это то, что в версии .Net Core 3.1 изменилось подключения сервисов необходимое для работы MVC.

Теперь вместо

{% highlight c# %}
 services.AddMvc();
{% endhighlight %}

мы имеем

{% highlight c# %}
services.AddControllersWithViews();
{% endhighlight %}

Второе нововведение — это изменение настройки маршрутизации.

В .Net Core 2.x настройка маршрутизации проходила следующим образом:

{% highlight c# %}
app.UseMvc(routers =>
{
    routers.MapRoute(
        name: null,
        template: "{controller=Home}/{action=Index}/{id?}");
});
{% endhighlight %}

В новой версии .Net Core настройка маршрута происходит следующим способом:

{% highlight c# %}
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        name: "default",
        pattern: "{Controller=Home}/{Action=Index}"
    );
    endpoints.MapBlazorHub();
});
{% endhighlight %}

Как по мне такой вариант инициализации машрутов лучше отображается действительность. По мимо MVC приложения также имеются и другие типы приложения. Например: Razor Pages, Web API и Blazor. Именно, поэтому настройка маршрута через метод, который называется *UseMvc()*, не логична.

Далее необходимо подготовить основную модель для онлайн кинотеатра, которая будет отображаться реальный фильм:

{% highlight c# %}
public class Movie
{
    public int Id { get; set; }

    public string Name { get; set; }

    public string Description { get; set; }

    public MovieType Type { get; set; }
}
{% endhighlight %}

Так же надо ввести типы имеющихся фильмов. Это можно сделать с помощью *MovieType*:

{% highlight c# %}
public enum MovieType
{
    Action,
    Western,
    Comedy,
    Horror
}
{% endhighlight %}

Далее нужно реализовать отображения фильмов, которые имеются в кинотеатре. Для этого воспользуемся Blazor Component. Компонент будет принимать модель фильма, отображать имя и описания конкретного фильма ну и с эмитируем постер фильма (для этого я буду отображать "заглушку"). Так же компонент должен реагировать на взаимодействия с мышью, т.е. изменить при наведения мыши. 

В итоге получилось так:

{% highlight html %}
@inject NavigationManager NavigationManager

<div class="card w-25 p-1 m-1 @Class" @onmouseover="OnMouseOver" @onmouseout="OnMouseOut" @onclick="@(()=>OnClick(Model.Id))">
    <svg class="bd-placeholder-img card-img-top" width="100%" height="180"
         xmlns="http://www.w3.org/2000/svg" preserveAspectRatio="xMidYMid slice"
         focusable="false" role="img" aria-label="Placeholder: Image cap">
        <title>Placeholder</title>
        <rect width="100%" height="100%" fill="#868e96"></rect>
        <text x="40%" y="50%" fill="#dee2e6" dy=".3em">Image</text>
    </svg>
    <div class="card-body movie-body">
        <h5 class="card-title">@Model.Name</h5>
        <p class="card-text">@Model.Description</p>
    </div>
</div>

@code {
    [Parameter]
    public Movie Model { get; set; }

    public string Class { get; set; }

    public void OnMouseOver() => Class = "movie-selected";

    public void OnMouseOut() => Class = "";

    public void OnClick(int id) => NavigationManager.NavigateTo($"/movie/card?movieId={id}", true);
}
{% endhighlight %}

Из исходного кода компонента видно, что он не представляет ничего сложного. В данном компоненте происходит обработка таких событий как: onmouseover, onmouseout и onclick. 

После написания данного компонента нужно его проверить. Для этого запустим приложения ..., и оно не работает. Точнее будет, если я скажу, что компонент, который я написал работает только на первой страницы. 

Поиске на этот вопрос не дали ничего. Мне даже пришлось [спросить](https://github.com/dotnet/aspnetcore/issues/18588#event-2986623146) у команды, которая занимается разработкой Blazor. 

И после ответа от команды Blazor я понял в чем была моя ошибка. Оказывается, что бы Blazor Component работал во всем приложении нужно указать специальный тэг:

{% highlight html %}
<base href="~/" />
{% endhighlight %}

Результат:

![Movie card]({{ site.url }}/images/p10/2.gif)

После этого я реализовал фильтрацию фильмов по категориям (по типам) и открытие "карточки" фильма. Для этого я создал *MovieController.cs*, которые имеет следующее методы:

{% highlight c# %}
public class MovieController : Controller
{
    private readonly ICinemaRepository _repository;
    public MovieController(ICinemaRepository repository)
        => _repository = repository;

    public IActionResult List(MovieType type)
    {
        ViewBag.Title = type;
        return View(_repository.GetMovies(type));
    }

    public IActionResult Card(int movieId)
    {
        var movie = _repository.GetMovie(movieId);
        ViewBag.Title = movie.Name;
        return View(movie);
    }
}
{% endhighlight %}

Код реализации репозитория можно посмотреть в [GitHub](https://github.com/denmaklucky/Examples/blob/master/BlazorServer/OnlineCinema/OnlineCinema/Models/CinemaRepository.cs).

У каждого фильма есть свои оценки/комментарии от пользователей. Я хочу реализовать что-то подобное и в моем приложение. Для начало создадим сущность, которая будет отображать комментарии для определенного фильма. Данная сущность показана ниже

{% highlight c# %}
public class Comment
{
    public int Id { get; set; }

    public int MovieId { get; set; }

    public string Author { get; set; }

    public string Value { get; set; }

    public DateTime CreatedOn { get; set; }
}
{% endhighlight %}

Так же я реализую контроллер, который будет возвращать комментарии для определённого фильма и добавлять их. Ниже показать код контроллера *CommentController.cs*: 
{% highlight c# %}
public class CommentController : Controller
{
    private readonly ICinemaRepository _repository;
    public CommentController(ICinemaRepository repository)
        => _repository = repository;

    public IEnumerable<Comment> GetComments(int movieId)
        => _repository.GetComments(movieId);

    [HttpPost]
    public IActionResult AddComment([FromBody]Comment comment)
    {
        _repository.AddComment(comment);
        return Ok();
    }
}
{% endhighlight %}

Так же нужно реализовать компонент для комментариев. Он должен уметь отображать существующее комментарии к определённому фильму, в карточке которого я сейчас нахожусь, и дать возможность пользователю добавлять новые комментарии. Вот что получилось:

{% highlight html %}
@using System.Net.Http
@inject IHttpClientFactory HttpClientFactory

<EditForm Model="@NewComment" OnSubmit="OnCreateNewComment">

    <div class="form-group">
        <label>Author</label>
        <InputText class="form-control" @bind-Value="NewComment.Author" />
    </div>

    <div class="form-group">
        <label>Comment</label>
        <InputTextArea class="form-control" @bind-Value="@NewComment.Value" />
    </div>

    <div class="float-right">
        <button class="btn btn-success" type="submit">Add</button>
    </div>

</EditForm>

@if (Comments == null)
{
    <span>Loading...</span>
}
else
{
    @foreach (var comment in Comments)
    {
        <div>
            <div>
                <span class="comment-author">@comment.Author</span>
                <span class="comment-createdate">@comment.CreatedOn.ToShortDateString()</span>
            </div>
            <div>@comment.Value</div>
        </div>

        <br />
    }
}

@code {
    private HttpClient _client;

    public List<Comment> Comments { get; set; }

    [Parameter]
    public int MovieId { get; set; }

    public Comment NewComment { get; set; } = new Comment();

    public async Task OnCreateNewComment()
    {
        NewComment.CreatedOn = DateTime.Now;
        NewComment.MovieId = MovieId;
        Comments.Add(NewComment);

        var result = await _client.PostAsync("/Comment/AddComment", 
            new StringContent(Newtonsoft.Json.JsonConvert.SerializeObject(NewComment), System.Text.Encoding.UTF8, "application/json"));

        NewComment = new Comment();
    }

    protected override async Task OnInitializedAsync()
    {
        _client = HttpClientFactory.CreateClient("Cinema Api");

        var response = await _client.GetAsync($"/Comment/GetComments?movieId={MovieId}");
        var content = await response.Content.ReadAsStringAsync();

        Comments = Newtonsoft.Json.JsonConvert.DeserializeObject<List<Comment>>(content);
    }
}
{% endhighlight %}

Как видно из кода выше *CommentsComponent.razor* представляет из себя EditForm и список уже существующих комментариев. EditForm - это стандартный компонент, который представляет из себя надстройку над существующей form из Html.Это надстройка позволяет реагировать на submit только, когда Model прошла валидацию (OnValidSubmit="") или наоборот, когда модель не прошла валидцию (OnInvalidSubmit=""). Эти методы дают возможность контролировать внешний вид компонента при валидации модели. Но в я использую услвоную нажатие на кнопку без проверки входящей модели.

Так же в секции @code видно два метода, которые взаимодействуют с сервером при помощи HttClient. Первое взаимодействия происходит в методе *OnInitializedAsync*. В данном методе происходит запрос всех комментариев по данному фильму. Второе взаимодействия происходит в методе *OnCreateNewComment*. В этом методе я отправляю новый комментарий на сервер.

Ниже можно посмотреть результат того:

![Comments]({{ site.url }}/images/p10/3.gif)

Вместо вывода:

Blazor+ASP.NET Core MVC сдружить можно, но у меня возникли некоторые проблемы с этим. Первое с чем я столкнулся это то, что в к версии .NET Core < 3.1.x есть проблема при передачи параметра в компонент, т.е. при таком виде 

{% highlight c# %}
@(await Html.RenderComponentAsync<MovieCard>(RenderMode.ServerPrerendered, new { Model = movie }))
{% endhighlight %}

происходило исключения. Это проблема решается путем обновления .NET Core до версии 3.1. 

Вторая не приятная ситуация, с которой пришлось столкнуться — это необходимость указания тэга в мастере страниц, для того чтобы компоненты Blazor работали во всем приложение.

В целом мне очень понравилось комбинировать ASP.NET Core MVC и Blazor Component. Надеюсь, при написание вашего проекта у вас не возникнет таких проблем.

[Ссылка на репозиторий](https://github.com/denmaklucky/Examples/blob/master/BlazorServer/OnlineCinema/).