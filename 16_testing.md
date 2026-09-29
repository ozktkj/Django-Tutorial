# 16. テスト

## 基本編のテストはそのまま通る

基本編の `accounts/tests.py` は、**1 行も変えずに**発展編でも通ります。
テストが URL 名とレスポンス（ステータスコード・リダイレクト先・表示内容）だけを確認していて、ビューが CBV か FBV かに依存していないからです。

このように、**中身の書き方ではなく外から見た動きをテストしておく**と、今回のような書き換え（リファクタリング）を安心して行えます。

## FBV で自分で書いた部分のテスト

CBV では Django が引き受けていて、FBV では自分で書いた部分は、テストで確かめておきます。

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

| テスト | 確認していること |
| --- | --- |
| `test_login_redirects_to_next` | 安全な `next` にはリダイレクトする |
| `test_login_ignores_external_next` | 外部サイトの `next` は無視する（オープンリダイレクト対策） |
| `test_next_is_passed_to_template` | `next` がテンプレートの hidden 項目に渡る |
| `test_wrong_password_shows_error` | パスワードが違うとログインできず、エラーが出る |
| `test_authenticated_user_is_redirected_from_login` | ログイン済みならログイン画面を表示しない |
| `test_login_adds_message` | メッセージが登録される |
| `test_session_key_changes_on_login` | ログイン時にセッション ID が変わる |
| `test_logout_shows_message_on_login_page` | ログアウト後のログイン画面にメッセージが出る |
| `ProfileEditViewTests` | ログイン必須、更新できる、重複メールは拒否、初期値は自分の情報 |

### 新しく出てきた書き方

- **`get_messages(res.wsgi_request)`**: リダイレクトのレスポンスから、登録されたメッセージを取り出します。
- **`follow=True`**: リダイレクトを最後までたどり、最終的な画面のレスポンスを返します。`assertRedirects` と `assertContains` を 1 回のリクエストで確かめられます。
- **`self.user.refresh_from_db()`**: DB から最新の値を読み直します。ビューで更新したあとの値を確認するときに必要です。

## 実行

```bash
python manage.py test
```

```
Ran 19 tests in ...s

OK
```

基本編の 7 件と、この章の 12 件です。

## 練習問題

1. `_get_safe_next_url()` で `url_has_allowed_host_and_scheme()` のチェックを外し、`next_url` をそのまま返すように変えてみましょう。どのテストが通らなくなるか確かめたら、元に戻します。
2. `logout_view` から `@require_POST` を外すと、基本編の `tests.py` のどのテストが失敗するでしょうか。
3. `ProfileForm` の `fields` に `"is_staff"` を加えると、何が起きるでしょうか。なぜ `fields` を絞る必要があるのかを考えてみましょう。

<details>
<summary>解答例</summary>

1. `test_login_ignores_external_next` が通らなくなります（結果は `ERROR`）。`https://evil.example.com/` へリダイレクトしてしまい、`assertRedirects` がリダイレクト先を取得しようとして「テストクライアントは外部の URL を取得できない」という `ValueError` を出すからです。
2. `AuthFlowTests.test_login_and_logout` が失敗します。GET でもログアウトできてしまい、405 ではなく 302 が返るからです。
3. 利用者が画面から、あるいはフォームの送信内容を書き換えて、自分を管理者にできてしまいます。**利用者向けのフォームでは、変更してよい項目だけを `fields` に書きます**。

</details>

## まとめ

| 項目 | CBV で書く場合 | FBV で書く場合 |
| --- | --- | --- |
| フォームの GET/POST 分岐 | 親クラスがやる | `if request.method == "POST":` |
| ログイン後のリダイレクト | `LoginView` が `next` を安全に処理する | `url_has_allowed_host_and_scheme()` を自分で呼ぶ |
| ログアウトを POST だけにする | `LogoutView` が 5.0 から POST のみ | `@require_POST` |
| ログイン必須 | `LoginRequiredMixin` | `@login_required` |
| ログイン済みユーザーのリダイレクト | `redirect_authenticated_user=True` | `if request.user.is_authenticated:` |
