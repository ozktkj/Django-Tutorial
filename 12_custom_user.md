# 12. カスタムユーザーモデル

基本編 02章の復習です。コードは基本編とまったく同じです。

## モデル

```python
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

| 項目 | ポイント |
| --- | --- |
| `AbstractUser` の継承 | username・パスワード・権限まわりをそのまま引き継ぐ |
| `email` の上書き | 標準では重複可・空欄可。`unique=True` にして必須にした |
| 追加項目 | `nickname`（09章のプロフィール編集で使う）、`birth_date` |
| `__str__` | 画面やテストで表示される名前。ニックネームがあればそちらを使う |

項目を増やしすぎないことも大切です。プロフィール情報が多くなるなら、`OneToOneField` で別モデルに分けます。

## マイグレーション

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

`accounts_customuser` テーブルが作られます。`auth_user` は作られません。

## 動作確認

```bash
python manage.py shell -c "from django.contrib.auth import get_user_model; print(get_user_model())"
# <class 'accounts.models.CustomUser'>
```

## ユーザーモデルの参照方法

| 場所 | 書き方 |
| --- | --- |
| `models.py` の `ForeignKey` / `ManyToManyField` | `settings.AUTH_USER_MODEL` |
| ビュー・フォーム・テストなど | `get_user_model()` |

`from django.contrib.auth.models import User` は使いません。カスタムユーザーに差し替えた時点で、そのモデルは使われないからです。

なお、この教材のビューでは `get_user_model()` すらほとんど登場しません。ログイン中のユーザーは `request.user` で取れるからです。

## 復習ポイント

- `AbstractUser` は「項目を足したいだけ」のとき。認証の仕組みから変えたいときは `AbstractBaseUser`
- `makemigrations` → `migrate` の順
- ユーザーモデルは直接 import せず、`settings.AUTH_USER_MODEL` か `get_user_model()` で参照する
