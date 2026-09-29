# 13. フォームと管理画面

基本編 03章の復習です。ビューを書く前に、フォームと管理画面を用意しておきます。

## フォーム

```python
# accounts/forms.py
from django import forms
from django.contrib.auth.forms import (
    AdminUserCreationForm,
    UserChangeForm,
    UserCreationForm,
)

from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):
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

| フォーム | 使う場所 |
| --- | --- |
| `CustomUserCreationForm` | サイトの新規登録画面（08章） |
| `CustomAdminUserCreationForm` | 管理画面のユーザー追加 |
| `CustomUserChangeForm` | 管理画面のユーザー編集 |

09章で、利用者が自分の情報を編集するための `ProfileForm` を追加します。

> **⚠️ `add_form` に使うフォームに注意**
> Django 5.1 以降、管理画面のユーザー追加画面には `usable_password` という項目があります。この項目を持っているのは `AdminUserCreationForm` だけです。
> `UserCreationForm` を継承したフォームを `add_form` に指定すると、追加画面を開いたときに次のエラーになります。
>
> ```
> FieldError: Unknown field(s) (usable_password) specified for CustomUser.
> ```
>
> `manage.py check` では見つからず、画面を開いて初めて分かるので注意してください。

## 管理画面

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from .forms import CustomAdminUserCreationForm, CustomUserChangeForm
from .models import CustomUser


@admin.register(CustomUser)
class CustomUserAdmin(UserAdmin):
    add_form = CustomAdminUserCreationForm
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
| `add_form` / `add_fieldsets` | ユーザー追加 |
| `form` / `fieldsets` | ユーザー編集 |

`fieldsets` に書いていない項目は表示されません。逆に、標準の `fieldsets` にすでにある項目（`email` など）を編集画面の `fieldsets` に書き足すと、重複エラー（`admin.E012`）になります。

## 動作確認

```bash
python manage.py createsuperuser
python manage.py runserver
```

`http://127.0.0.1:8000/admin/` を開き、ユーザーの追加と編集ができることを確認します。ここで作ったユーザーは、次章からのログイン確認にそのまま使えます。

この時点で `/` を開くと 404 になります。ここから先が、この教材の本題です。

## 復習ポイント

- 標準のフォームは Meta を継承して `model` と `fields` を差し替える
- 管理画面の `add_form` には `AdminUserCreationForm` 系を使う
- `UserAdmin` を継承すると、パスワードのハッシュ化などをそのまま使える
