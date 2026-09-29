# 14. ログアウトとアクセス制限

## ログアウト

```python
@require_POST
def logout_view(request):
    logout(request)
    messages.info(request, "ログアウトしました。")
    return redirect(settings.LOGOUT_REDIRECT_URL)
```

- `@require_POST`: POST 以外のリクエストには **405 Method Not Allowed** を返します。基本編の `LogoutView` と同じく、GET でのログアウトを防ぎます（Django 5.0 以降の `LogoutView` の動きに合わせています）。
- `logout(request)`: セッションのデータをすべて削除し、`request.user` を `AnonymousUser` にします。
- テンプレートは基本編の `base.html` のまま（`method="post"` と `{% csrf_token %}` を使ったフォーム）で動きます。

> `@require_POST` を付け忘れると、`<img src="https://あなたのサイト/logout/">` のようなタグを他のサイトに置かれただけで、見た人がログアウトさせられてしまいます。

## ログインしている人だけに見せる

```python
@login_required
def home_view(request):
    return render(request, "accounts/home.html")
```

`@login_required` は、CBV の `LoginRequiredMixin` にあたるデコレーターです。未ログインなら `settings.LOGIN_URL` へ、`?next=元のURL` を付けてリダイレクトします。

### デコレーターの順番

デコレーターは**下から順に**関数を包みます。そのため、一番上に書いたものが最初にチェックされます。

```python
@login_required                          # ① 最初にログインを確認
@require_http_methods(["GET", "POST"])   # ② 次にメソッドを確認
def profile_edit_view(request):
    ...
```

`@login_required` を一番上に書いておけば、未ログインの人には、ほかのチェックより先にログイン画面を案内できます。

### よく使うアクセス制限

| デコレーター | 条件 |
| --- | --- |
| `@login_required` | ログインしている |
| `@permission_required("app.change_model")` | その権限を持っている |
| `@user_passes_test(lambda u: u.is_staff)` | 関数が `True` を返す |
| `@staff_member_required`（`django.contrib.admin.views.decorators`） | `is_staff` が `True` |

## メッセージフレームワーク

「ログインしました」のような、**次の画面に 1 回だけ表示するメッセージ**には、`django.contrib.messages` を使います。`startproject` の初期設定で有効になっています。

```python
messages.success(request, "ログインしました。")
messages.info(request, "ログアウトしました。")
messages.error(request, "エラーが発生しました。")
```

テンプレートでは `messages` で取り出します。一度表示したメッセージは消えます。

```html
<!-- templates/base.html に追加 -->
{% if messages %}
  <ul class="messages">
    {% for message in messages %}
      <li class="{{ message.tags }}">{{ message }}</li>
    {% endfor %}
  </ul>
{% endif %}
```

`message.tags` には `success` や `info` が入るので、CSS で色を付けられます。

> **ログアウトとメッセージ:** `logout()` はセッションを削除しますが、メッセージはレスポンスを返すときにクッキーへ保存されるので（初期設定の `FallbackStorage` はクッキーを優先して使います）、ログアウトしたあとの画面にも表示されます。
> ただし、メッセージが大きくてクッキーに入りきらず、セッションに保存される場合は、ログアウトで一緒に消えることがあります。このため、`logout()` のあとで登録するようにしておくと確実です。
