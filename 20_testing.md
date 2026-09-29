# 20. テスト

画面がひととおりできたので、テストを書きます。基本編 05・06章の復習と、FBV で自分で書いた部分の確認です。

## 認証フローのテスト

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

このテストは、**基本編（CBV 版）のものとまったく同じ**です。1 行も変えずにそのまま通ります。
URL 名とレスポンス（ステータスコード・リダイレクト先・表示内容）だけを確認していて、ビューが CBV か FBV かに依存していないからです。

**外から見た動きをテストしておくと、中身の書き方を変えても同じテストで確認できます。** 今回のような作り直しを安心して行えるのは、このためです。

## FBV で自分で書いた部分のテスト

CBV では Django が引き受けていた処理は、自分で書いた以上、自分でテストします。

```python
# accounts/test_fbv.py
"""関数ベースビューで自前実装した部分のテスト"""
from django.contrib.auth import get_user_model
from django.contrib.messages import get_messages
from django.test import TestCase
from django.urls import reverse

User = get_user_model()
PASSWORD = "S3cure-pass!"


class LoginViewTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(
            username="taro", email="taro@example.com", password=PASSWORD
        )

    def login(self, next_url=None):
        data = {"username": "taro", "password": PASSWORD}
        if next_url is not None:
            data["next"] = next_url
        return self.client.post(reverse("accounts:login"), data)

    def test_login_redirects_to_next(self):
        res = self.login(next_url=reverse("accounts:profile_edit"))
        self.assertRedirects(res, reverse("accounts:profile_edit"))

    def test_login_ignores_external_next(self):
        # オープンリダイレクト対策: 外部サイトには飛ばさない
        res = self.login(next_url="https://evil.example.com/")
        self.assertRedirects(res, reverse("accounts:home"))

    def test_next_is_passed_to_template(self):
        res = self.client.get(reverse("accounts:login") + "?next=/profile/")
        self.assertEqual(res.context["next"], "/profile/")

    def test_wrong_password_shows_error(self):
        res = self.client.post(
            reverse("accounts:login"), {"username": "taro", "password": "wrong"}
        )
        self.assertEqual(res.status_code, 200)
        self.assertTrue(res.context["form"].non_field_errors())
        self.assertNotIn("_auth_user_id", self.client.session)

    def test_authenticated_user_is_redirected_from_login(self):
        self.client.force_login(self.user)
        res = self.client.get(reverse("accounts:login"))
        self.assertRedirects(res, reverse("accounts:home"))

    def test_login_adds_message(self):
        res = self.login()
        messages = [m.message for m in get_messages(res.wsgi_request)]
        self.assertIn("ログインしました。", messages)

    def test_session_key_changes_on_login(self):
        # login() はセッション固定攻撃を防ぐため、セッション ID を作り直す
        self.client.get(reverse("accounts:login"))
        session = self.client.session
        session["dummy"] = 1
        session.save()
        before = session.session_key
        self.login()
        self.assertNotEqual(before, self.client.session.session_key)


class LogoutViewTests(TestCase):
    def test_logout_shows_message_on_login_page(self):
        user = User.objects.create_user(username="taro", email="t@example.com", password="x")
        self.client.force_login(user)
        res = self.client.post(reverse("accounts:logout"), follow=True)
        self.assertRedirects(res, reverse("accounts:login"))
        self.assertContains(res, "ログアウトしました。")


class ProfileEditViewTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(
            username="taro", email="taro@example.com", password=PASSWORD
        )
        User.objects.create_user(username="jiro", email="jiro@example.com", password=PASSWORD)

    def test_requires_login(self):
        res = self.client.get(reverse("accounts:profile_edit"))
        self.assertRedirects(res, f"{reverse('accounts:login')}?next=/profile/")

    def test_update_profile(self):
        self.client.force_login(self.user)
        res = self.client.post(reverse("accounts:profile_edit"), {
            "nickname": "たろう", "email": "new@example.com", "birth_date": "2000-01-02",
        })
        self.assertRedirects(res, reverse("accounts:home"))
        self.user.refresh_from_db()
        self.assertEqual(self.user.nickname, "たろう")
        self.assertEqual(self.user.email, "new@example.com")

    def test_duplicate_email_is_rejected(self):
        self.client.force_login(self.user)
        res = self.client.post(reverse("accounts:profile_edit"), {
            "nickname": "", "email": "jiro@example.com", "birth_date": "",
        })
        self.assertEqual(res.status_code, 200)
        self.assertIn("email", res.context["form"].errors)

    def test_initial_values_are_current_user(self):
        self.client.force_login(self.user)
        res = self.client.get(reverse("accounts:profile_edit"))
        self.assertEqual(res.context["form"].instance, self.user)
        self.assertContains(res, "taro@example.com")
```

| テスト | 確認していること | 対応する章 |
| --- | --- | --- |
| `test_login_redirects_to_next` | 安全な `next` にはリダイレクトする | 05 |
| `test_login_ignores_external_next` | 外部サイトの `next` は無視する | 05 |
| `test_next_is_passed_to_template` | `next` がテンプレートに渡る | 05 |
| `test_wrong_password_shows_error` | 間違ったパスワードではログインできない | 05 |
| `test_authenticated_user_is_redirected_from_login` | ログイン済みならログイン画面を出さない | 05 |
| `test_login_adds_message` | メッセージが登録される | 06 |
| `test_session_key_changes_on_login` | ログイン時にセッション ID が変わる | 05 |
| `test_logout_shows_message_on_login_page` | ログアウト後の画面にメッセージが出る | 06 |
| `ProfileEditViewTests` | ログイン必須・更新・重複メール・初期値 | 09 |

### この章で出てくる書き方

| 書き方 | 説明 |
| --- | --- |
| `get_messages(res.wsgi_request)` | リダイレクトのレスポンスから、登録されたメッセージを取り出す |
| `follow=True` | リダイレクトを最後までたどり、最終的な画面を返す |
| `refresh_from_db()` | DB から最新の値を読み直す |
| `force_login(user)` | ログインフォームを通さずにログイン状態にする |

## 実行

```bash
python manage.py test
```

```
Ran 19 tests in ...s

OK
```

`tests.py` の 7 件と `test_fbv.py` の 12 件です。

## 練習問題

1. `_get_safe_next_url()` の `url_has_allowed_host_and_scheme()` によるチェックを外し、`next_url` をそのまま返すようにするとどうなるでしょうか。テストを実行して確かめたら、元に戻します。
2. `logout_view` から `@require_POST` を外すと、どのテストが通らなくなるでしょうか。
3. `ProfileForm` の `fields` に `"is_staff"` を加えると、何が起きるでしょうか。

<details>
<summary>解答例</summary>

1. `test_login_ignores_external_next` が通らなくなります（結果は `ERROR`）。`https://evil.example.com/` へリダイレクトしてしまい、`assertRedirects` がそのリダイレクト先を取得しようとして「テストクライアントは外部の URL を取得できない」という `ValueError` を出すからです。
2. `AuthFlowTests.test_login_and_logout` が通らなくなります。GET でもログアウトできてしまい、405 ではなく 302 が返るからです。
3. 利用者が画面から、あるいは送信内容を書き換えて、自分を管理者にできてしまいます。**利用者向けのフォームでは、変更してよい項目だけを `fields` に書きます。**

</details>
