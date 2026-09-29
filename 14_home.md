# 14. 最初のビュー: ホーム画面

ここから関数ベースビュー（FBV）を書いていきます。まずは一番単純なビューで、全体のつながりを確認します。

## ビューは「リクエストを受け取ってレスポンスを返す関数」

Django のビューの正体は、次の形の関数です。

```python
def ビュー名(request):
    return レスポンス
```

`request` には、送られてきた HTTP リクエストの情報が入っています。

| 属性 | 中身 |
| --- | --- |
| `request.method` | `"GET"` や `"POST"` |
| `request.GET` | クエリ文字列（`?next=/profile/` など） |
| `request.POST` | フォームの送信内容 |
| `request.user` | ログイン中のユーザー。未ログインなら `AnonymousUser` |

CBV も最終的には、この形の関数に変換されて呼ばれます（`as_view()` がその変換を行います）。

## ホームビュー

```python
# accounts/views.py
from django.shortcuts import render


def home_view(request):
    return render(request, "accounts/home.html")
```

`render(request, テンプレート名, コンテキスト)` は、テンプレートを描画して `HttpResponse` を返すショートカットです。
第 3 引数を省略しているのは、テンプレートで使う `user` が、コンテキストプロセッサーによって自動で渡されるからです。

> この時点では、ログインしていなくてもホームを見られます。07章で `@login_required` を付けて制限します。

## URL

アプリ側と、プロジェクト側の両方を書きます。

```python
# accounts/urls.py（新規作成）
from django.urls import path

from . import views

app_name = "accounts"

urlpatterns = [
    path("", views.home_view, name="home"),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("accounts.urls")),
]
```

- **`.as_view()` は付けません。** 関数そのものを渡します。
- `app_name = "accounts"` により、URL 名は `accounts:home` のようになります（名前空間）。
- URL 名を付けておくと、テンプレートでは `{% url 'accounts:home' %}`、Python 側では `reverse("accounts:home")` で URL を組み立てられます。URL のパスを変えても、書き換えるのは `urls.py` だけで済みます。

## テンプレート

まずは最小限のベーステンプレートを作ります。ヘッダーの中身は、ログイン画面を作る 05章から少しずつ足していきます。

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>{% block title %}Django認証チュートリアル{% endblock %}</title>
</head>
<body>
  <header>
  </header>
  <main>
    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

```html
{% extends "base.html" %}
{% block title %}ホーム{% endblock %}
{% block content %}
<h1>ようこそ、{{ user }} さん</h1>
<ul>
  <li>ユーザー名: {{ user.username }}</li>
  <li>メール: {{ user.email }}</li>
  <li>ニックネーム: {{ user.nickname|default:"未設定" }}</li>
</ul>
{% endblock %}
```

`{{ user }}` は、ログインしていれば `CustomUser`、していなければ `AnonymousUser` です。`AnonymousUser` には `nickname` がないので、この時点で未ログインのまま開くと、ニックネームの欄は空で表示されます（テンプレートでは、存在しない属性はエラーにならず空文字になります）。

## 動作確認

```bash
python manage.py runserver
```

`http://127.0.0.1:8000/` にホーム画面が表示されれば成功です。

## この章のまとめ

FBV で画面を 1 つ作る手順は、いつも同じです。

1. `views.py` に関数を書く
2. `urls.py` に `path()` を足す
3. テンプレートを作る

次章から、この手順を繰り返しながら認証まわりを組み立てていきます。
