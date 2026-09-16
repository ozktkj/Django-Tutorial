# 04. 新規登録・ログイン・ログアウト

ログイン・ログアウトは `django.contrib.auth.views` の汎用ビューを使い、新規登録だけ `CreateView` で作ります。

## ビュー

```python
# accounts/views.py
from django.contrib.auth import login
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, TemplateView

from .forms import CustomUserCreationForm


class SignUpView(CreateView):
    form_class = CustomUserCreationForm
    template_name = "accounts/signup.html"
    success_url = reverse_lazy("accounts:home")

    def form_valid(self, form):
        response = super().form_valid(form)
        login(self.request, self.object)  # 登録後そのままログイン状態にする
        return response


class HomeView(LoginRequiredMixin, TemplateView):
    template_name = "accounts/home.html"
```

- `super().form_valid()` の中で `form.save()` が呼ばれ、`self.object` に保存したユーザーが入ります。パスワードのハッシュ化は `UserCreationForm.save()` がやってくれます。
- `login()` を呼ぶとセッションが作られます。登録後にログイン画面へ戻したい場合は、この行を削除して `success_url` を変えます。
- `LoginRequiredMixin` は**継承リストの一番左**に書きます。未ログインだと `LOGIN_URL` に `?next=/` を付けてリダイレクトします。

## URL

```python
# accounts/urls.py
from django.contrib.auth import views as auth_views
from django.urls import path

from . import views

app_name = "accounts"

urlpatterns = [
    path("", views.HomeView.as_view(), name="home"),
    path("signup/", views.SignUpView.as_view(), name="signup"),
    path(
        "login/",
        auth_views.LoginView.as_view(
            template_name="accounts/login.html",
            redirect_authenticated_user=True,
        ),
        name="login",
    ),
    path("logout/", auth_views.LogoutView.as_view(), name="logout"),
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

- `redirect_authenticated_user=True`: ログイン済みの人がログイン画面を開くと、`LOGIN_REDIRECT_URL` へ移動します。
- `LoginView` のテンプレートは、指定しなければ `registration/login.html` が使われます。
- `include("django.contrib.auth.urls")` を使うと、パスワード変更・リセットの URL もまとめて登録できます。ただし名前空間は付きません（`login`, `logout` などの名前になる）。

## ⚠️ Django 5.0 以降: ログアウトは POST のみ

Django 4.1 で GET によるログアウトが非推奨になり、**5.0 で削除**されました。`<a href="/logout/">` のリンクでは **405 Method Not Allowed** になります。
CSRF 対策（他のサイトに置かれたリンクで勝手にログアウトさせられる攻撃を防ぐ）のためです。

ログアウトは必ず、`{% csrf_token %}` を入れたフォームで POST 送信します。

## テンプレート

### base.html

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
    {% if user.is_authenticated %}
      <span>{{ user }} さん</span>
      <form action="{% url 'accounts:logout' %}" method="post" style="display:inline">
        {% csrf_token %}
        <button type="submit">ログアウト</button>
      </form>
    {% else %}
      <a href="{% url 'accounts:login' %}">ログイン</a>
      <a href="{% url 'accounts:signup' %}">新規登録</a>
    {% endif %}
  </header>
  <main>
    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

`user` は `django.contrib.auth.context_processors.auth`（初期設定で有効）によって、すべてのテンプレートで使えます。未ログインのときは `AnonymousUser` が入ります。

### login.html

```html
<!-- templates/accounts/login.html -->
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

hidden の `next` を入れておくと、`LoginRequiredMixin` で飛ばされてきたときに、ログイン後に元のページへ戻れます。
ここに外部サイトの URL を入れられても、`LoginView` が同じホストかどうかを確認するので、そのまま外部へ飛ばされることはありません。

### signup.html

```html
<!-- templates/accounts/signup.html -->
{% extends "base.html" %}
{% block title %}新規登録{% endblock %}
{% block content %}
<h1>新規登録</h1>
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">登録する</button>
</form>
{% endblock %}
```

### home.html

```html
<!-- templates/accounts/home.html -->
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

## 画面の流れ

```
未ログインで /  ──▶ /login/?next=/ ──(ログイン成功)──▶ /
/signup/ ──(登録成功・自動ログイン)──▶ /
[ログアウト] ボタン (POST /logout/) ──▶ /login/
```
