# Django カスタムユーザー＆ログイン/ログアウト チュートリアル

`AbstractUser` を継承したカスタムユーザーを作り、新規登録・ログイン・ログアウトまでを実装します。

- 対象: Django の基本（プロジェクト/アプリ、CBV、テンプレート）を知っている人
- 環境: Python 3.10+ / Django 5.2 LTS
- 完成コード: `sample/`（`python manage.py test` で動作確認済み）

## 目次

| 章 | 内容 |
| --- | --- |
| [01. プロジェクト作成と最初の注意点](01_setup.md) | なぜ最初に作るのか、`AUTH_USER_MODEL` |
| [02. カスタムユーザーモデル](02_custom_user.md) | `AbstractUser` 継承、マイグレーション |
| [03. フォームと管理画面](03_forms_admin.md) | `UserCreationForm` / `UserAdmin` の拡張 |
| [04. 新規登録・ログイン・ログアウト](04_auth_views.md) | `LoginView` / `LogoutView` / `SignUpView` |
| [05. テストの基礎](05_testing_basics.md) | テストの仕組み、`TestCase`、アサーション、テストクライアント |
| [06. テストと動作確認](06_testing.md) | 認証フローのテスト、よくあるエラー |

## 完成時の構成

```
sample/
├── manage.py
├── requirements.txt
├── config/
│   ├── settings.py
│   └── urls.py
├── accounts/
│   ├── admin.py
│   ├── forms.py
│   ├── models.py
│   ├── tests.py
│   ├── test_basics.py
│   ├── urls.py
│   ├── views.py
│   └── migrations/0001_initial.py
└── templates/
    ├── base.html
    └── accounts/
        ├── home.html
        ├── login.html
        └── signup.html
```

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
| `/admin/` | 管理画面 |
