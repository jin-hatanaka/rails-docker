# このリポジトリについて

HC の Docker 課題で GitHub のプロジェクトをクローンして、Docker Compose を使ってクローンした Rails アプリをコンテナ化するというものです。  
</br>

# 環境構築方法

## コンテナ化したいプロジェクトを用意する

Use this template で Create a new repository を選択し、自分のアカウント配下にリポジトリを作成します。
<img width="1240" alt="スクリーンショット 2024-05-24 5 08 16" src="https://github.com/jin-hatanaka/daily_report/assets/107024256/36efca96-1fac-4772-ae38-ff88a94d7cfb">

## Dockerfile を用意する

Rails 用の Dockerfile を作成します。

```Dockerfile
FROM ruby:3.2.2
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    nodejs \
    postgresql-client \
    yarn

WORKDIR /rails-docker
COPY Gemfile Gemfile.lock /rails-docker
RUN bundle install
```

FROM: ベースとなる ruby のイメージを取ってきます。

```
FROM ruby:3.2.2
```

RUN: 必要なパッケージをインストールします。

```
RUN apt-get update && apt-get install -y \
    packages \
```

WORKDIR: 指定したフォルダに移動します。なければ、フォルダを作成し、そこに移動します。

```
WORKDIR /rails-docker
```

COPY: ファイルをコンテナにコピーします。

```
COPY Gemfile Gemfile.lock /rails-docker
```

RUN: コピーした Gemfile を元に rails に必要なパッケージ(gem)をインストールする。

```
RUN bundle install
```

## docker-compose.yml を用意する

次に docker-compose.yml を作成します。

```docker-compose.yml
# docker-composeのversionを指定
version: '3'

# db-dataというvolumeを作成
volumes:
  db-data:

# ここから下にserviceを定義
services:
  # webというserviceを定義
  web:
    # docker buildと同じ。カレントディレクトリを指定
    build: .
    # サービスコンテナが起動したときに実行するコマンドを指定
    # ">" -> 折り返して表示。 "-c" -> 複数コマンドを実行
    command: >
      bash -c "rails db:create &&
              rails db:migrate &&
              rails s -b 0.0.0.0"
    # docker run -pと同じ。portを指定
    ports:
      - '3000:3000'
    # docker run -vと同じ。マウント先を指定
    volumes:
      - '.:/rails-docker'
    # コンテナ側に環境変数を設定
    environment:
      - 'DATABASE_PASSWORD=postgres'
    # docker run -itと同じ
    tty: true
    stdin_open: true
    # db serviseを作成後、docker runする
    depends_on:
      - db
    # web serviceからdb serviceにアクセスできる
    links:
      - db

  # dbというserviceを定義
  db:
    # imageを取得する
    image: postgres:12
    # マウント先を指定
    volumes:
      - 'db-data:/var/lib/postgresql/data'
    # パスワードを設定
    environment:
      - 'POSTGRES_PASSWORD=postgres'
```

## databese.yml を編集する

rails の config/database.yml を編集します。

```database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  user: postgres
  port: 5432
  password: <%= ENV.fetch("DATABASE_PASSWORD") %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

## コンテナを起動する

コンテナを起動させます。  
-d(detach)でコンテナがバックグラウンドで起動します。

```
docker-compose up -d
```

最後にブラウザから localhost:3000 にアクセスして rails アプリが起動しているか確認しましょう。
