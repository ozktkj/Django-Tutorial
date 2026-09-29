# 19. プロフィール編集ビュー

基本編にはなかった画面です。「ログイン中の人が自分の情報を編集する」という、会員サイトでよくある処理を作ります。
新規作成のフォームとの違いは `instance=` だけです。

## フォーム

```python
# accounts/forms.py（追記。先頭に from django import forms を追加）
class ProfileForm(forms.ModelForm):
    """ログイン中のユーザーが自分で編集できる項目だけを持つフォーム"""

    class Meta:
        model = CustomUser
        fields = ("nickname", "email", "birth_date")
        widgets = {"birth_date": forms.DateInput(attrs={"type": "date"})}
```

- 管理画面用の `CustomUserChangeForm` は使いません。`is_staff` や `is_superuser` など、**利用者に変えさせたくない項目**が入っているからです。
- `fields` には編集してよい項目だけを書きます。`fields = "__all__"` は使いません。
- `type="date"` を指定すると、ブラウザのカレンダー入力が使えます。

## ビュー

```python
# accounts/views.py（追記）
@login_required
@require_http_methods(["GET", "POST"])
def profile_edit_view(request):
    if request.method == "POST":
        form = ProfileForm(request.POST, instance=request.user)
        if form.is_valid():
            form.save()
            messages.success(request, "プロフィールを更新しました。")
            return redirect("accounts:home")
    else:
        form = ProfileForm(instance=request.user)

    return render(request, "accounts/profile_edit.html", {"form": form})
```

```python
# accounts/urls.py（追記）
    path("profile/", views.profile_edit_view, name="profile_edit"),
```

```html
{% extends "base.html" %}
{% block title %}プロフィール編集{% endblock %}
{% block content %}
<h1>プロフィール編集</h1>
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">保存する</button>
</form>
{% endblock %}
```

ヘッダーにリンクを足して、`base.html` は完成です。

```html
    {% if user.is_authenticated %}
      <span>{{ user }} さん</span>
      <a href="{% url 'accounts:profile_edit' %}">プロフィール編集</a>
      <form action="{% url 'accounts:logout' %}" method="post" style="display:inline">
```

## instance= の働き

| | 新規作成（08章） | 編集（この章） |
| --- | --- | --- |
| GET | `Form()` | `Form(instance=request.user)` → 今の値が初期表示される |
| POST | `Form(request.POST)` | `Form(request.POST, instance=request.user)` |
| `form.save()` | 新規作成（INSERT） | 上書き（UPDATE） |

### 編集対象を URL から受け取らない

`/users/<pk>/edit/` のように URL で ID を受け取ると、ID を書き換えるだけで他人の情報を編集できてしまうおそれがあります。
自分の情報を編集する画面では、**`request.user` を対象にする**のが安全です。URL に ID が出てこないので、確認の書き忘れも起きません。

### メールアドレスの重複チェック

`email` は `unique=True` なので、`ModelForm` が `is_valid()` の中で重複を確認します。
`instance` に自分自身を渡しているため、**自分のメールアドレスのまま保存しても重複扱いにはなりません**。

## 動作確認

1. ログインして `/profile/` を開く → 今の値が入っている
2. ニックネームを変えて保存する → ホームとヘッダーの表示が変わる
3. 他のユーザーが使っているメールアドレスを入れる → エラーになる
4. ログアウトして `/profile/` を開く → `/login/?next=/profile/` へ移動する
