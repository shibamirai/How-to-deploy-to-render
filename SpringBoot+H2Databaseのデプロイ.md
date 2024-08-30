# SpringBootアプリをRenderへデプロイする

H2Databaseを使ったSpringBootアプリをRenderへデプロイする方法を示します。Webサーバを立てずにSpringBootに含まれている簡易Webサーバで実行する方法です。Render上でDockerを使ってデプロイします。

## Dockerfileを作成する

以下の内容で、Java環境のDockerfileをプロジェクト直下に作成します。下記はJava 17の環境ですが、Java 22などにしたい場合は maven:3-eclipse-temurin-22, eclipse-temurin:22-alpine などに置き換えてください。

```conf
FROM maven:3-eclipse-temurin-17 AS build
COPY . .
RUN mvn clean package -Dmaven.test.skip=true
FROM eclipse-temurin:17-alpine
COPY --from=build /target/[ARTIFACTID]-[VERSION].jar demo.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "demo.jar"]
```

※ [ARTIFACTID]と[VERSION]は、pom.xmlの`<artifactId>`とその下の`<version>`の値に置き換えてください。`SpringBootSample-0.0.1-SNAPSHOT.jar`のようになります。

## GitHubリポジトリへの登録

ここまでの変更をGitHubリポジトリにプッシュします。RenderはGitHubリポジトリと連携してデプロイを行うので、まだGitHubリポジトリを用意していない方はここで用意して、プロジェクトをプッシュしておいてください。

※Eclipseの「プロジェクトの共用」からデフォルト設定でGitリポジトリを作成すると、プロジェクトフォルダの上に余分なフォルダが作られるので、「プロジェクトの親フォルダー内のリポジトリーを使用または作成」にチェックを入れて作成してください。「Creation of repositories in the Eclipse workspace is not recommended」という警告が出ますが無視しても問題ありません。

## Renderのアカウント作成

Renderのアカウントがまだない方は、[Renderアカウント作成.md](Renderアカウントの作成.md)を参考にアカウントを作成してください。

## デプロイ（Web Serviceの作成）

RenderでWeb Serviceを新規に作成してデプロイを行います。[Webアプリのデプロイ(共通).md](Webアプリのデプロイ(共通).md)の手順に従ってデプロイしてください。その際、Web Serviceの設定で下記の設定も行ってください。

### SpringBootデプロイ用のWeb Serviceの設定

- Language：`Docker`を選択します。
