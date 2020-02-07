---
layout: post
title: Рецепт идеального приложения
comments: true
---

Blazor... Сколько шума наделала эта технология. Все хотели посмотреть на вторую попытку Microsoft привнести C# в создания интерфейсов. Не ужели у них это получилось? Давай разберемся.

Как всегда я создам приложения, но на этот раз не буду создавать приложения по шаблону **Blazor App**. На этот раз я создам приложения по типу ASP.NET Core MVC и добавлю в него Blazor Component.

Моё приложения будет эмитировать онлайн кинотеатр. 

Первое что надо сделать - это создать новый проект по типу Empty.

![Empty template]({{ site.url }}/images/p10/1.png) 

`При создания проект обратите особое внимания на версию .Net Core. Версия должна быть >= 3.1.x`

Далее нужно настроить приложения для работы как с инфраструктурой MVC и Blazor.

Для этого добавим в класс *Startup.cs* следующее:

~~~c#
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
~~~

Первое, что бросается в глаза (ну лично для меня) так это то, что в версии .Net Core 3.1 изменилось подключения сервисов необходимое для работы MVC.

Теперь вместо 

```c#
 services.AddMvc();
```

мы имеем

```c#
services.AddControllersWithViews();
```





