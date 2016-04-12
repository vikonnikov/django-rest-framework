# Быстрый старт

Мы собираемся создать простой API, который позволял бы админам просматривать или редактировать пользователей и группы.

## Настройка проекта

Создайте новый Django проект `tutorial` и приложение `quickstart`

    # Create the project directory
    mkdir tutorial
    cd tutorial

    # Create a virtualenv to isolate our package dependencies locally
    virtualenv env
    source env/bin/activate  # On Windows use `env\Scripts\activate`

    # Install Django and Django REST framework into the virtualenv
    pip install django
    pip install djangorestframework

    # Set up a new project with a single application
    django-admin.py startproject tutorial .  # Note the trailing '.' character
    cd tutorial
    django-admin.py startapp quickstart
    cd ..

Мигрируйте текущую схему базы данных:

    python manage.py migrate

Давайте создадим пользователя `admin` с паролем `password123`. Позже мы будем использовать этого пользователя для авторизации.

    python manage.py createsuperuser

Теперь, когда мы инициализировали базу и создали пользователя, давайте перейдем в директорию приложения и начнем кодить...

## Сериализаторы

Во-первых, давайте добавим несколько сериализаторов. Создадим новый модуль `tutorial/quickstart/serializers.py` that we'll use for our data representations.

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

И в конце мы добавим вьюхи для авторизации. Они могут пригодиться для авториции, если вы будете использовать вирзуальные формы для вашего API.

## Настройки

We'd also like to set a few global settings.  We'd like to turn on pagination, and we want our API to only be accessible to admin users.  The settings module will be in `tutorial/settings.py`

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

Великолемно, все оч просто!

If you want to get a more in depth understanding of how REST framework fits together head on over to [the tutorial][tutorial], or start browsing the [API guide][guide].

[readme-example-api]: ../#example
[image]: ../img/quickstart.png
[tutorial]: 1-serialization.md
[guide]: ../#api-guide
[httpie]: https://github.com/jakubroztocil/httpie#installation
