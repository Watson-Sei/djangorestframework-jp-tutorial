# チュートリアル 5: リレーションシップとハイパーリンクAPI

現在のところ、API内のリレーションシップは主キーを使用して表現されています。このチュートリアルでは、リレーションシップにハイパーリンクを使用する代わりに API の結合性と発見性を向上させます。

## APIのルートのエンドポイントを作成する

今のところ、'snippets' と 'users' のエンドポイントはありますが、APIへのエントリーポイントは一つもありません。

しかし、APIのエントリーポイントは1つしかありません。1つを作成するには、通常の関数ベースのビューと先ほど紹介した `@api_view` デコレータを使用します。`snippets/views.py`に追加します:
```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```

ここで注目すべきことが2つあります。第一に、完全に修飾されたURLを返すためにRESTフレームワークの `reverse` 関数を使用していること、第二に、URLのパターンは便利な名前で識別され、後ほど `snippets/urls.py` で宣言します。

## ハイライトされたスニペットのエンドポイントを作成する

pastebin APIにまだ欠けているもう一つの明白なものは、エンドポイントをハイライトするコードです。

他のすべてのAPIエンドポイントとは異なり、JSONを使用するのではなく、HTML表現を提示するだけです。REST フレームワークが提供する HTML レンダラには 2 つのスタイルがあり、1 つはテンプレートを使用してレンダリングされた HTML を扱うためのもので、もう 1 つは事前にレンダリングされた HTML を扱うためのものです。2つ目のレンダラーは、このエンドポイントで使用したいものです。

コードハイライトビューを作成する際に考慮しなければならないもう一つのことは、使用できる具体的な汎用ビューが存在しないということです。

オブジェクトインスタンスを返すのではなく、オブジェクトインスタンスのプロパティを返しています。

具象的な汎用ビューを使う代わりに、インスタンスを表現するための基底クラスを使い、独自の `.get()` メソッドを作成します。`snippets/views.py` の中に追加します。

```python
from rest_framework import renderers
from rest_framework.response import Response

class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = [renderers.StaticHTMLRenderer]

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```

いつものように、作成した新しいビューをURLconfに追加する必要があります。新しいAPIルートのURLパターンを `snippets/urls.py` に追加します。

```python
path('', views.api_root),
```

そして、スニペットのハイライトのURLパターンを追加します:
```python
path('snippets/<int:pk>/highlight/', views.SnippetHighlight.as_view()),
```

## API のハイパーリンク

エンティティ間のリレーションシップを扱うことは、Web API デザインの中でも特に難しいことの一つです。リレーションシップを表現する方法はいくつかあります。

- 主キーの使用。
- エンティティ間のハイパーリンクの使用
- 関連するエンティティの一意の識別スラッグ・フィールドを使用する。
- 関連するエンティティの既定の文字列表現を使用する。
- 関連するエンティティを親表現の中に入れ子にする。
- その他のカスタム表現。

RESTフレームワークはこれらのスタイルをすべてサポートしており、フォワード・リレーションシップやリバース・リレーションシップにまたがって適用したり、ジェネリックな外部キーのようなカスタム・マネージャーにまたがって適用したりすることができます。

この場合、エンティティ間のハイパーリンクスタイルを使用したいと思います。そのために、既存の `ModelSerializer` の代わりに `HyperlinkedModelSerializer` を拡張するようシリアライザを変更します。

HyperlinkedModelSerializer`には、`ModelSerializer`との以下の違いがある。

- デフォルトでは `id` フィールドを含まない。

- HyperlinkedIdentityField`を用いて `url` フィールドを含む。

- リレーションシップは `PrimaryKeyRelatedField` の代わりに `HyperlinkedRelatedField` を使う。

既存のシリアライザをハイパーリンクを使用するように簡単に書き換えることができます。あなたの `snippets/serializers.py` の中に追加します:
```python
class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ['url', 'id', 'highlight', 'owner',
                  'title', 'code', 'linenos', 'language', 'style']


class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ['url', 'id', 'username', 'snippets']
```

新しい `'highlight'` フィールドを追加したことに注目してください。このフィールドは `url` フィールドと同じタイプですが、`'snippet-detail'` urlパターンの代わりに `'snippet-highlight'` urlパターンを指します。

ここでは、`'.json'`のようなフォーマット接尾辞付きのURLが含まれているので、`highlight`フィールドで、フォーマット接尾辞付きのハイパーリンクが返される場合は、`'.html'`接尾辞を使うべきであることを示しておく必要があります。

## URL パターンに名前をつけていることを確認する

ハイパーリンクAPIを使うのであれば、URLパターンに名前を付ける必要があります。どの URL パターンに名前を付ける必要があるのか見てみましょう。

- APIのルートは `'user-list' `と `'snippet-list'` を参照する。

- スニペットシリアライザは `'snippet-highlight'` を参照するフィールドを含む。

- ユーザシリアライザには `'snippet-detail'` を参照するフィールドがある。

- スニペットとユーザのシリアライザには `'url'` フィールドが含まれており、デフォルトでは `'{model_name}-detail'` を参照します。

これらの名前をすべて URLconf に追加した後、最終的な `snippets/urls.py` ファイルは以下のようになります。

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

# API endpoints
urlpatterns = format_suffix_patterns([
    path('', views.api_root),
    path('snippets/',
        views.SnippetList.as_view(),
        name='snippet-list'),
    path('snippets/<int:pk>/',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    path('snippets/<int:pk>/highlight/',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    path('users/',
        views.UserList.as_view(),
        name='user-list'),
    path('users/<int:pk>/',
        views.UserDetail.as_view(),
        name='user-detail')
])
```

ページネーションの追加

ユーザーやコードスニペットのリストビューは、かなり多くのインスタンスを返すことになるかもしれないので、実際には結果をページ分割して、APIクライアントが個々のページをステップスルーできるようにしたいと考えています。

`tutorial/settings.py`ファイルを少し修正することで、デフォルトのリストスタイルをページ分割を使用するように変更することができます。以下の設定を追加します。

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

RESTフレームワークの設定は、`REST_FRAMEWORK`という名前の単一の辞書設定に名前空間を設定していることに注意してください。

必要に応じてページ分割スタイルをカスタマイズすることもできますが、この場合はデフォルトのままにしておきます。

## APIの閲覧


ブラウザを開いてブラウジング可能なAPIに移動すると、リンクをたどるだけでAPIを操作できるようになっていることがわかります。

また、スニペットインスタンスの「ハイライト」リンクを見ることができ、ハイライトされたコードのHTML表現に移動することができます。

チュートリアルのパート6では、APIを構築するために必要なコード量を減らすためにViewSetsとRoutersを使用する方法を見ていきます。

[チュートリアル1（シリアル化）](https://github.com/Watson-Sei/djangorestframework-jp-tutorial)

[チュートリアル２（関数ベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README2.md)

[チュートリアル３（クラスベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README3.md)

[チュートリアル４（認証と権限）](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README4.md)

[チュートリアル５（リレーショナルシップとハイパーリンク）](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README5.md)