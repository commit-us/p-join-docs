# P Join · API 레퍼런스

P Join의 팀 검토용 HTTP API 설계입니다. 요청·응답 필드, 예시와 화면별 담당 정보는 [p-join-api 카탈로그](../p-join-api/index.html#catalog)에서 확인합니다. 데이터 관계는 [ERD](07_erd.md), 화면 ID는 [IA](02_ia.md)를 참고합니다.

- [공통 API 계약](#api-contract)
- [API 목록](#endpoint-list)
- [대체된 API](#replaced-endpoints)
- [ERD 검토](#schema-review)
- [팀 합의 항목](#open-decisions)

<a id="api-contract"></a>

## 공통 API 계약

> **팀 합의 완료 (2026-11-25)** 아래 인증·보안·오류·세션 처리 규칙은 P Join의 **내부 API 전체에 적용하는 공통 계약**입니다. 개별 API의 이전 예시와 충돌하면 이 공통 계약을 우선하며, 카탈로그의 상세 예시도 이에 맞춰 동기화해야 합니다.

- **경로 표준 (미확정)** 일반 API는 `/api`, S14 GitHub API는 `/api/v1`을 사용합니다. S11·S13 명세에는 루트 상대 경로가 포함됩니다. 배포 호스트와 경로 표준은 별도 팀 합의 항목입니다.
- **① 인증 방식 (확정)** 로그인 필수인 모든 P Join API(프로젝트·지원·멤버·북마크·좋아요·알림·GitHub 연동 API 포함)는 **HttpOnly 세션 쿠키**로 인증합니다. 공개 API는 비로그인 접근을 허용합니다. Express 세션과 PostgreSQL 세션 저장소는 기존 구현 제안으로 유지합니다. GitHub App 설치 토큰은 서버에만 보관하며 브라우저에 전달하지 않습니다.
- **② CSRF 보호 (확정)** 세션 쿠키를 사용하는 상태 변경 요청(`POST`·`PUT`·`PATCH`·`DELETE`)은 **CSRF 토큰과 Origin 검증**을 적용합니다. OAuth 콜백은 `state`, GitHub 웹훅은 `X-Hub-Signature-256` HMAC 등 전용 검증을 사용합니다. 실제 발급·전달 방법과 토큰 헤더명은 구현 단계에서 상세 계약을 확정합니다.
- **③ 인증·인가 오류 (확정)** 세션이 없거나 만료된 경우 **HTTP 401 + `UNAUTHENTICATED`**, 로그인은 됐으나 작업 권한이 없는 경우 **HTTP 403 + `FORBIDDEN`**을 사용합니다. 로그인 실패 시의 `INVALID_CREDENTIALS` 같은 개별 도메인 오류는 별개로 유지합니다. 리소스 존재 여부를 공개하면 안 되는 경우에는 정책에 따라 404를 사용할 수 있습니다.
- **④ 오류 응답 JSON (확정)** P Join 내부 API의 오류 응답은 `{"error":{"code":"...","message":"...","requestId":"..."}}` 형태로 통일합니다. `requestId`는 선택 필드입니다. **성공 응답 형식**과 **외부 GitHub REST API의 원본 응답**은 이 결정의 대상이 아닙니다.
- **⑥ 세션 만료 및 작성 중 데이터 (확정)** 비활동 만료 24시간·절대 만료 7일을 기존 정책으로 유지하고, 자동 Polling은 비활동 시간을 갱신하지 않습니다. 인증이 필요한 저장 요청에 401이 오면 프론트엔드는 **작성 중인 입력값을 보존하고 재로그인을 안내**합니다. 재로그인 후 최신 데이터·권한을 확인하고 **사용자가 다시 저장**하도록 하며, 변경 요청을 무조건 자동 재전송하지 않습니다. 화면 이동·새로고침을 넘는 초안 보존 방법은 프론트엔드 구현에서 별도로 정합니다.

**공통 오류 응답 예시**

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "접근 권한이 없습니다.",
    "requestId": "req_123"
  }
}
```

<a id="endpoint-list"></a>

## API 목록

카탈로그의 메서드와 경로를 그대로 표기합니다. 외부 API는 대상 호스트를 함께 표시합니다.

- [인증·사용자](#api-group-1)
- [프로젝트](#api-group-2)
- [지원·멤버](#api-group-3)
- [알림](#api-group-4)
- [일정·이슈·활동](#api-group-5)
- [GitHub 저장소·PR](#api-group-6)

<a id="api-group-1"></a>

### 인증·사용자

#### `POST /api/auth/signup` · 회원가입

로그인 ID·이메일·닉네임으로 계정을 만듭니다.

- **권한** 비로그인
- **명세** 문서 제안
- **요청 본문** loginId, password, email, nickname
- **대표 성공 응답** 201 Created
- **검증·동작** loginId: 소문자·숫자·밑줄 4–20자, password 8–128자, nickname 2–20자. 이메일은 형식·중복만 확인하며 소유권 인증은 하지 않습니다. Argon2id 해시. 400 INVALID_INPUT · 409 DUPLICATE_ACCOUNT.
- **오류 응답** `400` `INVALID_INPUT` · `409` `DUPLICATE_ACCOUNT`

#### `POST /api/auth/login` · 로그인

loginId와 비밀번호로 세션을 시작합니다.

- **권한** 비로그인
- **명세** 문서 제안
- **요청 본문** loginId, password
- **대표 성공 응답** 200 OK + HttpOnly 세션 쿠키
- **검증·동작** 이메일 로그인은 지원하지 않습니다. 성공 시 세션 ID를 재발급합니다. 15분 내 5회 실패 제한 및 IP 제한. 400 INVALID_INPUT · 401 INVALID_CREDENTIALS · 429 TOO_MANY_REQUESTS.
- **계약 메모** Users 명세는 loginId만 받습니다. 기획안 F01의 이메일 로그인은 별도 확인이 필요합니다. 성공 본문의 세부 필드는 인증 상태 응답을 참고한 예시입니다.
- **오류 응답** `400` `INVALID_INPUT` · `401` `INVALID_CREDENTIALS` · `429` `TOO_MANY_REQUESTS`

#### `POST /api/auth/logout` · 로그아웃

현재 서버 세션을 무효화하고 쿠키를 제거합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **대표 성공 응답** 204 No Content
- **검증·동작** CSRF 토큰·Origin 검사. 500 INTERNAL_SERVER_ERROR.
- **오류 응답** `500` `INTERNAL_SERVER_ERROR`

#### `GET /api/auth/me` · 인증 상태 조회

현재 세션의 사용자 기본 정보를 반환합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **대표 성공 응답** 200 {id,loginId,nickname}
- **검증·동작** 비활동 24시간·절대 만료 7일. 자동 polling은 비활동 시간을 갱신하지 않습니다. 401 UNAUTHENTICATED.
- **오류 응답** `401` `UNAUTHENTICATED`

#### `GET /api/users/me` · 내 프로필 조회

비밀번호 해시를 제외한 본인 프로필을 반환합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **대표 성공 응답** 200 {id,loginId,email,nickname,bio,avatarUrl,websiteUrl}
- **검증·동작** 본인 프로필만 조회. 401 UNAUTHENTICATED.
- **오류 응답** `401` `UNAUTHENTICATED`

#### `GET /users/{userId}` · 사용자 공개 프로필

프로젝트 팀원 패널의 “프로필 보기” 링크에서 사용합니다.

- **권한** 공개 범위 정책 미정
- **명세** 문서 제안
- **관련 화면** S10, S11 · F13, F15
- **대표 성공 응답** 200 사용자 공개 프로필
- **검증·동작** S11 문서는 이 경로를 다른 파트의 프로필 API로 참조합니다. Users 명세에는 타인 프로필 조회 계약이 없으므로 공개 필드·인가·prefix를 추가로 확정해야 합니다.
- **계약 메모** 팀원 카드에서 접근하는 공개 프로필의 필드와 조회 권한은 검토안입니다.
- **오류 응답** `404` 프로필을 찾을 수 없음 · 초안

#### `PATCH /api/users/me` · 내 프로필 수정

허용된 프로필 필드만 변경합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **요청 본문** nickname?, bio?, websiteUrl?
- **대표 성공 응답** 200 수정된 프로필
- **검증·동작** nickname 2–20자 고유, bio 최대 500자, websiteUrl은 HTTP(S). 다른 필드는 거부. 400 INVALID_INPUT · 409 DUPLICATE_NICKNAME.
- **계약 메모** 수정 응답은 내 프로필 조회와 같은 형태로 예시를 구성했습니다. 프로필 필드의 null·초기화 규칙은 명세에서 확인이 필요합니다.
- **오류 응답** `400` `INVALID_INPUT` · `401` `UNAUTHENTICATED` · `409` `DUPLICATE_NICKNAME`

#### `POST /api/users/me/avatar` · 프로필 사진 등록·변경

이미지를 검사한 뒤 전용 스토리지에 저장합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **요청 본문** multipart image (PNG/JPG, ≤5 MB)
- **대표 성공 응답** 200 {avatarUrl}
- **검증·동작** 실제 MIME·시그니처 검증, 이미지 재인코딩, 안전한 무작위 파일명. 400 INVALID_FILE · 413 FILE_TOO_LARGE.
- **오류 응답** `400` `INVALID_FILE` · `401` `UNAUTHENTICATED` · `413` `FILE_TOO_LARGE`

#### `PATCH /api/users/me/password` · 비밀번호 변경

현재 비밀번호 확인 후 새 비밀번호로 갱신합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **요청 본문** currentPassword, newPassword
- **대표 성공 응답** 204 + 세션 쿠키 삭제
- **검증·동작** 8–128자, 기존 값과 달라야 함. authVersion 증가를 원자 처리해 모든 기존 세션 무효화. 400 INVALID_INPUT / INVALID_CURRENT_PASSWORD / SAME_PASSWORD.
- **계약 메모** 비밀번호 요청 필드명은 기존 통합안 표기를 사용합니다.
- **오류 응답** `400` `INVALID_INPUT · INVALID_CURRENT_PASSWORD · SAME_PASSWORD` · `401` `UNAUTHENTICATED`

#### `GET /api/users/me/github` · 내 GitHub 계정 연결 상태 조회

내 GitHub 계정 연결 상태와 프로필 정보를 반환합니다.

- **권한** 로그인 필요
- **명세** 문서 제안 · 누락 보완
- **관련 화면** S05 · F17
- **대표 성공 응답** 200 {status,account}
- **검증·동작** 계정 연결과 프로젝트별 저장소 연결 상태는 별도로 조회합니다.
- **오류 응답** `401` `UNAUTHENTICATED`

#### `GET /api/users/me/github/connect` · 내 GitHub 계정 연결 시작

로그인 세션의 사용자에 대해 GitHub 계정 연결을 시작합니다.

- **권한** 로그인 필요
- **명세** 문서 제안 · 누락 보완
- **관련 화면** S05 · F17
- **대표 성공 응답** 302 Location
- **검증·동작** 계정 연동 방식이 결정되면 콜백 경로를 명세합니다.
- **계약 메모** 로그인한 사용자가 이 경로로 이동하면 GitHub 계정 연결을 시작합니다. 콜백 경로는 계정 연동 방식이 결정되면 명세합니다.
- **오류 응답** `401` `UNAUTHENTICATED`

#### `DELETE /api/users/me/github` · 내 GitHub 계정 연결 해제

내 P Join 계정에서 GitHub 계정 연결을 해제합니다.

- **권한** 로그인 필요
- **명세** 문서 제안 · 누락 보완
- **관련 화면** S05 · F17
- **대표 성공 응답** 204 No Content
- **검증·동작** 사용자가 해제를 확인한 뒤 호출합니다.
- **계약 메모** 현재 사용자의 계정 연결을 해제합니다. 프로젝트 저장소 연결·해제는 프로젝트별 저장소 API가 담당합니다.
- **오류 응답** `401` `UNAUTHENTICATED`

<a id="api-group-2"></a>

### 프로젝트

#### `POST /api/projects` · 프로젝트 생성

프로젝트와 기술 스택 연결을 트랜잭션으로 생성합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **요청 본문** title, description, detailTitle?, detailDescription?, coverImageUrl?, recruitmentStartAt?, recruitmentEndAt?, techStackIds?
- **대표 성공 응답** 201 프로젝트 및 기술 스택
- **검증·동작** title 2–100자, description 10–5000자. 모집 시작·마감은 둘 다 입력하거나 생략, 시작≤마감. techStackIds 중복 없이 최대 10개. Idempotency-Key는 제안. 400 INVALID_PROJECT_INPUT / INVALID_TECH_STACK · 409 IDEMPOTENCY_CONFLICT.
- **계약 메모** Projects의 생성 응답 예시는 기술 항목에 ID와 이름을 사용합니다. 기술 연결 문서는 displayOrder도 예시로 제공합니다.
- **오류 응답** `400` `INVALID_PROJECT_INPUT · INVALID_TECH_STACK` · `401` `UNAUTHENTICATED` · `403` `FORBIDDEN` · `409` `IDEMPOTENCY_CONFLICT`

#### `GET /api/projects/me` · 내 프로젝트 목록

소유 프로젝트를 페이지 단위로 조회합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **쿼리** page=1, size=20 (max 100), sort=createdAt_desc, status?
- **대표 성공 응답** 200 {items,page,size,total,totalPages}
- **검증·동작** 논리 삭제 제외. 400 INVALID_PAGINATION.
- **오류 응답** `400` `INVALID_PAGINATION` · `401` `UNAUTHENTICATED`

#### `GET /api/projects/{projectId}` · 공개 프로젝트 상세

프로젝트 내용, 모집 정보, 기술 스택, 오너의 공개 정보와 전체 좋아요 수를 조회합니다.

- **권한** 공개
- **명세** 문서 제안
- **관련 화면** S02 · F07–F09
- **대표 성공 응답** 200 프로젝트 + owner + likeCount
- **⑤ 소유자 공개 정보 (팀 합의 완료)** 응답의 `owner`는 `{ userId: UUID, nickname: string, avatarUrl: string | null }`을 포함합니다. 프로필 사진이 없으면 `avatarUrl: null`을 반환하며, 이메일·로그인 ID·비밀번호 해시 등 비공개 계정 정보는 포함하지 않습니다. 비로그인 사용자에게도 같은 공개 정보를 제공합니다.
- **검증·동작** 로그인 여부와 관계없이 같은 응답을 반환합니다. 스택은 displayOrder 오름차순. 400 INVALID_PROJECT_ID · 404 PROJECT_NOT_FOUND.
- **계약 메모** 로그인 여부와 관계없이 같은 공개 필드를 반환합니다. 좋아요·북마크·지원의 내 상태는 로그인한 경우 프로젝트별 내 상태 조회에서 가져옵니다.
- **오류 응답** `400` `INVALID_PROJECT_ID` · `404` `PROJECT_NOT_FOUND`

#### `PATCH /api/projects/{projectId}` · 프로젝트 수정

소유자 프로젝트의 전달 필드만 수정하고 기술 연결도 반영합니다.

- **권한** 소유자
- **명세** 문서 제안
- **요청 본문** 수정 필드 일부 + techStackIds? + version?
- **대표 성공 응답** 200 수정된 프로젝트
- **검증·동작** version은 ERD 추가 제안. 기술 FK·중복·최대 10개 확인, 전체 트랜잭션. 400 · 401 · 403 · 404 · 409 VERSION_CONFLICT.
- **오류 응답** `400` `INVALID_PROJECT_INPUT · INVALID_TECH_STACK · INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `403` `FORBIDDEN` · `404` `PROJECT_NOT_FOUND` · `409` `VERSION_CONFLICT`

#### `PATCH /api/projects/{projectId}/status` · 프로젝트 상태 변경

lifecycleStatus를 허용된 상태 전이로 변경합니다.

- **권한** 소유자
- **명세** 문서 제안
- **요청 본문** lifecycleStatus, version?
- **대표 성공 응답** 200 {projectId,lifecycleStatus,version?,updatedAt}
- **검증·동작** COMPLETED 등 종료 상태와 모집기간 기반 모집 상태는 별도 개념. 상태값·전이는 팀 합의 전 제안. 400 INVALID_STATUS · 409 INVALID_STATUS_TRANSITION.
- **계약 메모** 기획안 F09는 날짜에 따른 모집 상태와 완료·중단 상태를 구별합니다. 상태값과 전이 정책은 팀 검토 대상입니다.
- **오류 응답** `400` `INVALID_STATUS · INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `403` `FORBIDDEN` · `404` `PROJECT_NOT_FOUND` · `409` `INVALID_STATUS_TRANSITION`

#### `DELETE /api/projects/{projectId}` · 프로젝트 삭제

연관 이력을 고려해 논리 삭제를 제안합니다.

- **권한** 소유자
- **명세** 문서 제안
- **대표 성공 응답** 204 No Content
- **검증·동작** deletedAt은 ERD 추가 제안. 지원·멤버 연관 데이터 정책 확정 필요. 404 PROJECT_NOT_FOUND · 409 DELETE_CONFLICT.
- **오류 응답** `400` `INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `403` `FORBIDDEN` · `404` `PROJECT_NOT_FOUND` · `409` `DELETE_CONFLICT`

#### `GET /api/tech-stacks` · 기술 스택 검색

선택 가능한 공통 기술 마스터를 검색합니다.

- **권한** 공개
- **명세** 문서 제안
- **쿼리** search? (이름·searchKeywords)
- **대표 성공 응답** 200 {items:[{techStackId,name,searchKeywords,officialLogoUrl}]}
- **검증·동작** 이름순 정렬 제안. 분류 필드는 ERD에 없어 분류 필터 보류. isActive도 추가 제안. 400 INVALID_SEARCH_QUERY.
- **오류 응답** `400` `INVALID_SEARCH_QUERY`

#### `PUT /api/projects/{projectId}/bookmark` · 프로젝트 북마크 등록

내 북마크를 멱등하게 등록합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **대표 성공 응답** 201 최초 / 200 이미 등록 {isBookmarked:true}
- **검증·동작** (userId,projectId) 중복 방지; 재등록은 최초 저장 시각 보존. 400 INVALID_PROJECT_ID · 401 UNAUTHENTICATED · 404 PROJECT_NOT_FOUND.
- **계약 메모** 로그인 필수. 인증 수단과 전달 방식은 팀 공통 인증 명세를 따릅니다.
- **오류 응답** `400` `INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `404` `PROJECT_NOT_FOUND`

#### `DELETE /api/projects/{projectId}/bookmark` · 프로젝트 북마크 해제

내 북마크를 해제합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **대표 성공 응답** 200 {isBookmarked:false}
- **검증·동작** 이미 해제된 경우도 성공. 400 INVALID_PROJECT_ID · 401 · 404.
- **계약 메모** 로그인 필수. 인증 수단과 전달 방식은 팀 공통 인증 명세를 따릅니다.
- **오류 응답** `400` `INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `404` `PROJECT_NOT_FOUND`

#### `GET /api/me/bookmarks` · 내 북마크 목록

북마크한 프로젝트를 최근 저장순으로 조회합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **쿼리** limit=6 (1–50), cursor?
- **대표 성공 응답** 200 {items:[{projectId,title,bookmarkedAt}],nextCursor}
- **검증·동작** createdAt 내림차순, 동률 projectId 내림차순. 빈 목록은 items=[] 및 nextCursor=null. 400 INVALID_LIMIT / INVALID_CURSOR · 401.
- **계약 메모** 로그인 필수. 인증 수단과 전달 방식은 팀 공통 인증 명세를 따릅니다.
- **오류 응답** `400` `INVALID_LIMIT · INVALID_CURSOR` · `401` `UNAUTHENTICATED`

#### `PUT /api/projects/{projectId}/like` · 프로젝트 좋아요 등록

내 좋아요를 멱등하게 추가하고 전체 건수를 돌려줍니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **대표 성공 응답** 201 최초 / 200 중복 {isLiked:true,likeCount}
- **검증·동작** 복합 PK(userId,projectId)로 중복 집계 방지. 400 INVALID_PROJECT_ID · 401 · 404.
- **계약 메모** 로그인 필수. 인증 수단과 전달 방식은 팀 공통 인증 명세를 따릅니다.
- **오류 응답** `400` `INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `404` `PROJECT_NOT_FOUND`

#### `DELETE /api/projects/{projectId}/like` · 프로젝트 좋아요 취소

내 좋아요를 취소하고 전체 건수를 반환합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **대표 성공 응답** 200 {isLiked:false,likeCount}
- **검증·동작** 이미 취소돼도 전체 수를 정확히 반환. 400 INVALID_PROJECT_ID · 401 · 404.
- **계약 메모** 로그인 필수. 인증 수단과 전달 방식은 팀 공통 인증 명세를 따릅니다.
- **오류 응답** `400` `INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `404` `PROJECT_NOT_FOUND`

#### `GET /api/projects` · 공개 프로젝트 목록

공개 공고를 검색하고 기술 스택·모집 상태별로 조회합니다.

- **권한** 공개
- **명세** 문서 제안 · 누락 보완
- **관련 화면** S01 · F06, F09
- **쿼리** q?, techStackIds?, recruitmentStatus?, sort?, limit?, cursor?
- **대표 성공 응답** 200 {items,nextCursor}
- **검증·동작** 스크롤 추가 조회는 직전 nextCursor를 사용합니다. 기본 정렬은 최신 생성순입니다.
- **계약 메모** 최신 생성순으로 조회하고 생성 시각이 같으면 projectId 내림차순으로 정렬합니다. 프로젝트 분야 필터는 저장 필드와 분류값을 확정한 뒤 적용합니다.
- **오류 응답** `400` `INVALID_QUERY · INVALID_CURSOR`

#### `GET /api/projects/{projectId}/me` · 프로젝트별 내 상태 조회

로그인한 사용자의 좋아요·북마크·지원 상태를 조회합니다.

- **권한** 로그인 필요
- **명세** 문서 제안 · 분리
- **관련 화면** S02 · F07, F08
- **대표 성공 응답** 200 {isLiked,isBookmarked,myApplication}
- **검증·동작** 지원 이력이 없으면 myApplication은 null입니다.
- **계약 메모** 현재 로그인 세션의 사용자 기준으로 조회합니다. 비로그인 화면은 공개 상세만 호출합니다.
- **오류 응답** `400` `INVALID_PROJECT_ID` · `401` `UNAUTHENTICATED` · `404` `PROJECT_NOT_FOUND`

#### `GET /api/me/projects` · 참여 중인 프로젝트 목록

사용자 대시보드에서 본인의 프로젝트 멤버십을 조회합니다.

- **권한** 로그인 필요
- **명세** 문서 제안 · 누락 보완
- **관련 화면** S06 · F14
- **쿼리** limit?, cursor?
- **대표 성공 응답** 200 {items,nextCursor}
- **검증·동작** 프로젝트 멤버십을 기준으로 조회합니다.
- **계약 메모** 현재 사용자의 프로젝트 멤버십을 기준으로 조회합니다. 참여 시각 내림차순이며, 같은 시각에는 projectId 내림차순으로 정렬합니다.
- **오류 응답** `400` `INVALID_LIMIT · INVALID_CURSOR` · `401` `UNAUTHENTICATED`

<a id="api-group-3"></a>

### 지원·멤버

#### `POST /api/projects/{projectId}/applications` · 프로젝트 지원

ERD의 applications 행을 생성합니다.

- **권한** 로그인 필요
- **명세** ERD 초안
- **관련 화면** S02 · M12 · F08
- **요청 본문** comment, privacyConsentAt
- **대표 성공 응답** 201 Application
- **검증·동작** ERD에 정의된 필드 기반 초안. 지원 가능 상태, 중복 지원 UNIQUE 제약, 동의 시각 검증은 팀 결정. 원문 ERD 기반 추론.
- **계약 메모** 기존 카탈로그의 privacyConsentAt을 유지했습니다. 체크박스의 동의 확인과 서버의 기록 시각을 어떤 값으로 교환할지는 확인이 필요합니다.
- **오류 응답** `400` 코멘트 또는 동의 입력 오류 · `401` 로그인 필요 · `404` `PROJECT_NOT_FOUND` · `409` 중복 지원 또는 지원할 수 없는 모집 상태

#### `GET /api/projects/{projectId}/applications` · 프로젝트 지원자 목록

프로젝트에 접수된 지원을 조회합니다.

- **권한** 프로젝트 OWNER
- **명세** ERD 초안
- **관련 화면** S10 · F13
- **쿼리** status?, page=1, size=20
- **대표 성공 응답** 200 {items,total,page,size}
- **검증·동작** applicantId, status, handledAt 및 생성시각 반환 제안. 상태 enum과 정렬 기준 확정 필요.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `PROJECT_NOT_FOUND`

#### `GET /api/me/applications` · 내 지원 내역

로그인 사용자의 프로젝트 지원을 조회합니다.

- **권한** 로그인 필요
- **명세** ERD 초안
- **관련 화면** S06 · F08, F14
- **쿼리** status?, page=1, size=20
- **대표 성공 응답** 200 {items,total,page,size}
- **검증·동작** applicantId는 세션 사용자로 제한. 해당 조회 경로는 ERD만으로 도출한 제안.
- **오류 응답** `401` 로그인 필요

#### `GET /api/projects/{projectId}/applications/{applicationId}` · 지원 상세

지원 내용과 처리 상태를 조회합니다.

- **권한** 지원자 본인 또는 프로젝트 OWNER
- **명세** ERD 초안
- **관련 화면** S06, S10 · F08, F13
- **대표 성공 응답** 200 Application
- **검증·동작** 다른 사용자 지원의 비공개 comment 노출 방지. 404 APPLICATION_NOT_FOUND.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `APPLICATION_NOT_FOUND` 지원을 찾을 수 없음

#### `GET /api/projects/{projectId}/members` · 프로젝트 멤버 목록

project_members와 users를 조인해 팀원을 조회합니다.

- **권한** 프로젝트 멤버 또는 공개 정책
- **명세** ERD 초안
- **관련 화면** S11, S12 · F15, F16
- **대표 성공 응답** 200 {items:[{userId,nickname,profileImageUrl,role,joinedAt}]}
- **검증·동작** role은 ERD에 enum 미정. 이메일 등 개인정보는 필요한 권한만 노출.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `PROJECT_NOT_FOUND`

#### `GET /api/projects/{projectId}/members/{userId}` · 프로젝트 멤버 상세

프로젝트 내 멤버 역할과 가입 정보를 조회하며, 대시보드에서 필요하면 최근 활동을 함께 조회합니다.

- **권한** 프로젝트 멤버
- **명세** ERD 초안
- **관련 화면** S11, S12 · F15, F16
- **쿼리** include=latestActivity?
- **대표 성공 응답** 200 {projectId,userId,role,joinedAt,user,latestActivity?}
- **검증·동작** 멤버십과 프로젝트 접근 검증. 개인정보는 사용자 프로필 공개 범위 따름.
- **계약 메모** include=latestActivity로 대시보드 모달용 팀원 정보를 조회합니다. 활동이 없으면 latestActivity는 null이고, 직무·상태·연락처 표시는 검토안입니다.
- **오류 응답** `400` 지원하지 않는 include 값 · `401` 로그인 필요 · `403` 접근 권한 없음 · `404` 프로젝트 멤버를 찾을 수 없음

#### `PATCH /api/projects/{projectId}/members/{userId}` · 멤버 역할 변경

OWNER가 멤버의 role을 변경합니다.

- **권한** OWNER
- **명세** ERD 초안
- **관련 화면** S12 · F16
- **요청 본문** role
- **대표 성공 응답** 200 ProjectMember
- **검증·동작** ERD에는 role 컬럼만 있고 허용 역할과 변경 제약은 미정. 본인 권한 상승·마지막 OWNER 제거를 막는 정책 필요.
- **계약 메모** 변경 가능한 세부 역할은 검토 중입니다. 소유권 변경은 별도 소유권 이전 API의 계약을 따릅니다.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` 프로젝트 멤버를 찾을 수 없음

#### `DELETE /api/projects/{projectId}/members/{userId}` · 멤버 제거

프로젝트에서 멤버십을 제거합니다.

- **권한** OWNER
- **명세** ERD 초안
- **관련 화면** S12 · F16
- **대표 성공 응답** 204 No Content
- **검증·동작** 프로젝트별 권한, OWNER 자신 제거·소유권 이전 정책 필요. 실제 API 미작성, ERD 기반 제안.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` 프로젝트 멤버를 찾을 수 없음 · `409` 소유자 또는 제거할 수 없는 멤버

#### `DELETE /api/projects/{projectId}/members/me` · 프로젝트 나가기

본인의 프로젝트 멤버십을 종료합니다.

- **권한** 프로젝트 멤버
- **명세** ERD 초안
- **관련 화면** S11, S12 · F15, F16
- **대표 성공 응답** 204 No Content
- **검증·동작** OWNER의 나가기/소유권 이전은 별도 정책 필요. ERD에는 이 동작이 명시되지 않아 제안 상태.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `409` 소유권 이전이 필요한 상태

#### `PUT /api/projects/{projectId}/owner` · 프로젝트 소유권 이전

기존 OWNER가 새 OWNER로 소유권을 넘깁니다.

- **권한** 현재 OWNER + 대상은 프로젝트 멤버
- **명세** ERD 초안
- **관련 화면** S11, S12 · F15, F16
- **요청 본문** newOwnerId
- **대표 성공 응답** 200 Project
- **검증·동작** R&R에 소유권 이전이 있으나 세부 계약은 문서에 없습니다. projects.ownerId와 project_members.role을 원자적으로 갱신할지 합의해야 하는 ERD 기반 초안.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` 프로젝트 멤버를 찾을 수 없음 · `409` 소유권을 이전할 수 없는 상태

#### `POST /api/projects/{projectId}/applications/bulk-reject` · 지원 일괄 거절

확인한 지원 건 중 아직 대기 중인 건을 거절합니다.

- **권한** 프로젝트 소유자
- **명세** ERD 초안 · 누락 보완
- **관련 화면** S10 · M04 · F13
- **요청 본문** applicationIds
- **대표 성공 응답** 200 {rejectedCount,skippedApplicationIds}
- **검증·동작** 모달에서 확인한 ID만 대상으로 합니다. 다른 프로젝트의 지원은 포함할 수 없습니다.
- **계약 메모** 모달 확인 뒤 새로 접수된 지원은 대상에 포함하지 않습니다. 확인한 대상 중 아직 대기 중인 지원만 거절하고, 처리된 대상은 응답에 별도로 표시합니다.
- **오류 응답** `400` 지원 ID 목록이 비어 있거나 형식이 올바르지 않음 · `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `APPLICATION_NOT_FOUND` 지원을 찾을 수 없음

#### `PATCH /api/projects/{projectId}/applications/{applicationId}/decision` · 지원 승인·거절

프로젝트 소유자가 대기 중인 지원을 승인하거나 거절합니다.

- **권한** 프로젝트 소유자
- **명세** ERD 초안 · 분리
- **관련 화면** S10 · F13
- **요청 본문** status: ACCEPTED 또는 REJECTED
- **대표 성공 응답** 200 Application
- **검증·동작** 승인 시 지원 상태 갱신과 멤버 등록을 같은 트랜잭션에서 처리합니다.
- **계약 메모** 승인 시 지원 상태 갱신과 멤버 등록을 한 트랜잭션에서 처리합니다.
- **오류 응답** `400` 처리 상태가 올바르지 않음 · `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `APPLICATION_NOT_FOUND` 지원을 찾을 수 없음 · `409` 이미 처리되었거나 허용되지 않는 상태 전이 · 초안

#### `PATCH /api/projects/{projectId}/applications/{applicationId}/cancel` · 내 지원 취소

지원자 본인이 대기 중인 지원을 취소합니다.

- **권한** 지원자 본인
- **명세** ERD 초안 · 분리
- **관련 화면** S02, S06 · F08
- **대표 성공 응답** 200 Application
- **검증·동작** 요청을 실행한 사용자의 본인 지원과 대기 상태를 확인합니다.
- **계약 메모** 지원자 본인의 대기 중 지원만 취소할 수 있습니다.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `APPLICATION_NOT_FOUND` 지원을 찾을 수 없음 · `409` 이미 처리되었거나 허용되지 않는 상태 전이 · 초안

<a id="api-group-4"></a>

### 알림

#### `GET /api/notifications` · 알림 목록

내 알림을 최신순으로 조회합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **관련 화면** S15 · F20
- **쿼리** page=1, limit=20, filter=all|unread
- **대표 성공 응답** 200 {items,page,limit,total}
- **검증·동작** Polling 30–60초. 자동 조회는 세션 비활동 타이머를 갱신하지 않습니다. 400 INVALID_QUERY · 401 UNAUTHENTICATED.
- **오류 응답** `400` `INVALID_QUERY` · `401` `UNAUTHENTICATED`

#### `GET /api/notifications/unread-count` · 미확인 알림 수

내 readAt=null 알림 수를 반환합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **관련 화면** S15 · F20
- **대표 성공 응답** 200 {count:number}
- **검증·동작** 자동 polling은 세션 활동 시각 갱신에서 제외. 401 UNAUTHENTICATED.
- **오류 응답** `401` `UNAUTHENTICATED`

#### `PATCH /api/notifications/{id}/read` · 알림 읽음 처리

읽지 않은 내 알림에 readAt을 기록합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **관련 화면** S15 · F20
- **대표 성공 응답** 200 {id,readAt,...}
- **검증·동작** 반복 요청에도 최초 readAt 유지, 소유자 검사. 401 · 404 NOT_FOUND.
- **계약 메모** 원문 예시가 명시한 id·readAt을 표시했습니다. 그 외 응답 필드는 원문에서 생략되어 있습니다.
- **오류 응답** `401` `UNAUTHENTICATED` · `404` `NOT_FOUND`

#### `PATCH /api/notifications/read-all` · 알림 전체 읽음

내 미확인 알림을 조건부 일괄 갱신합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **관련 화면** S15 · F20
- **대표 성공 응답** 200 {updatedCount:number}
- **검증·동작** userId와 readAt=null 조건으로 updateMany. 401 UNAUTHENTICATED.
- **오류 응답** `401` `UNAUTHENTICATED`

#### `DELETE /api/notifications/{id}` · 읽은 알림 삭제

읽은 본인 알림만 삭제합니다.

- **권한** 로그인 필요
- **명세** 문서 제안
- **관련 화면** S15 · F20
- **대표 성공 응답** 204 No Content
- **검증·동작** 미확인 알림 삭제는 거부. 400 NOT_READ · 401 · 404 NOT_FOUND. 알림은 30일 후 정기 삭제 제안.
- **오류 응답** `400` `NOT_READ` 미확인 알림 삭제 불가 · `401` `UNAUTHENTICATED` · `404` `NOT_FOUND`

<a id="api-group-5"></a>

### 일정·이슈·활동

#### `GET /projects/{projectId}/dashboard` · 프로젝트 대시보드

프로젝트·주간 목표·활동·도구 수·멤버·다음 일정을 묶어 반환합니다.

- **권한** OWNER, MEMBER
- **명세** 문서 제안
- **관련 화면** S11 · F15
- **대표 성공 응답** 200 {project,weeklyGoal,latestActivity,tools,members,nextSchedule,meta}
- **검증·동작** nextSchedule 없으면 null. github 미연결이면 openIssues/openPullRequests null. 캐시 TTL 60초 제안, 변경 이벤트에 무효화. 원문 경로 prefix 없음.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `PROJECT_NOT_FOUND`

#### `GET /projects/{projectId}/activities` · 최근 활동 목록

커밋·PR·이슈 활동을 최신순 커서 조회합니다.

- **권한** OWNER, MEMBER
- **명세** 문서 제안
- **관련 화면** S11 · F15
- **쿼리** limit=10 (max 50), cursor?, type?
- **대표 성공 응답** 200 {items,nextCursor}
- **검증·동작** type 예: COMMIT, PR_*, ISSUE_*. 중복 수집은 외부 ID/유니크 키로 방지.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `PROJECT_NOT_FOUND`

#### `GET /projects/{projectId}/schedules` · 일정 목록

프로젝트 일정을 scheduledAt 오름차순으로 조회합니다.

- **권한** OWNER, MEMBER
- **명세** 문서 제안
- **관련 화면** S11 · F15
- **쿼리** from?, to?, include=past?
- **대표 성공 응답** 200 {items:[Schedule]}
- **검증·동작** from/to 생략 시 오늘부터 30일.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` `PROJECT_NOT_FOUND`

#### `POST /projects/{projectId}/schedules` · 일정 등록

새 프로젝트 일정을 만듭니다.

- **권한** OWNER (제안)
- **명세** 문서 제안
- **관련 화면** S11 · F15
- **요청 본문** scheduledAt, platform, link?, title?
- **대표 성공 응답** 201 Schedule + Location
- **검증·동작** 시간은 현재 이후. platform: ZOOM, GOOGLE_MEET, DISCORD, SLACK, OFFLINE, ETC. OFFLINE 외 link 필수, https:// 최대 500자. title 기본값 “다음 위클리 일정”, 최대 100자. 403 FORBIDDEN_OWNER_ONLY · 422 SCHEDULE_IN_PAST.
- **오류 응답** `401` 로그인 필요 · `403` `FORBIDDEN_OWNER_ONLY` · `404` `PROJECT_NOT_FOUND` · `422` `SCHEDULE_IN_PAST` 일정 시각이 과거임

#### `PATCH /projects/{projectId}/schedules/{scheduleId}` · 일정 수정

바뀐 필드만 낙관적 잠금으로 수정합니다.

- **권한** OWNER (제안)
- **명세** 문서 제안
- **관련 화면** S11 · F15
- **요청 본문** version 필수 + scheduledAt/platform/link/title 일부
- **대표 성공 응답** 200 Schedule, version +1
- **검증·동작** 버전 불일치: 409 SCHEDULE_VERSION_CONFLICT + details.currentVersion/current. 캐시 무효화.
- **계약 메모** 수정할 값만 전달합니다. 진행 방식을 바꿀 때는 최종 일정의 링크 조건을 함께 확인합니다.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` 일정을 찾을 수 없음 · `422` `SCHEDULE_IN_PAST` 일정 시각이 과거임 · `409` `SCHEDULE_VERSION_CONFLICT` 일정 버전 충돌 · 응답 구조 초안

#### `DELETE /projects/{projectId}/schedules/{scheduleId}` · 일정 삭제

일정을 물리 삭제합니다.

- **권한** OWNER (제안)
- **명세** 문서 제안
- **관련 화면** S11 · F15
- **쿼리** version 필수 query
- **대표 성공 응답** 204 No Content
- **검증·동작** 복구 불가. 삭제 후 404 또는 멱등 204는 팀 결정. 버전 불일치 409.
- **계약 메모** 이미 삭제된 일정에 대한 404 또는 204 정책은 팀 확인이 필요합니다.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `409` `SCHEDULE_VERSION_CONFLICT` 일정 버전 충돌 · 응답 구조 초안

#### `GET /projects/{projectId}/issues` · 이슈 목록

GitHub 원본을 동기화한 이슈 사본을 조회합니다.

- **권한** OWNER, MEMBER
- **명세** 문서 제안
- **관련 화면** S13 · M10 · F17, F18
- **쿼리** state=open|closed|all, q?, sort=updated|created, page=1, size=20
- **대표 성공 응답** 200 {counts,items,page,size,totalItems,lastSyncedAt}
- **검증·동작** counts는 state 필터와 무관하게 open/closed 모두 반환. 저장소 미연결 409 GITHUB_NOT_CONNECTED.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `409` `GITHUB_NOT_CONNECTED` GitHub 저장소 미연결

#### `GET /projects/{projectId}/issues/{number}` · 이슈 상세

목록 필드에 body, createdAt, closedAt을 추가해 반환합니다.

- **권한** OWNER, MEMBER
- **명세** 문서 제안
- **관련 화면** S13 · M10 · F17, F18
- **대표 성공 응답** 200 Issue
- **검증·동작** 원본은 GitHub, DB는 read model. 저장소 미연결 409.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` 이슈를 찾을 수 없음 · `409` `GITHUB_NOT_CONNECTED` GitHub 저장소 미연결

#### `POST /projects/{projectId}/issues` · 이슈 생성

GitHub에 먼저 생성한 뒤 issues 사본을 갱신합니다.

- **권한** OWNER, MEMBER (제안)
- **명세** 문서 제안
- **관련 화면** S13 · M10 · F17, F18
- **요청 본문** title (1–256자), body? (≤65,536자)
- **대표 성공 응답** 201 Issue 상세
- **검증·동작** 생성 상태 OPEN. 원격 실패 시 DB 미저장, 502 GITHUB_UPSTREAM_ERROR.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `409` `GITHUB_NOT_CONNECTED` GitHub 저장소 미연결 · `422` 이슈 입력 검증 오류 · `502` `GITHUB_UPSTREAM_ERROR` GitHub 요청 실패

#### `PATCH /projects/{projectId}/issues/{number}` · 이슈 수정·닫기/재열기

GitHub의 최신 updated_at과 충돌 검사 후 수정합니다.

- **권한** OWNER, MEMBER (제안)
- **명세** 문서 제안
- **관련 화면** S13 · M10 · F17, F18
- **요청 본문** baseUpdatedAt, title?, body?, state? (OPEN|CLOSED)
- **대표 성공 응답** 200 Issue 상세
- **검증·동작** baseUpdatedAt보다 GitHub가 최신이면 409 ISSUE_SYNC_CONFLICT + details.latest. 자동 덮어쓰기 금지. 삭제 API 없음.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `404` 이슈를 찾을 수 없음 · `409` `GITHUB_NOT_CONNECTED` GitHub 저장소 미연결 · `502` `GITHUB_UPSTREAM_ERROR` GitHub 요청 실패 · `409` `ISSUE_SYNC_CONFLICT` 이슈 수정 충돌 · 응답 구조 초안

#### `GET /projects/{projectId}/github/sync` · GitHub 동기화 상태 조회

연결 상태, repository, status, lastSyncedAt, lastError를 반환합니다.

- **권한** OWNER, MEMBER
- **명세** 문서 제안
- **관련 화면** S09, S11, S13 · F15, F17, F18
- **대표 성공 응답** 200 {connected,repository,status,lastSyncedAt,lastError}
- **검증·동작** status: IDLE, RUNNING, FAILED.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음

#### `POST /projects/{projectId}/github/sync` · 수동 동기화 시작

비동기 동기화 작업을 시작합니다.

- **권한** OWNER
- **명세** 문서 제안
- **관련 화면** S09, S11, S13 · F15, F17, F18
- **대표 성공 응답** 202 {status:RUNNING}
- **검증·동작** 이미 실행 중이면 같은 응답. 프로젝트당 분당 1회, 초과 429 RATE_LIMITED. 중복 실행 방지.
- **오류 응답** `401` 로그인 필요 · `403` 접근 권한 없음 · `409` `GITHUB_NOT_CONNECTED` GitHub 저장소 미연결 · `429` `RATE_LIMITED` 프로젝트당 분당 1회 제한

#### `POST /webhooks/github` · GitHub 웹훅 수신

push, issues, pull_request, pull_request_review 이벤트를 비동기 큐로 전달합니다.

- **권한** X-Hub-Signature-256 HMAC
- **명세** 문서 제안
- **요청 본문** GitHub payload
- **대표 성공 응답** 202 Accepted
- **검증·동작** 서명 실패 401. X-GitHub-Delivery로 중복 방지. PR 리뷰 문서가 pull_request_review 이벤트를 추가로 정의합니다. 빠르게 202 반환 후 비동기 처리.
- **계약 메모** 요청 예시는 이벤트별 주요 필드 발췌입니다. 전체 페이로드와 이벤트별 속성은 GitHub Docs를 참고하세요. 202 성공 응답에는 본문 스키마가 정의되어 있지 않습니다.
- **오류 응답** `401` 서명 검증 실패

<a id="api-group-6"></a>

### GitHub 저장소·PR

#### `GET /api/v1/projects/{projectId}/github/repositories` · 연결 가능한 저장소 목록

설치된 GitHub App이 접근 가능한 저장소 후보를 검색합니다.

- **권한** PJOIN 로그인 + 프로젝트 권한
- **명세** 문서 제안
- **관련 화면** S07, S09 · F17
- **쿼리** query?, cursor?, limit=20
- **대표 성공 응답** 200 {success,data:{items,nextCursor},meta:{source:GITHUB_API}}
- **검증·동작** 설치 계정·query·cursor별 캐시, 연결·권한 변경 시 무효화. 토큰은 브라우저에 전달하지 않음.
- **계약 메모** 저장소 선택 화면에 필요한 후보 항목의 세부 필드는 보완안입니다. 후보 조회는 이 경로를 사용합니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `502` `GITHUB_UNAVAILABLE` GitHub 저장소 요청 실패

#### `PUT /api/v1/projects/{projectId}/github/repository` · 저장소 연결·교체

githubRepositoryId 기준으로 PJOIN 프로젝트에 저장소를 연결합니다.

- **권한** 프로젝트 관리자 + App 접근
- **명세** 문서 제안
- **관련 화면** S07, S09 · F17
- **요청 본문** githubRepositoryId (GitHub BIGINT)
- **대표 성공 응답** 201 최초 / 200 교체
- **검증·동작** GH-02 전용 저장소 문서 기준. UNIQUE projectId 정책 여부 확인. 403 PROJECT_FORBIDDEN · 404 REPOSITORY_NOT_ACCESSIBLE · 409 PROJECT_REPOSITORY_ALREADY_LINKED · 502 GITHUB_UNAVAILABLE.
- **계약 메모** Notion의 통합 응답 봉투에 연결 결과 예시를 보완했습니다. 초대 permission은 멤버 초대 API에서 사용합니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` `REPOSITORY_NOT_ACCESSIBLE` · `409` `PROJECT_REPOSITORY_ALREADY_LINKED` · `502` `GITHUB_UNAVAILABLE` GitHub 저장소 요청 실패

#### `GET /api/v1/projects/{projectId}/github/repository` · 연결 저장소 조회

로컬 read model 저장소 메타데이터를 반환합니다.

- **권한** 프로젝트 권한
- **명세** 문서 제안
- **관련 화면** S08, S09 · F11, F17
- **대표 성공 응답** 200 Repository
- **검증·동작** repositoryId, githubRepositoryId, fullName, htmlUrl?, defaultBranch, visibility?, syncStatus, connectedAt, lastSyncedAt.
- **계약 메모** 카탈로그의 저장소 메타데이터와 Notion의 저장소 매핑을 바탕으로 구성한 응답입니다. 공개 범위 등 추가 제안 필드를 표시했습니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` 연결된 저장소 없음 · 상태 코드 초안

#### `POST /api/v1/projects/{projectId}/github/repository/sync` · 저장소 메타데이터 동기화

GitHub 최신 메타데이터를 확인하고 로컬 값을 갱신합니다.

- **권한** 프로젝트 권한
- **명세** 문서 제안
- **관련 화면** S09 · F17
- **대표 성공 응답** 200 완료 / 202 {jobId}
- **검증·동작** ETag/If-None-Match 활용 가능. fullName/defaultBranch/visibility, lastSyncedAt/syncStatus 갱신.
- **계약 메모** 기존 카탈로그의 완료·접수 계약을 기준으로 결과 필드를 보완했습니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `502` `GITHUB_UNAVAILABLE` GitHub 저장소 요청 실패

#### `DELETE /api/v1/projects/{projectId}/github/repository` · 저장소 연결 해제

PJOIN 연결만 해제하고 원격 GitHub 저장소는 유지합니다.

- **권한** 프로젝트 관리자 (권한 미확정)
- **명세** 문서 제안
- **관련 화면** S09 · F17
- **대표 성공 응답** 204 No Content
- **검증·동작** sync 중단·캐시 무효화. PR/review 데이터 cascade 삭제 금지; 보존·재연결 정책 결정 필요.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족

#### `POST /api/v1/projects/{projectId}/members/{userId}/github-invitation` · 승인된 참여자 GitHub 초대

지원이 승인된 PJOIN 사용자를 연결 저장소에 초대합니다.

- **권한** PJOIN 세션 + 프로젝트 승인 권한 + GitHub App Administration write
- **명세** 문서 제안
- **관련 화면** S10, S12 · F13, F16, F17
- **요청 본문** permission (예: pull; PR 작성이 필요하면 push 검토)
- **대표 성공 응답** 201 Pending / 204일 때 실제 permission 재조회
- **검증·동작** 원격 201은 초대 생성일 뿐 수락 완료가 아닙니다. ACTIVE는 permission API에서 실제 접근을 확인한 뒤 반환합니다. 초대 이력은 별도 테이블에 저장하지 않습니다.
- **계약 메모** GitHub의 204는 P Join의 응답 코드로 확정돼 있지 않습니다. GitHub가 204를 반환한 경우 서버에서 실제 권한을 재조회합니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `502` `GITHUB_UNAVAILABLE` GitHub 저장소 요청 실패

#### `GET /api/v1/projects/{projectId}/members/{userId}/github-invitation` · GitHub 초대·접근 상태 조회

초대 상태와 실제 저장소 permission을 반환합니다.

- **권한** 프로젝트 권한
- **명세** 문서 제안
- **관련 화면** S10, S12 · F13, F16, F17
- **대표 성공 응답** 200 {invitationStatus,requestedPermission,actualPermission,accessConfirmedAt}
- **검증·동작** GitHub 204 응답/웹훅으로 실제 접근을 추정하지 않고 permission endpoint로 확인합니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `502` `GITHUB_UNAVAILABLE` GitHub 저장소 요청 실패

#### `GET https://api.github.com/installation/repositories` · GitHub App 설치 저장소 목록

App 설치 범위의 원격 저장소를 페이지네이션해 가져옵니다.

- **권한** 서버 전용 GitHub App installation token
- **명세** 문서 제안
- **쿼리** GitHub page/per_page
- **대표 성공 응답** GitHub REST 응답
- **검증·동작** 외부 GitHub REST endpoint. 후보 검색/연결에서만 호출하고 App token은 브라우저·일반 캐시에 노출하지 않습니다.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `401` 인증 필요 · `403` 접근 금지 · `422` 입력 검증 실패

#### `PUT https://api.github.com/repos/{owner}/{repo}/collaborators/{username}` · GitHub 저장소 협력자 초대

승인된 멤버를 선택 저장소에 초대하거나 permission을 요청합니다.

- **권한** 서버 전용 GitHub App token + Administration write
- **명세** 문서 제안
- **요청 본문** permission
- **대표 성공 응답** 201 초대 생성 / 204 기존 권한 가능
- **검증·동작** 204 후 collaborator permission을 다시 조회해 실제 접근 확인. 403/404 원인을 확인하고 429·일부 5xx는 Retry-After에 따라 재시도.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `403` 접근 금지 · `404` 리소스 없음 · `422` 입력 검증 실패

#### `GET https://api.github.com/repos/{owner}/{repo}/invitations` · GitHub 대기 초대 조회

저장소의 대기 invitation에서 대상과 상태를 확인합니다.

- **권한** 서버 전용 GitHub App token
- **명세** 문서 제안
- **쿼리** GitHub pagination
- **대표 성공 응답** GitHub REST 응답
- **검증·동작** 보조 검증 정보. 외부 GitHub REST endpoint.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.

#### `GET https://api.github.com/repos/{owner}/{repo}/collaborators/{username}/permission` · GitHub 실제 권한 조회

초대 대상의 현재 저장소 접근 permission을 확인합니다.

- **권한** 서버 전용 GitHub App token
- **명세** 문서 제안
- **대표 성공 응답** GitHub permission 응답
- **검증·동작** 초대 수락 여부 및 실제 권한의 최종 확인. webhook은 재조회 신호이며 최종 기준은 이 endpoint.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `404` 리소스 없음

#### `GET https://api.github.com/repos/{owner}/{repo}/pulls?state=open&sort=updated&per_page=30&page=1` · GitHub PR 목록 조회

GitHub 원본 PR을 상태·페이지별로 조회합니다.

- **권한** 서버 전용 GitHub App token
- **명세** 문서 제안
- **쿼리** state, sort, per_page, page
- **대표 성공 응답** GitHub REST response
- **검증·동작** PJOIN 화면은 보통 local read model에서 조회하고 동기화 시에 호출합니다.
- **계약 메모** 경로에 붙은 쿼리는 기존 호출 예시입니다. 필드 표에는 GitHub REST의 기본값을 표시했습니다. 전체 응답은 GitHub Docs를 참고하세요.
- **오류 응답** `422` 입력 검증 실패

#### `GET https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}` · GitHub PR 상세 조회

본문·브랜치·head SHA 등 원본을 가져옵니다.

- **권한** 서버 전용 GitHub App token
- **명세** 문서 제안
- **대표 성공 응답** GitHub REST response
- **검증·동작** 병합 전 최신 상태 검증에 사용.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `404` 리소스 없음 · `422` 입력 검증 실패 · `500` GitHub 내부 오류 · `503` 서비스 이용 불가

#### `POST https://api.github.com/repos/{owner}/{repo}/pulls` · GitHub PR 생성

원격 PR을 생성합니다.

- **권한** 서버 전용 GitHub App token + Pull requests write
- **명세** 문서 제안
- **요청 본문** title, head, base, body?, draft?
- **대표 성공 응답** GitHub REST response
- **검증·동작** GitHub 성공 후 PJOIN DB upsert.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `403` 접근 금지 · `422` 입력 검증 실패

#### `PATCH https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}` · GitHub PR 수정·종료

허용 필드를 수정하거나 state=closed로 닫습니다.

- **권한** 서버 전용 GitHub App token + Pull requests write
- **명세** 문서 제안
- **요청 본문** 허용 수정 필드, state?
- **대표 성공 응답** GitHub REST response
- **검증·동작** GitHub 성공 후 로컬 모델 갱신.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `403` 접근 금지 · `422` 입력 검증 실패

#### `GET https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}/reviews` · GitHub PR 리뷰 목록 조회

원본 리뷰 상태·커밋 SHA를 가져옵니다.

- **권한** 서버 전용 GitHub App token
- **명세** 문서 제안
- **쿼리** GitHub pagination
- **대표 성공 응답** GitHub REST response
- **검증·동작** 병합 직전 fresh fetch와 리뷰 동기화에 사용.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.

#### `GET https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}` · GitHub 특정 리뷰 조회

원격 리뷰 상세를 조회합니다.

- **권한** 서버 전용 GitHub App token
- **명세** 문서 제안
- **대표 성공 응답** GitHub REST response
- **검증·동작** 리뷰 상세 PJOIN 경로와 ID 매핑 합의 필요.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `404` 리소스 없음

#### `POST https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}/reviews` · GitHub 리뷰 제출

승인·변경 요청·일반 의견을 GitHub에 씁니다.

- **권한** 서버 전용 GitHub App token + Pull requests write
- **명세** 문서 제안
- **요청 본문** body, event=APPROVE|REQUEST_CHANGES|COMMENT
- **대표 성공 응답** GitHub REST response
- **검증·동작** PJOIN 리뷰 제출 성공 후 GitHub review ID 기준 upsert.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `403` 접근 금지 · `422` 입력 검증 실패

#### `PUT https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/dismissals` · GitHub 리뷰 dismiss

PRD와 권한이 허용할 때만 원격 리뷰를 dismiss합니다.

- **권한** 서버 전용 GitHub App token + 적절한 권한
- **명세** 문서 제안
- **요청 본문** message?
- **대표 성공 응답** GitHub REST response
- **검증·동작** PJOIN API에는 현재 확정된 dismiss 경로가 없습니다. 제품 범위/권한 합의 후 추가.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `404` 리소스 없음 · `422` 입력 검증 실패

#### `PUT https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}/merge` · GitHub PR 병합

head SHA 조건을 포함해 원격 merge를 요청합니다.

- **권한** 서버 전용 GitHub App token + Pull requests write (필요 Contents 권한 검토)
- **명세** 문서 제안
- **요청 본문** sha=expectedHeadSha, merge_method?
- **대표 성공 응답** GitHub merge response
- **검증·동작** GitHub branch protection이 최종 판정. 요구되는 권한은 실제 endpoint 문서와 저장소 정책으로 확인.
- **계약 메모** GitHub REST 응답에서 P Join 연동에 사용하는 주요 필드를 발췌했습니다. 전체 스키마와 상태 코드는 GitHub Docs에서 확인할 수 있습니다.
- **오류 응답** `403` 접근 금지 · `404` 리소스 없음 · `405` 병합 불가 · `409` head SHA 불일치 · `422` 입력 검증 실패

#### `GET /api/v1/projects/{projectId}/github/pull-requests` · PR 목록

GitHub 원본을 동기화한 로컬 read model을 페이지 조회합니다.

- **권한** 프로젝트 멤버
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **쿼리** state=open, cursor?, limit=20
- **대표 성공 응답** 200 {success,data:{items,nextCursor},meta:{source,stale}}
- **검증·동작** items는 githubPrNumber,title,status,authorLogin,baseBranch,headBranch,headCommitSha,syncedAt 등을 포함.
- **계약 메모** PR 문서는 data.items / data.nextCursor를, GitHub 통합 기획안의 GH-04 예시는 data 배열과 별도 pageInfo를 사용합니다. 두 응답 형태는 팀 확인이 필요합니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음

#### `GET /api/v1/projects/{projectId}/github/pull-requests/{prNumber}` · PR 상세

본문·브랜치·상태·현재 head SHA를 조회합니다.

- **권한** 프로젝트 멤버
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **대표 성공 응답** 200 PullRequest
- **검증·동작** 프로젝트-저장소-PR 소속 확인. ERD 제안 필드: UNIQUE(repositoryId,githubPrNumber), authorLogin, syncedAt.
- **계약 메모** PR 상세는 저장소 내 GitHub PR 번호로 조회합니다. 응답 필드는 목록 예시와 Notion의 PR 필드 매핑을 참고해 보완했습니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음

#### `POST /api/v1/projects/{projectId}/github/pull-requests` · PR 생성

GitHub REST에 생성한 뒤 성공 응답으로 로컬 upsert합니다.

- **권한** 프로젝트 멤버 + GitHub 권한
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **요청 본문** title, head, base, body?, draft?
- **대표 성공 응답** 201 Created
- **검증·동작** head/base 브랜치 검증. GitHub 성공 전 로컬 성공 저장 금지. 요청 중복 방지.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음 · `422` GitHub 입력 검증 실패 · `502` `GITHUB_UPSTREAM_ERROR` GitHub 요청 실패

#### `PATCH /api/v1/projects/{projectId}/github/pull-requests/{prNumber}` · PR 수정·종료

허용 필드를 GitHub에 반영하고 로컬 read model을 갱신합니다.

- **권한** 프로젝트 멤버 + GitHub 권한
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **요청 본문** 허용 수정 필드; 종료 시 state=closed
- **대표 성공 응답** 200 OK
- **검증·동작** 병합은 전용 merge 경로 사용. 403, 404, 409, 422, 429, upstream 오류.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음 · `409` PR 변경 충돌 · `422` 입력 검증 실패 · `429` 호출 한도 초과 · `502` `GITHUB_UPSTREAM_ERROR` GitHub 요청 실패

#### `GET /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/reviews` · PR 리뷰 목록

동기화된 PR 리뷰 이력을 반환합니다.

- **권한** 프로젝트 멤버
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **대표 성공 응답** 200 {success,data:{items:[Review]}}
- **검증·동작** Review는 githubReviewId, reviewerLogin, status, comment, reviewedCommitSha, submittedAt, syncedAt 포함 제안.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음

#### `GET /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/reviews/{reviewId}` · PR 리뷰 상세

특정 리뷰의 상태·본문·대상 커밋을 조회합니다.

- **권한** 프로젝트 멤버
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **대표 성공 응답** 200 Review
- **검증·동작** PR은 저장소 내 GitHub 번호로, 리뷰는 P Join 내부 리뷰 ID로 식별합니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음

#### `POST /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/reviews` · PR 리뷰 제출

리뷰 의견/승인/변경 요청을 GitHub에 제출합니다.

- **권한** 프로젝트 구성원 + GitHub App 권한
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **요청 본문** event=APPROVE|REQUEST_CHANGES|COMMENT, body
- **대표 성공 응답** 201 Review
- **검증·동작** GitHub 성공 후 githubReviewId 기준 upsert. pending은 제출 완료로 표시하지 않음. 작성자 본인의 승인은 병합 승인에 포함하지 않음.
- **계약 메모** 리뷰 전용 문서 예시에는 event와 body, 통합 기획안 예시에는 commitId가 추가됩니다. 커밋 지정 필수 여부는 확인이 필요합니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음 · `422` 리뷰 입력 검증 오류

#### `GET /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/merge-readiness` · 병합 사전 점검

최신 PR·head·reviews를 동기화해 차단 사유를 확인합니다.

- **권한** 프로젝트 멤버
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **대표 성공 응답** 200 readiness report
- **검증·동작** 캐시된 승인만으로 통과시키지 않습니다. requiredPeerApprovals=1은 예시이며 팀 승인 수·자격 정책 미정.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음

#### `PUT /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/merge` · PR 병합

GitHub 최신 상태와 승인 게이트를 재검증한 뒤 병합합니다.

- **권한** 허용된 프로젝트 멤버 + GitHub 권한
- **명세** 문서 제안
- **관련 화면** S14 · F19
- **요청 본문** expectedHeadSha, mergeMethod?
- **대표 성공 응답** 200 {merged,sha,message}
- **검증·동작** GH-11 상세 기획 기준 PUT. GitHub merge API의 sha 조건에 expectedHeadSha를 전달. 다른 API 초안은 POST를 사용해 충돌 상태. 제품 범위 승인 후 활성화.
- **계약 메모** GH-11의 PUT 계약과 응답 예시입니다. PR 개별 문서에는 내부 UUID 경로와 POST 대안이 있습니다.
- **오류 응답** `401` P Join 인증 필요 · `403` `PROJECT_FORBIDDEN` 프로젝트 권한 부족 · `404` PR 또는 저장소 리소스 없음 · `405` `GITHUB_RULES_NOT_MET` · `409` `HEAD_CHANGED · PEER_REVIEW_REQUIRED`

<a id="replaced-endpoints"></a>

## 대체된 API

| 이전 계약 | 현재 제안 |
| --- | --- |
| `PATCH /api/projects/{projectId}/applications/{applicationId}/status` | `PATCH /api/projects/{projectId}/applications/{applicationId}/decision`<br>`PATCH /api/projects/{projectId}/applications/{applicationId}/cancel` |
| `GET /projects/{projectId}/members/{userId}/summary` | `GET /api/projects/{projectId}/members/{userId}` |
| `GET /api/v1/projects/{projectId}/github/repositories/available` | `GET /api/v1/projects/{projectId}/github/repositories` |
| `GET /api/v1/projects/{projectId}/github/pull-requests/{pullRequestId}` | `GET /api/v1/projects/{projectId}/github/pull-requests/{prNumber}` |
| `GET /api/v1/projects/{projectId}/github/pull-requests/{pullRequestId}/reviews` | `GET /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/reviews` |
| `POST /api/v1/projects/{projectId}/github/pull-requests/{pullRequestId}/reviews` | `POST /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/reviews` |
| `PUT /api/v1/projects/{projectId}/github/pull-requests/{pullRequestId}/merge` | `PUT /api/v1/projects/{projectId}/github/pull-requests/{prNumber}/merge` |

<a id="schema-review"></a>

## ERD 검토

기존 구조는 [07_erd.md](07_erd.md)를 참고합니다. 아래에는 카탈로그에서 검토 중인 모델을 정리합니다.

- **project_tech_stacks** 프로젝트와 기술 카탈로그 사이의 다대다 연결입니다. `(projectId, techStackId)` 복합 PK와 `displayOrder`를 사용합니다. API의 `techStackIds`는 이 관계에 행으로 저장합니다.
- **github_repositories** `githubRepositoryId` UNIQUE, `htmlUrl`, `visibility`, `createdAt/updatedAt`를 둡니다. `projectId`도 UNIQUE로 프로젝트당 연결 하나를 보장합니다. ETag와 토큰은 이 테이블에 저장하지 않습니다.
- **pull_requests** `authorLogin` 필수 스냅샷, `syncedAt`, UNIQUE(`repositoryId`, `githubPrNumber`) 및 상태·갱신시각 인덱스를 둡니다. `authorUserId`는 users FK입니다.
- **pull_request_reviews** `githubReviewId` UNIQUE, `reviewerLogin`, `reviewedCommitSha`, `syncedAt`를 둡니다. `reviewerUserId`는 users FK입니다. 중복 webhook은 원격 ID 기준 upsert합니다.
- **applications** 기존 필드: `applicationId`, `projectId`, `applicantId`, `comment`, `privacyConsentAt`, `status`, `handledByUserId`, `handledAt`, 생성·수정 시각.
- **project_members** 복합 PK `(projectId, userId)`, `role`, `joinedAt`. 역할 값, 팀 탈퇴·강퇴·소유권 이전 정책은 ERD에 없으므로 관련 APIs는 검토 초안입니다.

<a id="open-decisions"></a>

## 팀 합의 항목

- **인증·오류 규칙 (합의 완료, 세부 명세 동기화 필요)** P Join 내부 보호 API의 HttpOnly 세션 쿠키, 상태 변경 시 CSRF·Origin 검증, 401 `UNAUTHENTICATED` / 403 `FORBIDDEN`, 오류 JSON `{error:{code,message,requestId?}}`로 확정했습니다. 카탈로그와 도메인별 기존 예시에 남은 Bearer·평면 오류·기타 오류 봉투 표현은 수정해야 합니다. **성공 응답 JSON 형식** 통일은 별도 미확정 항목입니다.
- **경로 버전 및 회원관리** `/api`, `/api/v1`, 루트 상대 경로가 혼재합니다. Applications·Members의 세부 API는 ERD 기반 초안으로 제안했으며 상태값·역할·승인 권한이 필요합니다.
- **저장소 연결과 교체 정책** 후보 목록은 `/github/repositories`로 정리했습니다. `PUT …/github/repository`로 연결할 때 기존 연결을 교체할 수 있는 조건은 팀 확인이 필요합니다.
- **PR 병합 정책** PR 상세·리뷰·병합 경로는 `{prNumber}`를 사용합니다. GH-11의 PUT과 `expectedHeadSha`를 기준으로, 병합 권한과 허용할 병합 방식을 팀에서 정해야 합니다.
- **Projects 스키마·상태** `version`, `deletedAt`, 상태 전이 및 모집 기간과 lifecycleStatus 관계는 추가 제안입니다. 모집 상태를 종료 상태와 분리할지 합의가 필요합니다.
- **ERD와 API 필드 불일치** Users API의 `avatarUrl`은 ERD의 `profile_image_key`로 만든 공개 URL입니다. `websiteUrl`과 `portfolio_url`의 매핑은 확정이 필요합니다. Notifications API는 `message`·`targetUrl`을 쓰지만 ERD는 `actorId`·`targetType`·`targetId`·`expiresAt`을 둡니다. S11 Schedule은 `scheduledAt/link`, ERD는 `startsAt/endsAt/meetingUrl`입니다.
- **ERD 관계와 보존 정책** 프로젝트당 GitHub 저장소 연결은 하나입니다. disconnect 후 PR/리뷰 데이터 보존, 승인 인원·자격, applications 중복 지원 제약, 멤버/owner 전환 및 연관 데이터 삭제를 확정해야 합니다.
- **GitHub 초대 상태 보존** 초대 상태는 GitHub API에서 실시간 조회하며 ERD에 별도 초대 테이블을 두지 않습니다. 장기 이력이 필요해지면 `project_members` 확장을 별도로 검토합니다.
- **REST·GraphQL 및 캐시** REST를 기본으로 하고 GraphQL은 측정된 병목에서만 검토합니다. ETag/If-None-Match와 304, `Retry-After` 우선·제한된 지수 backoff, webhook HMAC·delivery 중복 방지를 적용합니다. SHA-256은 cache key 용도, HMAC-SHA256은 webhook 서명 검증 용도이며 서로 대체할 수 없습니다.
- **응답 필드·삭제·만료 정책** Notifications API의 읽음 전용 삭제와 30일 만료, 프로젝트 논리 삭제, disconnected 저장소의 데이터 보존 및 Schedules 물리 삭제 규칙을 실제 FK/운영 정책과 맞춰야 합니다.
