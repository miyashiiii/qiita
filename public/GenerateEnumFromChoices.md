---
title: Strawberry Djangoを使ってDjangoのChoicesを自動でGraphQLのEnum化する
tags:
  - Django
  - GraphQL
  - strawberry
  - StrawberryDjango
private: false
updated_at: '2023-07-20T18:05:05+09:00'
id: f93c4c57a51dbf8ecfd7
organization_url_name: null
slide: false
---


## はじめに

Strawberry Djangoでは、`@strawberry.django.type`デコレータをつけたクラスを作成すると、DjangoのModelに対応したGraphqlのtypeを生成します。この時各フィールドの型は `auto` を指定するとModelの定義に対応した型を自動で設定してくれます。
[Defining Types - Strawberry Django](https://strawberry-graphql.github.io/strawberry-graphql-django/guide/types/)

この記事は、DjangoのModelの[choices](https://docs.djangoproject.com/en/4.2/ref/models/fields/#choices)を使用したフィールドを、自動でGraphQLのEnumに変換してくれる `GENERATE_ENUMS_FROM_CHOICES`というオプションの紹介です

## Strawberry Djangoとは
[Strawberry](https://strawberry.rocks/)はPython製のGraphQLライブラリ。  
そのDjango向け拡張が[Strawberry Django](https://strawberry-graphql.github.io/strawberry-graphql-django/)です。
(ライブラリ名としては`strawberry-graphql-django`)

## コード例
### 動作確認バージョン
- django: 3.2.19
- strawberry-graphql: 0.194.1
- strawberry-graphql-django: 0.13.0


### settings.py
```py
STRAWBERRY_DJANGO = {
    "GENERATE_ENUMS_FROM_CHOICES": True,
}
```

### models.py
Taskモデルの`priority`フィールドに`choices`パラメータを設定
```py
class Task(models.Model):
    PRIORITY_CHOICES = [
        ("L", "LOW"),
        ("M", "MEDIUM"),
        ("H", "HIGH"),
    ]
    title = models.CharField(max_length=200)
    priority = models.CharField(
        max_length=1,
        choices=PRIORITY_CHOICES,
        default="M",
    )
```

もしくは、[`TextChoices`](https://docs.djangoproject.com/en/4.2/ref/models/fields/#enumeration-types)を使ったEnumライクな指定の仕方もできます。
```py
class PriorityChoices(models.TextChoices):
    LOW = "L", "LOW"
    MEDIUM = "M", "MEDIUM"
    HIGH = "H", "HIGH"

class Task(models.Model):
    title = models.CharField(max_length=200)
    priority = models.CharField(
        max_length=2,
        choices=PriorityChoices.choices,
        default=PriorityChoices.MEDIUM,
    )
```

### schema.py
`tasks`クエリを作成。カスタムのリゾルバ処理は作成せず、単にTaskを取ってくるだけ
```py
@strawberry_django.type(Task)
class TaskType:
    title: strawberry.auto
    priority: strawberry.auto


@strawberry.type
class Query:
    tasks: list[TaskType] = strawberry.django.field()


schema = strawberry.Schema(query=Query,)
```


### 出力されるスキーマ
以下のように、Enumが自動生成される。
choicesパラメータに渡した各タプルの1つ目が値、2つ目がフィールドのdescriptionとして扱われている
```graphql
enum MyappTaskPriorityEnum {
  """Low"""
  L

  """Medium"""
  M

  """High"""
  H
}

type Query {
  tasks: [TaskType!]!
}

type TaskType {
  title: String!
  priority: MyappTaskPriorityEnum!
}
```


:::note warn
今回のコード例ではCharFieldに対してchoicesを指定しています。
`IntegerField`に対して`choices`を指定して`auto`で変換する形では、Enum変換されず、単にIntになるようです。
また、執筆時点で最新の0.14.0では[`IntegerChoices`](https://docs.djangoproject.com/en/4.2/ref/models/fields/#enumeration-types)を使ったchoicesの指定をすると、エラーで落ちます
`IntegerField`を使用しているフィールドがあるPJでの導入には注意してください。

※今回これに気づいて修正PR出しました。次回リリースバージョン以降ではタプルを使っての指定と同様にIntが返るようになると思います。
:::

:::note warn
ドキュメントでは`TextChoices`,`IntegerChoices`を使う際は`GENERATE_ENUMS_FROM_CHOICES`ではなく[`django-choices-field`](https://strawberry-graphql.github.io/strawberry-graphql-django/integrations/choices-field/)というライブラリを使ってModel側フィールドを定義するよう推奨されており、こちらの導入を検討するとよいかと思います。
ただし`django-choices-field`は現状個人のリポジトリとなっておりstar数も少なく、個人的には正直本番投入するにはためらってしまう面もあります。
:::


## `GENERATE_ENUMS_FROM_CHOICES`を使わない場合
ちなみに、この設定を使わずにchoicesをスキーマ上でEnumで表現したい場合、以下のようにEnumクラスとその生成処理を自前で書く必要がある。
(プロジェクトによっては汎用変換処理を作ってそう)

schema.py:
```py
@strawberry.enum
class PriorityEnum(strawberry.Enum):
    LOW = 'L'
    MEDIUM = 'M'
    HIGH = 'H'

@strawberry.type
class TaskType:
    title: str
    description: str
    priority: PriorityEnum

def convert_task(task: Task) -> TaskType:
  """TaskTypeへの変換処理"""
    priority = task.priority
    if priority == PriorityChoices.LOW:
        converted_priority = PriorityEnum.LOW
    elif priority == PriorityChoices.MEDIUM:
        converted_priority = PriorityEnum.MEDIUM
    elif priority == PriorityChoices.HIGH:
        converted_priority = PriorityEnum.HIGH
    else:
        converted_priority = None

    return TaskType(
        title=task.title,
        description=task.description,
        priority=converted_priority,
    )
class Query:

    @strawberry.field
    def tasks(self, info) -> List[TaskType]:
        tasks = Task.objects.all()
        return [convert_task(task) for task in tasks]
```
