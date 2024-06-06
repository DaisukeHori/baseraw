# DockerとBaserawのセットアップガイド

このガイドは、EC2インスタンスのUbuntu24.04上でDockerとbaserawをセットアップし、再起動後に自動的に起動するように設定するためのステップバイステップの手順を示しています。

## 前提条件

- EC2インスタンスでUbuntu24.04を開始していてClIを操作できる状態であること
- EC2上でSSH,80,443,8080のTCPポートを0.0.0.0/0（任意の場所）からアクセスできるようにセキュリティグループを設定しておくこと
- ドメインを取得すること
- EC2のパブリック IPv4 DNS　（ec2-xx-xx-xxx-xxx.ap-northeast-1.compute.amazonaws.com#こんな感じ）をサブドメイン（一般的にはactivepieces.xxxx.com）にCNAMEで紐づけておくこと

### 余談：

- t3nano:V2Core0.5GB:$5/月
- t3micro:V2Core1GB:$10/月
- t3small:V2Core2GB:$20/月
- t3midium:V2Core4GB:$40/月


## ステップ1: 古いDocker関連パッケージの削除

以下のコマンドを実行して、古いDocker関連パッケージを削除します。

```sh
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove -y $pkg; done
```

## ステップ2: Dockerリポジトリの設定とインストール

1. 必要なパッケージのインストール

```sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

2. DockerのGPGキーを追加

```sh
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

3. Dockerリポジトリの追加

```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

4. Dockerと関連パッケージのインストール

最新バージョンのインストールコマンドを入れます。
```sh
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

5. Dockerの動作確認

```sh
sudo docker run hello-world
```

###ここまでの参考

https://docs.docker.com/engine/install/ubuntu/


## ステップ3: ユーザーをDockerグループに追加

```sh
sudo usermod -aG docker ${USER}
```

この変更を反映するために、再ログインするか次のコマンドを実行します：

```sh
su - ${USER}　#なんか知らんがこれ動かないし、動かなくても動いた。
#あるいは↓
newgrp docker
```

## ステップ4: Baserowのセットアップ

### 1. 必要なディレクトリとファイルの準備

まず、Baserowのインストール用ディレクトリを作成します。

```sh
mkdir ~/baserow
cd ~/baserow
```

### 2. Docker Composeファイルと設定ファイルのダウンロード

Baserowの公式リポジトリから必要なファイルをダウンロードします。

```sh
curl -o docker-compose.yml https://gitlab.com/baserow/baserow/-/raw/master/docker-compose.yml
curl -o .env https://gitlab.com/baserow/baserow/-/raw/master/.env.example
curl -o Caddyfile https://gitlab.com/baserow/baserow/-/raw/master/Caddyfile
```

### 3. 環境変数ファイル (.env) の編集

`.env`ファイルを開き、必要な環境変数を設定します。ここでは、`SECRET_KEY`、`DATABASE_PASSWORD`、および`REDIS_PASSWORD`を設定します。

`SECRET_KEY`、`DATABASE_PASSWORD`、および`REDIS_PASSWORD`はそれぞれなんでもいいです。ユーザーが決めるものです。
`BASEROW_PUBLIC_URL`と`BASEROW_CADDY_ADDRESSES`は自分で設定したドメインにしてください

```sh
nano .env
```

次の内容を追加または編集します：

```env
#4つは追記
SECRET_KEY=supersecuresecretkey1234567890!@#$%
DATABASE_PASSWORD=securepassword123
REDIS_PASSWORD=anothersecurepassword456
BASEROW_PUBLIC_URL=https://baserow.revol-one.com
#これだけ新規
BASEROW_CADDY_ADDRESSES=https://baserow.revol-one.com
```

### 4. Caddyfileの編集

次に、`Caddyfile`を編集して正しい設定を行います。以下の内容をコピーして貼り付けます。

your-email@example.comは Let's Encryptの通知を受け取るためのメールアドレスです。自分のメアドなんでもOKです。

下記コマンドをメモ帳に貼って書き換えてから貼り付けて使ってください
```sh
cat <<EOL > /home/ubuntu/baserow/Caddyfile
{
    email your-email@example.com  # Let's Encryptの通知を受け取るためのメールアドレス
}

https://baserow.revol-one.com {
    tls {
        on_demand
    }

    handle /api/* {
        reverse_proxy http://backend:8000
    }

    handle /ws/* {
        reverse_proxy http://backend:8000
    }

    handle_path /media/* {
        @downloads {
            query dl=*
        }
        header @downloads Content-disposition "attachment; filename={query.dl}"

        file_server {
            root /baserow/media/
        }
    }

    handle_path /static/* {
        file_server {
            root /baserow/static/
        }
    }

    reverse_proxy http://web-frontend:3000
}
EOL

```
### 5.Docker Composeファイルの編集

次に、docker-compose.ymlファイルを以下の内容に編集します。

```sh
nano /home/ubuntu/baserow/docker-compose.yml
```

services:セクションを見つけ、以下の内容に全部を書き換えます。

起動順書の依存関係の追加とCaddyfileのディレクトリ、BASEROW_CADDY_ADDRESSES:-:を80から443へ書き替えています。

```yaml
services:
  # A caddy reverse proxy sitting in-front of all the services. Responsible for routing
  # requests to either the backend or web-frontend and also serving user uploaded files
  # from the media volume.
  caddy:
    image: caddy:2
    restart: unless-stopped
    environment:
      # Controls what port the Caddy server binds to inside its container.
      BASEROW_CADDY_ADDRESSES: ${BASEROW_CADDY_ADDRESSES:-:443} #80から443へ変更
      PRIVATE_WEB_FRONTEND_URL: ${PRIVATE_WEB_FRONTEND_URL:-http://web-frontend:3000}
      PRIVATE_BACKEND_URL: ${PRIVATE_BACKEND_URL:-http://backend:8000}
      BASEROW_PUBLIC_URL: ${BASEROW_PUBLIC_URL:-}
    ports:
      - "${HOST_PUBLISH_IP:-0.0.0.0}:${WEB_FRONTEND_PORT:-80}:80"
      - "${HOST_PUBLISH_IP:-0.0.0.0}:${WEB_FRONTEND_SSL_PORT:-443}:443"
    volumes:
      - /home/ubuntu/baserow/Caddyfile:/etc/caddy/Caddyfile　 #$PWDから/home/ubuntu/baserow/へ変更
      - media:/baserow/media
      - caddy_config:/config
      - caddy_data:/data
    depends_on: #追記
      - backend #追記
    networks:
      local:
```


### 6. Docker ComposeでBaserowを起動

次に、Docker Composeを使用してBaserowを起動します。

```sh
sudo docker compose up -d
```

### 7. ログの確認

Caddyが正しく起動しているかどうか、ログを確認します。

```sh
sudo docker compose logs caddy
```

### 8. EC2インスタンスのセキュリティグループ設定

EC2インスタンスのセキュリティグループに、以下のポートが開いていることを確認します：

- SSH: 22
- HTTP: 80
- HTTPS: 443

### 9. ドメインの設定

ドメインプロバイダで、`baserow.revol-one.com`がEC2インスタンスのパブリックIPアドレスまたはパブリックDNSにポイントしていることを確認します。

## ステップ5: Dockerの自動起動設定

再起動後にDockerコンテナが自動的に起動するように設定します。

```sh
sudo systemctl enable docker
```

これで、EC2インスタンスが再起動された場合でも、Dockerコンテナは自動的に起動します。

## ステップ6: Baserowへのアクセス

ブラウザを開き、以下のURLにアクセスしてBaserowに接続します：

```url
https://baserow.revol-one.com
```

これで、BaserowのセットアップとSSL化が完了し、安全にアクセスできるようになります。




### おまけ
Docker Composeの停止コマンド

```sh
sudo docker compose down
```

## エラーの時は

以下の手順で問題の原因を特定し、解決します。

### 1. サービスのログを確認

各サービスのログを確認して、具体的なエラーメッセージを取得します。以下のコマンドを順に実行してください。

```sh
sudo docker compose logs backend
sudo docker compose logs web-frontend
sudo docker compose logs db
sudo docker compose logs redis
sudo docker compose logs celery
sudo docker compose logs celery-export-worker
sudo docker compose logs celery-beat-worker
```

### 2. 共通の問題と解決方法

#### データベース接続の問題
- `backend`サービスのログにデータベース接続エラーが表示されている場合、データベースが正しく起動していることを確認してください。

```sh
sudo docker compose logs db
```

データベースコンテナが正しく起動していない場合、以下のようなエラーが表示されることがあります。
- データベースの設定（ユーザー名、パスワード、データベース名）が正しいか確認します。
- `.env`ファイルの設定を再確認します。

#### Redis接続の問題
- `backend`サービスのログにRedis接続エラーが表示されている場合、Redisが正しく起動していることを確認してください。

```sh
sudo docker compose logs redis
```

#### フロントエンドの問題
- `web-frontend`サービスのログにエラーが表示されている場合、以下のコマンドを実行して詳細を確認します。

```sh
sudo docker compose logs web-frontend
```

#### マイグレーションの問題
- データベースのマイグレーションが必要な場合があります。以下のコマンドを実行してマイグレーションを実行します。

```sh
sudo docker compose exec backend python manage.py migrate
```

### 3. 依存関係の確認
- `docker-compose.yml`ファイルで各サービスが正しく依存していることを確認します。

### 4. コンテナの再起動
すべてのサービスが正しく起動していることを確認した後、再度コンテナを再起動します。

```sh
sudo docker compose down
sudo docker compose up -d
```

### 5. 再度アクセスの確認
これで問題が解決し、`https://baserow.revol-one.com`にアクセスできることを確認してください。
