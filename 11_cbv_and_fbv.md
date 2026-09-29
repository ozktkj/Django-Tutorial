# 01. CBV と FBV

## 2 つの書き方

Django のビューは、クラスでも関数でも書けます。どちらも「リクエストを受け取ってレスポンスを返す」という点は同じです。

| | クラスベースビュー（CBV） | 関数ベースビュー（FBV） |
| --- | --- | --- |
| 書く量 | 少ない（決まった処理は親クラスがやる） | 多い（すべて自分で書く） |
| 処理の流れ | メソッドに分かれていて、追うには親クラスを読む必要がある | 上から下へ読めば分かる |
| カスタマイズ | オーバーライドするメソッドを知っている必要がある | 普通の Python のように書き換えられる |
| 向いている場面 | 一覧・詳細・作成・更新などの定型処理 | 画面ごとに処理が大きく違う、複数のフォームを扱う、など |

どちらが正しいということはありません。実際の現場でも両方が使われています。
FBV で一度書いてみると、CBV が何をしてくれているのかが分かり、CBV のカスタマイズもしやすくなります。

## 基本編のビューとの対応

| 機能 | 基本編（CBV） | 発展編（FBV） |
| --- | --- | --- |
| 新規登録 | `SignUpView(CreateView)` | `signup_view` |
| ログイン | `auth_views.LoginView` | `login_view` |
| ログアウト | `auth_views.LogoutView` | `logout_view` + `@require_POST` |
| ホーム | `HomeView(LoginRequiredMixin, TemplateView)` | `home_view` + `@login_required` |
| プロフィール編集 | なし | `profile_edit_view`（追加） |

## CBV が裏でやっていたこと

たとえば `LoginView` は、次のような処理をまとめて引き受けています。FBV では、これを自分で書くことになります。

1. GET なら空のフォーム、POST なら送信内容を入れたフォームを作る
2. フォームの検証（ユーザー名とパスワードの確認）
3. 成功したら `login()` でセッションにユーザーを記録する
4. `next` パラメータが**安全な URL なら**そこへ、そうでなければ `LOGIN_REDIRECT_URL` へリダイレクトする
5. 失敗したらエラー付きのフォームで同じ画面を返す
6. `redirect_authenticated_user=True` なら、ログイン済みの人をリダイレクトする

特に 4 の「安全な URL か」の確認は、書き忘れるとセキュリティ上の問題になります（03章）。

## 使う主な関数

| 関数・デコレーター | import 元 | 役割 |
| --- | --- | --- |
| `render()` | `django.shortcuts` | テンプレートを描画してレスポンスを返す |
| `redirect()` | `django.shortcuts` | URL 名・URL・モデルを受け取り、リダイレクトする |
| `login()` / `logout()` | `django.contrib.auth` | セッションにユーザーを記録する / 消す |
| `@login_required` | `django.contrib.auth.decorators` | 未ログインなら `LOGIN_URL` へ飛ばす |
| `@require_POST` など | `django.views.decorators.http` | 受け付ける HTTP メソッドを制限する |
| `messages` | `django.contrib.messages` | 次の画面に 1 回だけ表示するメッセージ |

## views.py の import

```python
# accounts/views.py
from django.contrib import messages
from django.contrib.auth import login, logout
from django.contrib.auth.decorators import login_required
from django.contrib.auth.forms import AuthenticationForm
from django.conf import settings
from django.shortcuts import redirect, render, resolve_url
from django.utils.http import url_has_allowed_host_and_scheme
from django.views.decorators.http import require_http_methods, require_POST

from .forms import CustomUserCreationForm, ProfileForm
```

## URL

関数を登録するときは、`.as_view()` は付けません。

```python
from django.urls import path

from . import views

app_name = "accounts"

urlpatterns = [
    path("", views.home_view, name="home"),
    path("signup/", views.signup_view, name="signup"),
    path("login/", views.login_view, name="login"),
    path("logout/", views.logout_view, name="logout"),
    path("profile/", views.profile_edit_view, name="profile_edit"),
]
```

URL 名（`accounts:login` など）は基本編と同じにしているので、テンプレートや `settings.py` の `LOGIN_URL` などは変更不要です。
