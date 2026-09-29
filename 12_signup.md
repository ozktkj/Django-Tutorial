# 02. 新規登録ビュー

## コード

```python
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

## FBV の基本形: GET と POST の分岐

フォームを扱う FBV は、ほぼ必ず次の形になります。

```python
if request.method == "POST":
    form = SomeForm(request.POST)       # 送信内容を入れる
    if form.is_valid():                 # 検証
        ...                             # 保存など
        return redirect(...)            # 成功 → リダイレクト
else:
    form = SomeForm()                   # GET → 空のフォーム

return render(request, "...", {"form": form})  # GET、または検証失敗
```

ポイントは最後の `render()` です。**GET のときと、POST で検証に失敗したときの両方**がここを通ります。
検証に失敗したときの `form` にはエラー情報が入っているので、テンプレートの `{{ form.as_p }}` がエラーメッセージ付きで表示されます。

### 成功したら必ずリダイレクトする

保存に成功したあとに `render()` で画面を返すと、ブラウザを再読み込みしたときに同じ POST がもう一度送られ、二重登録になります。
成功したら `redirect()` で別の URL に GET させるのが決まりです（**PRG パターン**: Post / Redirect / Get）。

## 各行の解説

| コード | 説明 |
| --- | --- |
| `@require_http_methods(["GET", "POST"])` | GET と POST 以外（PUT など）は 405 を返す |
| `if request.user.is_authenticated:` | ログイン済みの人に登録画面は不要なのでホームへ |
| `CustomUserCreationForm(request.POST)` | 基本編で作ったフォームをそのまま使う |
| `user = form.save()` | ユーザーを作成。パスワードのハッシュ化はフォームの中で行われる |
| `login(request, user)` | 作成したユーザーでログイン状態にする |
| `messages.success(...)` | 次の画面に表示するメッセージを登録する（04章） |

`CreateView` 版では `form_valid()` をオーバーライドして `login()` を足していました。FBV では、処理の流れの中にそのまま書けます。

> **補足:** `form.save()` で作ったユーザーをそのまま `login()` に渡せるのは、どの認証バックエンドを使うか Django が判断できるからです。`AUTHENTICATION_BACKENDS` を複数設定している場合は、`login(request, user, backend="django.contrib.auth.backends.ModelBackend")` のように指定する必要があります。
