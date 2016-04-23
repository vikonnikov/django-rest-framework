# Часть 4: Авторизация и права доступа

Сейчас наш API не имеет ни каких ограничений и каждый может редактировать или удалять сниппеты. Нам нужно реализовать немного более продвинутый функционал:

* Сниппеты должны быть связаны с их создателем
* Только авторизованные пользователи могут создавать сниппеты
* Только создатель может редактировать или удалять
* Неавторизованные пользователи могуть имеет доступ только на чтение

## Добавляем информацию в нашу модель

Нам внести некоторые изменения в код класса нашей модели `Snippet`.
Давайте добавим несколько новых полей. Одно из полей будет содержать пользователя, создавшего сниппет. Второе поле будет использоваться для хранения html с подсвеченным синтаксисом нашего фрашмента кода.

Добавим следующие два поля в модель `Snippet` из файла `models.py`.

    owner = models.ForeignKey('auth.User', related_name='snippets')
    highlighted = models.TextField()

Также следует позаботиться о том, чтобы во время сохранения модели, поле `highlighted` заполнялось подсвеченным кодом, для этого мы будем использовать библиотеку `pygments`.

Нам необходимо импортировать следующие модули:

    from pygments.lexers import get_lexer_by_name
    from pygments.formatters.html import HtmlFormatter
    from pygments import highlight

Теперь у нашей модели мы переопределяем метод  `.save()`:

    def save(self, *args, **kwargs):
        """
        Use the `pygments` library to create a highlighted HTML
        representation of the code snippet.
        """
        lexer = get_lexer_by_name(self.language)
        linenos = self.linenos and 'table' or False
        options = self.title and {'title': self.title} or {}
        formatter = HtmlFormatter(style=self.style, linenos=linenos,
                                  full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super(Snippet, self).save(*args, **kwargs)

После этого мы должны обновить таблицы нашей базы данных.
Обычно создается и накатывается новая миграция, но в нашем случае давайте просто удалим старую базу и создадим новую.

    rm -f tmp.db db.sqlite3
    rm -r snippets/migrations
    python manage.py makemigrations snippets
    python manage.py migrate

Для тестирования нашего API создайте несколько новых пользователей. Наиболее быстрый способ создания нового пользователя - использование команды `createsuperuser`.

    python manage.py createsuperuser

## Добавим API для модели пользователя `User`

Теперь у нас есть несколько пользователей, с которыми мы будем работать. Напишем сериализатор для объектов пользователей. В файл `serializers.py` добавим следующий код:

    from django.contrib.auth.models import User

    class UserSerializer(serializers.ModelSerializer):
        snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

        class Meta:
            model = User
            fields = ('id', 'username', 'snippets')

Добавим в сериализатор поле `'snippets'`, отвечающее за обрабную связь с моделью `User` и содержащее сниппеты, созданные данным пользователем. Данное поле не добавляется по умолчанию в `ModelSerializer`, поэтому мы сами добавляем его в сериализатор.

Давайте добавим еще парочку представлений в файл `views.py`. Нам нужны представления, предназначенные только для отображения пользователей, поэтому мы используем общие представления `ListAPIView` и `RetrieveAPIView`.

    from django.contrib.auth.models import User


    class UserList(generics.ListAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer


    class UserDetail(generics.RetrieveAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

Не забудьте импортировать класс `UserSerializer`

	from snippets.serializers import UserSerializer

Ну и что бы у нас наконец получилось API, добавим полученные представления, в конфиг URL адресов. Добавьте следующие паттерны в файл `urls.py`.

    url(r'^users/$', views.UserList.as_view()),
    url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),

## Associating Snippets with Users

Right now, if we created a code snippet, there'd be no way of associating the user that created the snippet, with the snippet instance.  The user isn't sent as part of the serialized representation, but is instead a property of the incoming request.

The way we deal with that is by overriding a `.perform_create()` method on our snippet views, that allows us to modify how the instance save is managed, and handle any information that is implicit in the incoming request or requested URL.

On the `SnippetList` view class, add the following method:

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

The `create()` method of our serializer will now be passed an additional `'owner'` field, along with the validated data from the request.

## Обновляем сериализатор

Теперь каждый сниппет связан с тем пользователем, который его создал. Давайте обновим наш сериализатор `SnippetSerializer`. Добавим еще одно поле в описание нашего сериализатора в модуль `serializers.py`:

    owner = serializers.ReadOnlyField(source='owner.username')

**Note**: Make sure you also add `'owner',` to the list of fields in the inner `Meta` class.

This field is doing something quite interesting.  The `source` argument controls which attribute is used to populate a field, and can point at any attribute on the serialized instance.  It can also take the dotted notation shown above, in which case it will traverse the given attributes, in a similar way as it is used with Django's template language.

The field we've added is the untyped `ReadOnlyField` class, in contrast to the other typed fields, such as `CharField`, `BooleanField` etc...  The untyped `ReadOnlyField` is always read-only, and will be used for serialized representations, but will not be used for updating model instances when they are deserialized. We could have also used `CharField(read_only=True)` here.

## Добавляем права доступа к представлению

Теперь нам хотелось бы сделать так, чтобы только авторизованные пользователи могли редактировать, обновлять или удалять сниппеты.

REST framework includes a number of permission classes that we can use to restrict who can access a given view.  In this case the one we're looking for is `IsAuthenticatedOrReadOnly`, which will ensure that authenticated requests get read-write access, and unauthenticated requests get read-only access.

First add the following import in the views module

    from rest_framework import permissions

Then, add the following property to **both** the `SnippetList` and `SnippetDetail` view classes.

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,)

## Добавляем авторизацию для браузерного API

If you open a browser and navigate to the browsable API at the moment, you'll find that you're no longer able to create new code snippets.  In order to do so we'd need to be able to login as a user.

We can add a login view for use with the browsable API, by editing the URLconf in our project-level `urls.py` file.

Add the following import at the top of the file:

    from django.conf.urls import include

And, at the end of the file, add a pattern to include the login and logout views for the browsable API.

    urlpatterns += [
        url(r'^api-auth/', include('rest_framework.urls',
                                   namespace='rest_framework')),
    ]

The `r'^api-auth/'` part of pattern can actually be whatever URL you want to use.  The only restriction is that the included urls must use the `'rest_framework'` namespace. In Django 1.9+, REST framework will set the namespace, so you may leave it out.

Now if you open up the browser again and refresh the page you'll see a 'Login' link in the top right of the page.  If you log in as one of the users you created earlier, you'll be able to create code snippets again.

Once you've created a few code snippets, navigate to the '/users/' endpoint, and notice that the representation includes a list of the snippet pks that are associated with each user, in each user's 'snippets' field.

## Object level permissions

Really we'd like all code snippets to be visible to anyone, but also make sure that only the user that created a code snippet is able to update or delete it.

To do that we're going to need to create a custom permission.

In the snippets app, create a new file, `permissions.py`

    from rest_framework import permissions


    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        Custom permission to only allow owners of an object to edit it.
        """

        def has_object_permission(self, request, view, obj):
            # Read permissions are allowed to any request,
            # so we'll always allow GET, HEAD or OPTIONS requests.
            if request.method in permissions.SAFE_METHODS:
                return True

            # Write permissions are only allowed to the owner of the snippet.
            return obj.owner == request.user

Now we can add that custom permission to our snippet instance endpoint, by editing the `permission_classes` property on the `SnippetDetail` view class:

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

Make sure to also import the `IsOwnerOrReadOnly` class.

    from snippets.permissions import IsOwnerOrReadOnly

Now, if you open a browser again, you find that the 'DELETE' and 'PUT' actions only appear on a snippet instance endpoint if you're logged in as the same user that created the code snippet.

## Авторизация при использовании API

Поскольку мы установили права доступа к нашему API, нам необходимо авторизоваться, если мы хотим отредактировать какой-нибдуь из сниппетов. Мы не установили ни один из [классов авторизации][authentication], но, тем не менее, по-умолчанию применяются несколько классов `SessionAuthentication` and `BasicAuthentication`.

Когда для доступа к API мы используем браузер, у нас есть возможность авторизоваться, после чего в рамках установленной сессии мы можем отправлять авторзованные запросы.

If we're interacting with the API programmatically we need to explicitly provide the authentication credentials on each request.

If we try to create a snippet without authenticating, we'll get an error:

    http POST http://127.0.0.1:8000/snippets/ code="print 123"

    {
        "detail": "Authentication credentials were not provided."
    }

We can make a successful request by including the username and password of one of the users we created earlier.

    http -a tom:password123 POST http://127.0.0.1:8000/snippets/ code="print 789"

    {
        "id": 5,
        "owner": "tom",
        "title": "foo",
        "code": "print 789",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

## Итоги

Теперь у нас есть довольно токо настроенный набор ограничений для доступа к нашемeу Web API, мы добавили API для просмора списка всех имеющихся пользователей, а также пользователей создавших тот или иной сниппет.

В [главе 5][tut-5] мы обратим внимание на то, как объединить все в единое целое, создав API для наших подсвеченных сниппетов. Также мы попробуем улучшить связи внути нашего API за счет использования ссылочных связей.

[authentication]: ../api-guide/authentication.md
[tut-5]: 5-relationships-and-hyperlinked-apis.md
