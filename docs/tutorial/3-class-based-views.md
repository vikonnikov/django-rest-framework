# Часть 3: Представления на основе классов

Также мы можем разрабатывать представления для нашего API в виде классов, что более предпочтительно чем вьюхи-функции. Представления в виде классов имеют дополнительный встроенный функционал и позволяют придеживаться принципа [DRY][dry].

## Переработка API на базе представлений в виде классов

Давайте перепишем наше корневое представление в виде класса. Для этого потребуется немного изменить наш код в файле `views.py`.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from django.http import Http404
    from rest_framework.views import APIView
    from rest_framework.response import Response
    from rest_framework import status


    class SnippetList(APIView):
        """
        List all snippets, or create a new snippet.
        """
        def get(self, request, format=None):
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        def post(self, request, format=None):
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

Похоже на наш предыдущий код, но, как вы могли заметить, мы разделили наш код по типу HTTP методов. Поправим код представления для отдельного сниппета `views.py`.

    class SnippetDetail(APIView):
        """
        Retrieve, update or delete a snippet instance.
        """
        def get_object(self, pk):
            try:
                return Snippet.objects.get(pk=pk)
            except Snippet.DoesNotExist:
                raise Http404

        def get(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        def put(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        def delete(self, request, pk, format=None):
            snippet = self.get_object(pk)
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

Выглядит хорошо и про прежнему наш код похож на представления-функции.

Поскольку мы используем представления на основе классов нам придеться немного изменить файл `urls.py`.

    from django.conf.urls import url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.SnippetList.as_view()),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

Готово. Если теперь запустить отладочный сервер, то можно увидеть, что все работает точно также, как и до рефракторинга.

## Использование примесей

Возможность повторного использования реализованной логики, является одним из преимуществ использования представлений-классов.

Операции создания/получения/обновления/удаления являются общими для любого API, работающего с моделями и объектами данных. Поэтому подобный функционал уже реализован в REST фреймворке в классах примесей.

Давайте посмотрим как мы можем использовать примеси в нашем коде м снова отредактируем файл `views.py`.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import mixins
    from rest_framework import generics

    class SnippetList(mixins.ListModelMixin,
                      mixins.CreateModelMixin,
                      generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.list(request, *args, **kwargs)

        def post(self, request, *args, **kwargs):
            return self.create(request, *args, **kwargs)

Мы унаследовали наше представление от класса `GenericAPIView`, а также добавили еще два класса `ListModelMixin` и `CreateModelMixin`.

Класс `GenericAPIView` обеспечивает весь базовый функционал, а `ListModelMixin` и `CreateModelMixin` добавляют два метода `.list()` и `.create()`.  Данные методы мы явным образом связываем с методами `get` и `post`, которые отрабатывают при запросах соответсвующего типа.  Все достаточно просто.

    class SnippetDetail(mixins.RetrieveModelMixin,
                        mixins.UpdateModelMixin,
                        mixins.DestroyModelMixin,
                        generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        def put(self, request, *args, **kwargs):
            return self.update(request, *args, **kwargs)

        def delete(self, request, *args, **kwargs):
            return self.destroy(request, *args, **kwargs)

Подобным образом мы правим `SnippetDetail`. Снова мы используем класс `GenericAPIView` для обеспечения базового функионала представления и добавляем примиси, в которых реализованы методы `.retrieve()`, `.update()` and `.destroy()`.

## Использование вьюх на основе базовых представлений REST фреймворка

При использовании примесей наш код стал немного компактее и короче, давайте сделаем еще один шаг.
В REST фреймворке уже предусмотрено несколько базовых представлений, сочетающих в себе функционал примесей. Снова отредактируем файл `views.py`.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import generics


    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer


    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

Ничего себе, это просто невероятно короткий и читый код, вполне соответсвующий принципам Django.

Далее перейдите к [части 4][tut-4], из неё вы узнаете о авторизации и правах доступа к нашему  API.

[dry]: http://en.wikipedia.org/wiki/Don't_repeat_yourself
[tut-4]: 4-authentication-and-permissions.md
