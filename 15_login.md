# 15. ログインビュー

この教材の中心です。基本編では `auth_views.LoginView` の 1 行で済んでいた処理を、自分で書きます。

## LoginView がやっていたこと

1. GET なら空のフォーム、POST なら送信内容を入れたフォームを作る
2. ユーザー名とパスワードを確認する
3. 成功したら `login()` でセッションにユーザーを記録する
4. `next` が**安全な URL なら**そこへ、なければ `LOGIN_REDIRECT_URL` へリダイレクトする
5. 失敗したら、エラー付きのフォームで同じ画面を返す
6. ログイン済みの人が開いたらリダイレクトする（`redirect_authenticated_user=True` のとき）

これを順に書いていきます。

## コード

```python
# accounts/views.py（追記）
from django.conf import settings
from django.contrib import messages
from django.contrib.auth import login
from django.contrib.auth.forms import AuthenticationForm
from django.shortcuts import redirect, render, resolve_url
from django.utils.http import url_has_allowed_host_and_scheme
from django.views.decorators.http import require_http_methods


def _get_safe_next_url(request):
    """next パラメータが同じサイト内の URL なら返す。外部サイトなら None"""
    next_url = request.POST.get("next") or request.GET.get("next")
    if next_url and url_has_allowed_host_and_scheme(
        url=next_url,
        allowed_hosts={request.get_host()},
        require_https=request.is_secure(),
    ):
        return next_url
    return None


@require_http_methods(["GET", "POST"])
def login_view(request):
    if request.user.is_authenticated:
        return redirect("accounts:home")

    if request.method == "POST":
        form = AuthenticationForm(request, data=request.POST)
        if form.is_valid():
            login(request, form.get_user())
            messages.success(request, "ログインしました。")
            return redirect(
                _get_safe_next_url(request) or resolve_url(settings.LOGIN_REDIRECT_URL)
            )
    else:
        form = AuthenticationForm(request)

    return render(
        request,
        "accounts/login.html",
        {"form": form, "next": _get_safe_next_url(request) or ""},
    )
```

```python
# accounts/urls.py（追記）
    path("login/", views.login_view, name="login"),
```

## フォームを扱う FBV の基本形

```python
if request.method == "POST":
    form = フォーム(request.POST)   # 送信内容を入れる
    if form.is_valid():             # 検証
        ...                         # 成功時の処理
        return redirect(...)        # 成功 → リダイレクト
else:
    form = フォーム()               # GET → 空のフォーム

return render(request, "...", {"form": form})   # GET、または検証失敗
```

最後の `render()` は、**GET のときと、POST で検証に失敗したときの両方**が通ります。
検証に失敗したフォームにはエラー情報が入っているので、`{{ form.as_p }}` がそのままエラーメッセージを表示してくれます。

## AuthenticationForm

ユーザー名とパスワードを確認するフォームです。内部で `authenticate()` を呼び、ユーザーが存在し、パスワードが合っていて、`is_active` が `True` のときだけ `is_valid()` が `True` になります。

**引数の渡し方がほかのフォームと違います。**

```python
form = AuthenticationForm(request, data=request.POST)   # 1 つ目は request
```

- 1 つ目の引数は `request` です。`AuthenticationForm(request.POST)` と書くと、POST データが `request` として扱われ、正しく動きません。
- 送信内容は `data=` で渡します。
- 成功したら `form.get_user()` でユーザーを取り出します。

### authenticate() を直接使う書き方

```python
from django.contrib.auth import authenticate, login

user = authenticate(request, username=username, password=password)
if user is not None:
    login(request, user)
```

仕組みは分かりやすいのですが、入力チェック・エラーメッセージ・`is_active` の確認などを自分で書くことになります。特別な理由がなければ `AuthenticationForm` を使います。

## login() がしていること

- セッションに、ユーザーの ID と使った認証バックエンドを保存する
- **セッション ID を作り直す**: ログイン前のセッション ID を攻撃者が知っていても、ログイン後には使えなくなる（セッション固定攻撃の対策）
- CSRF トークンを作り直す
- `user_logged_in` シグナルを送り、`last_login` を更新する

「ログイン状態」の正体は、このセッションです。

## next パラメータとオープンリダイレクト

07章で `@login_required` を付けると、未ログインの人は `/login/?next=/profile/` のような URL へ飛ばされます。ログイン後は `next` の画面へ戻すのが親切です。

ただし、`next` の値をそのまま使ってはいけません。

```python
# ❌ 危険
return redirect(request.GET.get("next"))
```

次のような URL を踏ませると、本物のサイトでログインした直後に、偽サイトへ送り込めてしまいます（**オープンリダイレクト**）。

```
https://本物のサイト/login/?next=https://偽サイト/
```

利用者は本物のサイトでログインしたつもりなので、偽サイトで「もう一度パスワードを入力してください」と出されると信じてしまいます。

### url_has_allowed_host_and_scheme()

URL が許可したホストのものかを確かめる関数です。`LoginView` も内部で同じものを使っています。

| `next` の値 | 判定 |
| --- | --- |
| `/profile/` | ✅ 同じサイト内 |
| `https://evil.example.com/` | ❌ 別のホスト |
| `//evil.example.com/` | ❌ スキームを省略した外部 URL |
| `javascript:alert(1)` | ❌ 許可していないスキーム |

`require_https=request.is_secure()` を指定すると、HTTPS のページからは HTTPS の URL にしか移動しません。

この確認を `_get_safe_next_url()` という関数にまとめておくと、`next` を扱うビューが増えても同じ処理を使い回せます。

### GET と POST の両方から読む

```python
next_url = request.POST.get("next") or request.GET.get("next")
```

- 画面を開いたとき（GET）は、`?next=...` から受け取る
- フォームを送信したとき（POST）は、hidden 項目から受け取る

そのため、テンプレートで hidden 項目に入れて渡します。

## テンプレート

```html
{% extends "base.html" %}
{% block title %}ログイン{% endblock %}
{% block content %}
<h1>ログイン</h1>
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <input type="hidden" name="next" value="{{ next }}">
  <button type="submit">ログイン</button>
</form>
{% endblock %}
```

`LoginView` はコンテキストに `next` を自動で入れてくれていました。FBV では `render()` に自分で渡します。

ヘッダーにログイン状態を表示するようにします。

```html
<!-- templates/base.html のヘッダー -->
  <header>
    {% if user.is_authenticated %}
      <span>{{ user }} さん</span>
    {% else %}
      <a href="{% url 'accounts:login' %}">ログイン</a>
    {% endif %}
  </header>
```

## redirect() と resolve_url()

```python
return redirect(_get_safe_next_url(request) or resolve_url(settings.LOGIN_REDIRECT_URL))
```

`settings.LOGIN_REDIRECT_URL` には `"accounts:home"` という URL 名が入っています。`resolve_url()` は、URL 名でも URL でも受け取って実際の URL に変換します。
（`redirect()` 自体も URL 名を扱えるので省略できますが、「URL に変換してから使う」ことを明示しています。）

## 動作確認

1. `/login/` を開く
2. 03章で作った管理ユーザーでログインする → ホームへ移動し、ヘッダーに名前が出る
3. パスワードをわざと間違える → 同じ画面にエラーが表示される
4. `/login/?next=/admin/` を開いてログインする → 管理画面へ移動する
5. `/login/?next=https://example.com/` を開いてログインする → **ホームへ移動する**（外部 URL は無視される）

ログアウトはまだ作っていないので、状態を戻したいときはブラウザのクッキーを消すか、プライベートウィンドウを使ってください。次章で作ります。
