# チュートリアル 4: 認証と権限

現在のところ、私たちのAPIにはコードスニペットの編集や削除ができる人に関する制限はありません。それを確実にするために、もう少し高度な動作をさせたいと考えています。

- コードスニペットは常に作成者に関連付けられています。
- 認証されたユーザのみがスニペットを作成できる。
- スニペットの作成者のみが更新や削除を行うことができる。
- 認証されていないリクエストは完全な読み取り専用のアクセス権を持つべきです。

## モデルに情報を追加する

```Snippet```モデルクラスにいくつか変更を加えてみましょう。まず、いくつかのフィールドを追加します。フィールドの一つは、コードスニペットを作成したユーザーを表すために使用します。もう1つのフィールドは、コードのハイライトされたHTML表現を保存するために使用されます。

```models.py```の```Snippet```モデルに以下の2つのフィールドを追加します。

```python
owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
highlighted = models.TextField()
```

また、モデルを保存する際には、pygmentsのコードハイライトライブラリを使用して、ハイライトされたフィールドを入力する必要があります。

私たちは、いくつかの追加のインポートが必要になるでしょう。
```python
from pygments.lexers immport get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight
```

これで、モデルクラスに `````.save()````` メソッドを追加することができます:

```python
def save(self, *args, **kwargs):
    """
    コードスニペットのハイライトされたHTML表現を作成するには、`pygments` ライブラリを使用します。
    """
    lexer = get_lexer_by_name(self.language)
    linenos = 'table' if self.linenos else False
    options = {'title': self.title} if self.title else {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super(Snippet, self).save(*args, **kwargs)
```

これがすべて終わったら、データベースのテーブルを更新する必要があります。通常はそのためにデータベースの移行を作成しますが、このチュートリアルの目的のために、データベースを削除して再起動してみましょう。

```shell script
rm -f db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate
```

また、APIのテストに使用するために、いくつかの異なるユーザーを作成したいと思うかもしれません。これには ``createsuperuser```` コマンドを使うのが一番手っ取り早い方法です。
```shell script
python manage.py createsuperuser
```

ユーザーモデルにエンドポイントを追加する

これで、いくつかのユーザーが手に入ったので、それらのユーザーの表現をAPIに追加してみましょう。新しいシリアライザの作成は簡単です。
```serializers.py```に以下を追加します。

```python
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ['id', 'username', 'snippets']
```
``` 'snippets' ``` は User モデル上での逆リレーションであるため、```ModelSerializer``` クラスを使用する際にはデフォルトでは含まれません,そのための明示的なフィールドを追加する必要がありました。

```views.py```にいくつかのビューを追加します。ユーザー表現には読み取り専用のビューを使いたいので、```ListAPIView```と```RetrieveAPIView```の一般的なクラスベースのビューを使います。

```python
from django.contrib.auth.models import User


class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

Make sure to also import the ```UserSerializer``` class
```python
from snippets.serializers import UserSerializer
```

最後に、これらのビューをURL confから参照してAPIに追加する必要があります。```snippets/urls.py```のパターンに以下を追加します。
```python
path('users/', views.UserList.as_view()),
path('users/<int:pk>/', views.UserDetail.as_view()),
```

## スニペットとユーザーの関連付け

今のところ、コードスニペットを作成した場合、スニペットを作成したユーザーとスニペットのインスタンスを関連付ける方法はありません。ユーザーはシリアライズされた表現の一部として送信されるのではなく、受信したリクエストのプロパティとして送信されます。

これを処理する方法は、スニペットビューの `.perform_create()` メソッドをオーバーライドすることで、インスタンス保存の管理方法を変更したり、リクエストやリクエストされたURLに暗黙のうちに含まれる情報を処理したりすることができます。

SnippetListビュークラスに、以下のメソッドを追加します:
```python
def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```
シリアライザの `create()` メソッドには、リクエストの検証済みデータと一緒に追加の ` 'owner' ` フィールドが渡されます。

## シリアライザの更新

スニペットは作成したユーザーに関連付けられているので、SnippetSerializerを更新して反映させてみましょう。serializers.pyのシリアライザ定義に以下のフィールドを追加します:
```python
owner = serializers.ReadOnlyField(source='owner.username')
```

️注意: 内部`Meta`クラスのフィールドのリストに` 'owner' `を追加してください。

このフィールドは非常に興味深いことをしています。`source` 引数は、どの属性をフィールドに入力するかを制御し、シリアライズされたインスタンス上の任意の属性を指し示すことができます。
この場合、Django のテンプレート言語で使用されているのと同様の方法で、与えられた属性を辿ります。

今回追加したフィールドは、`CharField`や`BooleanField`などの他の型付きフィールドとは対照的に、untyped `ReadOnlyField`クラスです。untyped `ReadOnlyField`は常に読み取り専用で、シリアライズされた表現には使用されますが、デシリアライズされたときのモデルインスタンスの更新には使用されません。ここでも`CharField(read_only=True)`を使用することができました。

## ビューに必要なパーミッションを追加する

コードスニペットがユーザーに関連付けられたので、認証されたユーザーだけがコードスニペットを作成、更新、削除できるようにしたいと思います。

REST フレームワークには、与えられたビューに誰がアクセスできるかを制限するために使用できるいくつかのパーミッションクラスが含まれています。この場合、私たちが探しているのは `IsAuthenticatedOrReadOnly` で、認証されたリクエストは読み書きのアクセスを、認証されていないリクエストは読み取り専用のアクセスを得ることを保証します。

まず、viewsモジュールに以下のようなインポートを追加します。

```python
from rest_framework import permissions
```

次に、以下のプロパティを `SnippetList` と `SnippetDetail` ビュークラスの両方に追加します。

```python
permission_classes = [permissions.IsAuthenticatedOrReadOnly]
```

## Browsable APIにログインを追加する

とりあえずブラウザを開いてブラウジング可能なAPIに移動してみると、新しいコードスニペットを作成できなくなっていることがわかります。そのためには、ユーザーとしてログインできるようにする必要があります。

プロジェクトレベルの `urls.py` ファイルの URLconf を編集することで、ブラウジング可能な API で使用するためのログインビューを追加することができます。

ファイルの先頭に以下のようなインポートを追加します:
```python
from django.conf.urls import include
```

そして、ファイルの最後に、ブラウジング可能なAPIのログインとログアウトのビューを含むパターンを追加します。

```python
urlpatterns += [
    path('api-auth/', include('rest_framework.urls')),
]
```

> パターンの `'api-auth/'` の部分は、実際には使用したい任意の URL を指定することができます。


もう一度ブラウザを開いてページを更新すると、ページの右上に「ログイン」のリンクが表示されます。先ほど作成したユーザーの一人としてログインすると、再びコードスニペットを作成できるようになります。

いくつかのコードスニペットを作成したら、'/users/' エンドポイントに移動して、各ユーザーの 'snippets' フィールドに、各ユーザーに関連付けられたスニペット ID のリストが含まれていることに気づきます。

## オブジェクトレベルのパーミッション

本当にすべてのコードスニペットを誰でも見られるようにしたいのですが、コードスニペットを作成したユーザーだけが更新や削除ができるようにしてください。

そのためには、カスタムパーミッションを作成する必要があります。

スニペットアプリで、新しいファイル`permissions.py`を作成します。
```python
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    オブジェクトの所有者のみが編集できるようにするためのカスタムパーミッション。
    """
    
    def has_object_permission(self, request, view, obj):
        # 読み込み権限は任意のリクエストに対して許可されています。
        # なので、常に GET, HEAD, OPTIONS リクエストを許可しています。
        if request.method in permissions.SAFE_METHODS:
            return True

        # 書き込み権限はスニペットの所有者にのみ与えられます。
        return obj.owner == request.user
```
`SnippetDetail` ビュークラスの `permission_classes` プロパティを編集して、カスタムパーミッションを snippet インスタンスのエンドポイントに追加することができます。

```python
permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                      IsOwnerOrReadOnly]
```

また、`IsOwnerOrReadOnly`クラスも必ずインポートしてください。

```python
from snippets.permissions import IsOwnerOrReadOnly
```

さて、再度ブラウザを開くと、コードスニペットを作成したユーザーと同じユーザーとしてログインしている場合にのみ、'DELETE'と'PUT'アクションがスニペットインスタンスのエンドポイントに表示されることがわかります。

## API での認証

API にはパーミッションが設定されているので、スニペットを編集したい場合は API へのリクエストを認証する必要があります。 認証クラスを設定していないので、現在はデフォルトの `SessionAuthentication` と `BasicAuthentication`が適用されています。

Web ブラウザを介して API と対話すると、ログインすることができ、ブラウザのセッションがリクエストに必要な認証を提供してくれます。

API をプログラムで操作する場合は、各リクエストで認証情報を明示的に提供する必要があります。

認証せずにスニペットを作成しようとするとエラーになります:
```shell script
http POST http://127.0.0.1:8000/snippets/ code="print(123)"

{
    "detail": "Authentication credentials were not provided."
}
```

以前に作成したユーザーのうちの1人のユーザー名とパスワードを含めることで、リクエストを成功させることができます。

```shell script
http -a admin:password123 POST http://127.0.0.1:8000/snippets/ code="print(789)"

{
    "id": 1,
    "owner": "admin",
    "title": "foo",
    "code": "print(789)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

## 概要

これで、Web API のパーミッションをかなり細かく設定することができ、システムのユーザーと作成したコードスニペットのエンドポイントを設定することができました。

チュートリアルのパート5では、ハイライトされたスニペットのHTMLエンドポイントを作成することですべてを結びつけ、システム内のリレーションシップにハイパーリンクを使用することでAPIのまとまりを向上させる方法を見ていきます。

[チュートリアル２（関数ベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial)

[チュートリアル３（クラスベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README3.md)