# P Join · 기술 스택

## 기술 스택

팀 개발 환경은 2026년 11월 25일 기준으로 아래 버전을 사용합니다.

| 영역 | 기술 | 고정 버전 |
| --- | --- | --- |
| 프런트엔드 | Next.js | [16.4.6](https://github.com/vercel/next.js/releases/tag/v16.4.6) |
| UI 런타임 | React / React DOM | [19.3.1](https://www.npmjs.com/package/react/v/19.3.1) |
| 백엔드 | Express.js | [5.3.0](https://github.com/expressjs/express/releases/tag/v5.3.0) |
| 데이터베이스 | PostgreSQL | [18.7](https://www.postgresql.org/docs/release/18.7/) |
| ORM | Prisma CLI, Prisma Client, PostgreSQL 어댑터 | [7.10.1](https://www.prisma.io/docs/orm/release-status) |
| PostgreSQL 드라이버 | `pg` | [8.23.1](https://www.npmjs.com/package/pg/v/8.23.1) |
| JavaScript 런타임 | Node.js | [24.22.3 LTS](https://nodejs.org/en/download/archive/v24.22.3) |
| 패키지 매니저 | npm | [12.2.3](https://docs.npmjs.com/cli-commands/npm/) |

Prisma는 정식 지원 중인 7 계열을 사용합니다. PostgreSQL은 Prisma의 지원 목록에 명시된 18 계열의 최신 패치인 18.7을 사용합니다. Node.js 24.22.3은 Next.js, Prisma, npm의 런타임 요구 사항을 충족합니다. [Prisma 릴리스 상태](https://www.prisma.io/docs/orm/release-status) · [Prisma 지원 데이터베이스](https://docs.prisma.io/docs/orm/reference/supported-databases) · [Node.js 요구 사항](https://github.com/npm/cli/blob/v12.2.3/package.json#L263)

## Windows · PowerShell

### Node.js와 npm 맞추기

NVM for Windows가 없다면 아래 명령으로 설치합니다. 설치 후 관리자 PowerShell을 새로 열고 다음 블록을 실행합니다. [NVM for Windows 설치 안내](https://github.com/nvm-windows/nvm/wiki)

```powershell
winget install --exact --id CoreyButler.NVMforWindows
```

```powershell
nvm install 24.22.3
nvm use 24.22.3
npm.cmd install --global npm@12.2.3
node --version
npm.cmd --version
```

### 앱 패키지 버전 맞추기

각 명령은 해당 앱의 `package.json`이 있는 디렉터리에서 실행합니다.

```powershell
# 프런트엔드
npm.cmd install --save-exact next@16.4.6 react@19.3.1 react-dom@19.3.1
npm.cmd ls next react react-dom --depth=0

# 백엔드
npm.cmd install --save-exact express@5.3.0 '@prisma/client@7.10.1' '@prisma/adapter-pg@7.10.1' pg@8.23.1
npm.cmd install --save-dev --save-exact prisma@7.10.1
npm.cmd ls express prisma '@prisma/client' '@prisma/adapter-pg' pg --depth=0
```

### PostgreSQL 18.7 실행 및 확인

[Docker Desktop](https://docs.docker.com/get-docker/desktop/)을 설치하고 실행한 상태에서 작업합니다. `docker run`은 처음 한 번 실행합니다. 이후에는 `docker start pjoin-postgres`로 같은 컨테이너를 시작합니다.

```powershell
docker run --name pjoin-postgres --detach `
  --publish 127.0.0.1:5432:5432 `
  --env POSTGRES_USER=pjoin --env POSTGRES_PASSWORD=pjoin_local_dev --env POSTGRES_DB=pjoin `
  --volume pjoin-pg18:/var/lib/postgresql `
  postgres:18.7-bookworm

docker exec pjoin-postgres pg_isready -U pjoin -d pjoin
docker exec pjoin-postgres psql -U pjoin -d pjoin -Atc "SHOW server_version"
```

## macOS · zsh

### Node.js와 npm 맞추기

[nvm 설치 안내](https://github.com/nvm-sh/nvm#installing-and-updating)에 따라 nvm을 설치한 뒤 새 터미널을 열고 실행합니다.

```zsh
nvm install 24.22.3
nvm use 24.22.3
npm install --global npm@12.2.3
node --version
npm --version
```

### 앱 패키지 버전 맞추기

각 명령은 해당 앱의 `package.json`이 있는 디렉터리에서 실행합니다.

```zsh
# 프런트엔드
npm install --save-exact next@16.4.6 react@19.3.1 react-dom@19.3.1
npm ls next react react-dom --depth=0

# 백엔드
npm install --save-exact express@5.3.0 @prisma/client@7.10.1 @prisma/adapter-pg@7.10.1 pg@8.23.1
npm install --save-dev --save-exact prisma@7.10.1
npm ls express prisma @prisma/client @prisma/adapter-pg pg --depth=0
```

### PostgreSQL 18.7 실행 및 확인

[Docker Desktop](https://docs.docker.com/get-docker/desktop/)을 설치하고 실행한 상태에서 작업합니다. `docker run`은 처음 한 번 실행합니다. 이후에는 `docker start pjoin-postgres`로 같은 컨테이너를 시작합니다.

```zsh
docker run --name pjoin-postgres --detach \
  --publish 127.0.0.1:5432:5432 \
  --env POSTGRES_USER=pjoin --env POSTGRES_PASSWORD=pjoin_local_dev --env POSTGRES_DB=pjoin \
  --volume pjoin-pg18:/var/lib/postgresql \
  postgres:18.7-bookworm

docker exec pjoin-postgres pg_isready -U pjoin -d pjoin
docker exec pjoin-postgres psql -U pjoin -d pjoin -Atc "SHOW server_version"
```

## 공통 개발 설정

- 위 명령의 로컬 개발용 Prisma 연결 문자열은 `DATABASE_URL=postgresql://pjoin:pjoin_local_dev@127.0.0.1:5432/pjoin?schema=public`입니다.
- PostgreSQL 초기화 직후에는 `pg_isready`가 준비됐다고 표시한 뒤 서버 버전을 확인합니다. 양쪽 OS에서 같은 [PostgreSQL 공식 이미지 태그](https://github.com/docker-library/docs/blob/master/postgres/README.md#supported-tags-and-respective-dockerfile-links)를 사용합니다.
- 의존성을 처음 고정하거나 변경할 때 `package.json`과 `package-lock.json`을 함께 반영합니다. 기존 잠금 파일을 재현할 때는 Windows에서 `npm.cmd ci`, macOS에서 `npm ci`를 사용합니다.

## ERD 및 API 계약

테이블 관계, 필드와 제약은 [데이터베이스 ERD](07_erd.md)에서 확인합니다. 엔드포인트와 오류 응답은 [API 레퍼런스](08_api_reference.md#api-contract)를 참고합니다.
