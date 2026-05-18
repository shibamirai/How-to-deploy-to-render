# Django + SQLite のデプロイ

SQLite を使った Django アプリを Render へデプロイする方法を示します。SQLite ではなく PostgreSQL を使った場合の方法は[公式サイト](https://render.com/docs/deploy-django)で紹介してあるので、そちらをご覧ください。

## デプロイに必要なライブラリのインストール

1. django-environ のインストール

    SECRET_KEY や DB のパスワードなど外部に公開したくない設定を .env に記述して Git の対象外としておき、その設定を環境変数として読み込めるようにするためのライブラリ

    ```cmd
    pip install django-environ
    ```

1. Whitenoise のインストール

    本番環境で CSS、JavaScript、画像などの静的ファイルを、Nginx などの複雑な外部サーバを使わずに、Python Web アプリケーションプロセス内で手軽に効率的に配信するためのライブラリ

    ```cmd
    pip install 'whitenoise[brotli]'
    ```

    インストール後、setting.py に以下を追加する

    ```python
    ・・・省略・・・
    MIDDLEWARE = [
        'django.middleware.security.SecurityMiddleware',
        'whitenoise.middleware.WhiteNoiseMiddleware',   # 追加
        ・・・

    ・・・省略・・・

    STATIC_URL = '/static/'

    # 以下を追加
    if not DEBUG:
        STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
        STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
    ```

1. Python 向け Web アプリケーションサーバのインストール

    Render では Django アプリを Gunicorn と Uvicorn を使って運用する

    ```cmd
    pip install gunicorn
    pip install uvicorn
    ```

1. ライブラリのリストの更新

    このプロジェクトで使用しているライブラリのリストを requirements.txt に記録し、デプロイ時にインストールする

    ```cmd
    pip freeze > requirements.txt
    ```

## Render デプロイ用の設定

1. SECRET_KEY など外部に公開すべきでない情報は .env ファイルに設定し、Git の管理対象外とします。SECRET_KEY の他、後で作成する superuser のアカウント情報もここに設定しておきます。

    .env
    ```env
    SECRET_KEY='実際のSECRET_KEYの値を設定'
    SUPERUSER_NAME=スーパーユーザの名前
    SUPERUSER_EMAIL=スーパーユーザのメールアドレス
    SUPERUSER_PASS=スーパーユーザのパスワード
    ```

    .gitignore に .env を追加

    ```
    __pycache__/
    *.pyc
    ・・・省略・・・
    .env
    ```

1. setting.py で以下の設定を行います

    - .env ファイルから環境変数を読み込み、SECRET_KEY には環境変数の値を設定
    - DEBUG を False に変更
    - ALLOWED_HOSTS にあらゆるドメインを対象とする '*' を入力

    setting.py
    ```python
    import os
    import environ      # 追加
    from pathlib import Path

    # Build paths inside the project like this: BASE_DIR / 'subdir'.
    BASE_DIR = Path(__file__).resolve().parent.parent

    # 追加
    env = environ.Env()
    env.read_env(os.path.join(BASE_DIR, ".env"))
    # ここまで

    ・・・省略・・・

    # 変更
    SECRET_KEY = env('SECRET_KEY')
    DEBUG = False
    ALLOWED_HOSTS = ['*']

    ・・・省略・・・

    # 追加
    SUPERUSER_NAME = env("SUPERUSER_NAME")
    SUPERUSER_EMAIL = env("SUPERUSER_EMAIL")
    SUPERUSER_PASS = env("SUPERUSER_PASS")

    ```

## カスタムコマンドの作成

Render の無料プランではシェル機能が使えないため、サーバ上で手動で superuser を作成することができません。そのため superuser 作成用のカスタムコマンドを作成し、後で作成する build.sh で実行できるようにします。

プロジェクトの任意のアプリ（ここでは customauth とする）の中に management/commands ディレクトリを用意し、superuser.py ファイルを作成します

customauth/management/commands/superuser.py
```
import os
from django.core.management.base import BaseCommand
from django.contrib.auth import get_user_model
from django.conf import settings

User = get_user_model()

class Command(BaseCommand):
    def handle(self, *args, **options):
        if not User.objects.filter(email=settings.SUPERUSER_EMAIL).exists():
            User.objects.create_superuser(
                email=settings.SUPERUSER_EMAIL,
                name=settings.SUPERUSER_NAME,
                password=settings.SUPERUSER_PASS
            )
```

## スクリプトファイルの作成

Render 上でデプロイが行われたときに実行するコマンドのスクリプトを build.sh という名前でプロジェクト直下に用意します

build.sh
```python
set -o errexit
pip install -r requirements.txt
python manage.py collectstatic --no-input
python manage.py migrate
python manage.py superuser
```

## render.yaml ファイルの作成

Render 特有のファイルで、事前にどういった条件でデプロイするのか、その設定内容を記入します

render.yaml
```
services:
  - type: web
    plan: free
    name: 任意の名前
    runtime: python
    region: singapore
    buildCommand: "./build.sh"
    startCommand: "python -m gunicorn 設定フォルダの名前.asgi:application -k uvicorn.workers.UvicornWorker"
    envVars:
      - key: SECRET_KEY
        generateValue: true
      - key: WEB_CONCURRENCY
        value: 4

```

※`設定フォルダの名前`とは、setting.py の WSGI_APPLICATION に設定されている `xxx..wsgi.application' の xxx のこと。
`django-admin startproject <プロジェクト名>` でプロジェクトを作成した場合はプロジェクト名と同じになっている。

## GitHub リポジトリへの登録

ここまでの変更を GitHub リポジトリにプッシュします。Render は GitHub リポジトリと連携してデプロイを行うので、まだ GitHub リポジトリを用意していない方はここで用意して、プロジェクトをプッシュしておいてください。

## Renderのアカウント作成

Renderのアカウントがまだない方は、[Renderアカウント作成.md](Renderアカウントの作成.md)を参考にアカウントを作成してください。


## デプロイ（Blueprintを使用）

Render へのデプロイは Blueprint を利用して行います。Render で Blueprint インスタンスを作成すると、render.yaml ファイルの内容に従ってデプロイが行われます。

1. Render にログインし「Blueprints」から「New Blueprint Instance」をクリック
1. GitHub で作成したリポジトリが表示されるはずなので、そのリポジトリ横にある「connect」をクリック
1. 表れた画面の Blueprint Name に任意の名前を入力し、「Deploy Blueprint」をクリック
1. デプロイが始まるが、環境変数を設定していないので失敗するはず
1. 次に Dashboard に戻るとこのプロジェクトが service に追加されているはずなので、それをクリックする
1. 「Environments」で「Edit」をクリックし、.env に記述していた下記の設定を追加する
    - SECRET_KEY （すでに追加されていれば不要）
    - SUPERUSER_NAME
    - SUPERUSER_EMAIL
    - SUPERUSER_PASS
1. 保存するとデプロイが再開し、今度は成功するはず。


    
