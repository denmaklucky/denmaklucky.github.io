---
layout: post
title: Full version of MasterViewControl
comments: true
---

После релиза [UWP](https://docs.microsoft.com/ru-ru/windows/uwp/get-started/whats-a-uwp) Microsoft показала как должны выглядить современные приложения на Windows 10. И так вопрос: что должно быть у приложения, чтобы оно соответствовало универсальной платформе? Канечно же **адаптивный пользовательский интерфейс**, так как UWP может запускаться на различных устройствах, с различной диагональю экрана (или [без](https://denmaklucky.github.io/windows-10-iot-core/)).

![one_platform]({{ site.url }}/images/p4/5.png "UWP")

 Например, для рассмотрения я взял стандартный клиент для почты. У него имеется два состояния (на самом деле у данного клиента есть три состояния, но для моего примера важны только два):

"Narrow"

![standart_client_state_1]({{ site.url }}/images/p4/1.png "Narrow status")

"Wide"

![standart_client_state_2]({{ site.url }}/images/p4/2.png "Wide status")

При этом, в разных состояниях меняется и **UX** ~~мастер UI/UX~~:

"Narrow"

![standart_client_state_2]({{ site.url }}/images/p4/6.gif "Narrow")

"Wide"

![standart_client_state_2]({{ site.url }}/images/p4/7.gif "Wide")

Покопаясь в интернете я нашел, что-то... А точнее описание такого паттерна поведения GUI, который называется [Master/details pattern](https://docs.microsoft.com/ru-ru/windows/uwp/design/controls-and-patterns/master-details).

![master_details_pattern]({{ site.url }}/images/p4/4.png "Adaptive layout")

Т.е. есть уже готовое решение, которое можно использовать для написания приложения на UWP. Нужно всего лишь добавить шаблон, пару стилей и "вуаля" - всё готово. Но не всё так просто, как казалось на первый взгляд.
Первом делом я расмотрел то, что предлагает Microsoft. А именно [Master/details sample](https://github.com/Microsoft/Windows-universal-samples/tree/master/Samples/XamlMasterDetail).

Дальше решение от Microsoft. Слабонервных попрошу уйти.

```xml 
<Grid x:Name="LayoutRoot" Loaded="LayoutRoot_Loaded">
    <VisualStateManager.VisualStateGroups>
        <VisualStateGroup x:Name="AdaptiveStates" CurrentStateChanged="AdaptiveStates_CurrentStateChanged">
            <VisualState x:Name="DefaultState">
                <VisualState.StateTriggers>
                    <AdaptiveTrigger MinWindowWidth="720" />
                </VisualState.StateTriggers>
            </VisualState>

        <VisualState x:Name="NarrowState">
            <VisualState.StateTriggers>
                   <AdaptiveTrigger MinWindowWidth="0" />
            </VisualState.StateTriggers>

            <VisualState.Setters>
                <Setter Target="MasterColumn.Width" Value="*" />
                <Setter Target="DetailColumn.Width" Value="0" />
                <Setter Target="MasterListView.SelectionMode" Value="None" />
            </VisualState.Setters>
        </VisualState>
        </VisualStateGroup>
    </VisualStateManager.VisualStateGroups>

        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <Grid.ColumnDefinitions>
            <ColumnDefinition x:Name="MasterColumn" Width="320" />
            <ColumnDefinition x:Name="DetailColumn" Width="*" />
        </Grid.ColumnDefinitions>

        <TextBlock
            Text="My Items"
            Margin="12,8,8,8"
            Style="{ThemeResource TitleTextBlockStyle}" />

        <ListView .../>

        <ContentPresenter .../>
</Grid>
```
Теперь можно выдохнуть.

Давайте поставим все точки над **i**. Что нужно делать-то?
Взять Grid. Разбить этот Grid на несколько колонок. В нем определить VisualStateManager, которые будут реагировать на изменения размера. И в случае, когда ширина Grid'a будет меньше определенного размера, "захлопнуть" колонку с DetailView и показать DetailViewPage.

Почему это решение нельзя назвать полноценным `MasterDetailView`?

Во-первых, **ContentPresenter'y** был задан определенный шаблон, и из-за этого в DetailView можно поместить только элемент из ListView. Следовательно, можно забыть об использовании **CommandBar'a** в MasterView.
Во-вторых, так как **ContentPresenter** представляет из себя обычный контейнер для отображения содержимого ListView, то можно забыть о навигации внутри DetailView (как это реализовано в стандартном почтовом клиенте). 
Microsoft как-бы сказали:"Вот так вот должно выглядить приложение. Вот такой паттерн нужно использовать. Но готового контрола для реализации этого паттерна в приложениях у нас нет, но вы можете его сделать сами! Никто Вам не мешает."

Ок. Я так и сделаю.

Что мне нужно от `full MasterDetailView`? 

Во-первых, у данного представления должно быть два рабочих состояния, как у стандартного клиента для почты.
Во-вторых, в данном представлении должны отсутсвовать недочеты, которые есть в примере от Microsoft.

Может кто-нибудь уже написал что-то похожее? Откроем исходники [Unigram](https://github.com/UnigramDev/Unigram) ~~потомучто!~~.
Поведения приложения Unigram меня устраевает, но их решения слишком неоправданно сложное (кому интересно, может посмотреть в исходники). Разработчики данного приложения используют `NavigationService`, что для приложения с нестрогой MVVM архитектурой необязательно.

Ну что же? Раз решил давай напиши. 

Мне нужен контрол, состоящий из двух частей: MasterView и DetailView. При этом в MastrView'e должен находится не только ListView, а любой элемент, который разработчик захочет. А DatailView должен содержать **Frame**, чтобы можно осуществлять навигацию по страницам.
Для начала возьмем и создадим класс, который назавем, ну например, `MasterDetailView` и который наследуется от ContentControl. Почему наш класс должен наследоваться от **ContentControl**? В MasterDetailView у нас будет **ContentPresenter**. А как говорил один ~~легендарный~~ .net разработчик: "Лучший контейнер для ContentPresenter'a - это ContentControl".

В конструкторе я пропишу следующее:

```csharp
public MasterDetailView()
{
    DefaultStyleKey = typeof(MasterDetailView);

    Loaded += OnLoaded;
    Unloaded += OnUnloaded;
}

```

Для чего нам нужны обработчики для событий `Loaded` и `Unloaded`? В этих обработчиках я буду подписываться (и отписываться) на события кнопки **Back**, примерно так.

```csharp
SystemNavigationManager.GetForCurrentView().BackRequested += OnBackRequested;
```

В методе `OnBackRequested()` я буду переходить на предыдущую страницу.
Что нам ещё нужно? Мне нужно, чтобы внешний пользователь,тот кто использует мой контрол мог узнать текущие состоние. Для этого создадим обычный `enum`, который характеризует эти состояния.

```csharp
public enum MasterDetailVisualState : byte
{
    Narrow,
    Wide
}
```

А в классе `MasterDetailView` будет свойство **CurrentState**, которое показывает текущее состояние. 
Теперь нам надо менять свойства CurrentState при изменении в VisualStateGroup. Для этого переопределим метод
`OnApplyTemplate()` и я напишу в нем следующее.

```csharp
protected override void OnApplyTemplate()
{
    _masterPresenter = (ContentPresenter)GetTemplateChild("MasterPresenter");
    _detailPresenter = (Frame)GetTemplateChild("DetailPresenter");
    _stateGroup = (VisualStateGroup)GetTemplateChild("AdaptiveStates");
    
    _stateGroup.CurrentStateChanged += OnCurrentStateChanged;
    CurrentState = _stateGroup.CurrentState.Name == "WideState" ? MasterDetailVisualState.Wide :
    MasterDetailVisualState.Narrow;
}
```

Здесь я подписываюсь на событие **CurrentStateChanged** у VisualStateGroup. Данное событие возникает, когда происходит изменение состония. 
В методе `OnCurrentStateChanged()` меняю значение у свойства CurrentState. Его реализация

```csharp
private void OnCurrentStateChanged(object sender, VisualStateChangedEventArgs e)
{
    if(e.NewState.Name == "WideState")
    {
        CurrentState = MasterDetailVisualState.Wide;
    }

    if(e.NewState.Name == "NarrowState")
    {
        CurrentState = MasterDetailVisualState.Narrow;
    }
    
    UpdateView();
    ViewStateChanged?.Invoke(this, EventArgs.Empty);
}
```

Далее нужно реализовать метод, который бы изменял видимость у **_masterPserenter** и **_detailPresenter**.  
В нем не будет ничего сложного. Назову-ка я его `UpdateView()`.

```csharp
private void UpdateView()
{
    if (CurrentState == MasterDetailVisualState.Filled)
    {
        _masterPresenter.Visibility = Visibility.Visible;
        _detailPresenter.Visibility = Visibility.Visible;
    }
    
    if (CurrentState == MasterDetailVisualState.Narrow && _detailFrame.Content == null)
    {
        _masterPresenter.Visibility = Visibility.Visible;
        _detailPresenter.Visibility = Visibility.Collapsed;
    }
    
    if (CurrentState == MasterDetailVisualState.Filled && _detailFrame.Content != null)
    {
        _masterPresenter.Visibility = Visibility.Collapsed;
        _detailPresenter.Visibility = Visibility.Visible;
    }
}
```

Ой! Чуть не забыл показать реализацию метода `OnBackRequested()`.

```csharp
private void OnBackRequested(object sender, BackRequestedEventArgs e)
{
    if (CurrentState == MasterDetailVisualState.Filled)
    {
        if (_detailFrame.CanGoBack)
        {
            _detailFrame.GoBack();
        }
    }
    else
    {
        if (_detailFrame.BackStack.Count > 1)
        {
            if (_detailFrame.CanGoBack)
            {
                _detailFrame.GoBack();
            }
        }
        else
        {
            _detailPresenter.Visibility = Visibility.Collapsed;
            _masterPresenter.Visibility = Visibility.Visible;
            _detailFrame.BackStack.Clear();
        }
    }
}
```

Теперь пожалуй один из главных моментов, так как у меня не используется **NavigationService**, то моему контролу нужно как-то переходить на другие страницы. 
Для это я создам в классе MasterDetailView метод `Navigate()`, который принимал был тип страницы и параметры, которые необходимы для перехода.

```csharp
public void Navigate(Type typePage, object args)
{
    UpdateView();
    _detailFrame.Navigate(typePage, args);
}
```

Этот метод не идеален, но для примера сойдет.
Теперь нужно определить шаблона для моего контрола. Определю-ка я его в нутри стиля.

```xml
<Style TargetType="Controls:MasterDetailView">
        <Setter Property="BorderBrush" Value="{ThemeResource SystemControlForegroundBaseLowBrush}"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Controls:MasterDetailView">
                    <Grid Background="{ThemeResource ApplicationPageBackgroundThemeBrush}">
                        <VisualStateManager.VisualStateGroups>
                            <VisualStateGroup x:Name="AdaptiveStates">
                                <VisualState x:Name="WideState">
                                    <VisualState.StateTriggers>
                                        <AdaptiveTrigger MinWindowWidth="820"/>
                                    </VisualState.StateTriggers>
                                    <VisualState.Setters>
                                        <Setter Target="MasterColumn.Width" Value="260*" />
                                        <Setter Target="DetailColumn.Width" Value="540*" />
                                        <Setter Target="MasterColumn.MinWidth" Value="72" />
                                        <Setter Target="MasterColumn.MaxWidth" Value="540" />
                                        <Setter Target="DetailPresenter.(Grid.Column)" Value="1"/>
                                    </VisualState.Setters>
                                </VisualState>
                                <VisualState x:Name="NarrowState">
                                    <VisualState.StateTriggers>
                                        <AdaptiveTrigger MinWindowWidth="0"/>
                                    </VisualState.StateTriggers>
                                </VisualState>
                            </VisualStateGroup>
                        </VisualStateManager.VisualStateGroups>
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition x:Name="MasterColumn"/>
                            <ColumnDefinition x:Name="DetailColumn" Width="0"/>
                        </Grid.ColumnDefinitions>
                        <ContentPresenter x:Name="MasterPresenter"
                                          Content="{TemplateBinding Content}"
                                          ContentTemplate="{TemplateBinding ContentTemplate}"
                                          ContentTemplateSelector="{TemplateBinding ContentTemplateSelector}"
                                          ContentTransitions="{TemplateBinding ContentTransitions}"
                                          DataContext="{TemplateBinding DataContext}"
                                          HorizontalContentAlignment="Stretch"
                                          VerticalContentAlignment="Stretch"/>

                        <Frame x:Name="DetailPresenter" Background="{TemplateBinding Background}">
                            <TextBlock Text="Ничего не выбрано, пожалуйста выберите" HorizontalAlignment="Center"  VerticalAlignment="Center"/>
                        </Frame>
                    </Grid>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
</Style>
```
Давай-те посмотрим на шаблон по внимательней. Что я изменил?

Во-первых, в качестве **MasterView** выступает ContentPresenter. А это означает, что в нем можно поместить всё что угодно.
Во-вторых, в качестве **DetailView** выступает Frame (да здравствует навигация без NavigationService).

Итог:

"Narrow"

![full_masterdetailview]({{ site.url }}/images/p4/8.gif "Narrow status")

"Wide"

![full_masterdetailview]({{ site.url }}/images/p4/9.gif "Wide status")

Чуть не забыл, я добавил капельку [акрила](https://docs.microsoft.com/ru-ru/windows/uwp/design/style/acrylic).