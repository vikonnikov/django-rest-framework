# Часть 2: Запросы и Ответы

Итак, начиная с этого момента, мы действительно начинаем погружаться в работу с ядром Django REST фреймворка.
Давайте поговорим о нескольких основоплагающих компонентах.

## Объекты запроса

REST фреймворк использует объект `Request`, унаследованный от стандартного `HttpRequest`, и предоставляет более гибкие возможности обработки запросов. Основной функционал объекта `Request` заложен в аттрибут `request.data`, напоминающий `request.POST`, но более удобный для работы с Web API.

    request.POST  # Only handles form data.  Only works for 'POST' method.
    request.data  # Handles arbitrary data.  Works for 'POST', 'PUT' and 'PATCH' methods.

## Объекты ответа

В REST фреймворке используется объект `Response`, унаследованный от `TemplateResponse`, содержащий неотрендеренный контент и в зависимости от запрашиваемого клиентом типа контента ему возвращется соответсвующий ответ.

    return Response(data)  # Renders to content type as requested by the client.

## Коды состояния

Использование числовых кодов ошибок не всегда удобно и понятно. Поэтому в REST фреймворке предусметрен более явный способ описания ошибок, например, `HTTP_400_BAD_REQUEST` в модуле `status`.

## Обертки для API-вьюх

В REST фреймворке предусмотрены две обертки позволяющие вам создавать вьюхи для API.


1. Декоратор `@api_view` для работы с вьюхами-функциями.
2. Класс `APIView` для работы с вьюхами на базе классов.

Данные обертки предоставляют некоторый функционал, который позволяет передавать в ваши вьюхи объекты `Request`и добавляет контекст в объекты `Response`, который в зависимости от типа запарашиваемого контента вернет нам данные в нужном формате.

Также обертки позволяют контролировать доступ к методам вьюхи и возвращают `405 Method Not Allowed` когда это необходимо, кроме того возвращают исключение `ParseError`, возникающее при обращении к `request.data`, если на вход были переданы неверные данные.

## Объединение всех компонентов

Ок, давайте продвинемся дальше и воспользуемся перечисленными компонентами для того чтобы написать несколько вьюх.

Нам больше не нужен класс `JSONResponse` из модуля `views.py`, поэтому мы можем смело его удалить.  А теперь давайте пеерпишем наш код.

    from rest_framework import status
    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer


    @api_view(['GET', 'POST'])
    def snippet_list(request):
        """
        List all snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        elif request.method == 'POST':
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

Наша вьюха это не более чем улучшенный вариант предыдущей версии примера. Этот код выглядит немного компактнее и очень похож на код, который бы мы использовали для работы с API Django форм. Кроме того мы используем именованные коды состояний, что позволяет сделать наш код более поняным.

А вот вьюха для отдельного сниппета:

    @api_view(['GET', 'PUT', 'DELETE'])
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a snippet instance.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        elif request.method == 'PUT':
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        elif request.method == 'DELETE':
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

Все это выглядит очень знакомым, поскольку не сильно отличается от работы со стандарными вьюхами Django.

Отметим что больше мы не указываем нашим ответам и запросам тип контента. `request.data` может обрабатывать входящие запросы как с `json` так и с другими форматами данных. Точно также мы возвращаем объект ответа с данными и REST фреймворк сам рендерит и возвращает нам данные в нужном для нас формате.

## Добавляем суффиксы форматирования в URLs 

Для того чтобы возвращаемые ответы содержали данные в нужном для нас формате, давайте добавим поддержку суффиксов форматирования.  Суффиксы форматирования задаются в URL адресах, которые может обрабатывать наш API, например, [http://example.com/api/items/4/.json][json-url].

Давайте добавим ключевой аргумент `format` в обе наши вьюхи.

    def snippet_list(request, format=None):

and

    def snippet_detail(request, pk, format=None):

Немного откорректируем файл `urls.py` и в дополнение к нашим адресам добавим `format_suffix_patterns`.

    from django.conf.urls import url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

Нам не нужно добавлять какие-то новые паттерны для адресов, этот способ максимально прост и позволяет нам задавать формат возвращаемых данных довольно очевидным способом.

## Как это выглядит?

Go ahead and test the API from the command line, as we did in [tutorial part 1][tut-1].  Everything is working pretty similarly, although we've got some nicer error handling if we send invalid requests.

We can get a list of all of the snippets, as before.

    http http://127.0.0.1:8000/snippets/

    HTTP/1.1 200 OK
    ...
    [
      {
        "id": 1,
        "title": "",
        "code": "foo = \"bar\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      },
      {
        "id": 2,
        "title": "",
        "code": "print \"hello, world\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      }
    ]

We can control the format of the response that we get back, either by using the `Accept` header:

    http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
    http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML

Or by appending a format suffix:

    http http://127.0.0.1:8000/snippets.json  # JSON suffix
    http http://127.0.0.1:8000/snippets.api   # Browsable API suffix

Similarly, we can control the format of the request that we send, using the `Content-Type` header.

    # POST using form data
    http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

    {
      "id": 3,
      "title": "",
      "code": "print 123",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

    # POST using JSON
    http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

    {
        "id": 4,
        "title": "",
        "code": "print 456",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

Теперь откройте ваше API в браузере, перейдя по ссылке [http://127.0.0.1:8000/snippets/][devserver].

### Браузабельность =D

Because the API chooses the content type of the response based on the client request, it will, by default, return an HTML-formatted representation of the resource when that resource is requested by a web browser.  This allows for the API to return a fully web-browsable HTML representation.

Having a web-browsable API is a huge usability win, and makes developing and using your API much easier.  It also dramatically lowers the barrier-to-entry for other developers wanting to inspect and work with your API.

See the [browsable api][browsable-api] topic for more information about the browsable API feature and how to customize it.

## Что дальше?

В [части 3][tut-3] вы узнаете о том как использовать представления на основе классов, а также увидите как использование базовых представлений позволяет уменьшить количество кода.

[json-url]: http://example.com/api/items/4/.json
[devserver]: http://127.0.0.1:8000/snippets/
[browsable-api]: ../topics/browsable-api.md
[tut-1]: 1-serialization.md
[tut-3]: 3-class-based-views.md
