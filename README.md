# djangorestframework-jp-tutorial

## チュートリアル 1: シリアル化

### 序章

このチュートリアルでは、Web API を強調したシンプルな pastebin コードの作成について説明します。途中で、RESTフレームワークを構成する様々なコンポーネントを紹介し、どのようにしてすべてが一緒に収まるのかを包括的に理解してもらいます。

チュートリアルはかなり深いので、始める前にクッキーとお気に入りのビールを一杯飲んでおいた方がいいでしょう。簡単な概要を知りたい場合は、代わりにクイックスタートのドキュメントを見てください。

注意: このチュートリアルのコードは、GitHub の encode/rest-framework-tutorial リポジトリにあります。完成した実装は、テスト用のサンドボックス版としてオンラインで公開されています。


## 新しい環境の設定

何かをする前に、venv を使って新しい仮想環境を作成します。これにより、パッケージの設定が他のプロジェクトからうまく分離されていることを確認します。

```sh
 python3 -m venv env 
 source env/bin/activate
```

これで仮想環境に入ったので、必要なパッケージをインストールすることができます。

```sh
 pip install django
 pip install djangorestframework 
 pip install pygments  # コードの強調表示に使用します
```

注: 仮想環境をいつでも終了するには、deactivate と入力します。詳細は venv のドキュメントを参照してください。

## 始めるには

さて、コーディングの準備が整いました。始めるために、新しいプロジェクトを作成して作業をしてみましょう。

```sh
 cd ~
 django-admin startproject tutorial
 cd tutorial
```

これが終われば、シンプルなWeb APIを作成するために使用するアプリを作成することができます。

```sh
 python manage.py startapp snippets
```

新しい snippets アプリと rest_framework アプリを INSTALLED_APPS に追加する必要があります。チュートリアル/settings.pyを編集してみましょう 

```
 INSTALLED_APPS = [
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
 ]
```

よし、準備はできた

## 連携するモデルの作成

このチュートリアルでは、コードスニペットを保存するためのシンプルなスニペットモデルを作成することから始めます。snippets/models.pyファイルを編集してください。注意: プログラミングの良い習慣にはコメントが含まれています。このチュートリアルコードのリポジトリ版にもありますが、ここではコード自体に焦点を当てるためにコメントを省略しています。

```python
 from django.db import models
 from pygments.lexers import get_all_lexers
 from pygments.styles import get_all_styles

 LEXERS = [item for item in get_all_lexers() if item[1]]
 LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
 STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


 class Snippet(models.Model):
     created = models.DateTimeField(auto_now_add=True)
     title = models.CharField(max_length=100, blank=True, default='')
     code = models.TextField()
     linenos = models.BooleanField(default=False)
     language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
     style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

     class Meta:
         ordering = ['created']
```

また、スニペットモデルの初期マイグレーションを作成して、初めてデータベースを同期させる必要があります。

```sh
 python manage.py makemigrations snippets
 python manage.py migrate 
```

## シリアライザクラスの作成

Web API を始めるために最初に必要なことは、スニペットインスタンスをシリアライズして json のような表現にデシリアライズする方法を提供することです。これは、Django のフォームに非常に似た動作をするシリアライザを宣言することで実現できます。スニペットディレクトリに serializers.py という名前のファイルを作成し、以下を追加します。

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        検証済みのデータを指定して新しい `Snippet` インスタンスを作成して返します。
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        既存の `Snippet` インスタンスを更新して返します。
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

シリアライザクラスの最初の部分は、シリアライズ/デシリアライズされるフィールドを定義します。create() メソッドと update() メソッドは、serializer.save() を呼び出したときにどのようにして完全なインスタンスが作成されるか、または修正されるかを定義します。

シリアライザクラスは Django Form クラスに非常に似ていて、必須、max_length、default などの様々なフィールドに同様の検証フラグを含みます。

フィールドフラグは、HTML へのレンダリング時のような特定の状況下でシリアライザがどのように表示されるかを制御することもできます。上の {'base_template': 'textarea.html'} フラグは、Django フォームクラスで widget=widgets.Textarea を使うのと同じです。これはチュートリアルで後述するように、ブラウジング可能な API をどのように表示するかを制御するのに特に便利です。

後ほど説明するように、ModelSerializer クラスを使うことで時間の節約にもなりますが、今はシリアライザの定義を明示的にしておきましょう。


## シリアライザでの作業

先に進む前に、新しいシリアライザクラスの使い方を説明します。Django シェルに入ってみましょう。

```sh
 python manage.py shell 
```

さて、いくつかのインポートができたら、作業するためのコードスニペットをいくつか作成してみましょう。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print("hello, world")\n')
snippet.save()
```

これで、いくつかの snippet インスタンスを遊べるようになりました。これらのインスタンスの一つをシリアライズしてみましょう。

``` 
serializer = SnippetSerializer(snippet)
serializer.data
# {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}
```

この時点で、モデルインスタンスをPythonネイティブのデータ型に変換しました。シリアライズ処理を最終的に行うために、データをjsonにレンダリングします。

``` 
content = JSONRenderer().render(serializer.data)
content
# b'{"id": 2, "title": "", "code": "print(\\"hello, world\\")\\n", "linenos": false, "language": "python", "style": "friendly"}'
```

デシリアライズも似たようなものです。最初にストリームをPythonネイティブのデータ型に解析します...

```python
import io

stream = io.BytesIO(content)
data = JSONParser().parse(stream)
```

...そして、それらのネイティブなデータ型を、完全に実装されたオブジェクトのインスタンスに復元します。

```python
serializer = SnippetSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# OrderedDict([('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()
# <Snippet: Snippet object>
```

APIがフォームの操作とどれだけ似ているかに注目してください。この類似性は、シリアライザを使用したビューを書き始めると、さらに明らかになるはずです。

モデルインスタンスの代わりにクエリセットをシリアライズすることもできます。そのためにはシリアライザの引数に many=True フラグを追加するだけです。

```python
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
# [OrderedDict([('id', 1), ('title', ''), ('code', 'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', ''), ('code', 'print("hello, world")'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```

## モデルシリアライザの使用
私たちの SnippetSerializer クラスは、Snippet モデルにも含まれる多くの情報を複製しています。コードをもう少し簡潔にまとめることができればいいのですが。

Django が Form クラスと ModelForm クラスの両方を提供しているのと同じように、REST フレームワークには Serializer クラスと ModelSerializer クラスの両方が含まれています。

ModelSerializer クラスを使ってシリアライザをリファクタリングしてみましょう。snippets/serializers.py というファイルを再度開き、SnippetSerializer クラスを以下のように置き換えます。

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style']
```

シリアライザが持つ素晴らしい特性の一つは、シリアライザのインスタンス内のすべてのフィールドを表示することで検査できるということです。Django シェルを python manage.py シェルで開き、以下のようにしてみてください。

```python
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
# SnippetSerializer():
#    id = IntegerField(label='ID', read_only=True)
#    title = CharField(allow_blank=True, max_length=100, required=False)
#    code = CharField(style={'base_template': 'textarea.html'})
#    linenos = BooleanField(required=False)
#    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
#    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```

重要なのは、ModelSerializerクラスは特に不思議なことをするわけではなく、単にシリアライザクラスを作成するためのショートカットであることを覚えておくことです。

・自動的に決定されるフィールドのセット。

・create() メソッドと update() メソッドのシンプルなデフォルト実装。

## Serializer を使って通常の Django ビューを書く
新しい Serializer クラスを使って API ビューを書く方法を見てみましょう。今のところ、REST フレームワークの他の機能は使わないので、通常の Django のビューとして書きます。

snippets/views.py ファイルを編集し、以下を追加します。

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```

APIのルートは、既存のスニペットをすべてリストアップしたり、新しいスニペットを作成したりするビューになる予定です。

```python
@csrf_exempt
def snippet_list(request):
    """
    すべてのコードスニペットをリストアップするか、新しいスニペットを作成します。
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
```

CSRF トークンを持たないクライアントからこのビューに POST できるようにしたいので、ビューを csrf_exempt としてマークする必要があることに注意してください。RESTフレームワークのビューはこれよりも実際にはより賢明な動作をしますが、今の目的のためにはこれで十分です。

また、個々のスニペットに対応し、スニペットを取得、更新、削除するために使用できるビューも必要です。

```python
@csrf_exempt
def snippet_detail(request, pk):
    """
    コードスニペットを取得、更新、または削除します。
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```

最後に、これらのビューを配線する必要があります。snippets/urls.pyを作成します。

```python
from django.urls import path
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>/', views.snippet_detail),
]
```

また、tutorial/urls.pyファイルのルートurlconfを配線して、スニペットアプリのURLを含める必要があります。

```python
from django.urls import path, include

urlpatterns = [
    path('', include('snippets.urls')),
]
```

現時点で適切に対処できていないエッジケースがいくつかあることは注目に値します。もし不正なjsonを送信したり、ビューが処理しないメソッドでリクエストが行われた場合、500の "server error "レスポンスが発生してしまいます。それでも、今のところはこれで大丈夫です。

## 初めての試みである Web API のテスト
これで、スニペットを提供するサンプルサーバを立ち上げることができるようになりました。

シェルを終了します...

```
quit()
```

...とDjangoの開発サーバーを立ち上げます。

```sh
python manage.py runserver

Validating models...

0 errors found
Django version 1.11, using settings 'tutorial.settings'
Development server is running at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
 ```

別のターミナルウィンドウで、サーバーをテストすることができます。

curl や httpie を使って API をテストすることができます。Httpie は Python で書かれたユーザーフレンドリーな http クライアントです。それをインストールしてみましょう。

httpie をインストールするには pip:
```sh
 pip install httpie
```

最後にスニペットの一覧を取得します。

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

また、IDを参照することで特定のスニペットを取得することもできます。

```
http http://127.0.0.1:8000/snippets/2/

HTTP/1.1 200 OK
...
{
  "id": 2,
  "title": "",
  "code": "print(\"hello, world\")\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```

同様に、これらのURLにWebブラウザでアクセスすることで、同じjsonを表示させることができます。

## 今どこにいるの？

Django の Forms API と似たような感じのシリアライズ API と、いくつかの通常の Django ビューを持っています。

API ビューは今のところ、json レスポンスを提供するだけで、特別なことは何もしていませんし、エラー処理のエッジケースもありますが、機能する Web API です。

チュートリアルのパート2では、どのようにして改善を始めることができるかを見ていきます。





## 翻訳更新日:2020/3/31

100％の精度で翻訳ができていません。間違っているところがあれば教えてください。
