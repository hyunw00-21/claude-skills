# Keycloak Admin API - 리소스 그룹 상세 참조

> Base URL: `{keycloak-host}/admin/realms/{realm}`
> 공식 OpenAPI: https://www.keycloak.org/docs-api/latest/rest-api/openapi.json

---

## 목차

1. [Users](#users)
2. [Clients](#clients)
3. [Roles](#roles)
4. [Groups](#groups)
5. [Realms Admin](#realms-admin)
6. [Identity Providers](#identity-providers)
7. [Client Scopes](#client-scopes)
8. [Role Mappings](#role-mappings)
9. [Attack Detection](#attack-detection)
10. [Organizations](#organizations)
11. [Authentication Management](#authentication-management)
12. [Protocol Mappers](#protocol-mappers)

---

## Users

```
GET    /users                         # 사용자 목록 (쿼리: username, email, firstName, lastName, search, exact, first, max)
POST   /users                         # 사용자 생성
GET    /users/{userId}                # 특정 사용자 조회
PUT    /users/{userId}                # 사용자 수정
DELETE /users/{userId}                # 사용자 삭제
GET    /users/{userId}/sessions       # 활성 세션 목록
DELETE /users/{userId}/sessions       # 모든 세션 종료 (강제 로그아웃)
GET    /users/{userId}/role-mappings  # 부여된 역할 조회
POST   /users/{userId}/role-mappings/realm          # Realm 역할 할당
DELETE /users/{userId}/role-mappings/realm          # Realm 역할 해제
POST   /users/{userId}/role-mappings/clients/{clientId}  # Client 역할 할당
PUT    /users/{userId}/reset-password                # 비밀번호 설정/변경
PUT    /users/{userId}/execute-actions-email         # 이메일 액션 전송 (VERIFY_EMAIL 등)
GET    /users/{userId}/credentials    # 자격증명 목록
DELETE /users/{userId}/credentials/{credentialId}   # 자격증명 삭제
GET    /users/{userId}/groups         # 사용자 그룹 조회
PUT    /users/{userId}/groups/{groupId}  # 그룹에 사용자 추가
DELETE /users/{userId}/groups/{groupId} # 그룹에서 사용자 제거
GET    /users/count                   # 사용자 수 조회
```

### 사용자 목록 쿼리 파라미터

| 파라미터 | 설명 | 예시 |
|---------|------|------|
| `username` | username 필터 (부분 일치, exact=true면 정확 일치) | `username=john` |
| `email` | email 필터 | `email=john@example.com` |
| `search` | username, email, firstName, lastName 통합 검색 | `search=john` |
| `exact` | 정확 일치 여부 | `exact=true` |
| `enabled` | 활성화 여부 필터 | `enabled=true` |
| `first` | 페이지 오프셋 | `first=0` |
| `max` | 페이지 크기 (기본 100, 최대 권장 500) | `max=50` |
| `briefRepresentation` | 간략 응답 (id, username만) | `briefRepresentation=true` |

---

## Clients

```
GET    /clients                       # Client 목록 (쿼리: clientId, search, first, max)
POST   /clients                       # Client 생성
GET    /clients/{client-uuid}         # 특정 Client 조회 (UUID로 조회)
PUT    /clients/{client-uuid}         # Client 수정
DELETE /clients/{client-uuid}         # Client 삭제
GET    /clients/{client-uuid}/client-secret    # Client Secret 조회
POST   /clients/{client-uuid}/client-secret    # Client Secret 재생성
GET    /clients/{client-uuid}/roles            # Client 역할 목록
POST   /clients/{client-uuid}/roles            # Client 역할 생성
GET    /clients/{client-uuid}/roles/{role-name} # 특정 역할 조회
PUT    /clients/{client-uuid}/roles/{role-name} # 역할 수정
DELETE /clients/{client-uuid}/roles/{role-name} # 역할 삭제
GET    /clients/{client-uuid}/service-account-user  # Service Account User 조회
GET    /clients/{client-uuid}/user-sessions    # 클라이언트 활성 세션 수
```

> **주의**: `/clients` 경로의 `{id}`는 내부 UUID이지 `clientId`가 아님.
> clientId로 먼저 UUID를 조회한 후 사용:
> `GET /clients?clientId=my-app&search=false` → 응답의 `id` 필드 사용

### 핵심 ClientRepresentation 필드

```json
{
  "clientId": "my-service",
  "name": "My Service",
  "enabled": true,
  "protocol": "openid-connect",
  "publicClient": false,
  "bearerOnly": false,
  "serviceAccountsEnabled": true,
  "standardFlowEnabled": false,
  "directAccessGrantsEnabled": false,
  "secret": "...",
  "redirectUris": ["http://localhost:8080/*"],
  "webOrigins": ["+"]
}
```

---

## Roles

```
GET    /roles                         # Realm 역할 목록
POST   /roles                         # Realm 역할 생성
GET    /roles/{role-name}             # 역할 조회 (name으로)
PUT    /roles/{role-name}             # 역할 수정
DELETE /roles/{role-name}             # 역할 삭제
GET    /roles/{role-name}/users       # 역할이 부여된 사용자 목록
GET    /roles/{role-name}/composites  # Composite 역할 목록
POST   /roles/{role-name}/composites  # Composite 역할 추가
GET    /roles-by-id/{role-id}         # 역할 조회 (ID로)
```

---

## Groups

```
GET    /groups                        # 그룹 목록 (쿼리: search, first, max)
POST   /groups                        # 최상위 그룹 생성
GET    /groups/{groupId}              # 그룹 조회
PUT    /groups/{groupId}              # 그룹 수정
DELETE /groups/{groupId}              # 그룹 삭제
POST   /groups/{groupId}/children     # 하위 그룹 생성
GET    /groups/{groupId}/members      # 그룹 멤버 목록
GET    /groups/{groupId}/role-mappings  # 그룹 역할 조회
POST   /groups/{groupId}/role-mappings/realm  # 그룹에 Realm 역할 할당
GET    /groups/count                  # 그룹 수 조회
```

---

## Realms Admin

```
GET    /admin/realms                  # 모든 Realm 목록
POST   /admin/realms                  # Realm 생성
GET    /admin/realms/{realm}          # Realm 조회
PUT    /admin/realms/{realm}          # Realm 수정
DELETE /admin/realms/{realm}          # Realm 삭제
GET    /admin/realms/{realm}/events   # 이벤트 조회 (쿼리: type, client, user, first, max)
DELETE /admin/realms/{realm}/events   # 이벤트 삭제
GET    /admin/realms/{realm}/admin-events  # Admin 이벤트 조회
POST   /admin/realms/{realm}/logout-all    # 전체 세션 종료
POST   /admin/realms/{realm}/partial-import  # 리소스 부분 임포트
```

> **주의**: Realm 목록은 `/admin/realms`이지 `/admin/realms/{realm}/realms`가 아님

---

## Identity Providers

```
GET    /identity-provider/instances          # IdP 목록
POST   /identity-provider/instances          # IdP 생성
GET    /identity-provider/instances/{alias}  # IdP 조회
PUT    /identity-provider/instances/{alias}  # IdP 수정
DELETE /identity-provider/instances/{alias}  # IdP 삭제
GET    /identity-provider/instances/{alias}/mappers  # IdP 매퍼 목록
POST   /identity-provider/instances/{alias}/mappers  # IdP 매퍼 생성
GET    /identity-provider/providers/{providerId}    # IdP 프로바이더 정보
```

---

## Client Scopes

```
GET    /client-scopes                       # Client Scope 목록
POST   /client-scopes                       # Client Scope 생성
GET    /client-scopes/{scopeId}             # Client Scope 조회
PUT    /client-scopes/{scopeId}             # Client Scope 수정
DELETE /client-scopes/{scopeId}             # Client Scope 삭제
GET    /default-default-client-scopes       # 기본 Default Client Scopes
PUT    /default-default-client-scopes/{scopeId}    # Default Client Scope 추가
DELETE /default-default-client-scopes/{scopeId}    # Default Client Scope 제거
GET    /clients/{client-uuid}/default-client-scopes # Client의 기본 스코프
PUT    /clients/{client-uuid}/default-client-scopes/{scopeId} # Client 스코프 추가
```

---

## Attack Detection (Brute Force)

```
DELETE /attack-detection/brute-force/users            # 전체 사용자 로그인 실패 기록 초기화
GET    /attack-detection/brute-force/users/{userId}   # 특정 사용자 브루트포스 상태 조회
DELETE /attack-detection/brute-force/users/{userId}   # 특정 사용자 잠금 해제
```

---

## Organizations (v25+)

> **Preview 기능** — Keycloak 25.0.0 이상에서 사용 가능

```
GET    /organizations                 # 조직 목록
POST   /organizations                 # 조직 생성
GET    /organizations/{orgId}         # 조직 조회
PUT    /organizations/{orgId}         # 조직 수정
DELETE /organizations/{orgId}         # 조직 삭제
GET    /organizations/{orgId}/members # 조직 멤버 목록
POST   /organizations/{orgId}/members # 멤버 추가
DELETE /organizations/{orgId}/members/{userId} # 멤버 제거
POST   /organizations/{orgId}/invitations  # 초대 생성
```

---

## Authentication Management

```
GET    /authentication/flows          # 인증 플로우 목록
POST   /authentication/flows          # 인증 플로우 생성
GET    /authentication/flows/{flowAlias}/executions  # 플로우 실행 단계 목록
PUT    /authentication/flows/{flowAlias}/executions  # 플로우 실행 단계 수정
GET    /authentication/required-actions    # Required Action 목록
PUT    /authentication/required-actions/{alias}  # Required Action 수정
```

---

## Protocol Mappers

```
GET    /clients/{client-uuid}/protocol-mappers/models  # 클라이언트 매퍼 목록
POST   /clients/{client-uuid}/protocol-mappers/models  # 매퍼 생성
GET    /client-scopes/{scopeId}/protocol-mappers/models  # 스코프 매퍼 목록
POST   /client-scopes/{scopeId}/protocol-mappers/models  # 스코프 매퍼 생성
```

---

## Pagination 공통 패턴

모든 목록 API에서 공통으로 사용:

```
GET /users?first=0&max=50     # 1페이지 (0~49번째)
GET /users?first=50&max=50    # 2페이지 (50~99번째)
GET /users/count              # 전체 수 확인 후 페이지 계산
```
