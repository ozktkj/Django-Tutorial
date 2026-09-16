# 01. プロジェクト作成と最初の注意点

## カスタムユーザーは「最初の migrate より前」に作る

Django 公式ドキュメントも、新規プロジェクトではデフォルトの `User` で足りそうでもカスタムユーザーを作るよう勧めています。

理由は、`AUTH_USER_MODEL` を途中で変えるのがとても大変だからです。

- `auth_user` を参照する外部キーや、`admin` / `auth` の初期マイグレーションが先に適用されてしまう
- 途中で変えると、マイグレーション履歴の整合が取れなくなる（`InconsistentMigrationHistory`）
- 開発中なら DB を作り直せば済みますが、本番運用中はデータ移行が必要になる

> **ルール:** `startproject` のあと、**`migrate` を実行する前に** カスタムユーザーを作って `AUTH_USER_MODEL` を設定する。

## プロジェクトとアプリの作成

```bash
mkdir sample && cd sample
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install "Django>=5.2,<6.0"

django-admin startproject config .
python manage.py startapp accounts
mkdir -p templates/accounts
```

ここではまだ `migrate` を実行しません。`runserver` を起動したときの「未適用のマイグレーションがあります」という警告も、今は無視して構いません。

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

| 設定 | 役割 |
| --- | --- |
| `AUTH_USER_MODEL` | `"アプリラベル.モデル名"` の形で指定する。`models.ForeignKey` のような import パスではない |
| `LOGIN_URL` | `@login_required` や `LoginRequiredMixin` で未ログインのときに飛ばす先 |
| `LOGIN_REDIRECT_URL` | ログイン後の遷移先（`next` パラメータがなければここ） |
| `LOGOUT_REDIRECT_URL` | ログアウト後の遷移先 |

`LOGIN_URL` などには URL パターン名（`名前空間:名前`）を書けます。

次章でモデルを作ります。
