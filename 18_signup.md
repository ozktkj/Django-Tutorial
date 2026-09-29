# 18. 新規登録ビュー

03章で作った `CustomUserCreationForm` を使って、利用者が自分で登録できる画面を作ります。

## コード

```python
# accounts/views.py（追記）
@require_http_methods(["GET", "POST"])
def signup_view(request):
    if request.user.is_authenticated:
        return redirect("accounts:home")

    if request.method == "POST":
        form = CustomUserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)
            messages.success(request, "登録が完了しました。")
            return redirect("accounts:home")
    else:
        form = CustomUserCreationForm()

    return render(request, "accounts/signup.html", {"form": form})
```

```python
# accounts/urls.py（追記）
    path("signup/", views.signup_view, name="signup"),
```

```html
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

ヘッダーにもリンクを足します。

```html
    {% else %}
      <a href="{% url 'accounts:login' %}">ログイン</a>
      <a href="{% url 'accounts:signup' %}">新規登録</a>
    {% endif %}
```

## 解説

構造は 05章のログインビューと同じです。違いは、フォームが保存を伴うことです。

| コード | 説明 |
| --- | --- |
| `form.save()` | ユーザーを作成する。パスワードのハッシュ化はフォームの中で行われる |
| `login(request, user)` | 作成したユーザーでそのままログイン状態にする |
| `return redirect(...)` | 成功したらリダイレクトする |

基本編の `CreateView` 版では、`form_valid()` をオーバーライドして `login()` を足していました。FBV では処理の流れの中にそのまま書けます。

### 成功したら必ずリダイレクトする（PRG パターン）

保存に成功したあとに `render()` で画面を返すと、ブラウザの再読み込みで同じ POST が再送され、二重登録になります。
**Post → Redirect → Get** の順にするのが決まりです。`redirect()` でリダイレクトを返せば、再読み込みしても送られるのは GET です。

### 認証バックエンドの指定

`form.save()` で作ったユーザーをそのまま `login()` に渡せるのは、使う認証バックエンドを Django が判断できるからです。
`AUTHENTICATION_BACKENDS` を複数設定している場合は、次のように明示します。

```python
login(request, user, backend="django.contrib.auth.backends.ModelBackend")
```

## 動作確認

1. `/signup/` を開いて登録する → そのままログイン状態でホームへ移動する
2. すでに使われているユーザー名で登録してみる → エラーが表示され、登録されない
3. パスワードを `password` のような単純なものにしてみる → パスワード検証（`AUTH_PASSWORD_VALIDATORS`）に引っかかる
