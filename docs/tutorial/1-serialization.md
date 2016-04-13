# Часть 1: Сериализация

## Введение

В  данном параграфе мы обсудим создание простого Web API для сервиса подсветки синатксиса фрагментов кода наподобии `pastebin`. Здесь будет представлено описание некоторых компонетов REST фреймворка, которое позволит вам составить представление о том как они работают и взаимодействуют.

Описание здесь приводится довольно подробное, так что, перед тем как начать изучение, запаситесь печенюшками и кружечкой вашего любимого напитка. Если вам будет достаточно только общего описания, то тогда лучше почитайте [быстрый старт][quickstart].

---

**Note**: Весь представленный код доступен на GitHub [tomchristie/rest-framework-tutorial][repo]. Также доступен работающей [демо-сервис][sandbox].

---

## Настрока окружения

Для начала создадим виртуальное окружение, используя [virtualenv]. Виртуальное окружение позволяет изолировать используемые нами пакеты от других проектов, над которыми мы работаем.

    virtualenv env
    source env/bin/activate

Теперь мы внутри нашего окружения и можем установить необходимые пакеты.

    pip install django
    pip install djangorestframework
    pip install pygments  # We'll be using this for the code highlighting

**Note:** Для того чтобы выйти из виртуального окружения, просто выполните команду `deactivate`.  Для получения более подробной информации рекоммендуем прочитать документацию по [virtualenv][virtualenv].

## Начало работы

Итак, теперь мы готовы.
Давайте создадим новый проект, с которым будем в дальнейшем работать.

    cd ~
    django-admin.py startproject tutorial
    cd tutorial

Создадим новое приложение, в котором и будем создавать наш простенький Web API.

    python manage.py startapp snippets

Добавим нашу аппликуху `snippets` и `rest_framework` в переменную `INSTALLED_APPS`.
Отредактируйте файл `tutorial/settings.py` следующим образом:

    INSTALLED_APPS = (
        ...
        'rest_framework',
        'snippets.apps.SnippetsConfig',
    )

## Создание модели

Для решения поставленной в данной обучалке задачи мы создадим модель `Snippet`, которая будет использоваться для хранения фрагментов кода. Отредактируем файл `snippets/models.py`.  Note: Хороший программист всегда оставляет комментарии в своем коде. Все комментарии к приведенному ниже коду вы найдете в репозитории данного приложения, здесь же мы решили опустить все комментарии, для того чтобы сосредоточиться именно на коде.

    from django.db import models
    from pygments.lexers import get_all_lexers
    from pygments.styles import get_all_styles

    LEXERS = [item for item in get_all_lexers() if item[1]]
    LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
    STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


    class Snippet(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        title = models.CharField(max_length=100, blank=True, default='')
        code = models.TextField()
        linenos = models.BooleanField(default=False)
        language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
        style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

        class Meta:
            ordering = ('created',)

Создадим миграцию для нашей новой модели и синхронизируем её базой данных.

    python manage.py makemigrations snippets
    python manage.py migrate

## Создание класса сериализатора

Для создания Web API необходимо предусмотреть способ сериализации и десериализации нашего объекта, например, в `json` представление. Для этого давайте создадим сериализаторы, которые по принципу работы очень похожи на Django-формы. В директории `snippets` создадим файл `serializers.py` и скопируем в него следующее содержимое.

    from rest_framework import serializers
    from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


    class SnippetSerializer(serializers.Serializer):
        pk = serializers.IntegerField(read_only=True)
        title = serializers.CharField(required=False, allow_blank=True, max_length=100)
        code = serializers.CharField(style={'base_template': 'textarea.html'})
        linenos = serializers.BooleanField(required=False)
        language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
        style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

        def create(self, validated_data):
            """
            Create and return a new `Snippet` instance, given the validated data.
            """
            return Snippet.objects.create(**validated_data)

        def update(self, instance, validated_data):
            """
            Update and return an existing `Snippet` instance, given the validated data.
            """
            instance.title = validated_data.get('title', instance.title)
            instance.code = validated_data.get('code', instance.code)
            instance.linenos = validated_data.get('linenos', instance.linenos)
            instance.language = validated_data.get('language', instance.language)
            instance.style = validated_data.get('style', instance.style)
            instance.save()
            return instance

В начале класса сериализатора мы описываем сериализуемые поля. Методы `create()` и `update()` будут вызваны при создании объекта или его модификации при вызове метода `serializer.save()`.

Класс сериализатора очень похож на класс Django `Form`. Описываеме в нем поля также могут содержать такие проверочные аттрибуты как `required`, `max_length` и `default`.

Аттрибуты полей также позволяют управлять отображением поля в сгенерированном HTML-представлении. Например, аттрибут `{'base_template': 'textarea.html'}` явлется эквивалентом `widget=widgets.Textarea` класса `Form`. Чуть позже мы увидим как использовать данные аттрибуты для управления отображением полей в браузерном API.

Мы можем сэкономить немного времени используя уже имеющиеся особенности класса `ModelSerializer`, но давайте пока оставим все как есть, в явном виде.

## Работа с сериализатором

Преждем чем мы двинемся дальше давайте немного поподробнее разберемся с классом сериализатора. Перейдите в режим интерактивной работы с Django.

    python manage.py shell
    
Ок, после того как мы сделали несколько импортов, создадим парочку сниппетов.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser

    snippet = Snippet(code='foo = "bar"\n')
    snippet.save()

    snippet = Snippet(code='print "hello, world"\n')
    snippet.save()

Теперь у нас есть несколько объектов с фрагменетами кода. Давайте взглянем на сериализованные данные из этих объектов.

    serializer = SnippetSerializer(snippet)
    serializer.data
    # {'pk': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}

Наш объект был преобразован к стандартным питоновским типам данных. Для преобразования полученных данных в `json` воспользуемся слудеющим кодом:

    content = JSONRenderer().render(serializer.data)
    content
    # '{"pk": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'

Процесс десериализации аналогичен. Сначала мы преобразовываем данные к стандартым питовским типам...

    from django.utils.six import BytesIO

    stream = BytesIO(content)
    data = JSONParser().parse(stream)

...а уже затем преобразуем их в объект

    serializer = SnippetSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # OrderedDict([('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
    serializer.save()
    # <Snippet: Snippet object>

Стоит отметить, что это очень похоже на API, используемое при работе с формами. Сходство станет еще более очевидным, когда мы начнем писать представления для нашего сериализатора.

Также мы можем сериализовать не только объект модели, но и их набор. Для этого необходимо добавить аргумент `many=True` в констурктор сериализатора.

    serializer = SnippetSerializer(Snippet.objects.all(), many=True)
    serializer.data
    # [OrderedDict([('pk', 1), ('title', u''), ('code', u'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('pk', 2), ('title', u''), ('code', u'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('pk', 3), ('title', u''), ('code', u'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]

## Использование класса ModelSerializers

Наш класса `SnippetSerializer` содержит много дублирующей информации, содержащейся в модели `Snippet`. Было бы неплохо, если бы наш код стал немного компактнее и короче.

Подобно тому как в Django реализованы классы `Form` и `ModelForm`, в REST-фреймворке реализованы классы `Serializer` и `ModelSerializer`.

Давайте взглянем на класс нашего сниппета, переписанный с использованием класса `ModelSerializer`.
Снова откройте файл `snippets/serializers.py` и замените старое описание класса `SnippetSerializer` на новое.

    class SnippetSerializer(serializers.ModelSerializer):
        class Meta:
            model = Snippet
            fields = ('id', 'title', 'code', 'linenos', 'language', 'style')

У сетриализатора есть одна полезная особенность - вы можете просмотреть все его поля, для этого достаточно вывести его представление. Откройте Django-консоль `python manage.py shell` и введите следующий код:

    from snippets.serializers import SnippetSerializer
    serializer = SnippetSerializer()
    print(repr(serializer))
    # SnippetSerializer():
    #    id = IntegerField(label='ID', read_only=True)
    #    title = CharField(allow_blank=True, max_length=100, required=False)
    #    code = CharField(style={'base_template': 'textarea.html'})
    #    linenos = BooleanField(required=False)
    #    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
    #    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...

Важно запомнить что класс `ModelSerializer` не содержит ни какой магии, а просто позволяет создавать сериализаторы используя сокращенную форму:

* Автоматически определяет набор необходимых полей.
* Содержит дефолтные методы `create()` и `update()`.

## Создание стандарнтых представлений Django для нашего сериализатора

Давайте посмотрим как же мы можем написать вьюхи для нашего API, используя наш сериализатор.
Пока что мы не будем использовать дополнительных возможностей REST-фреймворка и будем писать обычные вьюхи Django.

Начнем с того что напишем класс, наследуемый от HttpResponse, и предназначенный для обработки данных и выдачи их в формате `json`.

Отредактируем файл `snippets/views.py` и добавим в него следующий код.

    from django.http import HttpResponse
    from django.views.decorators.csrf import csrf_exempt
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer

    class JSONResponse(HttpResponse):
        """
        An HttpResponse that renders its content into JSON.
        """
        def __init__(self, data, **kwargs):
            content = JSONRenderer().render(data)
            kwargs['content_type'] = 'application/json'
            super(JSONResponse, self).__init__(content, **kwargs)

Пусть корнем нашего API будет служить вьюха, возвращающая список всех сниппетов или создающая новый сниппет.

    @csrf_exempt
    def snippet_list(request):
        """
        List all code snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return JSONResponse(serializer.data)

        elif request.method == 'POST':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data, status=201)
            return JSONResponse(serializer.errors, status=400)

Поскольку мы хотим выполнять POST запросы от клиентов, не имеющих CSRF-токена, мы декорируем нашу вьюху с помощью `csrf_exempt`.  Конечно на продакшене не стоит этого делать, но для нашей задачи это допустимо.

Создадим вьюху, которая будет выдавать информацию об отдельном сниппете, а кроме того может обновлять его или удалять.

    @csrf_exempt
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a code snippet.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return HttpResponse(status=404)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return JSONResponse(serializer.data)

        elif request.method == 'PUT':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(snippet, data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data)
            return JSONResponse(serializer.errors, status=400)

        elif request.method == 'DELETE':
            snippet.delete()
            return HttpResponse(status=204)

Теперь необходимо связать наши вьюхи с url-адресами. Создайте файл `snippets/urls.py`:

    from django.conf.urls import url
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
    ]

Также добавьте конфиг с новыми адресами в корневой конфиг `tutorial/urls.py`.

    from django.conf.urls import url, include

    urlpatterns = [
        url(r'^', include('snippets.urls')),
    ]

Стоит отметить, что в нашем коде есть места, которые могут отрабатывать не самым подходящим образом, попросту выдавать ошибку в определенной ситуации. Например, если вьюха получит искаженный `json` или будет использовать метод запроса, который вьюха не умеет обрабатывать, то мы получим ошибку сервера 500 "server error".  Тем не менее, оставим все как есть.

## Тестирование нашего Web API

Теперь мы можем запустить простой сервер, который будет обрабатывать наши сниппеты.

Выйдите из интерактивного режима...

	quit()

...и запустите отладочный сервер Django.

	python manage.py runserver

	Validating models...

	0 errors found
	Django version 1.8.3, using settings 'tutorial.settings'
	Development server is running at http://127.0.0.1:8000/
	Quit the server with CONTROL-C.

Откройте вторую консоль и уже в ней отправляйте запросы к серверу.

Для тестирования нашего API мы можем использовать консольные утилиты [curl][curl] или [httpie][httpie]. Httpie это простой и удобный http-клиент, написанный на Python. Давайте установим его.

Устаноите httpie с помощью pip:

    pip install httpie

Получим список всех сниппетов:

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

Или же мы можем получить отдельный сниппет, указав его id:

    http http://127.0.0.1:8000/snippets/2/

    HTTP/1.1 200 OK
    ...
    {
      "id": 2,
      "title": "",
      "code": "print \"hello, world\"\n",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

Если мы перейдем по ссылке в браузере то, увидим что будет выведен тот же самый json.

## Что же мы сделали

Мы сделали API для сериализации, причем выглядит оно очень похожим на API для форм Django, а также создали несколько стандарных вьюх.

Вьюхи для нашего API не делают ничего особенного, только обработка `json` запросов и обработка некоторых ошибок, но тем не менее мы имеем работающий Web API.

[Во второй части][tut-2] мы увидим как можно улучшить наш код.

[quickstart]: quickstart.md
[repo]: https://github.com/tomchristie/rest-framework-tutorial
[sandbox]: http://restframework.herokuapp.com/
[virtualenv]: http://www.virtualenv.org/en/latest/index.html
[tut-2]: 2-requests-and-responses.md
[httpie]: https://github.com/jakubroztocil/httpie#installation
[curl]: http://curl.haxx.se
