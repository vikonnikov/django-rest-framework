# Часть 5: Использование ссылочного связывания в API

В данный момент связи в нашем API представлены первичными ключами. В данной части, используя ссылочные связи, мы сделаем API чуть более явным и наглядным.

## Создание корневой точки входа API

Сейчас у нас есть точки входа для сниппетов 'snippets' и пользователей 'users', но у нас нет единой точки входа для API. Для её создания воспользуемся стандарным представлением на базе функции с описанным чуть ранее декоратором `@api_view`. Добавим в файл `snippets/views.py` следующее:

    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from rest_framework.reverse import reverse


    @api_view(['GET'])
    def api_root(request, format=None):
        return Response({
            'users': reverse('user-list', request=request, format=format),
            'snippets': reverse('snippet-list', request=request, format=format)
        })

Стоит обратить внимание на 2 момента. Во-первых, мы используем функцию `reverse`, возвращающую полностью определенные url-адреса; во-вторых, именованные паттерны, по которым формируются url-адреса, будут описаны чуть позже в файле `snippets/urls.py`. 

## Создание точек входа для подсвечиваемых сниппетов

В нашем приложении отсутствует API для оображения сниппетов с подсвеченным синтаксисом.

В отличии от всех остальных отчек входа API, нам не нужено возвращать JSON, вместо этого нам нужно получить HTML. REST фреймворк позволяет отрендерить HTML двумя способами: первый - используя шаблоны, и второй - используя заранее сгенерированный HTML. Мы воспользуемся вторым способом.

The other thing we need to consider when creating the code highlight view is that there's no existing concrete generic view that we can use.  We're not returning an object instance, but instead a property of an object instance.

При создании представления для отображения подсвеченного кода мы не сможем воспользоваться каким-то конкретным представлением из числа доступных, поскольку наше представление должно возвращать не объект, а его атрибут `snippet.highlighted`.

Вместо использования какого-то конкретного общего представления, мы воспользуемся базовым классом `generics.GenericAPIView` (от которого унаследованы общие представления) и переопределим метод `.get()`. Отредактируем модуль `snippets/views.py`:

    from rest_framework import renderers
    from rest_framework.response import Response

    class SnippetHighlight(generics.GenericAPIView):
        queryset = Snippet.objects.all()
        renderer_classes = (renderers.StaticHTMLRenderer,)

        def get(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

И как обычно после создания представлений нам необходимо добавить их в наш URLconf.
В модуль `snippets/urls.py` мы добавим url для  корневой точки входа:

    url(r'^$', views.api_root),

Также добавим url для подсветки синтаксиса сниппета:

    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),

## Использование ссылочного связывания в API

Dealing with relationships between entities is one of the more challenging aspects of Web API design.  There are a number of different ways that we might choose to represent a relationship:

Организация взаимосвязями между объектами является одним из наболее сложных аспектов проектирования Web API. Существует несколько способов отражения связи между объектами.

* Использование первичных ключей
* Использование ссылок между объектами
* Использование уникальных индетифицирущих полей со слагами для связанного объекта
* ИСпользование некторого строкового предстваления связанного объекта
* Встравивание связанного объекта в родительский
* Некоторые другие кастомные представления

REST framework supports all of these styles, and can apply them across forward or reverse relationships, or apply them across custom managers such as generic foreign keys.

In this case we'd like to use a hyperlinked style between entities.  In order to do so, we'll modify our serializers to extend `HyperlinkedModelSerializer` instead of the existing `ModelSerializer`.

The `HyperlinkedModelSerializer` has the following differences from `ModelSerializer`:

* It does not include the `pk` field by default.
* It includes a `url` field, using `HyperlinkedIdentityField`.
* Relationships use `HyperlinkedRelatedField`,
  instead of `PrimaryKeyRelatedField`.

We can easily re-write our existing serializers to use hyperlinking. In your `snippets/serializers.py` add:

    class SnippetSerializer(serializers.HyperlinkedModelSerializer):
        owner = serializers.ReadOnlyField(source='owner.username')
        highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

        class Meta:
            model = Snippet
            fields = ('url', 'highlight', 'owner',
                      'title', 'code', 'linenos', 'language', 'style')


    class UserSerializer(serializers.HyperlinkedModelSerializer):
        snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

        class Meta:
            model = User
            fields = ('url', 'username', 'snippets')

Notice that we've also added a new `'highlight'` field.  This field is of the same type as the `url` field, except that it points to the `'snippet-highlight'` url pattern, instead of the `'snippet-detail'` url pattern.

Because we've included format suffixed URLs such as `'.json'`, we also need to indicate on the `highlight` field that any format suffixed hyperlinks it returns should use the `'.html'` suffix.

## Именование URL-паттернов

If we're going to have a hyperlinked API, we need to make sure we name our URL patterns.  Let's take a look at which URL patterns we need to name.

* The root of our API refers to `'user-list'` and `'snippet-list'`.
* Our snippet serializer includes a field that refers to `'snippet-highlight'`.
* Our user serializer includes a field that refers to `'snippet-detail'`.
* Our snippet and user serializers include `'url'` fields that by default will refer to `'{model_name}-detail'`, which in this case will be `'snippet-detail'` and `'user-detail'`.

After adding all those names into our URLconf, our final `snippets/urls.py` file should look like this:

    from django.conf.urls import url, include
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    # API endpoints
    urlpatterns = format_suffix_patterns([
        url(r'^$', views.api_root),
        url(r'^snippets/$',
            views.SnippetList.as_view(),
            name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$',
            views.SnippetDetail.as_view(),
            name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
            views.SnippetHighlight.as_view(),
            name='snippet-highlight'),
        url(r'^users/$',
            views.UserList.as_view(),
            name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$',
            views.UserDetail.as_view(),
            name='user-detail')
    ])

    # Login and logout views for the browsable API
    urlpatterns += [
        url(r'^api-auth/', include('rest_framework.urls',
                                   namespace='rest_framework')),
    ]

## Добавляем разделение на страницы

The list views for users and code snippets could end up returning quite a lot of instances, so really we'd like to make sure we paginate the results, and allow the API client to step through each of the individual pages.

We can change the default list style to use pagination, by modifying our `tutorial/settings.py` file slightly.  Add the following setting:

    REST_FRAMEWORK = {
        'PAGE_SIZE': 10
    }

Note that settings in REST framework are all namespaced into a single dictionary setting, named 'REST_FRAMEWORK', which helps keep them well separated from your other project settings.

We could also customize the pagination style if we needed too, but in this case we'll just stick with the default.

## Просмотр API в браузере

If we open a browser and navigate to the browsable API, you'll find that you can now work your way around the API simply by following links.

You'll also be able to see the 'highlight' links on the snippet instances, that will take you to the highlighted code HTML representations.

In [part 6][tut-6] of the tutorial we'll look at how we can use ViewSets and Routers to reduce the amount of code we need to build our API.

[tut-6]: 6-viewsets-and-routers.md
