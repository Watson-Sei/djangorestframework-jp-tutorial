# チュートリアル 3: クラスベースのビュー

また、関数ベースのビューではなく、クラスベースのビューを使って API ビューを書くこともできます。これは強力なパターンで、共通の機能を再利用することができ、コードをDRYに保つのに役立ちます。

クラスベースのビューを使用して API を書き換える
ルートビューをクラスベースのビューに書き換えることから始めます。これに必要なのは、views.pyを少しリファクタリングすることです。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    すべてのスニペットをリストアップするか、新しいスニペットを作成します。
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

ここまでは順調です。先ほどのケースとよく似ていますが、異なるHTTPメソッド間の分離が良くなりました。views.pyのインスタンスビューも更新する必要があります。

```python
class SnippetDetail(APIView):
    """
    スニペットのインスタンスを取得、更新、削除します。
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

いい感じですね。繰り返しになりますが、今のところは関数ベースのビューとほとんど変わりません。

クラスベースのビューを使用しているので、snippets/urls.pyも少しリファクタリングする必要があります。

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.SnippetList.as_view()),
    path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

さて、これで完了です。開発サーバーを実行すれば、すべてが以前と同じように動作するはずです。

## ミックスインの使用

クラスベースのビューを使う大きな利点の一つは、再利用可能な動作を簡単に構成できることです。

これまで使用してきた作成/検索/更新/削除の操作は、私たちが作成するどのモデルバックAPIビューでも似たようなものになります。これらの共通の動作はRESTフレームワークのmixinクラスに実装されています。

ミキシングクラスを使ってビューを構成する方法を見てみましょう。再び```views.py```モジュールを見てみましょう。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import mixins
from rest_framework import generics

class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```
ここで何が起こっているのかを正確に調べてみましょう。GenericAPIView を使用してビューを構築し、ListModelMixin と CreateModelMixin を追加しています。

基底クラスはコア機能を提供し、mixinクラスは.list()と.create()アクションを提供します。そして、明示的に get メソッドと post メソッドを適切なアクションにバインドしています。ここまでは十分にシンプルです。

```python
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

よく似ています。ここでもGenericAPIViewクラスを使用してコア機能を提供し、.retrieve()、.update()、.destroy()アクションを提供するためにミックスインを追加しています。

## 一般的なクラスベースのビューを使用する

ミキシングクラスを使って、以前よりも少し少ないコードで使えるようにビューを書き換えましたが、さらに一歩進んでみましょう。RESTフレームワークは、すでにミックスインされている一般的なビューのセットを提供しており、これを使ってviews.pyモジュールをさらにスリム化することができます。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

うわー、かなり簡潔ですね。膨大な量を無料で手に入れることができましたし、私たちのコードは良くできていて、クリーンでイディオムな Django のように見えます。

次はチュートリアルのパート 4 に進み、API の認証とパーミッションの扱い方を見ていきます。

[チュートリアル1（シリアル化）](https://github.com/Watson-Sei/djangorestframework-jp-tutorial)

[チュートリアル２（関数ベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README2.md)

[チュートリアル３（クラスベースView)](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README3.md)

[チュートリアル４（認証と権限）](https://github.com/Watson-Sei/djangorestframework-jp-tutorial/blob/master/README4.md)