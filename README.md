# node-images

Docker Hub이 막힌 환경의 **71번 서버**에 Node.js 베이스 이미지를 반입하기 위한 저장소.

`docker save | gzip`로 만든 이미지 아카이브를 git으로 배포 → 로컬 PC로 받음 → scp로 71번 서버 전송 → `docker load`로 반입한다.

## 포함된 이미지

| 파일 | 이미지 태그 | 아키텍처 |
| --- | --- | --- |
| `node24.22.1-alpine-amd64.tar.gz` | `node:24.22.1-alpine` | linux/amd64 |
| `node24.22.1-slim-amd64.tar.gz` | `node:24.22.1-bookworm-slim` | linux/amd64 |
| `node22.23.4-alpine-amd64.tar.gz` | `node:22.23.4-alpine` | linux/amd64 |
| `node22.23.4-slim-amd64.tar.gz` | `node:22.23.4-bookworm-slim` | linux/amd64 |

> node 이미지는 GitHub 단일 파일 제한(100MB) 미만이라 분할 없이 통짜로 커밋한다.

### n8n 이미지

| 파일 | 이미지 태그 | 아키텍처 | 내장 Node |
| --- | --- | --- | --- |
| `n8n-2.44.6-amd64.tar.gz.part-aa` ~ `part-af` | `n8nio/n8n:2.44.6` | linux/amd64 | v26.12.1 |

> 압축해도 약 279MB라 50MB 단위로 분할했다. 원본 체크섬은 `n8n-2.44.6-amd64.tar.gz.sha256`.
> n8n 공식 이미지는 Node를 **자체 내장**하므로 위 node 베이스 이미지와 무관하게 그대로 실행된다.

Dockerfile에서는 반입한 태그를 그대로 쓴다:

```dockerfile
FROM node:24.22.1-alpine
# 또는
FROM node:24.22.1-bookworm-slim
# node 22가 필요하면
FROM node:22.23.4-alpine
FROM node:22.23.4-bookworm-slim
```

---

## 반입 절차

### 1. 사내 로컬 PC에서 git pull로 다운로드

```bash
# 최초 1회
git clone https://github.com/Nocharm/node-images.git
cd node-images

# 이미 받아둔 경우 최신화
git pull
```

### 2. scp로 71번 서버에 전송

```bash
# <USER>, <SERVER_71> 은 환경에 맞게 변경
scp node24.22.1-*.tar.gz node22.23.4-*.tar.gz <USER>@<SERVER_71>:/tmp/

# 포트가 22가 아니면: scp -P <PORT> ...
# 점프 호스트를 거쳐야 하면: scp -o ProxyJump=<USER>@<BASTION> ...
```

### 3. 71번 서버에 접속해서 docker load

```bash
ssh <USER>@<SERVER_71>
cd /tmp

# docker load는 gzip 압축을 자동 해제하므로 .tar.gz를 그대로 넣으면 된다
docker load -i node24.22.1-alpine-amd64.tar.gz   # → node:24.22.1-alpine
docker load -i node24.22.1-slim-amd64.tar.gz     # → node:24.22.1-bookworm-slim
docker load -i node22.23.4-alpine-amd64.tar.gz   # → node:22.23.4-alpine
docker load -i node22.23.4-slim-amd64.tar.gz     # → node:22.23.4-bookworm-slim
```

### 4. 반입 확인

```bash
docker images | grep node
# node   24.22.1-alpine          ...
# node   24.22.1-bookworm-slim   ...
# node   22.23.4-alpine          ...
# node   22.23.4-bookworm-slim   ...

docker run --rm node:24.22.1-alpine node -v   # v24.22.1
docker run --rm node:22.23.4-alpine node -v   # v22.23.4
```

확인 후 임시 파일 정리:

```bash
rm /tmp/node24.22.1-*.tar.gz /tmp/node22.23.4-*.tar.gz
```

---

## n8n 이미지 반입 (분할 파일)

```bash
# 1. 로컬 PC → 71번 서버 전송
scp n8n-2.44.6-amd64.tar.gz.part-* n8n-2.44.6-amd64.tar.gz.sha256 <USER>@<SERVER_71>:/tmp/

# 2. 71번 서버에서 재조립 + 무결성 확인
ssh <USER>@<SERVER_71>
cd /tmp
cat n8n-2.44.6-amd64.tar.gz.part-* > n8n-2.44.6-amd64.tar.gz
sha256sum -c n8n-2.44.6-amd64.tar.gz.sha256      # → OK

# 3. docker load
docker load -i n8n-2.44.6-amd64.tar.gz           # → n8nio/n8n:2.44.6
docker run --rm --entrypoint n8n n8nio/n8n:2.44.6 --version   # 2.44.6

# 4. 정리
rm /tmp/n8n-2.44.6-amd64.tar.gz*
```

기존 n8n 컨테이너는 compose 등에서 이미지 태그를 `n8nio/n8n:2.44.6` 으로 바꾼 뒤 재생성한다. **업그레이드 전 n8n 데이터 볼륨(`/home/node/.n8n`)과 DB는 반드시 백업**할 것.

---

## 참고: 이미지 아카이브를 새로 만드는 방법

인터넷이 되는 PC에서 이미지를 다시 뜰 때:

```bash
# amd64 플랫폼으로 pull (ARM Mac 등에서 작업 시 --platform 필수)
docker pull --platform linux/amd64 node:24.22.1-alpine
docker pull --platform linux/amd64 node:24.22.1-bookworm-slim

# save + gzip
docker save node:24.22.1-alpine        | gzip > node24.22.1-alpine-amd64.tar.gz
docker save node:24.22.1-bookworm-slim | gzip > node24.22.1-slim-amd64.tar.gz
```

100MB가 넘는 파일은 GitHub push가 막히므로 그때는 `split -b 50m <파일> <파일>.part-` 로 분할하고, 반입 측에서 `cat <파일>.part-* > <파일>` 로 재조립한다.
