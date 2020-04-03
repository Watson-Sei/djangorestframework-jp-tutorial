# チュートリアル 6: ViewSets & Routers

RESTフレームワークには`ViewSets`を扱うための抽象化が含まれており、開発者はAPIの状態やインタラクションのモデリングに集中でき、URLの構築は一般的な規約に基づいて自動的に処理されるようにしています。

`ViewSet` クラスは `View` クラスとほぼ同じものですが、`read` や `update` のような操作を提供し、`get` や `put` のようなメソッドハンドラを提供しないことを除いては、`View` クラスとほとんど同じです。

`ViewSet` クラスは最後の瞬間にのみメソッドハンドラのセットにバインドされ、ビューのセットにインスタンス化されます。

`ViewSet` クラスは、ビューのセットにインスタンス化されたときに、最後の瞬間にメソッドハンドラのセットにバインドされるだけで、通常は URL のコンフィグを定義する複雑な処理を行う `Router` クラスを使用します。

## ViewSetsを使用するためのリファクタリング

現在のビューのセットを、ビューのセットにリファクタリングしてみましょう。

まず最初に、`UserList` と `UserDetail` ビューを一つの `UserViewSet` にリファクタリングしてみましょう。2つのビューを削除して、1つのクラスに置き換えることができます。

```python
from rest_framework import viewsets

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    このビューセットは自動的に `list` と `detail` のアクションを提供します。
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

ここでは `ReadOnlyModelViewSet` クラスを使用して、デフォルトの '読み取り専用' 操作を自動的に提供しています。`queryset` と `serializer_class` 属性は通常のビューを使用していた時と全く同じように設定していますが、同じ情報を 2 つの別々のクラスに提供する必要はなくなりました。

次に、`SnippetList`、`SnippetDetail`、`SnippetHighlight`のビュークラスを置き換えます。3つのビューを削除して、1つのクラスに置き換えることができます。

```python
from rest_framework.decorators import action
from rest_framework.response import Response

class SnippetViewSet(viewsets.ModelViewSet):
    """
    このビューセットは、`list`, `create`, `retrieve`, `update`, `destroy` アクションを自動的に提供します。

    さらに、追加の `highlight` アクションも提供します。
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly]

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

今回は`ModelViewSet`クラスを使用して、デフォルトの読み書き操作の完全なセットを取得しています。

また、`@action` デコレータを使用して `highlight` という名前のカスタムアクションを作成していることに注目してください。このデコレータは、標準の `create`/`update`/`delete` スタイルに収まらないカスタムエンドポイントを追加するために使用することができます。

`@action` デコレータを使用するカスタムアクションはデフォルトで `GET` リクエストに応答します。アクションが `POST` リクエストに応答するようにしたい場合は `method` 引数を使うことができます。

デフォルトでは、カスタムアクションのためのURLはメソッド名自体に依存します。URL の構築方法を変更したい場合は、デコレータキーワードの引数に `url_path` を含めることができます。

## ViewSets を明示的に URL にバインドする

ハンドラメソッドがアクションにバインドされるのは、URLConfを定義したときだけです。何が起こっているのかを確認するために、まず ViewSets からビューのセットを明示的に作成してみましょう。

`snippets/urls.py` ファイルでは、`ViewSet` クラスを具体的なビューのセットにバインドしています。

```python
from snippets.views import SnippetViewSet, UserViewSet, api_root
from rest_framework import renderers

snippet_list = SnippetViewSet.as_view({
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({
    'get': 'list'
})
user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})
```

それぞれの `ViewSet` クラスから複数のビューを作成し、それぞれのビューに必要なアクションに http メソッドをバインドしていることに注目してください。

リソースを具体的なビューにバインドしたので、いつものようにURL confでビューを登録します。

```python
urlpatterns = format_suffix_patterns([
    path('', api_root),
    path('snippets/', snippet_list, name='snippet-list'),
    path('snippets/<int:pk>/', snippet_detail, name='snippet-detail'),
    path('snippets/<int:pk>/highlight/', snippet_highlight, name='snippet-highlight'),
    path('users/', user_list, name='user-list'),
    path('users/<int:pk>/', user_detail, name='user-detail')
])
```

## ルータの使用

View` クラスではなく `ViewSet` クラスを使っているので、実際には URL のコンフィギュレーションを自分たちで設計する必要はありません。リソースをビューやURLに配線するための規約は、`Router`クラスを使って自動的に処理することができます。必要なのは、適切なビューセットをルータに登録して、あとはルータに任せるだけです。

これが再配線された `snippets/urls.py` ファイルです。

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from snippets import views

# ルータを作成し、それにビューセットを登録します。
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet)
router.register(r'users', views.UserViewSet)

# APIのURLはルーターが自動で決定するようになりました。
urlpatterns = [
    path('', include(router.urls)),
]
```

ルータにビューセットを登録するのは、 urlpattern を提供するのと似ています。2つの引数、つまりビューの URL プレフィックスとビューセット自体を指定します。

使用している `DefaultRouter` クラスは API ルートビューも自動的に作成してくれるので、`views` モジュールから `api_root` メソッドを削除することができます。

## ビューとビューセットのトレードオフ

ビューセットを使用することは、本当に便利な抽象化になります。URL の規約が API 全体で一貫していることを保証し、書く必要のあるコードの量を最小限に抑え、URL の conf の仕様よりも API が提供するインタラクションや表現に集中することができます。

だからといって、それが常に正しいアプローチであるとは限りません。関数ベースのビューではなく、クラスベースのビューを使用する場合と同様のトレードオフがあります。ビューセットを使用することは、個別にビューを構築するよりも明確ではありません。