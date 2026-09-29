# 21. 付録: CBV との対応

基本編（CBV）と発展編（FBV）を並べて振り返ります。

## ビューの対応

| 機能 | 基本編（CBV） | 発展編（FBV） |
| --- | --- | --- |
| ホーム | `HomeView(LoginRequiredMixin, TemplateView)` | `home_view` + `@login_required` |
| ログイン | `auth_views.LoginView` | `login_view` |
| ログアウト | `auth_views.LogoutView` | `logout_view` + `@require_POST` |
| 新規登録 | `SignUpView(CreateView)` + `form_valid()` | `signup_view` |
| プロフィール編集 | （`UpdateView` で書ける） | `profile_edit_view` |

## 処理ごとの書き方

| 処理 | CBV | FBV |
| --- | --- | --- |
| GET/POST の分岐 | 親クラスがやる | `if request.method == "POST":` |
| フォームの生成 | `form_class` を指定 | 自分で `Form()` / `Form(request.POST)` |
| 成功時の処理 | `form_valid()` をオーバーライド | `if form.is_valid():` の中に書く |
| 遷移先 | `success_url` | `return redirect(...)` |
| `next` の安全な処理 | `LoginView` がやる | `url_has_allowed_host_and_scheme()` を自分で呼ぶ |
| ログイン必須 | `LoginRequiredMixin` | `@login_required` |
| ログイン済みの人の除外 | `redirect_authenticated_user=True` | `if request.user.is_authenticated:` |
| メソッド制限 | 定義していないメソッドは自動で 405 | `@require_POST` など |
| 編集対象の指定 | `get_object()` | `instance=request.user` |

## 行数の比較

この教材の `views.py` は、約 90 行です。基本編の `views.py` は 20 行ほどで、残りは Django が引き受けていました。
**FBV が長いのは、CBV が肩代わりしていた処理が見えるようになったためです。**

## どう使い分けるか

FBV が向いている場面:

- 画面ごとに処理が大きく違う
- 1 つの画面で複数のフォームを扱う
- 処理の流れを、上から下へ読めるようにしたい
- CBV のどのメソッドをオーバーライドすべきか調べる時間のほうが長くなりそうなとき

CBV が向いている場面:

- 一覧・詳細・作成・更新といった定型処理
- 同じような画面をいくつも作る（ミックスインで共通化できる）
- `LoginView` のように、**セキュリティ上の細かい処理を任せたい**とき

最後の点は特に大切です。この教材で書いた `next` の確認や、ログアウトの POST 限定は、`LoginView` / `LogoutView` を使っていれば最初から入っていました。
**仕組みを理解したうえで、任せられるものは任せる**のが、いちばん安全で楽な選び方です。

## さらに学ぶなら

- パスワード変更・リセット（`PasswordChangeView`, `PasswordResetView` / FBV なら `PasswordChangeForm`, `PasswordResetForm`）
- メールアドレスでのログイン（`AbstractBaseUser` + 自作 `UserManager`）
- 認証バックエンドの自作（`AUTHENTICATION_BACKENDS`）
- django-allauth によるソーシャルログイン
