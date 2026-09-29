# 15. プロフィール編集ビュー

発展編で新しく追加する画面です。「ログイン中のユーザー自身の情報を編集する」という、会員サイトでよくある処理を FBV で書きます。

## フォーム

```python
# accounts/forms.py に追加（先頭に from django import forms を追加）
class ProfileForm(forms.ModelForm):
    """ログイン中のユーザーが自分で編集できる項目だけを持つフォーム"""

    class Meta:
        model = CustomUser
        fields = ("nickname", "email", "birth_date")
        widgets = {"birth_date": forms.DateInput(attrs={"type": "date"})}
```

- 管理画面用の `CustomUserChangeForm` は使いません。`is_staff` や `username` など、**利用者に変えてほしくない項目**が入る可能性があるからです。
- `fields` には、編集してよい項目だけを書きます。`fields = "__all__"` は使いません。
- `type="date"` にすると、ブラウザのカレンダー入力が使えます。

## ビュー

```python
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

新規登録ビューとの違いは `instance=request.user` だけです。

| | 新規登録 | 編集 |
| --- | --- | --- |
| GET | `Form()` | `Form(instance=request.user)` → 今の値が入る |
| POST | `Form(request.POST)` | `Form(request.POST, instance=request.user)` → そのユーザーを更新 |
| `form.save()` | 新しく作成（INSERT） | 上書き（UPDATE） |

### 編集対象を URL から受け取らない

`/users/<pk>/edit/` のように URL で ID を受け取ると、ID を書き換えるだけで他人の情報を編集できてしまうおそれがあります。
自分の情報を編集する画面では、**`request.user` を対象にする**のが安全です。

### メールアドレスの重複チェック

`CustomUser.email` は `unique=True` なので、`ModelForm` は `is_valid()` の中で重複を確認します。
`instance` に自分自身を渡しているので、**自分のメールアドレスのまま保存する場合は重複扱いになりません**。

## テンプレート

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

`base.html` のヘッダーには、ログイン中だけ表示するリンクを追加しました。

```html
<span>{{ user }} さん</span>
<a href="{% url 'accounts:profile_edit' %}">プロフィール編集</a>
```
