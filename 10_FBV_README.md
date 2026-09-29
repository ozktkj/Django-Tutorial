# 【発展】関数ベースビューで作る カスタムユーザー＆ログイン/ログアウト

基本編で作ったものと同じ機能を、**関数ベースビュー（FBV）でゼロから作り直す**教材です。
基本編の復習をしながら、`LoginView` などのクラスベースビュー（CBV）が裏でやっていた処理を自分の手で書いていきます。

- 対象: 基本編（カスタムユーザー＆ログイン/ログアウト チュートリアル）を終えた人
- 環境: Python 3.10+ / Django 5.2 LTS
- 完成コード: `sample/`（`python manage.py test` で 19 件のテストが通ることを確認済み）

基本編のプロジェクトを書き換えるのではなく、**空のディレクトリから作り始めます**。
モデル・フォーム・管理画面（01〜03章）は基本編とまったく同じものを作るので、思い出しながら手を動かしてください。各章の最後に「復習ポイント」として要点をまとめています。

## 目次

| 章 | 内容 | 基本編との関係 |
| --- | --- | --- |
| [11. プロジェクトの作成と設定](11_setup.md) | venv、`startproject`、`AUTH_USER_MODEL` | 復習 |
| [12. カスタムユーザーモデル](12_custom_user.md) | `AbstractUser` の継承、マイグレーション | 復習 |
| [13. フォームと管理画面](13_forms_admin.md) | 登録フォーム、`UserAdmin` の拡張 | 復習 |
| [14. 最初のビュー: ホーム画面](14_home.md) | `render()`、URL、ベーステンプレート | **ここから FBV** |
| [15. ログインビュー](15_login.md) | `AuthenticationForm`、`login()`、`next` の安全な扱い | 新規 |
| [16. ログアウトとメッセージ](16_logout.md) | `@require_POST`、`logout()`、メッセージ | 新規 |
| [17. アクセス制限](17_access_control.md) | `@login_required`、デコレーターの順番 | 新規 |
| [18. 新規登録ビュー](18_signup.md) | GET/POST の分岐、PRG パターン | 新規 |
| [19. プロフィール編集ビュー](19_profile_edit.md) | `instance=` を使った更新 | 発展 |
| [20. テスト](20_testing.md) | 認証まわりのテスト | 復習＋新規 |
| [21. 付録: CBV との対応](21_cbv_compare.md) | 基本編のコードとの比較、使い分け | まとめ |

## 作る画面

| URL | 画面 | 作る章 |
| --- | --- | --- |
| `/` | ホーム（要ログイン） | 04・07 |
| `/login/` | ログイン | 05 |
| `/logout/` | ログアウト（POST のみ） | 06 |
| `/signup/` | 新規登録 | 08 |
| `/profile/` | プロフィール編集（要ログイン） | 09 |
| `/admin/` | 管理画面 | 03 |

各章の終わりで `runserver` を起動して動作を確認できるように、作る順番を組み立てています。

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
│   ├── test_fbv.py
│   ├── urls.py
│   ├── views.py
│   └── migrations/0001_initial.py
└── templates/
    ├── base.html
    └── accounts/
        ├── home.html
        ├── login.html
        ├── profile_edit.html
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
