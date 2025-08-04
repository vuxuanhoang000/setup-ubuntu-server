# Setup Ubuntu Server

## 1. Cài đặt Docker

### 1.1. Thiết lập Docker's apt repository.

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo mkdir /etc/apt/sources.list.d
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

## 1.2. Cài đặt các gói Docker.

- Để cài đặt phiên bản Docker Engine cụ thể, hãy bắt đầu bằng cách liệt kê các phiên bản có sẵn trong kho lưu trữ:

```bash
# List the available versions:
apt-cache madison docker-ce | awk '{ print $3 }'

5:28.3.2-1~ubuntu.24.04~noble
5:28.3.1-1~ubuntu.24.04~noble
...
```

- Chọn phiên bản mong muốn và cài đặt:

```bash
export VERSION_STRING="5:27.5.1-1~ubuntu.22.04~jammy"
sudo apt-get install docker-ce=${VERSION_STRING} docker-ce-cli=${VERSION_STRING} containerd.io docker-buildx-plugin docker-compose-plugin -y
```

- Kiểm tra phiên bản hiện tại của Docker

```bash
docker --version
```

## 1.3. Fix lỗi user hiện tại không có quyền truy cập vào Docker daemon thông qua socket /var/run/docker.sock

- Mô tả lỗi

```bash
docker ps -a

permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.47/containers/json?all=1": dial unix /var/run/docker.sock: connect: permission denied
```

### Cách khắc phục

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker ps -a
```

## 2. Cài đặt các Service bằng Docker.

### 2.1. Mongo

#### 2.1.1. Tạo một image mongo tùy chỉnh

```bash
mkdir mongo && touch mongo/Dockerfile
cat << EOF > mongo/Dockerfile
FROM mongo:7.0.21

RUN openssl rand -base64 756 > /data/replica.key
RUN chmod 400 /data/replica.key
RUN chown 999:999 /data/replica.key

CMD ["mongod", "--replSet", "rs0", "--keyFile", "/data/replica.key"]
EOF

docker build -t mongo -f mongo/Dockerfile .
```

#### 2.1.2. Chạy mongo

```bash
export MONGO_USERNAME=root
export MONGO_PASSWORD=<ROOT_PASSWORD>
docker run -d \
  --name mongo \
  -e MONGO_INITDB_ROOT_USERNAME=$MONGO_USERNAME \
  -e MONGO_INITDB_ROOT_PASSWORD=$MONGO_PASSWORD \
  -v mongo_configdb:/data/configdb \
  -v mongo_data:/data/db \
  --network=host \
  --restart=no \
  mongo
```

- Đợi Mongo vài giây để khởi tạo, sau đó tiếp tục.

```bash
docker exec -it mongo mongosh -u $MONGO_USERNAME -p $MONGO_PASSWORD --eval "rs.initiate()"
```

#### 2.1.3. Script tạo một user chỉ có quyền truy cập vào một cơ sở dữ liệu

```js
var DB_NAME = '<DB-NAME>';
var USERNAME = '<USERNAME>';
var PASSWORD = '<PASSWORD>'

use(DB_NAME);
db.createUser({
  user: USERNAME,
  pwd: PASSWORD,
  roles: [{ role: "dbOwner", db: DB_NAME }],
});

```

#### 2.1.4. Cài đặt MongoDB Database Tools

- Tải Database Tools .deb package tại [MongoDB Download Center](https://www.mongodb.com/try/download/database-tools)
- Ví dụ dưới đây dành cho `Ubuntu 22.04 x86_64`

```bash
curl -fsSL https://fastdl.mongodb.org/tools/db/mongodb-database-tools-ubuntu2204-x86_64-100.12.2.deb -o ~/mongodb-database-tools.deb
sudo apt install ~/mongodb-database-tools.deb
```

### 2.2. Mysql

#### 2.2.1. Cấu hình và chạy mysql

```bash
export MYSQL_ROOT_PASSWORD=<ROOT_PASSWORD>
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD \
  -v mysql_data:/var/lib/mysql \
  --network=host \
  --restart=no \
  mysql:8.4.3
```

#### 2.2.2. Cách tạo một user chỉ có quyền truy cập vào một cơ sở dữ liệu

```sql
CREATE USER '<USERNAME>'@'%' IDENTIFIED BY '<PASSWORD>';
CREATE DATABASE IF NOT EXISTS <DATABASE_NAME>;
GRANT ALL PRIVILEGES ON <DATABASE_NAME>.* TO '<USERNAME>'@'%';
FLUSH PRIVILEGES;
```

### 2.3. Redis

#### 2.3.1. Tạo một image redis tùy chỉnh

```bash
export REDIS_USERNAME=root
export REDIS_PASSWORD=<ROOT_PASSWORD>
mkdir redis && touch redis/Dockerfile
cat << EOF > redis/Dockerfile
FROM redis:7.4-alpine

RUN mkdir -p /etc/redis

# Append cấu hình vào redis.conf
RUN echo "user default off" >> /etc/redis/redis.conf
RUN echo "user $REDIS_USERNAME on >$REDIS_PASSWORD ~* +@all" >> /etc/redis/redis.conf

RUN chmod 400 /etc/redis/redis.conf
RUN chown 999:999 /etc/redis/redis.conf

CMD ["redis-server", "/etc/redis/redis.conf"]
EOF

docker build -t redis -f redis/Dockerfile .
```

#### 2.3.2. Chạy redis

```bash
docker run -d \
  --name redis \
  -v redis_data:/data \
  --network=host \
  --restart=no \
  redis
```

#### 2.4. MinIO (S3)

#### 2.4.1. Cấu hình và chạy minio

```bash
export MINIO_USER=root
export MINIO_PASSWORD=<ROOT_PASSWORD>
docker run -d \
  --name minio \
  -e MINIO_ROOT_USER=$MINIO_USER \
  -e MINIO_ROOT_PASSWORD=$MINIO_PASSWORD \
  -v minio_data:/data \
  --network=host \
  --restart=no \
  minio/minio:RELEASE.2025-04-22T22-12-26Z \
  server data --console-address ":9001"
```

#### 2.4.1. Cách tạo ACCESS KEY chỉ có quyền truy cập vào một BUCKET duy nhất trong MinIO

- Ở sidebar trái → chọn User → Access Keys
- Bấm nút **Create access key +**
- Tích chọn switch **Restrict beyond user policy**
- Dán đoạn JSON sau (thay YOUR-BUCKET bằng tên Bucket thật của bạn)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::<YOUR-BUCKET>",
        "arn:aws:s3:::<YOUR-BUCKET>/*"
      ]
    }
  ]
}
```

### 2.4. Nginx

#### 2.4.1. Tạo thư mục lưu trữ cấu hình nginx

```bash
mkdir -p ./nginx/conf.d ./nginx/ssl ./nginx/www

# Chạy container tạm để copy file default.conf ra host
docker run --rm nginx:1.28.0-alpine cat /etc/nginx/conf.d/default.conf > ./nginx/conf.d/default.conf
```

#### 2.4.2. Chạy nginx

```bash
docker run -d \
  --name nginx \
  -v "$(pwd)/nginx/conf.d":"/etc/nginx/conf.d" \
  -v "$(pwd)/nginx/ssl":"/etc/nginx/ssl" \
  -v "$(pwd)/nginx/www":"/var/www" \
  --network=host \
  --restart=no \
  nginx:1.28.0-alpine
```

## 3. Cài đặt Node

### 3.1. Cài đặt NVM (Node Version Manager)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

nvm install 20 --lts
nvm current
```

### 3.2. Cài đặt Yarn (Trình quản lý gói (package manager) dành cho JavaScript)

```bash
npm i -g yarn
yarn --version
```

### 3.3. Cài đặt PM2 (Process Manager for Node.JS)

```bash
npm i -g pm2
pm2 --version
```
