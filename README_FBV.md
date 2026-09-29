# 【発展】関数ベースビューで作る カスタムユーザー＆ログイン/ログアウト

基本編「カスタムユーザー＆ログイン/ログアウト チュートリアル」では、`LoginView` や `CreateView` などのクラスベースビュー（CBV）を使いました。
この発展編では、同じ機能を**関数ベースビュー（FBV）**で書き直します。CBV が裏でやってくれていた処理を自分で書くことで、Django の認証の仕組みを深く理解するのが目的です。

- 前提: 基本編を終えていること（カスタムユーザー・フォーム・管理画面・テストの基礎）
- 環境: Python 3.10+ / Django 5.2 LTS
- 完成コード: `sample/`（`python manage.py test` で 19 件のテストが通ることを確認済み）

## 目次

| 章 | 内容 |
| --- | --- |
| [11. CBV と FBV](11_cbv_and_fbv.md) | 基本編との違い、書き換える範囲 |
| [12. 新規登録ビュー](12_signup.md) | GET/POST の分岐、`form.is_valid()`、`login()` |
| [13. ログインビュー](13_login.md) | `AuthenticationForm`、`next` の安全な扱い |
| [14. ログアウトとアクセス制限](14_logout_and_access.md) | `@require_POST`、`@login_required`、メッセージ |
| [15. プロフィール編集ビュー](15_profile_edit.md) | `instance=` を使った更新フォーム |
| [16. テスト](16_testing.md) | FBV で自分で書いた部分のテスト |

## 基本編からの変更点

| ファイル | 変更 |
| --- | --- |
| `accounts/views.py` | **すべて関数ベースに書き直し** |
| `accounts/urls.py` | 関数を登録する形に変更、`profile/` を追加 |
| `accounts/forms.py` | `ProfileForm` を追加 |
| `templates/base.html` | メッセージ表示とプロフィール編集リンクを追加 |
| `templates/accounts/profile_edit.html` | 新規 |
| `accounts/test_fbv.py` | 新規 |
| `models.py` / `admin.py` / `settings.py` / その他のテンプレート | **変更なし** |

基本編の `accounts/tests.py`（7 件）も、そのまま通ります。URL 名と画面の動きを変えていないので、ビューの書き方を変えても同じテストで確かめられます。

## サンプルの動かし方

```bash
cd sample
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

| URL | 画面 |
| --- | --- |
| `/` | ホーム（要ログイン） |
| `/signup/` | 新規登録 |
| `/login/` | ログイン |
| `/profile/` | プロフィール編集（要ログイン） |
| `/admin/` | 管理画面 |
