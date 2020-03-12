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