# LaravelアプリをRenderへデプロイする

SQLiteおよびViteを使ったLaravelアプリをRenderへデプロイする方法を示します。SQLiteではなくPostgreSQLを使った場合の方法は[公式サイト](https://docs.render.com/deploy-php-laravel-docker)で紹介してあるので、そちらをご覧ください。

LaravelアプリはRender上でDockerを使ってデプロイします。

## デプロイのためのアプリの修正

1. ブラウザでの混合コンテンツ警告を回避するために、全てのアセットをHTTPSで提供するようにします。app/Providers/AppServiceProvider.phpを下記のように修正してください。

    ```php
    namespace App\Providers;

    use Illuminate\Routing\UrlGenerator;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        // ...

        public function boot(UrlGenerator $url)
        {
            if (env('APP_ENV') == 'production') {
                $url->forceScheme('https');
            }
        }
    }
    ```

1. Dockerの設定を行います。Laravelプロジェクトの直下に (1)Dockerfile (2).dockerignore (3)conf/nginx/nginx-site.confの３つのファイルを作成します。

    1. Dockerfile

        nginx-php-fpmをベースにしたDockerfileを作成します。インストールされるPHPなどのバージョンは[こちら](https://github.com/richarvey/nginx-php-fpm)でご確認ください。Viteの実行にはnpmが必要なので、ここでnpmもインストールしています。

        ```conf
        FROM richarvey/nginx-php-fpm:3.1.6

        # npmのインストール
        RUN apk add --no-cache npm

        COPY . .

        # Image config
        ENV SKIP_COMPOSER 1
        ENV WEBROOT /var/www/html/public
        ENV PHP_ERRORS_STDERR 1
        ENV RUN_SCRIPTS 1
        ENV REAL_IP_HEADER 1

        # Laravel config
        ENV APP_ENV production
        ENV APP_DEBUG false
        ENV LOG_CHANNEL stderr

        # Allow composer to run as root
        ENV COMPOSER_ALLOW_SUPERUSER 1

        CMD ["/start.sh"]
        ```

    1. .dockerignore

        Dockerイメージに含めないファイルを指定します。

        ```conf
        /node_modules
        /public/hot
        /public/storage
        /storage/*.key
        /vendor
        .env
        .phpunit.result.cache
        Homestead.json
        Homestead.yaml
        npm-debug.log
        yarn-error.log
        ```

    1. nginx-site.conf

        NGINXの設定を記述します。conf/nginx/ディレクトリを作成し、そこに配置します。

        ```conf
        server {
        # Render provisions and terminates SSL
        listen 80;

        # Make site accessible from http://localhost/
        server_name _;

        root /var/www/html/public;
        index index.html index.htm index.php;

        # Disable sendfile as per https://docs.vagrantup.com/v2/synced-folders/virtualbox.html
        sendfile off;

        # Add stdout logging
        error_log /dev/stdout info;
        access_log /dev/stdout;

        # block access to sensitive information about git
        location /.git {
            deny all;
            return 403;
        }

        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";

        charset utf-8;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location = /favicon.ico { access_log off; log_not_found off; }
        location = /robots.txt  { access_log off; log_not_found off; }

        error_page 404 /index.php;

        location ~* \.(jpg|jpeg|gif|png|css|js|ico|webp|tiff|ttf|svg)$ {
            expires 5d;
        }

        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/var/run/php-fpm.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param SCRIPT_NAME $fastcgi_script_name;
            include fastcgi_params;
        }

        # deny access to . files
        location ~ /\. {
            log_not_found off;
            deny all;
        }

        location ~ /\.(?!well-known).* {
            deny all;
        }
        }
        ```

1. アプリの起動時に実行されるデプロイスクリプトを作成します。scriptsディレクトリを作成し、その中に00-laravel-deploy.shという名前で作成します。アプリの起動に必要なcomposerやArtisanのコマンドに加え、Viteのインストールと実行を行い、SQLiteのデータベースファイルがアプリから読み書きできるようにパーミッションを変更します。

    ```sh
    #!/usr/bin/env bash
    echo "Running composer"
    composer global require hirak/prestissimo
    composer install --no-dev --working-dir=/var/www/html

    echo "Caching config..."
    php artisan config:cache

    echo "Caching routes..."
    php artisan route:cache

    echo "Running migrations..."
    php artisan migrate --force

    echo "Viteのインストールと実行"
    npm install
    npm run build

    echo "データベースファイルのパーミッションを読み書き可に変更"
    chmod 777 database
    chmod 777 database/database.sqlite
    ```

## GitHubリポジトリへの登録

ここまでの変更をGitHubリポジトリにプッシュします。RenderはGitHubリポジトリと連携してデプロイを行うので、まだGitHubリポジトリを用意していない方はここで用意して、プロジェクトをプッシュしておいてください。

## Renderのアカウント作成

Renderのアカウントがまだない方は、[Renderアカウント作成.md](Renderアカウントの作成.md)を参考にアカウントを作成してください。

## デプロイ（Web Serviceの作成）

RenderでWeb Serviceを新規に作成してデプロイを行います。[Webアプリのデプロイ(共通).md](Webアプリのデプロイ(共通).md)の手順に従ってデプロイしてください。その際、Web Serviceの設定で下記の設定も行ってください。

### Laravelデプロイ用のWeb Serviceの設定

- Language：`Docker`を選択します。
- Environment Variables：下記の環境変数を追加します。

    | NAME_OF_VARIABLE | value |
    |---|---|
    |APP_KEY|`php artisan key:generate --show`の出力をコピーして貼り付けてください。'base64:'から始まるランダムな文字列です。|
    |DB_CONNECTION|sqlite|
    |DB_HOST|127.0.0.1|
