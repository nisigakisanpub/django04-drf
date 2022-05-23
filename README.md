## DRF＋Vue.js＋JWT認証（Django REST framework）

### 構成・バージョン

```
/home/nisigaki/django04-drf/drf-vue2-sample
.
|-config/
| |-__init__.py
| |-settings.py
| |-urls.py
| |-wsgi.py
|-apiv1/
| |-__init__.py
| |-apps.py
| |-serializers.py
| |-urls.py
| |-views.py
|-shop/
| |-__init__.py
| |-admin.py
| |-apps.py
| |-migrations/
| | |-0001_initial.py
| | |-__init__.py
| |-models.py
|-templates/
|-manage.py
|-requirements.txt
.
|-frontend/
| |-babel.config.js
| |-package-lock.json
| |-package.json
| |-public/
| | |-favicon.ico
| | |-index.html
| |-src/
| | |-App.vue
| | |-components
| | | |-GlobalHeader.vue
| | | |-GlobalMessage.vue
| | |-main.js
| | |-router/
| | | |-index.js
| | |-services/
| | | |-api.js
| | |-store/
| | | |-index.js
| | |-views/
| |     |-HomePage.vue
| |     |-LoginPage.vue
| |-vue.config.js

python --version
→Python 3.10.0
pip --version
→pip 22.1 from /home/nisigaki/.pyenv/versions/3.10.0/lib/python3.10/site-packages/pip (python 3.10)

yarn --version
→1.22.18
npm --version
→8.10.0

npm list --depth=0 | grep vue
====
|-@vue/cli-plugin-babel@5.0.4
|-@vue/cli-plugin-eslint@5.0.4
|-@vue/cli-plugin-router@5.0.4
|-@vue/cli-plugin-vuex@5.0.4
|-@vue/cli-service@5.0.4
|-bootstrap-vue@2.22.0
|-eslint-plugin-vue@8.7.1
|-vue-router@3.5.4
|-vue-template-compiler@2.6.14
|-vue@2.6.14
|-vuex@3.6.2
====

pip list | grep -e Django -e django
====
Django                        4.0.1
django-cors-headers           3.12.0
django-extensions             3.1.5
django-templated-mail         1.1.1
django-widget-tweaks          1.4.12
djangorestframework           3.13.1
djangorestframework-simplejwt 4.8.0
social-auth-app-django        4.0.0
```


### 手順（Django側、コマンド）

```
// インストール
pip install Django
pip install djangorestframework
pip install djangorestframework-simplejwt
pip install djoser
pip install django-cors-headers

// 新規作成
django-admin startproject config .
python manage.py startapp shop
python manage.py startapp apiv1

// 動作検証

////バックエンド
python manage.py makemigrations shop
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver

////フロントエンド
npm install
npm run serve
```


### 手順（Django側、コーディング）

```
// setting.py 改修

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # 3rd party apps
    'rest_framework',
    'djoser',
    'corsheaders',
    # My applications
    'apiv1.apps.Apiv1Config',
    'shop.apps.ShopConfig',
     # rails routes -> manage.py show_urls
     "django_extensions",
]

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]

# REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

SIMPLE_JWT = {
    'AUTH_HEADER_TYPES': ('JWT',),
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
}

# CORS
CORS_ALLOW_ALL_ORIGINS = False
CORS_ALLOWED_ORIGINS = [
    'http://localhost:8080',
    'http://127.0.0.1:8080',
]

// shop アプリ作成

shop/models.py
====
import uuid
from django.db import models

class Book(models.Model):
    """本モデル"""

    class Meta:
        db_table = 'book'

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    title = models.CharField(verbose_name='タイトル', max_length=20)
    price = models.IntegerField(verbose_name='価格', null=True, blank=True)
    created_at = models.DateTimeField(verbose_name='登録日時',
                                      auto_now_add=True)

    def __str__(self):
        return self.title
====

shop/admin.py
====
from django.contrib import admin
from .models import Book

class BookModelAdmin(admin.ModelAdmin):
    list_display = ('title', 'price', 'id', 'created_at')
    ordering = ('-created_at',)
    readonly_fields = ('id', 'created_at')

admin.site.register(Book, BookModelAdmin)
====

// apivi アプリ作成

apiv1/serializers.py
====
from rest_framework import serializers
from shop.models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'price']
====

apiv1/views.py
====
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from shop.models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    """BookオブジェクトのCRUDをおこなうAPI"""

    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
====

// ルーティング設定

config/urls.py
====

from django.contrib import admin
from django.urls import path, re_path, include
from django.views.generic import TemplateView, RedirectView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', TemplateView.as_view(template_name='index.html')),
    path('api/v1/auth/', include('djoser.urls')),
    path('api/v1/auth/', include('djoser.urls.jwt')),
    path('api/v1/', include('apiv1.urls')),
    re_path('', RedirectView.as_view(url='/')),
]
====

apiv1/urls.py
====
from django.urls import path, include
from rest_framework import routers
from . import views

router = routers.DefaultRouter()
router.register('books', views.BookViewSet)

app_name = 'apiv1'
urlpatterns = [
    path('', include(router.urls)),
]
====
```


### ルーティング確認

```
python manage.py show_urls
====
/       django.views.generic.base.view
/       django.views.generic.base.view
/admin/ django.contrib.admin.sites.index        admin:index
/admin/<app_label>/     django.contrib.admin.sites.app_index    admin:app_list
/admin/<url>    django.contrib.admin.sites.catch_all_view
/admin/auth/group/      django.contrib.admin.options.changelist_view    admin:auth_group_changelist
/admin/auth/group/<path:object_id>/     django.views.generic.base.view
/admin/auth/group/<path:object_id>/change/      django.contrib.admin.options.change_view        admin:auth_group_change
/admin/auth/group/<path:object_id>/delete/      django.contrib.admin.options.delete_view        admin:auth_group_delete
/admin/auth/group/<path:object_id>/history/     django.contrib.admin.options.history_view       admin:auth_group_history
/admin/auth/group/add/  django.contrib.admin.options.add_view   admin:auth_group_add
/admin/auth/user/       django.contrib.admin.options.changelist_view    admin:auth_user_changelist
/admin/auth/user/<id>/password/ django.contrib.auth.admin.user_change_password  admin:auth_user_password_change
/admin/auth/user/<path:object_id>/      django.views.generic.base.view
/admin/auth/user/<path:object_id>/change/       django.contrib.admin.options.change_view        admin:auth_user_change
/admin/auth/user/<path:object_id>/delete/       django.contrib.admin.options.delete_view        admin:auth_user_delete
/admin/auth/user/<path:object_id>/history/      django.contrib.admin.options.history_view       admin:auth_user_history
/admin/auth/user/add/   django.contrib.auth.admin.add_view      admin:auth_user_add
/admin/autocomplete/    django.contrib.admin.sites.autocomplete_view    admin:autocomplete
/admin/jsi18n/  django.contrib.admin.sites.i18n_javascript      admin:jsi18n
/admin/login/   django.contrib.admin.sites.login        admin:login
/admin/logout/  django.contrib.admin.sites.logout       admin:logout
/admin/password_change/ django.contrib.admin.sites.password_change      admin:password_change
/admin/password_change/done/    django.contrib.admin.sites.password_change_done admin:password_change_done
/admin/r/<int:content_type_id>/<path:object_id>/        django.contrib.contenttypes.views.shortcut      admin:view_on_site
/admin/shop/book/       django.contrib.admin.options.changelist_view    admin:shop_book_changelist
/admin/shop/book/<path:object_id>/      django.views.generic.base.view
/admin/shop/book/<path:object_id>/change/       django.contrib.admin.options.change_view        admin:shop_book_change
/admin/shop/book/<path:object_id>/delete/       django.contrib.admin.options.delete_view        admin:shop_book_delete
/admin/shop/book/<path:object_id>/history/      django.contrib.admin.options.history_view       admin:shop_book_history
/admin/shop/book/add/   django.contrib.admin.options.add_view   admin:shop_book_add
/api/v1/        rest_framework.routers.view     apiv1:api-root
/api/v1/\.<format>/     rest_framework.routers.view     apiv1:api-root
/api/v1/auth/   rest_framework.routers.view     api-root
/api/v1/auth/\.<format>/        rest_framework.routers.view     api-root
/api/v1/auth/jwt/create/        rest_framework_simplejwt.views.view     jwt-create
/api/v1/auth/jwt/refresh/       rest_framework_simplejwt.views.view     jwt-refresh
/api/v1/auth/jwt/verify/        rest_framework_simplejwt.views.view     jwt-verify
/api/v1/auth/users/     djoser.views.UserViewSet        user-list
/api/v1/auth/users/<id>/        djoser.views.UserViewSet        user-detail
/api/v1/auth/users/<id>\.<format>/      djoser.views.UserViewSet        user-detail
/api/v1/auth/users/activation/  djoser.views.UserViewSet        user-activation
/api/v1/auth/users/activation\.<format>/        djoser.views.UserViewSet        user-activation
/api/v1/auth/users/me/  djoser.views.UserViewSet        user-me
/api/v1/auth/users/me\.<format>/        djoser.views.UserViewSet        user-me
/api/v1/auth/users/resend_activation/   djoser.views.UserViewSet        user-resend-activation
/api/v1/auth/users/resend_activation\.<format>/ djoser.views.UserViewSet        user-resend-activation
/api/v1/auth/users/reset_password/      djoser.views.UserViewSet        user-reset-password
/api/v1/auth/users/reset_password\.<format>/    djoser.views.UserViewSet        user-reset-password
/api/v1/auth/users/reset_password_confirm/      djoser.views.UserViewSet        user-reset-password-confirm
/api/v1/auth/users/reset_password_confirm\.<format>/    djoser.views.UserViewSet        user-reset-password-confirm
/api/v1/auth/users/reset_username/      djoser.views.UserViewSet        user-reset-username
/api/v1/auth/users/reset_username\.<format>/    djoser.views.UserViewSet        user-reset-username
/api/v1/auth/users/reset_username_confirm/      djoser.views.UserViewSet        user-reset-username-confirm
/api/v1/auth/users/reset_username_confirm\.<format>/    djoser.views.UserViewSet        user-reset-username-confirm
/api/v1/auth/users/set_password/        djoser.views.UserViewSet        user-set-password
/api/v1/auth/users/set_password\.<format>/      djoser.views.UserViewSet        user-set-password
/api/v1/auth/users/set_username/        djoser.views.UserViewSet        user-set-username
/api/v1/auth/users/set_username\.<format>/      djoser.views.UserViewSet        user-set-username
/api/v1/auth/users\.<format>/   djoser.views.UserViewSet        user-list
/api/v1/books/  apiv1.views.BookViewSet apiv1:book-list
/api/v1/books/<pk>/     apiv1.views.BookViewSet apiv1:book-detail
/api/v1/books/<pk>\.<format>/   apiv1.views.BookViewSet apiv1:book-detail
/api/v1/books\.<format>/        apiv1.views.BookViewSet apiv1:book-list
```

