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

Поскольку мы используем `ViewSet` раньше чем `View`, нам нет необходимости самим конфигурировать URL-ы. Для автоматизации связывания представлений и url-ов используется класс `Router`, для этого достаточно зарегистрировать наши представления в роутере.

Так должен выглядеть переписанный нами модуль `urls.py`.

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

Региcтрация `ViewSet` с помощью `Router` похожа на задание url в `urlpattern`. Мы указываем два аргумента - префикс для URL и пердставление.

Класс `DefaultRouter` автоматически создает корневую точку входа для нашего API, поэтому мы можем удалить метод `api_root` из модуля `views.py`.

## Что же лучше views или viewsets

`ViewSet`  позволяет не только минимизировать количество кода, но и следовать определенным соглашениям при описании url-ов.  `ViewSet` и `Router` позволяют автоматизировать конфигурирование URL-ов, что позволяет разработчику сосредоточиться именно на функционале API.

Но это конечно не означает что использование `ViewSet` является наиболее предпочтительным. Существуют случаи когда лучше использовать обычные представления вместо наборов. Описание представлений по одному является более явным способом чем использование наборов `ViewSet`.

## Результаты нашей работы

Написав совсем немного кода мы получили Web API для сохранения фрагментов кода, с возможностью работы из браузера, авторизацией, разграничением доступа и несколькими форматами отдаваемых данных.

Мы прошли все этапы процесса проектирования и рассмотрели способы доработки стандарных представлений Django под наши задачи.

Код нашего приложения вы найдете на [GitHub][repo], также доступна [демка][sandbox] сервиса.

## Открываем новые горизонты

Вот мы и дошли до конца нашего руководства. Приведем еще несколько ресурсов, где вы сможете продолжить ваше знакомство с REST-фреймворком:

* Contribute on [GitHub][github] by reviewing and submitting issues, and making pull requests.
* Join the [REST framework discussion group][group], and help build the community.
* Follow [the author][twitter] on Twitter and say hi.

**Теперь давайте сделаем что-нибудь грандиозное.**


[repo]: https://github.com/tomchristie/rest-framework-tutorial
[sandbox]: http://restframework.herokuapp.com/
[github]: https://github.com/tomchristie/django-rest-framework
[group]: https://groups.google.com/forum/?fromgroups#!forum/django-rest-framework
[twitter]: https://twitter.com/_tomchristie
