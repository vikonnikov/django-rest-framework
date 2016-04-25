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

## Связываем сниппеты с пользователями

В данный момент при создании сниппета пользователь создавший его не связывается сним

На данный момент у нас не реализован механизм связывания снипппета с создавшим его пользователем. Пользователь не передается в сериализванных данных, но при этом мы можем извлечь его из соответствующего свойства объекта запроса.

Для решения данной задачей мы можем переопределить метод представления `.perform_create()`s. Данный метод позволяет нам модифицировать процессс сохранения объекта, а также извлекать и обрабатывать любые неявные данные, передающиеся в запросах или присутсвующих в запрашиваемом URL.

Добавьте в класс `SnippetList` следующий метод:

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

Теперь в метод `create()`, вместе с остальными валидными данными из запроса, будет передано дополнительное поле `'owner'`.

## Обновляем сериализатор

Теперь каждый сниппет связан с тем пользователем, который его создал. Давайте обновим наш сериализатор `SnippetSerializer`. Добавим еще одно поле в описание нашего сериализатора в модуль `serializers.py`:

    owner = serializers.ReadOnlyField(source='owner.username')

**Note**: Убедитесь что вы добавили `'owner',` в список полей внутреннего метакласса `Meta`.

Данное поле делает что-то очень интересное. Аргумент используется для указания аттрибута объекта, значение которого будет извлечено и сериализовано. Данный аргемент может содежать имя любого атрибута сериализуемого объекта. Также можно задать путь к желаемому аттрибуту объекта, используя при этом точки между именами аттрибутов, мы можем проходить по вложенной цепочке свойств сериализуемого объекта.

В отличии от других полей для `'owner'` мы не указываем его тип, вместо этого мы используем класс `ReadOnlyField`. Поле, созданное на базе класса `ReadOnlyField`, может испольоваться только для чтения, его невозможно изменить при десериализации. Также мы можем использовать стандарный класс поля с указанием аргумента `read_only`, например, `CharField(read_only=True)`.

## Добавляем права доступа к представлению

Теперь нам хотелось бы сделать так, чтобы только авторизованные пользователи могли редактировать, обновлять или удалять сниппеты.

REST фреймворк содержит набор классов, которые мы можем использовать для установки прав доступа к представлениям. В нашем случае мы будем использовать класс `IsAuthenticatedOrReadOnly`, представляющий доступ на запись и чтение только авторизованным пользователям, неавторизованные пользователи имеют доступ на чтение.

В начало модуля с представлениями добавьте следующий импорт:

    from rest_framework import permissions

Затем, в классы `SnippetList` и `SnippetDetail` добавьте следующий аттрибут:

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,)

## Добавляем авторизацию для браузерного API

Если вы откроете браузер и попробуете перейти по одному из адресов нашего API, то вы заметите что больше не можете создавать новые сниппеты. Теперь для выполнения данного действия нам нужно авторизоваться.

Добавим представление для авторизации в нашем браузерном API. Для этого нам необходимо отредактировать модуль  `urls.py`.

Добавьте в начало файла импорт функции:

    from django.conf.urls import include

В конец файла добавьте паттерн, подключающей представления для авторизации и выхода из сеанса браузерного API.

    urlpatterns += [
        url(r'^api-auth/', include('rest_framework.urls',
                                   namespace='rest_framework')),
    ]

Вместо паттерна `r'^api-auth/'` вы можете использовать любой другой, главное задать пространство имен `namespace='rest_framework'`. In Django 1.9+, REST framework will set the namespace, so you may leave it out.

Если вы откроете браузер и обновите страницу, то в правом верхнем углу увидите ссылку на страницу авторизации 'Login'. После авторизации вы снова сможете создавать сниппеты.

После того как вы создадите несколько сниппетов, перейдите по ссылке '/users/' и убедитесь, что сериализованные данные каждого из пользователей, содержат аттрибут 'snippets', содержащий список, состоящий из идентификаторов сниппетов, связанных с пользователем.

## Установка прав доступа на уровне объекта

Все наши сниппеты доступны для чтения любому пользователю. Давайте сделаем так что редактировать или удалять сниппет может только создавший его пользователь.

Для этого нам необходимо создать кастомный класс для прав доступа.

В приложении `snippets`, создайте новый файл `permissions.py`

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

Добавим данный класс в список `permission_classes` нашего представления `SnippetDetail`:

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

Убедитесь что вы ипортировали класс `IsOwnerOrReadOnly`.

    from snippets.permissions import IsOwnerOrReadOnly

Откройте браузер и вы увидите что методы 'DELETE' и 'PUT' доступны только в том случае, если владелец сниппта совпадает с авторизованным пользователем.

## Авторизация при использовании API

Поскольку мы установили права доступа к нашему API, нам необходимо авторизоваться, если мы хотим отредактировать какой-нибдуь из сниппетов. Мы не установили ни один из [классов авторизации][authentication], но, тем не менее, по-умолчанию применяются несколько классов `SessionAuthentication` and `BasicAuthentication`.

Когда для доступа к API мы используем браузер, у нас есть возможность авторизоваться, после чего в рамках установленной сессии мы можем отправлять авторзованные запросы.

При программного использовании API нам необходимо в каждом запросе явным образом передавать параметры авторизации.

Если мы попытаемся создать сниппет, не передавая авторизационных параметров, мы получим ошибку:

    http POST http://127.0.0.1:8000/snippets/ code="print 123"

    {
        "detail": "Authentication credentials were not provided."
    }

Для выполнения успешного запроса нам необходимо передать имя и пароль пользователя.

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
