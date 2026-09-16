# 06. テストと動作確認

## テストコード

テストの書き方や実行結果の読み方は、[05. テストの基礎](05_testing_basics.md) を参照してください。

```python
# accounts/tests.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse

User = get_user_model()


class AuthFlowTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username="taro", email="taro@example.com", password="S3cure-pass!"
        )

    def test_user_model_is_custom(self):
        self.assertEqual(User.__name__, "CustomUser")

    def test_home_requires_login(self):
        res = self.client.get(reverse("accounts:home"))
        self.assertRedirects(res, f"{reverse('accounts:login')}?next=/")

    def test_signup_logs_in(self):
        res = self.client.post(reverse("accounts:signup"), {
            "username": "hanako", "email": "hanako@example.com", "nickname": "はな",
            "password1": "S3cure-pass!", "password2": "S3cure-pass!",
        })
        self.assertRedirects(res, reverse("accounts:home"))
        self.assertTrue(User.objects.filter(username="hanako").exists())
        self.assertContains(self.client.get(reverse("accounts:home")), "はな")

    def test_login_and_logout(self):
        res = self.client.post(reverse("accounts:login"),
                               {"username": "taro", "password": "S3cure-pass!"})
        self.assertRedirects(res, reverse("accounts:home"))
        # Django 5.0 以降、ログアウトは POST のみ
        self.assertEqual(self.client.get(reverse("accounts:logout")).status_code, 405)
        res = self.client.post(reverse("accounts:logout"))
        self.assertRedirects(res, reverse("accounts:login"))
        self.assertNotIn("_auth_user_id", self.client.session)


class AdminTests(TestCase):
    def setUp(self):
        self.admin = User.objects.create_superuser(
            username="admin", email="admin@example.com", password="S3cure-pass!"
        )
        self.client.force_login(self.admin)

    def test_admin_add_page(self):
        res = self.client.get(reverse("admin:accounts_customuser_add"))
        self.assertEqual(res.status_code, 200)
        self.assertContains(res, "usable_password")

    def test_admin_add_user(self):
        res = self.client.post(reverse("admin:accounts_customuser_add"), {
            "username": "jiro", "email": "jiro@example.com", "nickname": "じろう",
            "usable_password": "true",
            "password1": "S3cure-pass!", "password2": "S3cure-pass!",
        })
        self.assertEqual(res.status_code, 302)
        self.assertTrue(User.objects.get(username="jiro").check_password("S3cure-pass!"))

    def test_admin_change_page(self):
        res = self.client.get(
            reverse("admin:accounts_customuser_change", args=[self.admin.pk])
        )
        self.assertEqual(res.status_code, 200)
```

`AdminTests` では管理画面の追加・編集画面が実際に開けるかを確認しています。`usable_password` のエラーは `check` では検出できないので、このテストで防ぎます。

```bash
python manage.py test
```

```
Ran 16 tests in ...s  # 05章の test_basics.py の 9 件を含む

OK
```

## ブラウザでの確認

1. `/` を開く → `/login/?next=/` へ移動する
2. `/signup/` で登録する → ホームにニックネームが表示される
3. 「ログアウト」ボタンを押す → ログイン画面へ戻る
4. `/login/` でログインする → ホームへ移動する
5. `/admin/` で登録したユーザーを確認する

## よくあるエラー

| エラー | 原因と対処 |
| --- | --- |
| `InconsistentMigrationHistory: Migration admin.0001_initial is applied before its dependency accounts.0001_initial` | `AUTH_USER_MODEL` を設定する前に `migrate` した。開発中なら `db.sqlite3` を削除して migrate し直す |
| `AUTH_USER_MODEL refers to model 'accounts.CustomUser' that has not been installed` | `INSTALLED_APPS` に `accounts` がない、または書き方の誤り |
| `Manager isn't available; 'auth.User' has been swapped` | どこかで `from django.contrib.auth.models import User` を使っている。`get_user_model()` に置き換える |
| ログアウトで `405 Method Not Allowed` | GET で送っている。`method="post"` のフォームにする |
| ログアウトで `403 CSRF verification failed` | フォームに `{% csrf_token %}` がない |
| 管理画面のユーザー追加で `Unknown field(s) (usable_password) specified for CustomUser` | Django 5.1 以降、`add_form` に `UserCreationForm` 系を指定している。`AdminUserCreationForm` を継承したフォームにする（03章） |
| 登録フォームに追加項目が表示されない | `CustomUserCreationForm.Meta.fields` に項目を追加していない |
| `admin.E012 There are duplicate field(s) in 'fieldsets[...]'` | `email` など、標準の `fieldsets` にある項目を二重に書いている |

## 次のステップ

- メールアドレスでログインする（`AbstractBaseUser` + 自作 `UserManager`、または `USERNAME_FIELD = "email"`）
- パスワード変更・リセット（`PasswordChangeView`, `PasswordResetView`）
- プロフィール編集画面（`UpdateView` + `CustomUserChangeForm`）
- django-allauth を使ったソーシャルログイン
