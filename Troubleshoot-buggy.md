# TASK : Debugging the buggy app
------------------------------------------

## Typo Errors:
1. In GEMFILE: `boot-snap` → `bootsnap`
2. In `config/initializers/asserts`: `assetss` → `assets`
3. In `config/environments/production.rb`: `confgure` → `configure`

## Procedure:

1. Build a base `dockerfile.base`:

```dockerfile
FROM ruby:3.3.8-alpine

RUN apk add --no-cache \
    build-base \
    libxml2-dev \
    libxslt-dev \
    yaml-dev \
    mariadb-dev \
    git \
    nodejs \
    yarn \
    tzdata \
    bash

WORKDIR /app
```
2. Tag the Dockerfile.base as `ruby-3.3.8v1` and Build the dockerfile.app using it .

3. Build a appfile `dockerfile.app` :

```dockerfile
FROM ruby-3.3.8v1
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local without 'development test' && \
    bundle install --jobs 4 --retry 3
COPY . .
RUN bundle exec rake assets:precompile
EXPOSE 3000
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```
3. Configure `database.yml file`:
```
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
  username: <%= ENV["MYSQL_USER"] %>
  password: <%= ENV["MYSQL_PASSWORD"] %>
  host: <%= ENV["MYSQL_HOST"] %>
  port: <%= ENV.fetch("MYSQL_PORT", 3306) %>

production:
  <<: *default
  database: <%= ENV["MYSQL_DATABASE"] %>
```
4. Creating `Docker-compose.yml` :
```dockerfile
version: '3.9'

services:
  db:
    image: mysql:8.0
    container_name: buggy-db
    restart: always
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  web:
    build:
      context: .
      dockerfile: Dockerfile.app
    container_name: buggy-web
    depends_on:
      - db
    environment:
      RAILS_ENV: ${RAILS_ENV}
      DATABASE_HOST: ${DATABASE_HOST}
      DATABASE_NAME: ${MYSQL_DATABASE}
      DATABASE_USERNAME: ${MYSQL_USER}
      DATABASE_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "3000:3000"
    command: bundle exec puma -C config/puma.rb
    volumes:
      - .:/app

  nginx:
    image: nginx:stable-alpine
    container_name: buggy-nginx
    depends_on:
      - web
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./public:/app/public:ro

volumes:
  db_data:
```
5. Configure the `nginx.conf` to use nginx as a reverse proxy for the web container .
```# nginx.conf
events {}

http {
  include       mime.types;
  default_type  application/octet-stream;
  sendfile        on;
  keepalive_timeout  65;

  upstream rails_app {
    server web:3000;  # 'web' is your Rails container name in docker-compose.yml
  }

  server {
    listen 80;
    server_name localhost;

    root /app/public;

    location / {
      proxy_pass http://rails_app;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }

    location ~ ^/(assets|packs|images|fonts|favicon.ico) {
      root /app/public;
      gzip_static on;
      expires max;
      add_header Cache-Control public;
    }
  }
}
```

