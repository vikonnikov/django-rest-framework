# Часть 6: ViewSets и Routers

REST фреймворк содержит класс `ViewSets`, значительно облегчающий разработку API. Класс `ViewSets` автоматизирует процесс связывания url-паттернов с представленями на основе некоторых общих правил и позволяет разработчику сосредоточиться на моделировании состояний и взаимодействии API.

Классы `ViewSet` очень похожи на обычные представления `View`. Отличие состоит в том, что у `ViewSet` нет методов `get` или `put`, в место этого в них реализован механизм обработки операций `read` или `update`.

Связывание обработчиков и url-паттернов происходит в тот момент, когда `ViewSet` инициализируется в набор представлений. За конфигурирование url-ов отвечает класс `Router`.

## Рефакторинг кода с использованием ViewSets

Давайте отрефакторим наши представления с использованием набора представлений `ViewSet`.

Для начала отрефакторим представления `UserList` и `UserDetail` и объеденим их функционал в одном представлении `UserViewSet`. Удалим оба представления и заменим их одним классом:

    from rest_framework import viewsets

    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        This viewset automatically provides `list` and `detail` actions.
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

В данном случае мы мспользовали класс `ReadOnlyModelViewSet`, обеспечивающий работу только в 'read-only' режиме.  Мы по прежнему указываем аттрибуты `queryset` и `serializer_class` точно также как и при использовании обычных представлений,   теперь нам не нужно дублировать данные атрибуты в двух разных классах.

Next we're going to replace the `SnippetList`, `SnippetDetail` and `SnippetHighlight` view classes.  We can remove the three views, and again replace them with a single class.

    from rest_framework.decorators import detail_route

    class SnippetViewSet(viewsets.ModelViewSet):
        """
        This viewset automatically provides `list`, `create`, `retrieve`,
        `update` and `destroy` actions.

        Additionally we also provide an extra `highlight` action.
        """
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer
        permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                              IsOwnerOrReadOnly,)

        @detail_route(renderer_classes=[renderers.StaticHTMLRenderer])
        def highlight(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

        def perform_create(self, serializer):
            serializer.save(owner=self.request.user)

Мы использовали класс `ModelViewSet` с уже встроенным функционалом операций чтения и записи.

При создания кастомного метода `highlight` мы использовали декоратор `@detail_route`. Данный декоратор используются для создания точек входа API, отличающихся от стандартных `create`/`update`/`delete`.

Методы, созданные с помощью декоратора `@detail_route`, отвечают на `GET` запросы. Для того чтобы метод отвечал на `POST` запросы, в декоратор необходимо добавить аргумент `methods`.

Url-паттерны для дополнительных методов, зависят от имени метода. Для изменения url-паттерна, автоматически генерируемого при создании кастомного метода, можно передать в декоратор именованный аргумент `url_path`.

## Явное связывание ViewSets с URLs

Давайте создадим все представления `ViewSet` вручную и обычным способом свяжем их с url-паттернами в файле `urls.py`.

    from snippets.views import SnippetViewSet, UserViewSet, api_root
    from rest_framework import renderers

    snippet_list = SnippetViewSet.as_view({
        'get': 'list',
        'post': 'create'
    })
    snippet_detail = SnippetViewSet.as_view({
        'get': 'retrieve',
        'put': 'update',
        'patch': 'partial_update',
        'delete': 'destroy'
    })
    snippet_highlight = SnippetViewSet.as_view({
        'get': 'highlight'
    }, renderer_classes=[renderers.StaticHTMLRenderer])
    user_list = UserViewSet.as_view({
        'get': 'list'
    })
    user_detail = UserViewSet.as_view({
        'get': 'retrieve'
    })

Из каждого класса `ViewSet` мы создаем несколько представлений и связываем методы данных представлений с HTTP-методами.

Теперь зарегистрируем наши предствления в URL-конфиге.

    urlpatterns = format_suffix_patterns([
        url(r'^$', api_root),
        url(r'^snippets/$', snippet_list, name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
        url(r'^users/$', user_list, name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
    ])

## Использование роутеров

Because we're using `ViewSet` classes rather than `View` classes, we actually don't need to design the URL conf ourselves.  The conventions for wiring up resources into views and urls can be handled automatically, using a `Router` class.  All we need to do is register the appropriate view sets with a router, and let it do the rest.

Here's our re-wired `urls.py` file.

    from django.conf.urls import url, include
    from snippets import views
    from rest_framework.routers import DefaultRouter

    # Create a router and register our viewsets with it.
    router = DefaultRouter()
    router.register(r'snippets', views.SnippetViewSet)
    router.register(r'users', views.UserViewSet)

    # The API URLs are now determined automatically by the router.
    # Additionally, we include the login URLs for the browsable API.
    urlpatterns = [
        url(r'^', include(router.urls)),
        url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

Registering the viewsets with the router is similar to providing a urlpattern.  We include two arguments - the URL prefix for the views, and the viewset itself.

The `DefaultRouter` class we're using also automatically creates the API root view for us, so we can now delete the `api_root` method from our `views` module.

## Trade-offs between views vs viewsets

Using viewsets can be a really useful abstraction.  It helps ensure that URL conventions will be consistent across your API, minimizes the amount of code you need to write, and allows you to concentrate on the interactions and representations your API provides rather than the specifics of the URL conf.

That doesn't mean it's always the right approach to take.  There's a similar set of trade-offs to consider as when using class-based views instead of function based views.  Using viewsets is less explicit than building your views individually.

## Результаты нашей работы

With an incredibly small amount of code, we've now got a complete pastebin Web API, which is fully web browsable, and comes complete with authentication, per-object permissions, and multiple renderer formats.

We've walked through each step of the design process, and seen how if we need to customize anything we can gradually work our way down to simply using regular Django views.

You can review the final [tutorial code][repo] on GitHub, or try out a live example in [the sandbox][sandbox].

## Открываем новые горизонты

We've reached the end of our tutorial.  If you want to get more involved in the REST framework project, here are a few places you can start:

* Contribute on [GitHub][github] by reviewing and submitting issues, and making pull requests.
* Join the [REST framework discussion group][group], and help build the community.
* Follow [the author][twitter] on Twitter and say hi.

**Now go build awesome things.**


[repo]: https://github.com/tomchristie/rest-framework-tutorial
[sandbox]: http://restframework.herokuapp.com/
[github]: https://github.com/tomchristie/django-rest-framework
[group]: https://groups.google.com/forum/?fromgroups#!forum/django-rest-framework
[twitter]: https://twitter.com/_tomchristie
