---
layout: post
title: Рецепт идеального приложения 2
comments: true
---

При изучение ASP.NET MVC у меня всегда возникал вопрос: 

`А почему в качестве "веб-морды" нельзя использовать клиента, написаного спомощью синтаксика Razor?`

И на протяжении нескольких лет это было нельзя сделать.

Собственно, а теперь это можно сделать с помощью [Blazor Server](https://docs.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-3.1#blazor-server).

Создадим 3 звенное приложение: **клиент**- **сервер** - **база данных** (на самом деле будет 2 звенное приложение без БД).

В качестве клиента будет выступать Blazor Server, в качестве "бэка" будет выступать ASP.NET Core 3.1.

Для илюстрации возможностей Blazor Server в качестве веб-клиента, я создам приложения для работы с заметками.

Для начало создам сервер. Для создания сервера буду использовать шаблон "Empty".

![Empty template]({{ site.url }}/images/p11/1.png)

Далее необходимо создать инфруструктуру для работы приложения как Web Api. Для этого я создам директорию */Models*. Понятно из название, что в данной папке будет лежат данные, которые необходимы для приложения.

# Web Api

## Models

Основным сущностью моего приложения будет класс *Note*. Данный класс будет прдествлять из себя заметку.

{% highlight c#%}
    public class Note
    {
        public int Id { get; set; }

        public string Title { get; set; }

        public string Description { get; set; }
    }
{% endhighlight %}

Сущность заметки не представляет из себя ничего сложного:
 id - индетификатор объекта в БД
 title - наименования объекта
 description - "тело" заметке

## Repository

Далее нужно созадть механизм для работы с заметками. Для этого я создам репозиторий.

{% highlight c#%}
    public interface INoteRepository
    {
        //Возвращение всех заметок
        Task<List<Note>> GetAllNotes(CancellationToken token);

        //Возвращение конкретной заметки (по индентификатору)
        Task<Note> GetNote(int noteId, CancellationToken token);

        //Добавление или обновление заметки
        Task AddOrUpdateNote(Note note, CancellationToken token);

        //хз что😁
        Task DeleteNote(int noteId, CancellationToken token);
    } 
{% endhighlight %}

Реализацию самого репозитория можно посмотреть по [ссылки](https://github.com/denmaklucky/Examples/blob/master/BlazorServer/Notes/Note.WebApi/Models/Repositories/NoteRepsitory.cs).

## Controller

Без лишних слов, код контроллера

{% highlight c#%}
    [ApiController, Route("api/notes")]
    public class NotesController : ControllerBase
    {
        private readonly INoteRepository _repository;
        public NotesController(INoteRepository repository)
            => _repository = repository;

        [HttpGet, Route("list")]
        public Task<List<Models.Note>> GetAllNotes(CancellationToken token)
            => _repository.GetAllNotes(token);

        [HttpGet, Route("{noteId}")]
        public Task<Models.Note> GetNote(int noteId, CancellationToken token)
            => _repository.GetNote(noteId, token);

        [HttpPost, Route("createOrUpdate")]
        public async Task<IActionResult> GetNote(Models.Note note, CancellationToken token)
        {
            await _repository.AddOrUpdateNote(note, token);
            return Ok();
        }
        
        [HttpDelete, Route("delete/{noteId}")]
        public async Task<IActionResult> DeleteNote(int noteId, CancellationToken token)
        {
            await _repository.DeleteNote(noteId, token);
            return Ok();
        }
    }
{% endhighlight %}

Здесь следует упоменуть, что при создание контроллера я использовал аттрибут *Route*. Из этого следует, что при создания маршрута не нужно указывать шаблон для маршрута.

## Startup

Startup - это настройка сервисов необходимых для работы приложения, настройка самого приложения , и настройка среды. В данном случаи *Startup* представляет из себя почти базовую настройку приложения.

{% highlight c#%}
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddTransient<INoteRepository, NoteRepsitory>();
            services.AddControllers();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
{% endhighlight %}

---

Время проверки моей "Апишки" через [Postman](https://www.postman.com/).

![Web Api]({{ site.url }}/images/p11/2.gif)

Отлично! Контроллер отдает данные.

Но в текущей реализации есть один недостаток. Если я вызову мето `api/notes/note/{number}`, где number - индетификатор, которого нет, то я получу не красивую ошибку.

![Error]({{ site.url }}/images/p11/3.png)

Такая поробная информация очень полезная при отладки и её нужно записать в лог, но она абсалютна не нужна для клинета.

Для того, что выводить клиенту нужное сообщение я буду использовать **Middleware**. Суть промежуточного ПО в этом случаии, что я буду вызывать метод из MVC и ловить исключени, которые возникнуть входе извления Action'a.

Microsoft любезно предоставила возможность создавать Middleware как угодно. Еденственное ограничение: пользовательский Middleware должен принимать **RequestDelegate** и иметь метод с сигнатурой:

{% highlight c#%}
 Task Invoke(HttpContext context)
{% endhighlight %}

Вот само промежуточное ПО:

{% highlight c#%}
    public class HttpExceptionMiddleware
    {
        private readonly RequestDelegate _next;
        public HttpExceptionMiddleware(RequestDelegate dlt)
            => _next = dlt;

        public async Task Invoke(HttpContext context)
        {
            try
            {
                await _next(context);
            }
            catch (Exception ex)
            {
                await context.Response.WriteAsync(ex.Message);
            }
        }
    }
{% endhighlight %}

И добавляеи наш Middleware в класс Startup.cs:

{% highlight c#%}
 app.UseMiddleware<HttpExceptionMiddleware>();
{% endhighlight %}

И как результат я имею красивое сообщения для клиента об ошибке.

![The nice message for client]({{ site.url }}/images/p11/4.png)

# Client

Как я писал выше в качестве клиента я буду использовать Blazor Server. 

Для начала создадим проект на .NET Core 3.1 по типу `Empty`.

Первое что надо сделать настроить его для работы с Blazor. Для этого я внесу изменение в класс *Startup*.

{% highlight c#%}
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddRazorPages();
            services.AddServerSideBlazor();
            services.AddHttpClient();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();
            app.UseStaticFiles();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapFallbackToPage("/_Host");
                endpoints.MapBlazorHub();
            });
        }
    }
{% endhighlight %}

После этого нужно созадть директорию *Pages*. В данной директории будет храниться хост для запуска компонентов Blazor. 

Для того, чтобы все компоненты могли использовать одинаковые сборки, без директивы using в начале файла, нужно создать файл *_Imports.razor*.

После внесенных изменений решение выглядит следующим образом:

![The solution]({{ site.url }}/images/p11/5.png)

Я подготовил проект для старта и запуска Blazor. Далее нужно реализовать сам интерфейс. 

Я хочу сделать приложение похожее на изображение ниже:

![The design]({{ site.url }}/images/p11/5.png)

Слева у меня будет часть, где будут отображться всё существующее заметки. Справа будет карточка выделенной заметки.

Судя по всему мне нужно реализовать 3 компонента: лист с заметками, компонент самой заметки и карточка заметки.

`Компонент заметки - это то, как заметка будет представлена в списки, а карточка заметки - это отображение заметки, когда пользователь "кликает" на неё`

Первым делом я создам компонент *NoteComponent.razor*, который будет отвечать за отображение заметки в списки. Данный компонент будет принимать в качестве параметра объект *Note*, будет менять стиль при взаимодействии с указателем мыши, и будет выкидывать событие о клике.

Без слишних слов, код:

{% highlight html %}
<div class="@Class border-bottom" @onmouseover="OnMouseOver" @onmouseout="OnMouseOut" @onclick="@(()=>OnMouseClick(Model.Id))"
     pag-action="Edit" page-model="@Model.Id">
    <div class="note-title">
        @Model.Title
    </div>
    <div class="note-body">
        @Model.Description
    </div>
</div>

@code {

    public void OnMouseOver() => Class = "clr-selected";

    public void OnMouseOut() => Class = "";

    public Task OnMouseClick(int noteId) => OnClickNoteCallBack.InvokeAsync(Model);

    [Parameter]
    public Note Model { get; set; }

    [Parameter]
    public EventCallback<Note> OnClickNoteCallBack { get; set; }

    public string Class { get; set; }
}
{% endhighlight %}

Далее необходимо реализовать компонент, который отображаем список всё заметок, но для начало нужно реализовать сервис для "общение" с WebApi.

Сервис должен получать данные от WebApi по http, для этого ему нужен *HttpClient*. Т.к. при Dispose объекта HttpClient не всегда освобождается порт, я воспользует новой возможностью, который появился в .Net Core, IHttpClientFactory.

У HttpClient нет методов, которые позволели мне отправлять и получать данные от сервера в формате Json. Для того, чтобы получить данные в нужном мне формате мне пришлось бы сначала сделать запрос к сервера, затем считать контент и только потом десериализовать его. А т.к. сервис, в клиенте, будет неоднократно получать и отправлять данные серверу, то придется писать данный код много раз. Чтобы этого не далть я вынес данный код в методы расшения для HttCliet.

{% highlight c#%}
    public static class HttpClientExtensions
    {
        public static async Task<T> GetJsonAsync<T>(this HttpClient httpClient, string requestedUrl)
        {
            var response = await httpClient.GetAsync(requestedUrl);

            if (!response.IsSuccessStatusCode)
                throw new HttpRequestException($"Can't call {requestedUrl}; Status code {response.StatusCode}");

            var content = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject<T>(content);
        }

        public static async Task<bool> PostJsonAsync<T>(this HttpClient httpClient, string requestedUrl, T obj)
        {
            var stringContent = new StringContent(JsonConvert.SerializeObject(obj), Encoding.UTF8, "application/json");
            var response = await httpClient.PostAsync(requestedUrl, stringContent);
            return response.IsSuccessStatusCode;
        }
    }
{% endhighlight %}

Сам сервис, который находится в клиенте:

{% highlight c#%}
    public class NoteService : INoteService, IDisposable
    {
        //Адрес сервера
        private const string Server = "https://localhost:44350/api/";
        private readonly HttpClient _httpClient;

        public NoteService(IHttpClientFactory httpFactory)
            => _httpClient = httpFactory.CreateClient();


        public async Task CreateOrUpdate(Note note)
        {
            const string method = "notes/createOrUpdate";
            await _httpClient.PostJsonAsync<Note>($"{Server}{method}", note);
        }

        public Task<List<Note>> GetNotes()
        {
            const string method = "notes/list";
            return _httpClient.GetJsonAsync<List<Note>>($"{Server}{method}");
        }

        public async Task Delete(int noteId)
        {
            const string method = "notes/delete/";
            await _httpClient.DeleteAsync($"{Server}{method}{noteId}");
        }

        public void Dispose()
            => _httpClient?.Dispose();
    }
{% endhighlight %}

Как по мне выглядит очень сиспотично.

В самом компоненте нет ничего сложного. Если интеренсо то реализацию можно посмотреть [тут](https://github.com/denmaklucky/Examples/blob/master/BlazorServer/Notes/Notes/Components/NotePage.razor).

Вот и результат:

![The list of notes]({{ site.url }}/images/p11/7.gif)

Все что осталось для того, что реализовть то что я хотел изначально, так это карточку заметки.

В карточке нужно отображать свойства объекта такие как: Title, Description. Так же в карточке должна быть возможнасть управлять конретной заметки, а именно обновлять или удалять её.

Всё это реализовано [здесь](https://github.com/denmaklucky/Examples/blob/master/BlazorServer/Notes/Notes/Components/EditNote.razor).

Результат карточки:

![The list of notes]({{ site.url }}/images/p11/8.gif)

Заключение:

Наконец-то появилась возможность реализовывать Web-клиента на Razor.

Из основных недостатков можно отметить, что общение между сервером (где сидит клиент) и самим представление проиходит через html. Если реализовать интрефейс, в котором есть 2^10 строчек в таблице. И при каждом клике добавляется ещё столько же. В этом случаи сервер будет гонять **мегабайты** html.

Как всегда ссылка на [репозиторий](https://github.com/denmaklucky/Examples/tree/master/BlazorServer/Notes).