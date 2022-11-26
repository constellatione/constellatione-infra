# 별자리 인스턴스

별자리([constellati.one](https://constellati.one/))는 Fly.io 위에 올라간 Glitch 커스텀 Mastodon 인스턴스입니다. 이 프로젝트에서는 별자리 인스턴스를 설치하고 관리하는 방법을 정리하려고 합니다.

# 목차

1. 준비 사항
2. 설치
   - 앱 추가
   - PostgreSQL 배포
   - Redis 배포
   - 초기 Credentials 설정
   - 초기 데이터베이스 마이그레이션
   - S3 버킷 및 메일건 메일 설정
   - Sidekiq 배포
   - Rails 배포
  <!-- - ElasticSearch 배포 -->
3. 매뉴얼
   1. 사용자를 관리자 혹은 모더레이터로 지정하는 법
   2. 회원가입을 여닫는 법
   3. 데이터베이스 스냅샷 확인 및 수동 백업

# 준비 사항

1. Flyctl
2. Docker

# 설치

## 앱 추가

만약 앱이 없을 경우, Fly.io에 Redis, Sidekiq, Rails 애플리케이션을 각각 추가해줍니다.

```sh
fly apps create --name constellatione-setup -o 'constellati-one'
fly apps create --name constellatione-rails -o 'constellati-one'
fly apps create --name constellatione-streaming -o 'constellati-one'
fly apps create --name constellatione-sidekiq -o 'constellati-one'
fly apps create --name constellatione-redis -o 'constellati-one'
fly apps create --name constellatione-nginx -o 'constellati-one'
```

## PostgreSQL 배포

PostgreSQL은 매니지드 버전을 사용합니다.

```
fly pg create --region nrt --name constellatione-pg -o 'constellati-one'
```

이후 나오는 옵션에 대해서는 "Production - Highly available, 2x shared CPUs, 4GB RAM, 40GB disk"을 선택해주세요. 이는 기본적으로 두 대의 가상 머신을 만들 것이며, 한 대가 죽어도 고가용성 정책을 유지할 수 있을 것입니다.

>  TODO: 가격 보고 스펙을 내릴지 말지 결정할 것.

## Redis 배포

`./redis` 폴더 안에는 Redis 배포를 위한 것이 있습니다. Fly.io의 관리형 Redis는 Mastodon이 쓰는 Redis 기능과 100% 호환되지 않기 때문에 수동으로 배포합니다.

```sh
pushd ./redis
# 우선 메모리 512MB / 스토리지 10GB만 사용해봅니다. 안되면 늘리는걸로.
fly volumes create --region nrt --size 10 constellatione_redis
fly scale memory 512
fly deploy
popd
```

## 초기 Credentials 셋업

처음에만 사용합니다. 나중엔 마이그레이션을 어떻게 할지 고민 중...

```sh
# 꼭 저장해두세요.
SECRET_KEY_BASE=$(docker run --rm -it tootsuite/mastodon:latest bin/rake secret)
OTP_SECRET=$(docker run --rm -it tootsuite/mastodon:latest bin/rake secret)

fly secrets set OTP_SECRET=$OTP_SECRET SECRET_KEY_BASE=$SECRET_KEY_BASE --app constellatione-setup
fly secrets set OTP_SECRET=$OTP_SECRET SECRET_KEY_BASE=$SECRET_KEY_BASE --app constellatione-rails
fly secrets set OTP_SECRET=$OTP_SECRET SECRET_KEY_BASE=$SECRET_KEY_BASE --app constellatione-sidekiq

# 여기서 나오는 VAPID_PUBLIC_KEY, VAPID_PRIVATE_KEY도 저장해주세요.
eval $(docker run --rm -e OTP_SECRET=$OTP_SECRET -e SECRET_KEY_BASE=$SECRET_KEY_BASE -it tootsuite/mastodon:latest bin/rake mastodon:webpush:generate_vapid_key)

fly secrets set VAPID_PRIVATE_KEY=$VAPID_PRIVATE_KEY VAPID_PUBLIC_KEY=$VAPID_PUBLIC_KEY --app constellatione-setup
fly secrets set VAPID_PRIVATE_KEY=$VAPID_PRIVATE_KEY VAPID_PUBLIC_KEY=$VAPID_PUBLIC_KEY --app constellatione-rails
fly secrets set VAPID_PRIVATE_KEY=$VAPID_PRIVATE_KEY VAPID_PUBLIC_KEY=$VAPID_PUBLIC_KEY --app constellatione-sidekiq
```

## S3 버킷 및 메일건 메일 설정

1. AWS 계정에 S3 만들고... `fly secrets set AWS_ACCESS_KEY_ID="..." AWS_SECRET_ACCESS_KEY="..."`를 -rails, -sidekiq에다가 저장합니다. 
2. Mailgun 계정 설정 후... `fly secrets set SMTP_PORT="587" SMTP_SERVER="..." SMTP_LOGIN="..." SMTP_PASSWORD="..."`를 -rails, -sidekiq에다가 저장합니다.

## 초기 데이터베이스 마이그레이션

```sh
pushd setup
fly pg attach --app constellatione-setup constellatione-pg
fly secrets set DB_HOST="..." DB_USER="postgres" DB_PASS="..." --app constellatione-setup
fly secrets unset DATABASE_URL
fly scale memory 512
fly scale count 1
fly deploy --region nrt
fly destroy constellatione-setup
popd
```

## Rails 배포

```sh
pushd rails
fly pg attach --app constellatione-rails constellatione-pg
fly secrets set DB_HOST="..." DB_USER="postgres" DB_PASS="..." --app constellatione-rails
fly secrets unset DATABASE_URL
fly scale vm shared-cpu-1x
fly scale memory 1024
fly scale count 2
fly deploy --region nrt
popd
```

## Sidekiq 배포

```sh
pushd sidekiq
fly pg attach --app constellatione-sidekiq constellatione-pg
fly secrets set DB_HOST="..." DB_USER="postgres" DB_PASS="..." --app constellatione-sidekiq
fly secrets unset DATABASE_URL
fly scale vm shared-cpu-1x
fly scale memory 1024
fly scale count 2
fly deploy --region nrt
popd
```

## Streaming 배포

TBD

## Nginx 배포

TBD

## 도메인 연결하는 법

```sh
pushd nginx
fly ips allocate-v4
fly certs create constellati.one
popd
```

Fly.io 웹 콘솔에서 DNS 설정(A, CNAME)을 추가하고 기다립니다.
