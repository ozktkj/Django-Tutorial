# 05. テストの基礎

次の章で認証まわりのテストを書く前に、Django のテストの基本を押さえておきます。
この章のサンプルコードは `sample/accounts/test_basics.py` にまとめてあります。

## なぜテストを書くのか

画面を開いて確かめるだけでは、次のような問題があります。

- 確認に時間がかかり、コードを変えるたびに同じ操作を繰り返すことになる
- 確認し忘れた画面で問題が起きても気づかない
- Django のバージョンを上げたときに、何が壊れたのかが分からない

テストコードを書いておけば、コマンド 1 つで何十もの確認を数秒で繰り返せます。

## Django のテストの仕組み

Django のテストは、Python 標準の `unittest` をもとにしています。

### テストはどこに書くか

`python manage.py test` は、次の条件に合うものを自動で探して実行します。

| 対象 | 条件 |
| --- | --- |
| ファイル | `test*.py` という名前（`tests.py`, `test_basics.py` など） |
| クラス | `TestCase` などを継承したクラス |
| メソッド | 名前が `test` で始まるもの |

`test` で始まらないメソッドは実行されません。ファイルが増えてきたら、`tests/` パッケージ（`__init__.py` が必要）に分けることもできます。

```
accounts/
├── tests.py          # 06章の認証フローのテスト
└── test_basics.py    # この章のサンプル
```

### テスト用のデータベース

テストを実行すると、Django は**テスト専用のデータベースを新しく作り**、マイグレーションを当ててからテストを実行します。終わったら削除します。
開発用の `db.sqlite3` のデータは使われず、変更もされません。

さらに `TestCase` では、**テストメソッドが 1 つ終わるたびにデータの変更が取り消されます**（トランザクションのロールバック）。
そのため、あるテストで作ったユーザーが別のテストに残ることはなく、テストをどの順番で実行しても結果は変わりません。

### テストクラスの種類

| クラス | DB | 使いどころ |
| --- | --- | --- |
| `SimpleTestCase` | 使えない | URL の逆引き、DB を使わない関数など |
| `TestCase` | 使える（テストごとにロールバック） | **ほとんどの場合はこれ** |
| `TransactionTestCase` | 使える（テーブルを空にする） | トランザクションそのものを確認したいとき |
| `LiveServerTestCase` | 使える | Selenium や Playwright でブラウザを動かすとき |

迷ったら `TestCase` を使います。`SimpleTestCase` のテストで DB を使おうとすると、エラーで教えてくれます。

## テストの書き方

### サンプルコード

```python
# accounts/test_basics.py
"""05章「テストの基礎」のサンプル"""
from django.contrib.auth import get_user_model
from django.test import SimpleTestCase, TestCase
from django.urls import resolve, reverse

from . import views

User = get_user_model()


class UrlTests(SimpleTestCase):
    """DB を使わないテストは SimpleTestCase でよい"""

    def test_reverse_login(self):
        self.assertEqual(reverse("accounts:login"), "/login/")

    def test_signup_url_uses_signup_view(self):
        match = resolve("/signup/")
        self.assertEqual(match.func.view_class, views.SignUpView)


class CustomUserModelTests(TestCase):
    """DB を使うテストは TestCase"""

    @classmethod
    def setUpTestData(cls):
        # クラス内で 1 回だけ実行される（各テストの後にロールバックされる）
        cls.user = User.objects.create_user(
            username="taro", email="taro@example.com", password="S3cure-pass!"
        )

    def test_str_returns_username_when_no_nickname(self):
        # Arrange（準備）は setUpTestData で済んでいる
        # Act（実行）
        result = str(self.user)
        # Assert（検証）
        self.assertEqual(result, "taro")

    def test_str_returns_nickname(self):
        self.user.nickname = "たろう"
        self.assertEqual(str(self.user), "たろう")

    def test_password_is_hashed(self):
        self.assertNotEqual(self.user.password, "S3cure-pass!")
        self.assertTrue(self.user.check_password("S3cure-pass!"))

    def test_email_must_be_unique(self):
        from django.db import IntegrityError

        with self.assertRaises(IntegrityError):
            User.objects.create_user(
                username="taro2", email="taro@example.com", password="x"
            )


class ClientBasicsTests(TestCase):
    """テストクライアントの使い方"""

    def test_login_page_renders(self):
        res = self.client.get(reverse("accounts:login"))
        self.assertEqual(res.status_code, 200)
        self.assertTemplateUsed(res, "accounts/login.html")
        self.assertContains(res, "ログイン")
        self.assertIn("form", res.context)

    def test_signup_with_mismatched_passwords_shows_error(self):
        res = self.client.post(reverse("accounts:signup"), {
            "username": "hanako", "email": "hanako@example.com",
            "password1": "S3cure-pass!", "password2": "different-pass!",
        })
        # 失敗時はリダイレクトせず、同じ画面をエラー付きで返す
        self.assertEqual(res.status_code, 200)
        self.assertTrue(res.context["form"].errors)
        self.assertFalse(User.objects.filter(username="hanako").exists())

    def test_force_login(self):
        user = User.objects.create_user(username="jiro", email="j@example.com", password="x")
        self.client.force_login(user)  # パスワード入力を省略してログイン状態にする
        res = self.client.get(reverse("accounts:home"))
        self.assertEqual(res.status_code, 200)
        self.assertEqual(res.context["user"], user)
```

### テストの基本形: 準備・実行・検証

1 つのテストは、次の 3 段階で書くと読みやすくなります（Arrange / Act / Assert、略して AAA）。

```python
def test_str_returns_username_when_no_nickname(self):
    # Arrange（準備）: テストに必要なデータを用意する
    # Act（実行）: 確認したい処理を 1 つ実行する
    result = str(self.user)
    # Assert（検証）: 結果が期待どおりか確かめる
    self.assertEqual(result, "taro")
```

- **1 つのテストでは 1 つのことを確認する**のが基本です。失敗したときに、何が壊れたのかがすぐ分かります。
- メソッド名は長くなっても構いません。`test_str_returns_nickname` のように、何を確認するテストなのかが名前で分かるようにします。

### データの準備: setUp と setUpTestData

| メソッド | 実行されるタイミング | 向いているもの |
| --- | --- | --- |
| `setUp(self)` | **各テストの前**に毎回 | テストクライアントの設定、テストごとに作り直したいもの |
| `setUpTestData(cls)` | **クラスごとに 1 回** | 変更しないデータ（ユーザーなど）。DB への書き込み回数が減るので速い |

`setUpTestData` では `@classmethod` を付け、`self` ではなく `cls` に保存します。
`setUpTestData` で作ったデータを 1 つのテストの中で書き換えても、ほかのテストには影響しません（Django 3.2 以降、テストごとにコピーが使われます）。

### よく使うアサーション

| メソッド | 確認すること |
| --- | --- |
| `assertEqual(a, b)` | `a == b` |
| `assertNotEqual(a, b)` | `a != b` |
| `assertTrue(x)` / `assertFalse(x)` | `x` が真 / 偽 |
| `assertIn(a, b)` | `a in b` |
| `assertIsNone(x)` | `x is None` |
| `assertRaises(例外)` | `with` ブロックの中でその例外が起きる |

Django が追加しているもの:

| メソッド | 確認すること |
| --- | --- |
| `assertContains(res, text)` | ステータスが 200 で、HTML に `text` が含まれる |
| `assertNotContains(res, text)` | HTML に `text` が含まれない |
| `assertRedirects(res, url)` | `url` へリダイレクトし、その先が 200 を返す |
| `assertTemplateUsed(res, name)` | そのテンプレートで描画された |
| `assertFormError(form, field, errors)` | フォームの項目に、そのエラーが出ている |

`assertTrue(a == b)` とは書かずに `assertEqual(a, b)` を使います。失敗したときに、両方の値がメッセージに表示されるからです。

## テストクライアント

`TestCase` の中では `self.client` が使えます。これは**ブラウザの代わりになるもの**で、実際にサーバーを起動せずにビューへリクエストを送れます。

```python
res = self.client.get("/login/")                        # GET
res = self.client.post("/signup/", {"username": ...})   # POST（フォームの送信）
self.client.force_login(user)                           # パスワードなしでログイン状態にする
self.client.logout()                                    # ログアウト状態にする
```

戻り値のレスポンスからは、次の情報を取り出せます。

| 属性 | 中身 |
| --- | --- |
| `res.status_code` | 200, 302, 404 などのステータスコード |
| `res.content` | レスポンスの本文（バイト列） |
| `res.context["form"]` | テンプレートに渡されたコンテキスト |
| `res.url` | リダイレクト先（302 のとき） |

ポイント:

- **URL は `reverse()` で組み立てる**: `"/login/"` と直接書くと、`urls.py` を変えたときにテストも直す必要があります。
- **CSRF チェックは行われない**: テストクライアントからの POST では、`{% csrf_token %}` がなくても通ります。
- **`force_login()`**: ログインが前提のテストでは、毎回ログインフォームに POST するより速くて簡単です。ログイン処理そのものを確認したいときだけ、`/login/` に POST します。
- **失敗したフォーム送信は 200**: 入力エラーがあるとリダイレクトせず、同じ画面をエラー付きで返します。成功（302）と失敗（200）の両方をテストしておくと安心です。

## テストの実行

```bash
# すべて
python manage.py test

# アプリ単位 / ファイル単位 / クラス単位 / メソッド単位
python manage.py test accounts
python manage.py test accounts.test_basics
python manage.py test accounts.test_basics.CustomUserModelTests
python manage.py test accounts.test_basics.CustomUserModelTests.test_str_returns_nickname
```

よく使うオプション:

| オプション | 効果 |
| --- | --- |
| `-v 2` | テスト名を 1 件ずつ表示する |
| `--failfast` | 最初に失敗したところで止める |
| `-k 名前` | 名前にその文字列を含むテストだけ実行する（例: `-k str`） |
| `--keepdb` | テスト用 DB を消さずに残し、次回の実行を速くする |
| `--parallel auto` | CPU の数に合わせて並列で実行する |

### 実行結果の読み方

成功したとき:

```
Found 16 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
................
----------------------------------------------------------------------
Ran 16 tests in 3.866s

OK
Destroying test database for alias 'default'...
```

`.` が成功したテスト 1 件を表します。失敗は `F`、テストの途中で例外が起きた場合は `E` になります。

失敗したとき（期待値を `"jiro"` に書き換えた例）:

```
...F
======================================================================
FAIL: test_str_returns_username_when_no_nickname (accounts.test_basics.CustomUserModelTests.test_str_returns_username_when_no_nickname)
----------------------------------------------------------------------
Traceback (most recent call last):
  File ".../accounts/test_basics.py", line 37, in test_str_returns_username_when_no_nickname
    self.assertEqual(result, "jiro")
AssertionError: 'taro' != 'jiro'
- taro
+ jiro

----------------------------------------------------------------------
Ran 4 tests in 0.751s

FAILED (failures=1)
```

読むところは 3 つです。

1. `FAIL:` の行: **どのテスト**が失敗したか
2. `File ... line 37`: **どの行**で失敗したか
3. `AssertionError: 'taro' != 'jiro'`: **実際の値**（左）と**期待した値**（右）

| 表示 | 意味 | よくある原因 |
| --- | --- | --- |
| `FAIL` | アサーションが成り立たなかった | 実装のバグ、または期待値の誤り |
| `ERROR` | テストの途中で予期しない例外が起きた | テストコードの誤り、import の漏れ、URL 名の間違い |

## 練習問題

1. `test_basics.py` の期待値をわざと書き換えて、テストを失敗させてみましょう。表示を確認したら元に戻します。
2. `CustomUserModelTests` に、`nickname` を指定せずに作ったユーザーの `nickname` が空文字（`""`）であることを確認するテストを追加してみましょう。
3. `ClientBasicsTests` に、ログインしていない状態で `/signup/` を開くと 200 が返るテストを追加してみましょう。

<details>
<summary>解答例</summary>

```python
    # CustomUserModelTests に追加
    def test_nickname_defaults_to_empty(self):
        self.assertEqual(self.user.nickname, "")
```

```python
    # ClientBasicsTests に追加
    def test_signup_page_renders(self):
        res = self.client.get(reverse("accounts:signup"))
        self.assertEqual(res.status_code, 200)
        self.assertTemplateUsed(res, "accounts/signup.html")
```

</details>

次の章では、ここで学んだことを使って、新規登録・ログイン・ログアウト・管理画面のテストを書きます。
