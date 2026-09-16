# 03. フォームと管理画面

## なぜフォームを作り直すのか

`django.contrib.auth.forms.UserCreationForm` などの標準フォームは、Meta で `model = User`（標準のユーザー）を指しています。
カスタムユーザーの項目を入力させたいので、Meta を継承して `model` と `fields` を差し替えます。

## フォーム

```python
# accounts/forms.py
from django.contrib.auth.forms import (
    AdminUserCreationForm,
    UserChangeForm,
    UserCreationForm,
)

from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):
    """サイトの新規登録画面用"""

    class Meta(UserCreationForm.Meta):
        model = CustomUser
        fields = ("username", "email", "nickname")


class CustomAdminUserCreationForm(AdminUserCreationForm):
    """管理画面のユーザー追加用（Django 5.1+ の usable_password 項目を含む）"""

    class Meta(AdminUserCreationForm.Meta):
        model = CustomUser
        fields = ("username", "email", "nickname")


class CustomUserChangeForm(UserChangeForm):
    class Meta(UserChangeForm.Meta):
        model = CustomUser
        fields = ("username", "email", "nickname", "birth_date")
```

- `password1` / `password2` はフォームクラス側で定義されているので、`fields` に書く必要はありません。
- `class Meta(UserCreationForm.Meta)` と親の Meta を継承すると、`field_classes`（username 用の `UsernameField`）などが引き継がれます。

### ⚠️ Django 5.1 以降: 管理画面用の追加フォームは別に作る

Django 5.1 で、管理画面のユーザー追加画面に「パスワードによる認証」（`usable_password`）を選ぶ項目が加わりました。

- `UserAdmin.add_fieldsets` には `usable_password` が含まれる
- この項目を持っているのは `AdminUserCreationForm` だけで、`UserCreationForm` にはない

そのため、`add_form` に `UserCreationForm` を継承したフォームを指定すると、ユーザー追加画面を開いたときに次のエラーになります。

```
FieldError: Unknown field(s) (usable_password) specified for CustomUser.
Check fields/fieldsets/exclude attributes of class CustomUserAdmin.
```

`python manage.py check` では見つからず、画面を開いたときに初めてエラーになるので注意してください。
サイトの新規登録画面には `UserCreationForm` 系を、管理画面の `add_form` には **`AdminUserCreationForm` 系**を使い分けます。

## 管理画面

`UserAdmin` を継承すると、パスワードのハッシュ化や「パスワード変更」リンクなど、標準の管理画面の機能をそのまま使えます。

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from .forms import CustomAdminUserCreationForm, CustomUserChangeForm
from .models import CustomUser


@admin.register(CustomUser)
class CustomUserAdmin(UserAdmin):
    add_form = CustomAdminUserCreationForm  # AdminUserCreationForm 系を指定
    form = CustomUserChangeForm
    model = CustomUser
    list_display = ("username", "email", "nickname", "is_staff")

    # 編集画面に追加項目を表示
    fieldsets = UserAdmin.fieldsets + (
        ("追加情報", {"fields": ("nickname", "birth_date")}),
    )
    # 追加画面に追加項目を表示
    add_fieldsets = UserAdmin.add_fieldsets + (
        ("追加情報", {"fields": ("email", "nickname")}),
    )
```

| 属性 | 使われる画面 |
| --- | --- |
| `add_form` / `add_fieldsets` | ユーザー追加画面 |
| `form` / `fieldsets` | ユーザー編集画面 |

`fieldsets` に書いていない項目は画面に表示されません。追加した項目は忘れずにここへ加えます。

> **注意:** `email` は標準の `fieldsets`（「個人情報」セクション）にすでに含まれています。編集画面の `fieldsets` にもう一度書くと、項目が重複しているというエラー（`admin.E012`）になります。

## 動作確認

```bash
python manage.py createsuperuser
python manage.py runserver
```

`http://127.0.0.1:8000/admin/` にログインし、「ユーザー」の追加画面と編集画面にニックネームなどが表示され、実際にユーザーを追加できれば OK です。
