# 02. カスタムユーザーモデル

## AbstractUser と AbstractBaseUser

| 基底クラス | 向いている場面 |
| --- | --- |
| `AbstractUser` | 標準の `User`（username, email, first_name, is_staff など）に**項目を足したい** |
| `AbstractBaseUser` + `PermissionsMixin` | username をなくしてメールでログインするなど、**認証の仕組みから変えたい**。`UserManager` も自作する |

この教材では `AbstractUser` を使います。

## モデル

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class CustomUser(AbstractUser):
    """AbstractUser を継承したカスタムユーザー"""

    email = models.EmailField("メールアドレス", unique=True)
    nickname = models.CharField("ニックネーム", max_length=50, blank=True)
    birth_date = models.DateField("生年月日", null=True, blank=True)

    class Meta:
        verbose_name = "ユーザー"
        verbose_name_plural = "ユーザー"

    def __str__(self):
        return self.nickname or self.username
```

ポイント:

- **`email` の上書き**: `AbstractUser` の `email` は `blank=True` で重複もできます。ここでは `unique=True` にして必須・重複不可にしました。
- **`REQUIRED_FIELDS`**: `AbstractUser` では `["email"]` になっているので、`createsuperuser` でもメールを聞かれます。
- **`objects`**: `AbstractUser` は `UserManager` を持っているので、`create_user()` / `create_superuser()` はそのまま使えます。
- 項目を増やしすぎないこと。プロフィール情報が多い場合は、`OneToOneField` で別の `Profile` モデルに分けるのが一般的です。

## マイグレーション

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

`accounts/migrations/0001_initial.py` ができ、テーブル名は `accounts_customuser` になります。`auth_user` テーブルは作られません。

## ユーザーモデルの参照方法

カスタムユーザーを作ったら、コードの中で `CustomUser` を直接 import するのは避けます。

```python
# 外部キー（models.py）: 設定値の文字列を使う
from django.conf import settings

class Post(models.Model):
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
```

```python
# それ以外（views.py, tests.py など）: get_user_model() を使う
from django.contrib.auth import get_user_model

User = get_user_model()
```

| 場所 | 使うもの | 理由 |
| --- | --- | --- |
| `models.py` の FK / M2M | `settings.AUTH_USER_MODEL` | モデルを読み込む時点ではアプリの準備が終わっていないため |
| その他 | `get_user_model()` | 実際のモデルクラスが必要なため |

どちらもユーザーモデルの差し替えに強く、サードパーティアプリも同じ書き方をしています。
