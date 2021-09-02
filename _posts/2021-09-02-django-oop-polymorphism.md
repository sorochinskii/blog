---
title: "Моделирование полиморфизма в Django"
date: 2021-09-02 00:00:00 -0000
toc: true
toc_sticky: true
excerpt_separator: "<!--more-->"
categories: 
  - Blog
tags:
  - django
  - oop
  - translation
  - tutorial
---

Моделирование полиморфизма в реляционных базах данных является сложной задачей. В этой статье показаны несколько способов моделирования полиморфных объектов в реляционной базе данных с использованием Django ORM.

Оригинал статьи [Modeling Polymorphism in Django With Python](https://realpython.com/modeling-polymorphism-django-python/)

## Что такое полиморфизм?

Полиморфизм это способность объекта принимать множество форм. Частые примеры полиморфных объектов это потоки событий, различные типы пользователей и продукты на веб-сайте электронной коммерции. Полиморфная модель используется, если объекту требуется придать отличающуюся функциональность или информацию.

В приведенных выше примерах все события логируются для использования в будущем, но могут содержать различные данные. Все пользователи должны иметь возможность входа в систему, но могут иметь различные структуры профилей. На каждом веб-сайте электронной коммерции пользователь хочет поместить различные продукты в свою корзину.

## Почему моделирование полиморфизма сложная задача?

Существует множество путей моделирования полиморфизма. Некоторые подходы используют стандартные возможности Django ORM, а некоторые используют особые возможности Django ORM. Основные задачи с которыми можно столкнуться моделируя полиморфные объекты следующие:

- **как представить единичный полиморфный объект:** такие объекты имеют отличающиеся атрибуты. Django ORM привязывает атрибуты к колонкам в базе данных. В таком случае, каким образом Django должен привязывать атрибуты к колонкам в таблице? Должны ли различные объекты находится в одной таблице? Или надо иметь различные таблицы для этого?
- **Как ссылаться на экземпляры полиморфной модели:** чтобы использовать базу данных и возможности Django ORM, необходимо ссылаться на объекты внешними ключами. То как вы решите представить полиморфный объект является решающим значением для возможности ссылаться на него.

Чтобы по-настоящему понять проблемы моделирования полиморфизма, возьмем для примера превращение небольшого книжного магазина от первого вебсайта до большого онлайн магазина, продающего всевозможную продукцию.

> **Примечание:** для прохождения этого туториала, рекомендуется использовать PostgreSQL бэкенд, Django 2.x и Python3.
>
> Возможно попробовать с бэкендами других баз данных. В местах, где используются уникальные функции PostgreSQL будут показаны способы для других баз данных.

## Наивная реализация

Представьте, что у вас есть книжный магазинчик в хорошей части города сразу за кофейней и  вы хотите начать продавать книги онлайн.

Вы продаете только один тип продукта: книги. В онлайн магазине вы хотите показывать подробности о книгах, такие как название и цена. Вы хотите, чтобы пользователи просматривали веб-сайт и собирали множество книг, поэтому необходима корзина. Также надо отправлять книги пользователям, поэтому необходимо знать вес каждой книги для подсчета стоимости доставки.

Давайте создадим простую модель для книги:

  ```python
from django.contrib.auth import get_user_model
from django.db import models

class Book(models.Model):
    name = models.CharField(
        max_length=100,
    )
    price = models.PositiveIntegerField(
        help_text='in cents',
    )
    weight = models.PositiveIntegerField(
        help_text='in grams',
    )

    def __str__(self) -> str:
        return self.name

class Cart(models.Model):
    user = models.OneToOneField(
        get_user_model(),
        primary_key=True,
        on_delete=models.CASCADE,
    )
    books = models.ManyToManyField(Book)
  ```

 Для создания новой книги необходимо указать название, стоимость и вес:

```python
>>> from naive.models import Book
>>> book = Book.objects.create(name='Python Tricks', price=1000, weight=200)
>>> book
<Product: Python Tricks>` 
```

Для создания корзины, сначала необходимо ассоциировать ее с пользователем:

```python
>>> from django.contrib.auth import get_user_model
>>> haki = get_user_model().create_user('haki')

>>> from naive.models import Cart
>>> cart = Cart.objects.create(user=haki)
```

Затем пользователь может добавлять товары в корзину:

```python
>>> cart.products.add(book)
>>> cart.products.all()
<QuerySet [<Book: Python Tricks>]>
```

**Плюсы:**

- **Проще понять и поддерживать:** этого достаточно для одного типа продукта.

**Минусы:**

- **Ограничение только для однородных продуктов:** поддерживаются только продукты с одинаковым набором атрибутов. Полиморфизм в этом случае не допускается.

## Разреженная модель

С успехом вашего онлайн магазина, пользователи просят начать продавать также и электронные книги. Электронные книги великолепный продукт для вашего онлайн магазина и вы хотите начать продавать их прямо сейчас.

Отличия реальной книги от электронной:

- у электронной книги нет веса
- не нужна доставка.

Для поддержки существующей моделью `Book` электронных книг, необходимо добавить дополнительные поля:

```python
from django.contrib.auth import get_user_model
from django.db import models

class Book(models.Model):
    TYPE_PHYSICAL = 'physical'
    TYPE_VIRTUAL = 'virtual'
    TYPE_CHOICES = (
        (TYPE_PHYSICAL, 'Physical'),
        (TYPE_VIRTUAL, 'Virtual'),
    )
 type = models.CharField( max_length=20, choices=TYPE_CHOICES, ) 
    # Common attributes
    name = models.CharField(
        max_length=100,
    )
    price = models.PositiveIntegerField(
        help_text='in cents',
    )

    # Specific attributes
    weight = models.PositiveIntegerField(
        help_text='in grams',
    )
 download_link = models.URLField( null=True, blank=True, ) 
    def __str__(self) -> str:
        return f'[{self.get_type_display()}] {self.name}'

class Cart(models.Model):
    user = models.OneToOneField(
        get_user_model(),
        primary_key=True,
        on_delete=models.CASCADE,
    )
    books = models.ManyToManyField(
        Book,
    )
```

Во-первых добавлено поле типа книги. Во-вторых, добавлено поле URL для хранения ссылки на электронную книгу.

Для добавления физической книги сделайте следующее:

```python
>>> from sparse.models import Book
>>> physical_book = Book.objects.create(
...     type=Book.TYPE_PHYSICAL,
...     name='Python Tricks',
...     price=1000,
...     weight=200,
...     download_link=None,
... )
>>> physical_book
<Book: [Physical] Python Tricks>
```

Для добавления электронной книги сделайте следующее:

```python
>>> virtual_book = Book.objects.create(
...     type=Book.TYPE_VIRTUAL,
...     name='The Old Man and the Sea',
...     price=1500,
...     weight=0,
...     download_link='https://books.com/12345',
... )
>>> virtual_book
<Book: [Virtual] The Old Man and the Sea>		
```

Теперь ваши пользователи могут добавлять оба типа книг в корзину:

```
>>> from sparse.models import Cart
>>> cart = Cart.objects.create(user=user)
>>> cart.books.add(physical_book, virtual_book)
>>> cart.books.all()
<QuerySet [<Book: [Physical] Python Tricks>, <Book: [Virtual] The Old Man and the Sea>]>
```

Электронные книги - большой хит и вы решаете нанять сотрудников. Новые сотрудники с компьютерами "на Вы" и в базе данных начинают появляться странные данные:

```python
>>> Book.objects.create(
...     type=Book.TYPE_PHYSICAL,
...     name='Python Tricks',
...     price=1000,
...     weight=0,
...     download_link='http://books.com/54321',
... )
```

Эта книга имеет вес `0` и ссылку скачивания.

Другая книга имеет вес 100 грамм и не имеет ссылки скачивания:

```python
>>> Book.objects.create(
...     type=Book.TYPE_VIRTUAL,
...     name='Python Tricks',
...     price=1000,
...     weight=100,
...     download_link=None,
... )
```

Это не имеет смысла, а вы имеете проблему целостности данных. Для обработки таких проблем, добавим валидацию в модель:

```python
from django.core.exceptions import ValidationError

class Book(models.Model):

    # ...

    def clean(self) -> None:
        if self.type == Book.TYPE_VIRTUAL:
            if self.weight != 0:
                raise ValidationError(
                    'A virtual product weight cannot exceed zero.'
                )

            if self.download_link is None:
                raise ValidationError(
                    'A virtual product must have a download link.'
                )

        elif self.type == Book.TYPE_PHYSICAL:
            if self.weight == 0:
                raise ValidationError(
                    'A physical product weight must exceed zero.'
                )

            if self.download_link is not None:
                raise ValidationError(
                    'A physical product cannot have a download link.'
                )

        else:
            assert False, f'Unknown product type "{self.type}"'
```

Вы использовали  [механизм валидации Django](https://docs.djangoproject.com/en/2.1/ref/models/instances/#django.db.models.Model.clean). Метод `clean` вызывается автоматически только Django формами. Для объектов которые не создаются Django формами, необходимо явно валидировать объект.

Чтобы сохранить целостность модели `Book` нетронутой, нужно внести некоторые изменения в процесс создания книг:

```python
>>> book = Book(
...    type=Book.TYPE_PHYSICAL,
...    name='Python Tricks',
...    price=1000,
...    weight=0,
...    download_link='http://books.com/54321',
... )
>>> book.full_clean() ValidationError: {'__all__': ['A physical product weight must exceed zero.']} 
>>> book = Book(
...    type=Book.TYPE_VIRTUAL,
...    name='Python Tricks',
...    price=1000,
...    weight=100,
...    download_link=None,
... )
>>> book.full_clean() ValidationError: {'__all__': ['A virtual product weight cannot exceed zero.']}
```

Когда объекты создаются менеджером по-умолчанию (`Book.objects.create(...)`), Django создаст объект и сразу сохранит его в базу данных.

В вашем случае хочется валидировать объект перед сохранением в базу. Сначала создается объект `Book(...)`, затем валидируется `book.full_clean()`, и только потом сохраняем `book.save()`.

**Денормализация**

Разреженная модель это продукт [денормализации](https://en.wikipedia.org/wiki/Denormalization). В процессе денормализации вы встраиваете атрибуты из множества нормализованных моделей в единственную таблицу для лучшей производительности. Также денормализованная таблица обычно будет иметь множество колонок с null содержимым.

Денормализация часто используется для поддержки систем храненния данных в которых производительность чтения наиболее важна. В отличие от [OLTP систем](https://en.wikipedia.org/wiki/Online_transaction_processing), хранение данных не требует применения правил целостности, что делает денормализацию идеальной для них.

**Плюсы**

- **Легко разобраться и поддерживать:** обычно разреженная модель это первый шаг, когда определенным типам объектов требуется больше информации. Она очень интуитивна и проста в понимании.

**Минусы**

- **Невозможно использовать ограничение NOT NULL базы данных (Ограничение NOT NULL заставляет столбец не принимать нулевые значения):** Null значения используются для атрибутов, которые не определены для всех типов объектов.
- **Сложная логика валидации:** сложная логика валидации требуется для  применения правил целостности данных. Также такая логика требует больше тестов.
- **Большое количество Null-полей создают беспорядок:** представление множества типов продукции в единственной модели делает ее сложной для понимания и поддержки.
- **Новые типы требуют изменения схемы:** новые типы продукции требуют дополнительный полей и валидаций.

**Варианты использования**

Разреженная модель идеальна когда вы представляете гетерогенные объекты с бОльшим количеством общих атрибутов и с редким добавлением новых элементов.

## Полуструктурированная модель

Ваша книжная лавка добилась огромного успеха и продает все больше и больше книг. У вас есть книги различных жанров и издательств, электронные книги различных форматов, книги необычных форм и размеров и т.д.

При использовании разреженной модели вы добавляли все новые и новые поля для каждой новой продукции. Модель теперь имеет огромное количество null-полей. Новые разработчики имеют проблемы с поддержкой, а служащим требуется большое количество времени для заполнения.

 Для устранения беспорядка, вы принимаете решение хранить в модели только общие поля (`name` и `price`). Остальные поля вы храните в единственном `JSONField`:

```python
from django.contrib.auth import get_user_model
from django.contrib.postgres.fields import JSONField
from django.db import models

class Book(models.Model):
    TYPE_PHYSICAL = 'physical'
    TYPE_VIRTUAL = 'virtual'
    TYPE_CHOICES = (
        (TYPE_PHYSICAL, 'Physical'),
        (TYPE_VIRTUAL, 'Virtual'),
    )
    type = models.CharField(
        max_length=20,
        choices=TYPE_CHOICES,
    )

    # Common attributes
    name = models.CharField(
        max_length=100,
    )
    price = models.PositiveIntegerField(
        help_text='in cents',
    )

 extra = JSONField() 
    def __str__(self) -> str:
        return f'[{self.get_type_display()}] {self.name}'

class Cart(models.Model):
    user = models.OneToOneField(
        get_user_model(),
        primary_key=True,
        on_delete=models.CASCADE,
    )
    books = models.ManyToManyField(
        Book,
        related_name='+',
    )
```

> **JSONField:**
>
> В этом примере вы используете PostgreSQL, для которого Django имеет встроенную поддержку поля JSON в `django.contrib.postgres.fields`.
>
> Для других баз данных, таких как [SQLite](https://realpython.com/python-sqlite-sqlalchemy/) и MySQL существуют [packages](https://djangopackages.org/grids/g/json-fields/) для предоставления похожей функциональности.

Теперь модель `Book` освобождена от беспорядка. Общие атрибуты моделируются полями, а не общие хранятся в JSON-поле `extra`:

```python
>>> from semi_structured.models import Book
>>> physical_book = Book(
...     type=Book.TYPE_PHYSICAL,
...     name='Python Tricks',
...     price=1000,
...     extra={'weight': 200}, ... )
>>> physical_book.full_clean()
>>> physical_book.save()
<Book: [Physical] Python Tricks>

>>> virtual_book = Book(
...     type=Book.TYPE_VIRTUAL,
...     name='The Old Man and the Sea',
...     price=1500,
...     extra={'download_link': 'http://books.com/12345'}, ... )
>>> virtual_book.full_clean()
>>> virtual_book.save()
<Book: [Virtual] The Old Man and the Sea>

>>> from semi_structured.models import Cart
>>> cart = Cart.objects.create(user=user)
>>> cart.books.add(physical_book, virtual_book)
>>> cart.books.all()
<QuerySet [<Book: [Physical] Python Tricks>, <Book: [Virtual] The Old Man and the Sea>]>
```

Отсутствие беспорядка важно, но за это надо платить. Валидация теперь намного сложнее:

```python
from django.core.exceptions import ValidationError
from django.core.validators import URLValidator

class Book(models.Model):

    # ...

    def clean(self) -> None:

        if self.type == Book.TYPE_VIRTUAL:

            try:
                weight = int(self.extra['weight'])
            except ValueError:
                raise ValidationError(
                    'Weight must be a number'
                )
            except KeyError:
                pass
            else:
                if weight != 0:
                    raise ValidationError(
                        'A virtual product weight cannot exceed zero.'
                    )

            try:
                download_link = self.extra['download_link']
            except KeyError:
                pass
            else:
                # Will raise a validation error
                URLValidator()(download_link)

        elif self.type == Book.TYPE_PHYSICAL:

            try:
                weight = int(self.extra['weight'])
            except ValueError:
                raise ValidationError(
                    'Weight must be a number'
                 )
            except KeyError:
                pass
            else:
                if weight == 0:
                    raise ValidationError(
                        'A physical product weight must exceed zero.'
                     )

            try:
                download_link = self.extra['download_link']
            except KeyError:
                pass
            else:
                if download_link is not None:
                    raise ValidationError(
                        'A physical product cannot have a download link.'
                    )

        else:
            raise ValidationError(f'Unknown product type "{self.type}"')
```

Преимущество использования соответствующего поля, что оно валидирует тип. Оба и Django и Django ORM могут выполнять проверки типа используемго для поля. Для поля `JSONField` необходимо валидировать и тип и значение:

```python
>>> book = Book.objects.create(
...     type=Book.TYPE_VIRTUAL,
...     name='Python Tricks',
...     price=1000,
...     extra={'weight': 100},
... )
>>> book.full_clean()
ValidationError: {'__all__': ['A virtual product weight cannot exceed zero.']}
```

Другая проблема состоит в том, что не все базы данных имеют поддержку для запросов и индексирования значений в JSON-полях.

В PostgreSQL, например, можно запросить все книги с весом более `100`:

```python
>>> Book.objects.filter(extra__weight__gt=100)<QuerySet [<Book: [Physical] Python Tricks>]>
```

Другое ограничение, налагаемое при использовании JSON, заключается в том, что невозможно использовать ограничения базы данных, такие как не-null, уникальные и внешние ключи. Эти ограничения должны быть реализованы в приложении.

Полуструктурированный подход напоминает архитектуру [NoSQL] (https://en.wikipedia.org/wiki/NoSQL) и имеет множество своих преимуществ и недостатков. Поле JSON - это способ обойти строгую схему реляционной базы данных. Такой гибридный подход предоставляет нам гибкость для разделения многих типов объектов в единой таблице, сохраняя при этом некоторые преимущества реляционной, строго и сильно типизированной базы данных. Для многих распространенных вариантов использования NoSQL этот подход может оказаться более подходящим.

**Плюсы**

- **Уменьшение беспорядочности:** Общие поля хранятся в модели. Остальные - в JSON-поле.
- **Легкое добавление новых типов:** Новые типы продукции не требуют изменения в схеме.

**Минусы**

- **Сложная и специализированная валидация:** Валидация JSON поля требует валидации как типов, так и значений. Эти проблемы можно решить используя другие решения для валидации JSON данных. Например, [JSON schema](https://json-schema.org/).
- **Невозможность использовать ограничения базы данных:** таких, как не-null-поле, уникальные поля, поля внешних ключей, которые обеспечивают целостность данных на уровне базы данных.
- **Ограничения поддержки базы данных для JSON**: Не все вендоры баз данных поддерживают запросы и индексирование полей JSON.
- **Схема не применяется системой базы данных**: Изменения схемы могут потребовать обратной совместимости или оперативной миграции. Данные могут «гнить».
- **Нет глубокой интеграции с системой метаданных базы данных**: метаданные о полях не хранятся в базе данных. Схема применяется только на уровне приложения.

**Варианты использования**

Полуструктурированная модель идеальна в случаях, когда гетерогенные объекты не разделяют множество общих атрибутов и новые элементы добавляются в базу часто.

Классический вариант использования это хранение событий (логи, аналитика, хранилища событий). Большинство событий имеют дату, тип и метаданные, такие как устройств, юзер агент, пользователь и т.д. Данные для каждого типа хранятся и JSON-поле. Для аналитики и логов важна возможность добавления новых типов событий с минимальными затратами, поэтому полуструктурированная модель идеальна для этих случаев.

