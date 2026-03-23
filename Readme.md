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

RUN echo "zDw2R5tsTTjQD4AvE/bN89Oy1xkkA+WdPxuOgs5Rk4BdNnrKDwuHh+1PMuJMihwN" > /data/replica.key
RUN echo "UuAKxmDExlg5l5TquxPTaSlvGUMI5tFzz38CL2F3F8AQ3cy4+cpiCDlOlxtjmiR8" >> /data/replica.key
RUN echo "PUrdOmuAlyylVt1u01Vml8RNkQHh7dEgYdIAcJtzjCoBK7DDc8upYefCt19kIC39" >> /data/replica.key
RUN echo "j9fcgF51xJIXm6dTKqmZ4Kg83Yfjy++l8jiZ+0e59qwxoOvysP1AvszVTdg36bmv" >> /data/replica.key
RUN echo "p1FKzpvs5INlNZGKU+tZRqvFpWXkLgR087EOkIv7ltyf4MY+Fftrp/8vWpmzFOKq" >> /data/replica.key
RUN echo "uqB9s9DKvZytRrt5Nv48QKgup4eB6xtt25dh+pN8+G6uPxxnFo6AZ1nfnaOX5Q+B" >> /data/replica.key
RUN echo "TPaRwjTbFJxdpa0lK+ZdYksZccYzX8vOygRJSfqS9hC0WBOLJJGvnM91L1G+tzb6" >> /data/replica.key
RUN echo "GMCu71etmYSSr+BiR7mT+EfnWgQCKGaeEbhg/bbhMKmUSmGmI2z5xRjyqFPmlWQ1" >> /data/replica.key
RUN echo "vSdc2BgMiZW4vQD9S9rjDYFFZv3l20hzHaDG6CMFpKhb4LufWw1nOgVuiqsMJGTy" >> /data/replica.key
RUN echo "g6gh62hrrE1cXSobyZBm7lhbnZMP+fc/Nu5n4L1cSxFw+omhFb3/FYor7exCGWKr" >> /data/replica.key
RUN echo "UMOeH8iorw06vgU7cWbEYDOeTKSxNPrCfdtpFwGKCSPDuzYn7lLVlgYv2bnaHKlS" >> /data/replica.key
RUN echo "SP4G0Dcnm+uCFV146UB6Z2t3Hdqu2Eg5R1/w+aFWqvXone/bsWO6527RxZNfaJic" >> /data/replica.key
RUN echo "EA42TV9h++L4VOFvHwoMqsv0FgkvdajOIgHYKx2PGtKffxIRzldLn/lwduq4AmdY" >> /data/replica.key
RUN echo "OkJ0V844y664zqU7+4mwEdoLxgcB3LmAJ6LbCxWZpvhA+9TUTjuLJPy8SqtT/cq6" >> /data/replica.key
RUN echo "bQO0CYj8anwUvawxNET0srqYU54HZevK9z3usFsdZptiyElUMw3Fj9b370xs1Bgn" >> /data/replica.key
RUN echo "FTsc3TDJPSn89zkDmUe493IojE8McE411ok/s4CRywEjUahf" >> /data/replica.key
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
  --restart=always \
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
  --restart=always \
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
  --restart=always \
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
  --restart=always \
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

### 3.4. Cài đặt Ollama

```bash
docker run -d --gpus all --name ollama \
  --restart=always \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama:0.12.6
docker exec -it ollama ollama pull qwen2.5:1.5b
docker exec -it ollama ollama run qwen2.5:1.5b
```
