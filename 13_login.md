# 13. ログインビュー

## コード

```python
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

## AuthenticationForm

`AuthenticationForm` は、ユーザー名とパスワードの確認を行うフォームです。内部で `authenticate()` を呼び、ユーザーが存在してパスワードが合っていて、`is_active` が `True` のときだけ `is_valid()` が `True` になります。

ほかのフォームと引数の渡し方が違うので注意してください。

```python
form = AuthenticationForm(request, data=request.POST)  # 1 つ目は request
```

- 1 つ目の引数は `request` です。`AuthenticationForm(request.POST)` と書くと、POST のデータが `request` として扱われ、正しく動きません。
- 送信内容は `data=` キーワードで渡します。
- 検証に成功したら、`form.get_user()` でログインするユーザーを取り出します。

### authenticate() を直接使う場合

フォームを使わずに書くと、次のようになります。

```python
from django.contrib.auth import authenticate, login

user = authenticate(request, username=username, password=password)
if user is not None:
    login(request, user)
```

ただし、入力値のチェック、エラーメッセージ、`is_active` の確認などを自分で書く必要があります。特別な理由がなければ `AuthenticationForm` を使います。

## login() がしていること

- セッションにユーザーの ID と、使った認証バックエンドを保存する
- **セッション ID を作り直す**: ログイン前のセッション ID を攻撃者が知っていても、ログイン後には使えなくなります（セッション固定攻撃の対策）
- CSRF トークンを作り直す
- `user_logged_in` シグナルを送り、`last_login` を更新する

## next パラメータとオープンリダイレクト

`@login_required` で飛ばされてきたとき、URL は `/login/?next=/profile/` のようになります。ログイン後は `next` の URL へ戻すのが親切です。

しかし、`next` の値をそのまま `redirect()` に渡してはいけません。

```python
# ❌ 危険
return redirect(request.GET.get("next"))
```

次のようなリンクを踏まされると、本物のサイトでログインしたあと、偽サイトへ飛ばされてしまいます（**オープンリダイレクト**）。

```
https://本物のサイト/login/?next=https://偽サイト/
```

利用者は「本物のサイトでログインした」と思っているので、偽サイトで「もう一度パスワードを入力してください」と表示されると信じてしまいます。

### url_has_allowed_host_and_scheme

`url_has_allowed_host_and_scheme()` は、URL が許可したホストのものかどうかを確かめます。`LoginView` も内部で同じ関数を使っています。

| `next` の値 | 結果 |
| --- | --- |
| `/profile/` | ✅ 同じサイト内 |
| `https://evil.example.com/` | ❌ 別のホスト |
| `//evil.example.com/` | ❌ `https:` を省略した外部 URL |
| `javascript:alert(1)` | ❌ 許可していないスキーム |

`require_https=request.is_secure()` を指定すると、HTTPS のページからは HTTPS の URL にしか移動しません。

### テンプレートへ next を渡す

GET で `?next=/profile/` を受け取ったら、フォームの hidden 項目に入れて POST でも送られるようにします。

```html
<input type="hidden" name="next" value="{{ next }}">
```

テンプレートは基本編の `login.html` をそのまま使っています。`LoginView` はコンテキストに `next` を自動で入れてくれていましたが、FBV では `render()` に自分で渡します。

## redirect と resolve_url

```python
return redirect(_get_safe_next_url(request) or resolve_url(settings.LOGIN_REDIRECT_URL))
```

`settings.LOGIN_REDIRECT_URL` には `"accounts:home"` のような URL 名が入っています。`resolve_url()` は URL 名でも URL でも受け取って、実際の URL に変換します。
（`redirect()` も同じように URL 名を受け取れるので、この例では `resolve_url()` を省いても動きますが、「URL に変換してから使う」ことを明示しています。）
