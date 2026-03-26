---
name: keycloak-admin-api
description: >
  Keycloak Admin REST API 연동을 도와주는 스킬. 특히 테넌트(Tenant)마다 Realm을 분리하는
  Realm-per-Tenant 멀티테넌트 아키텍처, 애플리케이션에서 Keycloak으로의 단방향 데이터 Push 패턴,
  동적 Realm 라우팅, Realm 프로비저닝, 사용자/클라이언트/역할 관리를 빠르게 구현할 수 있도록 안내한다.
  사용자가 "Keycloak", "Admin API", "realm 설정", "멀티테넌트", "테넌트 realm", "tenant realm",
  "realm 생성", "사용자 관리", "client 등록", "토큰 발급", "KeycloakAdminClient",
  "keycloak-admin-client", "Spring Security OAuth2 Keycloak", "RestTemplate Keycloak",
  "WebClient Keycloak", "Keycloak 연동", "단방향 동기화", "Keycloak Push" 등을 언급할 때
  반드시 이 스킬을 사용한다.
  Java(Spring Boot), Python, curl 등 다양한 환경에서의 Admin API 통합 패턴을 제공한다.
---

# Keycloak Admin API - Multi-Tenant 연동 스킬

## 핵심 설계 원칙

1. **Realm-per-Tenant**: 테넌트마다 독립된 Realm을 생성 → 데이터 완전 격리
2. **단방향 Push**: 애플리케이션 → Keycloak 방향만 존재 (Keycloak 이벤트/콜백 수신 없음)
3. **동적 Realm 라우팅**: `TenantContext`에서 런타임에 Realm 이름 결정 (`@Value` 고정값 금지)
4. **Admin 토큰 단일 캐시**: `master` Realm의 서비스 계정 1개로 모든 테넌트 Realm 관리
5. **Idempotent 프로비저닝**: `409 Conflict` 방어로 중복 생성 안전 처리
6. **OpenAPI Spec 우선 참조**: `https://www.keycloak.org/docs-api/latest/rest-api/openapi.json`

---

## 아키텍처 개요

```
┌─────────────────────────────────────┐
│         Application (MSA)           │
│                                     │
│  TenantContext  ──▶  realmName 결정  │
│       │                             │
│  KeycloakAdminRouter                │
│  (단방향 Push: Admin API 호출)       │
└──────────────┬──────────────────────┘
               │  HTTP (Admin REST API)
               ▼
    ┌──────────────────────┐
    │    Keycloak Server   │
    │  ┌────────────────┐  │
    │  │  master realm  │  │  ← Admin 토큰 발급용 (직접 사용자 관리 금지)
    │  └────────────────┘  │
    │  ┌──────┐  ┌──────┐  │
    │  │ T001 │  │ T002 │  │  ← 테넌트별 Realm
    │  │realm │  │realm │  │     (사용자/역할/클라이언트 완전 격리)
    │  └──────┘  └──────┘  │
    └──────────────────────┘
```

**Realm 이름 규칙**: `tenant-{tenantId}` (예: `tenant-corp-abc`, `tenant-00123`)
- `master`와 충돌 방지를 위해 반드시 접두사 사용
- tenantId는 URL-safe 문자만 허용 (소문자, 숫자, 하이픈)

---

## Step 0 - Realm 프로비저닝 (테넌트 온보딩 시 1회 실행)

새 테넌트가 등록될 때 Realm을 생성하고 기본 리소스를 구성하는 흐름.

```
테넌트 생성 이벤트
      │
      ▼
1. Realm 생성          POST /admin/realms
2. 기본 Client 등록    POST /admin/realms/{realm}/clients
3. 기본 Role 생성      POST /admin/realms/{realm}/roles
4. (선택) 패스워드 정책 PUT  /admin/realms/{realm}
```

```http
POST /admin/realms
Authorization: Bearer {master-admin-token}
Content-Type: application/json

{
  "realm": "tenant-corp-abc",
  "displayName": "Corp ABC",
  "enabled": true,
  "registrationAllowed": false,
  "loginWithEmailAllowed": true,
  "duplicateEmailsAllowed": false,
  "passwordPolicy": "length(8) and digits(1)"
}
```

→ 상세 Java 구현 (RealmProvisioningService): [`references/integration-patterns.md`](references/integration-patterns.md)

---

## Step 1 - Admin 토큰 획득 (master Realm, 단일 캐시)

모든 테넌트 Realm 관리에 `master` Realm의 서비스 계정 토큰 1개를 사용한다.
토큰은 인스턴스 단위로 1개만 캐싱하며, 만료 30초 전 자동 갱신.

```bash
# master realm의 service account로 admin 토큰 발급
curl -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=my-admin-client" \
  -d "client_secret=YOUR_CLIENT_SECRET"
```

> **핵심 구분**: 토큰 발급 Realm은 항상 `master`. 조작 대상 Realm은 API 경로의 `{realm}` 으로 분리됨.

→ 토큰 캐시 구현: [`references/integration-patterns.md`](references/integration-patterns.md)

---

## Step 2 - 동적 Realm 라우팅

### 안티패턴 vs 올바른 패턴

```java
// ❌ 안티패턴 (고정 realm) — 멀티테넌트에서 사용 불가
@Value("${keycloak.target-realm}")
private String targetRealm;

// ✅ 올바른 패턴 (동적 realm)
String realmName = TenantRealmResolver.resolveCurrentRealm();
// → "tenant-corp-abc"
keycloak.realm(realmName).users().create(userRepresentation);
```

### Realm 이름 결정 방법 (우선순위 순)

| 방법 | 설명 | 사용 시나리오 |
|------|------|-------------|
| JWT Claim | `tenantId` claim에서 추출 | API 요청 처리 중 |
| Request Header | `X-Tenant-ID` 헤더 | 서비스 간 내부 통신 |
| ThreadLocal | `TenantContext.get()` | 비즈니스 로직 레이어 |
| 메서드 파라미터 | `syncUser(tenantId, user)` | 명시적 배치/이벤트 처리 |

→ TenantRealmResolver + TenantContext 구현: [`references/integration-patterns.md`](references/integration-patterns.md)

---

## Step 3 - 단방향 Push 패턴 (App → Keycloak)

애플리케이션 이벤트가 발생할 때 Keycloak으로만 Push한다.
Keycloak 이벤트 리스너/콜백을 앱에서 수신하는 방향은 이 스킬의 범위 밖이다.

### Push 트리거 매핑

| 애플리케이션 이벤트 | Keycloak 작업 | API |
|------------------|-------------|-----|
| 테넌트 생성 | Realm 프로비저닝 | `POST /admin/realms` |
| 사용자 가입 | 사용자 생성 | `POST /admin/realms/{realm}/users` |
| 사용자 정보 수정 | 사용자 업데이트 | `PUT /admin/realms/{realm}/users/{id}` |
| 비밀번호 변경 | 비밀번호 리셋 | `PUT /admin/realms/{realm}/users/{id}/reset-password` |
| 권한 부여 | Role 할당 | `POST /admin/realms/{realm}/users/{id}/role-mappings/realm` |
| 권한 회수 | Role 해제 | `DELETE /admin/realms/{realm}/users/{id}/role-mappings/realm` |
| 계정 비활성화 | 사용자 disable | `PUT /admin/realms/{realm}/users/{id}` (`enabled: false`) |
| 강제 로그아웃 | 세션 종료 | `POST /admin/realms/{realm}/users/{id}/logout` |
| 테넌트 비활성화 | Realm disable | `PUT /admin/realms/{realm}` (`enabled: false`) |
| 테넌트 삭제 | Realm 삭제 | `DELETE /admin/realms/{realm}` |

→ TenantKeycloakSyncService 구현: [`references/integration-patterns.md`](references/integration-patterns.md)

---

## Step 4 - API 리소스 그룹

자세한 내용은 → [`references/api-groups.md`](references/api-groups.md)

| 그룹 | 베이스 경로 | 주요 용도 |
|------|------------|---------|
| Realms Admin | `POST /admin/realms` | **Realm 생성** (테넌트 온보딩) |
| Users | `/admin/realms/{realm}/users` | 사용자 CRUD, 비밀번호, 역할 할당 |
| Clients | `/admin/realms/{realm}/clients` | Client 등록/조회/수정 |
| Roles | `/admin/realms/{realm}/roles` | Realm 역할 관리 |
| Groups | `/admin/realms/{realm}/groups` | 그룹 관리 및 멤버십 |
| Identity Providers | `/admin/realms/{realm}/identity-provider` | IdP 연동 |

> `{realm}` 은 항상 `TenantContext`에서 동적으로 결정된 값을 사용한다.

---

## Step 5 - 에러 처리

| HTTP 상태 | 원인 | 처리 방법 |
|-----------|------|---------|
| 401 | 토큰 만료 | 토큰 강제 무효화 → 재발급 → 1회 재시도 |
| 403 | 권한 부족 | master realm 서비스 계정에 `realm-management` 역할 확인 |
| 404 | Realm 없음 | 테넌트 온보딩 프로비저닝 완료 여부 점검 |
| 409 | 중복 리소스 | Idempotent 처리: 존재하면 skip 또는 update로 전환 |
| 400 | 잘못된 요청 | realm name 대문자/특수문자 포함 여부, 필수 필드 확인 |

---

## 개발 환경 주의사항

- **Realm name**: URL 경로에 사용되므로 소문자 + 하이픈만 허용. UUID와 다름
- **`master` Realm 보호**: 직접 사용자 생성/관리 금지. 오직 admin 토큰 발급용
- **OpenAPI Spec 오류**: 자동 클라이언트 생성 시 `--skip-validate-spec` 필요 (keycloak/keycloak#40400)
- **Pagination**: 목록 API는 `first` (offset), `max` (page size) 쿼리 파라미터 지원
- **테넌트 Realm 삭제**: 복구 불가. 삭제 전 `enabled: false` 상태 경유 권장
