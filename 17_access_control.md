# 17. アクセス制限

ホーム画面は、まだ誰でも見られる状態です。ログインした人だけに見せるようにします。

## @login_required

```python
# accounts/views.py
from django.contrib.auth.decorators import login_required


@login_required
def home_view(request):
    return render(request, "accounts/home.html")
```

デコレーターを 1 行足すだけです。CBV の `LoginRequiredMixin` にあたります。

未ログインの場合は、`settings.LOGIN_URL`（= `accounts:login`）へ、`?next=元のURL` を付けてリダイレクトします。
この `next` を 05章で安全に処理したので、ログインするとちゃんと元の画面に戻ります。

```
未ログインで /  ──▶ /login/?next=/ ──(ログイン成功)──▶ /
```

## デコレーターの順番

デコレーターは**下から順に**関数を包みます。したがって、**一番上に書いたものが最初に実行されます**。

```python
@login_required                          # ① 最初にログインを確認
@require_http_methods(["GET", "POST"])   # ② 次にメソッドを確認
def profile_edit_view(request):
    ...
```

`@login_required` を上に置いておけば、未ログインの人には、ほかの確認より先にログイン画面を案内できます。

## そのほかのアクセス制限

| デコレーター | 条件 | import 元 |
| --- | --- | --- |
| `@login_required` | ログインしている | `django.contrib.auth.decorators` |
| `@permission_required("accounts.change_customuser")` | その権限を持つ | 同上 |
| `@user_passes_test(lambda u: u.is_staff)` | 関数が `True` を返す | 同上 |
| `@staff_member_required` | `is_staff` が `True` | `django.contrib.admin.views.decorators` |

`@permission_required` と `@user_passes_test` は、条件を満たさないとき既定でログイン画面へ飛ばします。ログイン済みの人には 403 を返したい場合は、`raise_exception=True` を付けます。

## HTTP メソッドの制限

| デコレーター | 許可するメソッド |
| --- | --- |
| `@require_GET` | GET のみ |
| `@require_POST` | POST のみ |
| `@require_http_methods(["GET", "POST"])` | 指定したものだけ |

CBV では、定義したメソッド（`get`, `post`）以外は自動で 405 になります。FBV では、このデコレーターが同じ役割を果たします。

## 動作確認

1. ログアウトした状態で `/` を開く → `/login/?next=/` へ移動する
2. ログインする → ホームへ戻る
