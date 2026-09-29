# 16. ログアウトとメッセージ

## ログアウトビュー

```python
# accounts/views.py（追記）
from django.contrib.auth import logout
from django.views.decorators.http import require_POST


@require_POST
def logout_view(request):
    logout(request)
    messages.info(request, "ログアウトしました。")
    return redirect(settings.LOGOUT_REDIRECT_URL)
```

```python
# accounts/urls.py（追記）
    path("logout/", views.logout_view, name="logout"),
```

- `logout(request)`: セッションのデータをすべて削除し、`request.user` を `AnonymousUser` にします。
- `@require_POST`: POST 以外には **405 Method Not Allowed** を返します。

## なぜ POST だけにするのか

GET でログアウトできると、他人のページに置かれた次のようなタグを表示しただけで、ログアウトさせられてしまいます。

```html
<img src="https://あなたのサイト/logout/">
```

実害は小さいものの、これは CSRF（意図しない操作をさせられる攻撃）の一種です。Django も同じ理由で、**5.0 から `LogoutView` の GET を廃止しました**。基本編で `<a>` タグではなくフォームを使ったのはこのためです。

```html
<!-- templates/base.html のヘッダー（ログイン中） -->
<form action="{% url 'accounts:logout' %}" method="post" style="display:inline">
  {% csrf_token %}
  <button type="submit">ログアウト</button>
</form>
```

`{% csrf_token %}` を忘れると 403 になります。

## メッセージフレームワーク

「ログインしました」のように、**次の画面で 1 回だけ表示したいメッセージ**には `django.contrib.messages` を使います。`startproject` の初期設定で有効になっています。

```python
messages.success(request, "ログインしました。")
messages.info(request, "ログアウトしました。")
messages.warning(request, "…")
messages.error(request, "…")
```

登録したメッセージは、リダイレクト先の画面で取り出せます。

```html
<!-- templates/base.html の <header> と <main> の間に追加 -->
  {% if messages %}
    <ul class="messages">
      {% for message in messages %}
        <li class="{{ message.tags }}">{{ message }}</li>
      {% endfor %}
    </ul>
  {% endif %}
```

- 一度表示したメッセージは消えます。
- `message.tags` には `success` や `info` が入るので、CSS で色を分けられます。
- 05章のログインビューにも `messages.success()` を入れてあります。

> **保存場所:** 初期設定の `FallbackStorage` は、まずクッキーに保存し、入りきらない分をセッションに保存します。そのため、`logout()` でセッションを消してもメッセージは残りますが、確実にするために `logout()` の**あと**で登録しています。

## 完成した base.html

ここまでで、ベーステンプレートは次のようになります（プロフィール編集のリンクは 09章で足します）。

```html
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
    {% endif %}
  </header>
  {% if messages %}
    <ul class="messages">
      {% for message in messages %}
        <li class="{{ message.tags }}">{{ message }}</li>
      {% endfor %}
    </ul>
  {% endif %}
  <main>
    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

## 動作確認

1. ログインする → 「ログインしました。」と表示される
2. ログアウトボタンを押す → ログイン画面へ移動し、「ログアウトしました。」と表示される
3. ブラウザの URL に直接 `/logout/` と入力して開く（GET）→ **405 エラー**になる
