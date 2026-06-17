# node-images

Docker Hub이 막힌 환경의 **71번 서버**에 Node.js 베이스 이미지를 반입하기 위한 저장소.

`docker save | gzip`로 만든 이미지 아카이브를 git으로 배포 → 로컬 PC로 받음 → scp로 71번 서버 전송 → `docker load`로 반입한다.

## 포함된 이미지

| 파일 | 이미지 태그 | 아키텍처 |
| --- | --- | --- |
| `node22-alpine-amd64.tar.gz` | `node:22-alpine` | linux/amd64 |
| `node22-slim-amd64.tar.gz` | `node:22-bookworm-slim` | linux/amd64 |

> 통짜 파일과 20MB 단위 분할 파일(`*.part-*`)을 **함께** 커밋한다. 분할 파일은 100MB 제한이 빡빡하거나 통짜 전송이 실패할 때를 대비한 백업이며, `cat <파일>.part-* > <파일>` 로 재조립한다.

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
scp node22-alpine-amd64.tar.gz node22-slim-amd64.tar.gz <USER>@<SERVER_71>:/tmp/

# 포트가 22가 아니면: scp -P <PORT> ...
# 점프 호스트를 거쳐야 하면: scp -o ProxyJump=<USER>@<BASTION> ...
```

> 통짜 파일 전송이 실패하거나 경유 구간에서 100MB 초과가 막히면 아래 **[분할 파일로 반입하는 경우](#분할-파일로-반입하는-경우)** 절차를 사용한다.

### 3. 71번 서버에 접속해서 docker load

```bash
ssh <USER>@<SERVER_71>
cd /tmp

# docker load는 gzip 압축을 자동 해제하므로 .tar.gz를 그대로 넣으면 된다
docker load -i node22-alpine-amd64.tar.gz
docker load -i node22-slim-amd64.tar.gz
```

### 4. 반입 확인

```bash
docker images | grep node
# node   22-alpine          ...
# node   22-bookworm-slim   ...
```

확인 후 임시 파일 정리:

```bash
rm /tmp/node22-*.tar.gz
```

---

## 분할 파일로 반입하는 경우

통짜 `.tar.gz` 전송이 실패하거나 경유 구간에서 100MB 초과 파일이 막히면, 20MB 단위 분할 파일(`*.part-*`)을 사용한다.
흐름은 **분할 파일 전송 → 71번 서버에서 재조립(cat) → docker load** 순이다.

### 1. 분할 파일 전송

```bash
# alpine 분할: part-aa, ab, ac
scp node22-alpine-amd64.tar.gz.part-* <USER>@<SERVER_71>:/tmp/
# slim 분할: part-aa, ab, ac, ad
scp node22-slim-amd64.tar.gz.part-*  <USER>@<SERVER_71>:/tmp/
```

### 2. 71번 서버에서 재조립 (건집)

```bash
ssh <USER>@<SERVER_71>
cd /tmp

# part-* 를 순서대로 이어붙여 원본 .tar.gz 복원
cat node22-alpine-amd64.tar.gz.part-* > node22-alpine-amd64.tar.gz
cat node22-slim-amd64.tar.gz.part-*  > node22-slim-amd64.tar.gz
```

> `cat ...part-*` 는 셸이 part-aa, part-ab ... 알파벳순으로 정렬해 합치므로 순서를 따로 신경 쓸 필요 없다.

재조립이 깨지지 않았는지 확인 (gzip 무결성 검사):

```bash
gzip -t node22-alpine-amd64.tar.gz && echo "alpine OK"
gzip -t node22-slim-amd64.tar.gz  && echo "slim OK"
```

### 3. docker load

```bash
docker load -i node22-alpine-amd64.tar.gz   # → node:22-alpine
docker load -i node22-slim-amd64.tar.gz     # → node:22-bookworm-slim
docker images | grep node
```

### 4. 정리

```bash
rm /tmp/node22-*.tar.gz /tmp/node22-*.tar.gz.part-*
```

---

## 참고: 이미지 아카이브를 새로 만드는 방법

인터넷이 되는 PC에서 최신 이미지를 다시 뜰 때:

```bash
# amd64 플랫폼으로 pull (ARM Mac 등에서 작업 시 --platform 필수)
docker pull --platform linux/amd64 node:22-alpine
docker pull --platform linux/amd64 node:22-bookworm-slim

# save + gzip
docker save node:22-alpine        | gzip > node22-alpine-amd64.tar.gz
docker save node:22-bookworm-slim | gzip > node22-slim-amd64.tar.gz
```

100MB가 넘는 파일이 생기면 GitHub push가 막히므로 분할한다(반입 측에서 재조립):

```bash
# 분할 (20MB 단위)
split -b 20m node22-slim-amd64.tar.gz node22-slim-amd64.tar.gz.part-

# 재조립
cat node22-slim-amd64.tar.gz.part-* > node22-slim-amd64.tar.gz
```
