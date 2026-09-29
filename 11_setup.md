# 11. プロジェクトの作成と設定

まっさらな状態から始めます。基本編の 01章の復習です。

## 作成

```bash
mkdir sample && cd sample
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install "Django>=5.2,<6.0"

django-admin startproject config .
python manage.py startapp accounts
mkdir -p templates/accounts
```

```bash
pip freeze > requirements.txt      # または手で Django>=5.2,<6.0 と書く
```

**ここではまだ `migrate` を実行しません。** 理由は次のとおりです。

- `migrate` を実行すると、標準の `auth.User` 用のテーブルが作られる
- そのあとで `AUTH_USER_MODEL` を変えると、マイグレーションの履歴が食い違ってエラーになる（`InconsistentMigrationHistory`）

カスタムユーザーを使うと決めているなら、**最初の `migrate` より前に設定を済ませます**。

## settings.py

```python
# config/settings.py

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "accounts",  # 追加
]

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],  # 変更
        "APP_DIRS": True,
        # ...
    },
]

LANGUAGE_CODE = "ja"
TIME_ZONE = "Asia/Tokyo"

# --- カスタムユーザー / 認証設定 ---
AUTH_USER_MODEL = "accounts.CustomUser"
LOGIN_URL = "accounts:login"
LOGIN_REDIRECT_URL = "accounts:home"
LOGOUT_REDIRECT_URL = "accounts:login"
```

| 設定 | 使う場所（この教材で） |
| --- | --- |
| `AUTH_USER_MODEL` | `"アプリラベル.モデル名"` の形。02章で作るモデルを指す |
| `LOGIN_URL` | `@login_required` が未ログインの人を飛ばす先（07章） |
| `LOGIN_REDIRECT_URL` | ログイン成功後の遷移先（05章） |
| `LOGOUT_REDIRECT_URL` | ログアウト後の遷移先（06章） |

CBV では、これらの設定を `LoginView` などが自動で参照していました。FBV では**自分のコードから明示的に読み出します**。どの設定がいつ使われるのかが、コードを書くうちにはっきりします。

## 復習ポイント

- カスタムユーザーは、最初の `migrate` より前に用意する
- `INSTALLED_APPS` へのアプリ追加と `TEMPLATES` の `DIRS` を忘れない
- `LOGIN_URL` などには URL 名（`名前空間:名前`）を書ける
