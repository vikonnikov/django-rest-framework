# Быстрый старт

Итак, мы собираемся создать простой набор API-функций, который позволил бы пользователям с правами адмиистратора просматривать, редактировать или удалять других пользователей или группы. Под пользователями и группами бы подразумеваем стандарные джанговские объекты классов `User` и `Group`

## Настройка проекта

Создайте новый Django проект `tutorial` и приложение `quickstart`

    # Создадим директорию проекта
    mkdir tutorial
    cd tutorial

    # С помощью virtualenv создадим изолированную среду для нашего проекта
    virtualenv env
    source env/bin/activate  # On Windows use `env\Scripts\activate`

    # Установим Django и Django REST framework
    pip install django
    pip install djangorestframework

    # Создадим новый проект и новое приложение
    django-admin.py startproject tutorial .  # Note the trailing '.' character
    cd tutorial
    django-admin.py startapp quickstart
    cd ..

Структура нашего проекта будет выглядеть следующим образом:

    $ pwd
    <some path>/tutorial
    $ find .
    .
    ./manage.py
    ./tutorial
    ./tutorial/__init__.py
    ./tutorial/quickstart
    ./tutorial/quickstart/__init__.py
    ./tutorial/quickstart/admin.py
    ./tutorial/quickstart/apps.py
    ./tutorial/quickstart/migrations
    ./tutorial/quickstart/migrations/__init__.py
    ./tutorial/quickstart/models.py
    ./tutorial/quickstart/tests.py
    ./tutorial/quickstart/views.py
    ./tutorial/settings.py
    ./tutorial/urls.py
    ./tutorial/wsgi.py

Создание приложения внутри директории проекта может показаться не обычным, но использование пространства имен проекта позволяет избежать конфликта имен с внешним модулем (обсуждение данной темы выходит за рамки данного руководства).

Мигрируйте текущую схему базы данных:

    python manage.py migrate

Давайте создадим пользователя `admin` с паролем `password123`. Позже мы будем использовать этого пользователя для авторизации.

    python manage.py createsuperuser

Теперь, когда мы инициализировали базу и создали пользователя, давайте перейдем в директорию приложения и начнем кодить...

## Сериализаторы

Во-первых, давайте добавим несколько сериализаторов, которые мы будем использовать для получения определенного представления данных, хранимых в СУБД. Создадим новый модуль `tutorial/quickstart/serializers.py`.

    from django.contrib.auth.models import User, Group
    from rest_framework import serializers


    class UserSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = User
            fields = ('url', 'username', 'email', 'groups')


    class GroupSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Group
            fields = ('url', 'name')

В данном случае мы реализуем ссылочные связи с помощью базового сериализатора HyperlinkedModelSerializer. Вы можете использовать различные типы связей, в том числе и по внешнему ключу, но связь по ссылке наиболее удовлетворяет принципам REST.

## Представления

Давайте теперь создадим несколько вьюх. Откройте `tutorial/quickstart/views.py` и добавьте вот этот код.

    from django.contrib.auth.models import User, Group
    from rest_framework import viewsets
    from tutorial.quickstart.serializers import UserSerializer, GroupSerializer


    class UserViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows users to be viewed or edited.
        """
        queryset = User.objects.all().order_by('-date_joined')
        serializer_class = UserSerializer


    class GroupViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows groups to be viewed or edited.
        """
        queryset = Group.objects.all()
        serializer_class = GroupSerializer

Прежде чем создавать кучу отдельных вьюх для создания, удаления или редактирования объекта, мы можем унаследовать наши представления от класса `ViewSets`, объединяющего в себе все эти функции.

Использование наборов представлений позволяет не растаскивать логику по отдельным вьюхам, что зачастую очень удобно.

## URLs

Итак, а теперь давайте опишем URL-ы для нашего API в файле `tutorial/urls.py`...

    from django.conf.urls import url, include
    from rest_framework import routers
    from tutorial.quickstart import views

    router = routers.DefaultRouter()
    router.register(r'users', views.UserViewSet)
    router.register(r'groups', views.GroupViewSet)

    # Wire up our API using automatic URL routing.
    # Additionally, we include login URLs for the browsable API.
    urlpatterns = [
        url(r'^', include(router.urls)),
        url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

Поскольку мы используем набор представлений вместо обычной вьюхи, мы можем автоматически сгенерировать адреса для нашего API. Для этого достаточно зарегистрировать нашу вьюху с помощью роутера, который сгенирирует некоторый станджарный набор адресов.

Если хочется собственноручно задавать адреса для API, тогда следует отказаться от использования набора представлений и использовать обычные вьюхи.

И в конце мы добавим вьюхи для авторизации. Они также могут пригодиться для авториции, если вы будете использовать ваш API из браузера.

## Настройки

В файл `tutorial/settings.py` нужно добавить еще пару настроек. Мы бы хотели чтобы наше API было дсотупно только для пользователей с правами администратора. Также настроим постраничное отображение списков (по 10 объектов на странице).

    INSTALLED_APPS = (
        ...
        'rest_framework',
    )

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': ('rest_framework.permissions.IsAdminUser',),
        'PAGE_SIZE': 10
    }

Все, теперь все готово.

---

## Тестирование нашего API

Тепрь мы готовы оттеститить наше API. Давайте поднимем наш отладочный сервер

    python ./manage.py runserver

Мы можем получить доступ к нашему API, используя консольные утилиты, например, `curl`

    bash: curl -H 'Accept: application/json; indent=4' -u admin:password123 http://127.0.0.1:8000/users/
    {
        "count": 2,
        "next": null,
        "previous": null,
        "results": [
            {
                "email": "admin@example.com",
                "groups": [],
                "url": "http://127.0.0.1:8000/users/1/",
                "username": "admin"
            },
            {
                "email": "tom@example.com",
                "groups": [                ],
                "url": "http://127.0.0.1:8000/users/2/",
                "username": "tom"
            }
        ]
    }

Или, используя консольную команду [httpie][httpie]

    bash: http -a admin:password123 http://127.0.0.1:8000/users/

    HTTP/1.1 200 OK
    ...
    {
        "count": 2,
        "next": null,
        "previous": null,
        "results": [
            {
                "email": "admin@example.com",
                "groups": [],
                "url": "http://localhost:8000/users/1/",
                "username": "paul"
            },
            {
                "email": "tom@example.com",
                "groups": [                ],
                "url": "http://127.0.0.1:8000/users/2/",
                "username": "tom"
            }
        ]
    }


Или непосредственно в браузере...

![Quick start image][image]

Если вы работаете из браузера убедитесь что вы авторизовались в интерфейсе (смотрите в правом верхнем углу).

Великолепно, все оч просто!

Если вам итересно узнать больше подробностей о работе REST фреймворка, тогда пройдите [учебник][tutorial] или [руководство по API][guide].

[readme-example-api]: ../#example
[image]: ../img/quickstart.png
[tutorial]: 1-serialization.md
[guide]: ../#api-guide
[httpie]: https://github.com/jakubroztocil/httpie#installation
