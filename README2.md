## チュートリアル 2: リクエストとレスポンス
 ここからは、RESTフレームワークのコアとなる部分をカバーしていきます。ここでは、いくつかの重要なビルディングブロックを紹介します。

## リクエストオブジェクト

REST フレームワークでは、通常の HttpRequest を拡張し、より柔軟なリクエスト解析を提供する Request オブジェクトを導入しています。Request オブジェクトのコア機能は request.data 属性で、request.POST に似ていますが、Web API を扱う際にはより便利です。

``` python
request.POST  # フォームデータのみを扱います。 POST'メソッドでのみ動作します。
request.data   # 任意のデータを扱います。 POST'、'PUT'、'PATCH'メソッドで動作します。
```

## レスポンスオブジェクト

RESTフレームワークでは、レンダリングされていないコンテンツを受け取り、コンテンツネゴシエーションを使用してクライアントに返す正しいコンテンツタイプを決定するTemplateResponseのタイプであるResponseオブジェクトも導入されています。

```python
return Response(data) #クライアントが要求したコンテンツタイプにレンダリングします。
```

## ステータスコード

ビューで数値の HTTP ステータスコードを使用しても、常にわかりやすい読み物になるとは限りません。REST フレームワークは、ステータスモジュールの ```HTTP_400_BAD_REQUEST``` のように、各ステータスコードに対してより明示的な識別子を提供します。数字の識別子を使うのではなく、これらの識別子を全体的に使うのが良いでしょう。

## API ビューのラッピング

RESTフレームワークには、APIビューを書くために使用できる2つのラッパーが用意されています。

- 関数ベースのビューを扱うための @api_view デコレータ。

- クラスベースのビューを扱うための APIView クラス。

これらのラッパーは、Request インスタンスをビューで確実に受け取るようにしたり、コンテンツネゴシエーションを実行できるように Response オブジェクトにコンテキストを追加したりするなど、いくつかの機能を提供します。

ラッパーはまた、適切な場合には 405 Method Not Allowed レスポンスを返したり、不正な入力で request.data にアクセスした際に発生する ParseError 例外を処理したりといった動作も提供します。

## まとめて引っ張っていく

では、これらの新しいコンポーネントを使って、ビューを少しリファクタリングしてみましょう。

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer

@api_view(['GET',"POST"])
def snippet_list(request):
    """
    すべてのコードスニペットをリストアップするか、新しいスニペットを作成します。
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

インスタンス ビューは、以前の例よりも改善されています。少し簡潔になり、コードは Forms API を使用している場合と非常に似た感じになりました。また、名前付きのステータスコードを使用しているので、レスポンスの意味がより明確になります。

views.pyモジュール内の個々のスニペットのビューです。

```python
@api_view(['GET','PUT','DELETE'])
def snippet_detail(request, pk):
    """
    コードスニペットを取得、更新、または削除します。
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_404_NOT_FOUND)

    elif request.method == 'DELETE':
        snippet.delete()
        return Responses(status=status.HTTP_204_NO_CONTENT)
```
これは、通常の Django のビューを扱うのと大差ありません。

リクエストやレスポンスを特定のコンテンツタイプに明示的に結びつけていないことに注意してください。```request.data``` は ```json``` リクエストを扱うことができますが、他のフォーマットも扱うことができます。同様に、レスポンスオブジェクトをデータと共に返しますが、REST フレームワークがレスポンスを正しいコンテンツタイプにレンダリングしてくれるようにしています。

## URL にオプションの形式の接尾辞を追加する

レスポンスが単一のコンテンツタイプにハードワイヤードされなくなったことを利用するために、API エンドポイントにフォーマットサフィックスのサポートを追加してみましょう。フォーマットサフィックスを使用することで、指定したフォーマットを明示的に参照する URL が得られ、API は ```http://example.com/api/items/4.json``` のような URL を扱えるようになります。


以下のように、両方のビューにフォーマットキーワード引数を追加することから始めます。

```python
def snippet_list(request, format=None):
```
と
```python
def snippet_detail(request, pk, format=None):
```

既存のURLに加えて```format_suffix_patterns```のセットを追加するために、```snippets/urls.py```ファイルを少し更新します。

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

私たちは必ずしもこれらの余分なURLパターンを追加する必要はありませんが、特定のフォーマットを参照するためのシンプルでクリーンな方法を提供してくれます。

## どうですか？

チュートリアルパート1で行ったように、コマンドラインからAPIをテストしてみましょう。すべての動作はほぼ同じですが、無効なリクエストを送信した場合のエラー処理が改善されています。

以前のようにスニペットの一覧を取得することができるようになりました。

``` 
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
    "code": "print(\"hello, world\")\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```

返ってくるレスポンスのフォーマットを制御するには、Accept ヘッダを使用します。

``` 
http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
```

またはフォーマットの接尾辞をつけることで

```
http http://127.0.0.1:8000/snippets.json  # JSON suffix
http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
```

同様に、Content-Typeヘッダを使用して、送信するリクエストのフォーマットを制御することができます。

``` 
# フォームデータを使用したPOST
http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"

{
  "id": 3,
  "title": "",
  "code": "print(123)",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# JSONを使ったPOST
http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"

{
    "id": 4,
    "title": "",
    "code": "print(456)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

上記のhttpリクエストに--debugスイッチを追加すると、リクエストヘッダにリクエストタイプが表示されるようになります。

次に、http://127.0.0.1:8000/snippets/ にアクセスして、Web ブラウザで API を開きます。

## 閲覧性

API はクライアントのリクエストに基づいてレスポンスのコンテンツタイプを選択するため、デフォルトでは、リソースが Web ブラウザからリクエストされた場合、リソースの HTML 形式の表現を返します。これにより、API は完全にウェブブラウジング可能な HTML 表現を返すことができます。

ウェブブラウジング可能な API を持つことは、ユーザビリティの面で大きなメリットがあり、API の開発と使用をより簡単にすることができます。また、API の検査や作業を希望する他の開発者の参入障壁を劇的に下げることができます。

ブラウズ可能な API 機能とカスタマイズ方法の詳細については、ブラウズ可能な API トピックを参照してください。


## 次は何をするのでしょうか？

チュートリアルパート3では、クラスベースのビューの使用を開始し、ジェネリックビュー(generic views)を使用することで記述するコードの量を減らす方法を見ていきます。

[チュートリアル1（シリアル化）](https://github.com/Watson-Sei/djangorestframework-jp-tutorial)

[チュートリアル２（関数ベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README2.md)

[チュートリアル３（クラスベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README3.md)

[チュートリアル４（認証と権限）](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README4.md)