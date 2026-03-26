# Keycloak Admin API - Multi-Tenant 통합 패턴 참조

---

## 목차

1. [TenantContext & TenantRealmResolver](#tenant-routing)
2. [KeycloakAdminConfig (동적 realm 지원)](#keycloak-admin-config)
3. [RealmProvisioningService (테넌트 온보딩)](#realm-provisioning)
4. [TenantKeycloakSyncService (단방향 Push)](#unidirectional-sync)
5. [WebClient 직접 호출 방식](#webclient)
6. [토큰 관리 전략](#token-management)
7. [Python 멀티테넌트 패턴](#python)
8. [curl 스크립트](#curl)

---

## 1. TenantContext & TenantRealmResolver {#tenant-routing}

### TenantContext (ThreadLocal 기반)

```java
public final class TenantContext {

    private static final ThreadLocal<String> TENANT_ID = new ThreadLocal<>();

    private TenantContext() {}

    public static void set(String tenantId) {
        TENANT_ID.set(tenantId);
    }

    public static String get() {
        String tenantId = TENANT_ID.get();
        if (tenantId == null) {
            throw new IllegalStateException("TenantContext is not set for current thread");
        }
        return tenantId;
    }

    public static void clear() {
        TENANT_ID.remove();
    }
}
```

### TenantRealmResolver

```java
@Component
public class TenantRealmResolver {

    private static final String REALM_PREFIX = "tenant-";

    /**
     * 현재 TenantContext에서 Keycloak Realm 이름을 결정한다.
     * Realm 이름 규칙: "tenant-{tenantId}" (소문자, 하이픈 허용)
     */
    public String resolveCurrentRealm() {
        return toRealmName(TenantContext.get());
    }

    /**
     * tenantId → realm name 변환
     * 예: "Corp-ABC" → "tenant-corp-abc"
     */
    public String toRealmName(String tenantId) {
        return REALM_PREFIX + tenantId.toLowerCase().replaceAll("[^a-z0-9-]", "-");
    }

    /**
     * realm name → tenantId 역변환
     * 예: "tenant-corp-abc" → "corp-abc"
     */
    public String toTenantId(String realmName) {
        if (!realmName.startsWith(REALM_PREFIX)) {
            throw new IllegalArgumentException("Invalid realm name: " + realmName);
        }
        return realmName.substring(REALM_PREFIX.length());
    }
}
```

### Spring MVC Filter: X-Tenant-ID 헤더에서 TenantContext 설정

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TenantContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        try {
            HttpServletRequest httpRequest = (HttpServletRequest) request;
            String tenantId = extractTenantId(httpRequest);
            if (tenantId != null) {
                TenantContext.set(tenantId);
            }
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear(); // 반드시 ThreadLocal 정리
        }
    }

    private String extractTenantId(HttpServletRequest request) {
        // 우선순위 1: X-Tenant-ID 헤더
        String header = request.getHeader("X-Tenant-ID");
        if (header != null) return header;

        // 우선순위 2: JWT의 tenantId claim
        // Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        // if (auth instanceof JwtAuthenticationToken jwt) {
        //     return jwt.getToken().getClaimAsString("tenantId");
        // }

        return null;
    }
}
```

---

## 2. KeycloakAdminConfig (동적 realm 지원) {#keycloak-admin-config}

`Keycloak` 인스턴스 1개로 모든 테넌트 Realm을 관리한다.
`.realm(dynamicRealmName)` 호출로 런타임에 대상 Realm을 결정한다.

### 의존성 (Gradle)

```kotlin
implementation("org.keycloak:keycloak-admin-client:26.0.0") // 서버 버전과 일치시킬 것
```

### Config

```java
@Configuration
public class KeycloakAdminConfig {

    /**
     * Keycloak 인스턴스는 master realm으로 초기화.
     * 실제 조작 대상 realm은 keycloak.realm(realmName)으로 런타임에 지정.
     */
    @Bean
    public Keycloak keycloakAdminClient(
        @Value("${keycloak.admin.server-url}") String serverUrl,
        @Value("${keycloak.admin.client-id}") String clientId,
        @Value("${keycloak.admin.client-secret}") String clientSecret
    ) {
        return KeycloakBuilder.builder()
            .serverUrl(serverUrl)
            .realm("master")                    // 토큰 발급용 고정 realm
            .clientId(clientId)
            .clientSecret(clientSecret)
            .grantType("client_credentials")
            .build();
    }
}
```

### application.yml

```yaml
keycloak:
  admin:
    server-url: http://localhost:8080
    client-id: admin-service-client
    client-secret: ${KEYCLOAK_ADMIN_SECRET}
# keycloak.target-realm 설정 없음 — TenantContext에서 동적으로 결정
```

---

## 3. RealmProvisioningService (테넌트 온보딩) {#realm-provisioning}

테넌트가 처음 생성될 때 1회 실행되는 프로비저닝 서비스.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class RealmProvisioningService {

    private final Keycloak keycloak;
    private final TenantRealmResolver realmResolver;

    // 테넌트 온보딩 시 기본으로 생성할 Role 목록
    private static final List<String> DEFAULT_ROLES = List.of(
        "ROLE_ADMIN", "ROLE_AGENT", "ROLE_SUPERVISOR"
    );

    /**
     * 테넌트 Realm 전체 프로비저닝 (Realm + Client + Role)
     * Idempotent: 이미 존재하면 skip
     */
    public void provisionTenantRealm(String tenantId, String displayName) {
        String realmName = realmResolver.toRealmName(tenantId);
        log.info("Provisioning Keycloak realm: {} for tenant: {}", realmName, tenantId);

        createRealm(realmName, displayName);
        createDefaultClient(realmName, tenantId);
        createDefaultRoles(realmName);

        log.info("Realm provisioning complete: {}", realmName);
    }

    private void createRealm(String realmName, String displayName) {
        try {
            RealmRepresentation realm = new RealmRepresentation();
            realm.setRealm(realmName);
            realm.setDisplayName(displayName);
            realm.setEnabled(true);
            realm.setRegistrationAllowed(false);
            realm.setLoginWithEmailAllowed(true);
            realm.setDuplicateEmailsAllowed(false);
            realm.setPasswordPolicy("length(8) and digits(1) and upperCase(1)");
            realm.setAccessTokenLifespan(1800);          // 30분
            realm.setSsoSessionMaxLifespan(36000);       // 10시간

            keycloak.realms().create(realm);
            log.info("Created realm: {}", realmName);
        } catch (jakarta.ws.rs.ClientErrorException e) {
            if (e.getResponse().getStatus() == 409) {
                log.warn("Realm already exists, skipping: {}", realmName);
            } else {
                throw new RealmProvisioningException("Failed to create realm: " + realmName, e);
            }
        }
    }

    private void createDefaultClient(String realmName, String tenantId) {
        try {
            ClientRepresentation client = new ClientRepresentation();
            client.setClientId("app-" + tenantId);        // 테넌트별 고유 clientId
            client.setName("Application Client");
            client.setEnabled(true);
            client.setServiceAccountsEnabled(true);       // service account 활성화
            client.setAuthorizationServicesEnabled(false);
            client.setPublicClient(false);
            client.setStandardFlowEnabled(true);
            client.setDirectAccessGrantsEnabled(false);
            client.setRedirectUris(List.of("*"));         // 운영 시 구체적 URI로 제한

            try (Response response = keycloak.realm(realmName).clients().create(client)) {
                if (response.getStatus() == 201) {
                    log.info("Created client 'app-{}' in realm: {}", tenantId, realmName);
                } else if (response.getStatus() == 409) {
                    log.warn("Client already exists, skipping: app-{}", tenantId);
                }
            }
        } catch (Exception e) {
            throw new RealmProvisioningException("Failed to create client in realm: " + realmName, e);
        }
    }

    private void createDefaultRoles(String realmName) {
        for (String roleName : DEFAULT_ROLES) {
            try {
                RoleRepresentation role = new RoleRepresentation();
                role.setName(roleName);
                role.setDescription("Default role: " + roleName);
                keycloak.realm(realmName).roles().create(role);
                log.info("Created role '{}' in realm: {}", roleName, realmName);
            } catch (jakarta.ws.rs.ClientErrorException e) {
                if (e.getResponse().getStatus() == 409) {
                    log.warn("Role already exists, skipping: {} in {}", roleName, realmName);
                } else {
                    throw new RealmProvisioningException(
                        "Failed to create role: " + roleName + " in realm: " + realmName, e);
                }
            }
        }
    }

    /**
     * 테넌트 Realm 비활성화 (소프트 삭제 권장)
     */
    public void disableTenantRealm(String tenantId) {
        String realmName = realmResolver.toRealmName(tenantId);
        RealmRepresentation realm = keycloak.realm(realmName).toRepresentation();
        realm.setEnabled(false);
        keycloak.realm(realmName).update(realm);
        log.info("Disabled realm: {}", realmName);
    }

    /**
     * 테넌트 Realm 영구 삭제 (복구 불가 — 주의)
     */
    public void deleteTenantRealm(String tenantId) {
        String realmName = realmResolver.toRealmName(tenantId);
        keycloak.realm(realmName).remove();
        log.info("Deleted realm: {}", realmName);
    }
}
```

---

## 4. TenantKeycloakSyncService (단방향 Push) {#unidirectional-sync}

애플리케이션 이벤트 발생 시 Keycloak으로 Push하는 서비스.
Keycloak → 앱 방향의 이벤트/콜백은 포함하지 않는다.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class TenantKeycloakSyncService {

    private final Keycloak keycloak;
    private final TenantRealmResolver realmResolver;

    // ─────────────────────────────────────────────
    // 사용자 Push 작업
    // ─────────────────────────────────────────────

    /**
     * 사용자 생성 Push
     * 애플리케이션에서 사용자가 가입/생성될 때 호출
     */
    public String pushCreateUser(String tenantId, UserSyncCommand command) {
        String realmName = realmResolver.toRealmName(tenantId);

        UserRepresentation user = new UserRepresentation();
        user.setUsername(command.username());
        user.setEmail(command.email());
        user.setFirstName(command.firstName());
        user.setLastName(command.lastName());
        user.setEnabled(true);
        user.setEmailVerified(false);

        if (command.temporaryPassword() != null) {
            CredentialRepresentation credential = new CredentialRepresentation();
            credential.setType(CredentialRepresentation.PASSWORD);
            credential.setValue(command.temporaryPassword());
            credential.setTemporary(true);
            user.setCredentials(List.of(credential));
        }

        // Attributes: 앱 내부 userId 등 메타데이터 저장
        user.setAttributes(Map.of("appUserId", List.of(command.appUserId())));

        try (Response response = keycloak.realm(realmName).users().create(user)) {
            if (response.getStatus() == 201) {
                String location = response.getHeaderString("Location");
                String keycloakUserId = location.substring(location.lastIndexOf('/') + 1);
                log.info("User created in KC realm={}, keycloakUserId={}", realmName, keycloakUserId);
                return keycloakUserId;
            }
            if (response.getStatus() == 409) {
                // 이미 존재하면 조회 후 ID 반환 (Idempotent)
                log.warn("User already exists in KC, fetching existing: {}", command.username());
                return findKeycloakUserId(realmName, command.username());
            }
            throw new KeycloakSyncException("Failed to create user in KC, status: " + response.getStatus());
        }
    }

    /**
     * 사용자 정보 업데이트 Push
     * 애플리케이션에서 프로필 변경 시 호출
     */
    public void pushUpdateUser(String tenantId, String keycloakUserId, UserUpdateCommand command) {
        String realmName = realmResolver.toRealmName(tenantId);

        UserRepresentation user = keycloak.realm(realmName).users()
            .get(keycloakUserId).toRepresentation();

        if (command.email() != null)     user.setEmail(command.email());
        if (command.firstName() != null) user.setFirstName(command.firstName());
        if (command.lastName() != null)  user.setLastName(command.lastName());

        keycloak.realm(realmName).users().get(keycloakUserId).update(user);
        log.info("User updated in KC realm={}, userId={}", realmName, keycloakUserId);
    }

    /**
     * 비밀번호 변경 Push
     * 애플리케이션에서 비밀번호 변경 이벤트 발생 시 호출
     */
    public void pushResetPassword(String tenantId, String keycloakUserId,
                                  String newPassword, boolean temporary) {
        String realmName = realmResolver.toRealmName(tenantId);

        CredentialRepresentation credential = new CredentialRepresentation();
        credential.setType(CredentialRepresentation.PASSWORD);
        credential.setValue(newPassword);
        credential.setTemporary(temporary);

        keycloak.realm(realmName).users()
            .get(keycloakUserId)
            .resetPassword(credential);

        log.info("Password reset pushed to KC realm={}, userId={}", realmName, keycloakUserId);
    }

    /**
     * 계정 활성화/비활성화 Push
     */
    public void pushSetUserEnabled(String tenantId, String keycloakUserId, boolean enabled) {
        String realmName = realmResolver.toRealmName(tenantId);

        UserRepresentation user = keycloak.realm(realmName).users()
            .get(keycloakUserId).toRepresentation();
        user.setEnabled(enabled);

        keycloak.realm(realmName).users().get(keycloakUserId).update(user);
        log.info("User {} in KC realm={}, userId={}",
            enabled ? "enabled" : "disabled", realmName, keycloakUserId);
    }

    /**
     * 강제 로그아웃 Push (모든 세션 종료)
     */
    public void pushLogoutUser(String tenantId, String keycloakUserId) {
        String realmName = realmResolver.toRealmName(tenantId);
        keycloak.realm(realmName).users().get(keycloakUserId).logout();
        log.info("User sessions terminated in KC realm={}, userId={}", realmName, keycloakUserId);
    }

    // ─────────────────────────────────────────────
    // Role Push 작업
    // ─────────────────────────────────────────────

    /**
     * Role 할당 Push
     */
    public void pushAssignRole(String tenantId, String keycloakUserId, String roleName) {
        String realmName = realmResolver.toRealmName(tenantId);

        RoleRepresentation role = keycloak.realm(realmName).roles()
            .get(roleName).toRepresentation();

        keycloak.realm(realmName).users()
            .get(keycloakUserId)
            .roles().realmLevel()
            .add(List.of(role));

        log.info("Role '{}' assigned in KC realm={}, userId={}", roleName, realmName, keycloakUserId);
    }

    /**
     * Role 해제 Push
     */
    public void pushRevokeRole(String tenantId, String keycloakUserId, String roleName) {
        String realmName = realmResolver.toRealmName(tenantId);

        RoleRepresentation role = keycloak.realm(realmName).roles()
            .get(roleName).toRepresentation();

        keycloak.realm(realmName).users()
            .get(keycloakUserId)
            .roles().realmLevel()
            .remove(List.of(role));

        log.info("Role '{}' revoked in KC realm={}, userId={}", roleName, realmName, keycloakUserId);
    }

    // ─────────────────────────────────────────────
    // 내부 유틸
    // ─────────────────────────────────────────────

    private String findKeycloakUserId(String realmName, String username) {
        return keycloak.realm(realmName).users()
            .searchByUsername(username, true)
            .stream().findFirst()
            .map(UserRepresentation::getId)
            .orElseThrow(() -> new KeycloakSyncException("User not found in KC: " + username));
    }
}
```

### Command Records

```java
public record UserSyncCommand(
    String appUserId,
    String username,
    String email,
    String firstName,
    String lastName,
    String temporaryPassword  // null이면 비밀번호 미설정
) {}

public record UserUpdateCommand(
    String email,
    String firstName,
    String lastName
) {}
```

---

## 5. WebClient 직접 호출 방식 {#webclient}

`keycloak-admin-client` 라이브러리 없이 직접 HTTP 호출하는 방식.
Reactor/WebFlux 환경이거나 라이브러리 의존성을 피할 때 사용.

```java
@Component
@RequiredArgsConstructor
public class KeycloakAdminWebClient {

    private final WebClient webClient;
    private final KeycloakTokenManager tokenManager;     // 단일 캐시 (아래 참조)
    private final TenantRealmResolver realmResolver;

    @Value("${keycloak.admin.server-url}")
    private String serverUrl;

    // 사용자 생성 (Reactive)
    public Mono<String> createUser(String tenantId, UserRepresentation userRepresentation) {
        String realmName = realmResolver.toRealmName(tenantId);
        String uri = serverUrl + "/admin/realms/" + realmName + "/users";

        return tokenManager.getValidToken()
            .flatMap(token -> webClient.post()
                .uri(uri)
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(userRepresentation)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, response ->
                    response.bodyToMono(String.class)
                        .flatMap(body -> Mono.error(new KeycloakSyncException(
                            "KC API error " + response.statusCode() + ": " + body)))
                )
                .toBodilessEntity()
                .map(entity -> {
                    String location = entity.getHeaders().getFirst(HttpHeaders.LOCATION);
                    return location != null
                        ? location.substring(location.lastIndexOf('/') + 1)
                        : "";
                })
            );
    }

    // 사용자 조회 (Reactive)
    public Mono<List<UserRepresentation>> findUsers(String tenantId, String username) {
        String realmName = realmResolver.toRealmName(tenantId);

        return tokenManager.getValidToken()
            .flatMap(token -> webClient.get()
                .uri(serverUrl + "/admin/realms/" + realmName + "/users",
                    uriBuilder -> uriBuilder
                        .queryParam("username", username)
                        .queryParam("exact", true)
                        .build())
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .retrieve()
                .bodyToMono(new ParameterizedTypeReference<List<UserRepresentation>>() {})
            );
    }
}
```

---

## 6. 토큰 관리 전략 {#token-management}

**핵심**: master Realm의 서비스 계정 토큰은 1개만 캐싱한다.
대상 Realm이 여러 개여도 admin 토큰은 단일 인스턴스로 모든 Realm에 사용 가능하다.

### Spring Boot (Blocking)

```java
@Component
public class KeycloakTokenManager {

    private final RestTemplate restTemplate;
    private final AtomicReference<CachedToken> cachedToken = new AtomicReference<>();

    @Value("${keycloak.admin.server-url}")
    private String serverUrl;

    @Value("${keycloak.admin.client-id}")
    private String clientId;

    @Value("${keycloak.admin.client-secret}")
    private String clientSecret;

    public KeycloakTokenManager(RestTemplateBuilder builder) {
        this.restTemplate = builder.build();
    }

    public synchronized String getValidToken() {
        CachedToken current = cachedToken.get();
        if (current != null && current.isValid()) {
            return current.token();
        }
        TokenResponse newToken = fetchNewToken();
        cachedToken.set(new CachedToken(
            newToken.accessToken(),
            Instant.now().plusSeconds(newToken.expiresIn())
        ));
        return newToken.accessToken();
    }

    /** 401 응답 시 강제 무효화 후 재발급 */
    public synchronized void invalidate() {
        cachedToken.set(null);
    }

    private TokenResponse fetchNewToken() {
        String tokenUrl = serverUrl + "/realms/master/protocol/openid-connect/token";

        MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
        body.add("grant_type", "client_credentials");
        body.add("client_id", clientId);
        body.add("client_secret", clientSecret);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

        ResponseEntity<TokenResponse> response = restTemplate.postForEntity(
            tokenUrl, new HttpEntity<>(body, headers), TokenResponse.class);

        return Objects.requireNonNull(response.getBody());
    }

    private record CachedToken(String token, Instant expiresAt) {
        boolean isValid() {
            return Instant.now().isBefore(expiresAt.minus(Duration.ofSeconds(30)));
        }
    }

    public record TokenResponse(
        @JsonProperty("access_token") String accessToken,
        @JsonProperty("expires_in") long expiresIn
    ) {}
}
```

### WebFlux (Reactive) — Mono.cache() 활용

```java
@Component
public class ReactiveKeycloakTokenManager {

    private final WebClient webClient;
    private volatile Mono<String> cachedTokenMono = Mono.empty();
    private volatile Instant tokenExpiry = Instant.EPOCH;

    @Value("${keycloak.admin.server-url}") private String serverUrl;
    @Value("${keycloak.admin.client-id}") private String clientId;
    @Value("${keycloak.admin.client-secret}") private String clientSecret;

    public Mono<String> getValidToken() {
        if (Instant.now().isBefore(tokenExpiry.minus(Duration.ofSeconds(30)))) {
            return cachedTokenMono;
        }
        return refreshToken();
    }

    private synchronized Mono<String> refreshToken() {
        // double-check after acquiring lock
        if (Instant.now().isBefore(tokenExpiry.minus(Duration.ofSeconds(30)))) {
            return cachedTokenMono;
        }
        Mono<String> newToken = webClient.post()
            .uri(serverUrl + "/realms/master/protocol/openid-connect/token")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .body(BodyInserters.fromFormData("grant_type", "client_credentials")
                .with("client_id", clientId)
                .with("client_secret", clientSecret))
            .retrieve()
            .bodyToMono(KeycloakTokenManager.TokenResponse.class)
            .doOnNext(resp -> tokenExpiry = Instant.now().plusSeconds(resp.expiresIn()))
            .map(KeycloakTokenManager.TokenResponse::accessToken)
            .cache();

        this.cachedTokenMono = newToken;
        return newToken;
    }
}
```

---

## 7. Python 멀티테넌트 패턴 {#python}

```python
from keycloak import KeycloakAdmin, KeycloakOpenIDConnection


class KeycloakMultiTenantAdmin:
    """테넌트별 Realm을 동적으로 대상으로 하는 Admin 클라이언트"""

    REALM_PREFIX = "tenant-"

    def __init__(self, server_url: str, client_id: str, client_secret: str):
        self._connection = KeycloakOpenIDConnection(
            server_url=server_url,
            realm_name="master",
            client_id=client_id,
            client_secret_key=client_secret,
            verify=True
        )

    def to_realm_name(self, tenant_id: str) -> str:
        return self.REALM_PREFIX + tenant_id.lower().replace("_", "-")

    def _admin(self, tenant_id: str) -> KeycloakAdmin:
        """테넌트 ID를 받아 해당 Realm을 대상으로 하는 Admin 클라이언트 반환"""
        admin = KeycloakAdmin(connection=self._connection)
        admin.realm_name = self.to_realm_name(tenant_id)
        return admin

    # ─── Realm 프로비저닝 ───────────────────────────────

    def provision_tenant_realm(self, tenant_id: str, display_name: str) -> None:
        realm_name = self.to_realm_name(tenant_id)
        try:
            master_admin = KeycloakAdmin(connection=self._connection)
            master_admin.create_realm({
                "realm": realm_name,
                "displayName": display_name,
                "enabled": True,
                "registrationAllowed": False,
                "passwordPolicy": "length(8) and digits(1)"
            })
            print(f"Realm created: {realm_name}")
        except Exception as e:
            if "Conflict" in str(e):
                print(f"Realm already exists, skipping: {realm_name}")
            else:
                raise

        # 기본 Role 생성
        admin = self._admin(tenant_id)
        for role_name in ["ROLE_ADMIN", "ROLE_AGENT", "ROLE_SUPERVISOR"]:
            try:
                admin.create_realm_role({"name": role_name})
            except Exception:
                pass  # 이미 존재하면 skip

    # ─── 사용자 Push 작업 ──────────────────────────────

    def push_create_user(self, tenant_id: str, username: str, email: str,
                         first_name: str, last_name: str,
                         temporary_password: str | None = None) -> str:
        admin = self._admin(tenant_id)

        payload = {
            "username": username,
            "email": email,
            "firstName": first_name,
            "lastName": last_name,
            "enabled": True,
        }
        if temporary_password:
            payload["credentials"] = [{
                "type": "password",
                "value": temporary_password,
                "temporary": True
            }]

        user_id = admin.create_user(payload)
        print(f"User created: tenant={tenant_id}, userId={user_id}")
        return user_id

    def push_assign_role(self, tenant_id: str, user_id: str, role_name: str) -> None:
        admin = self._admin(tenant_id)
        role = admin.get_realm_role(role_name)
        admin.assign_realm_roles(user_id=user_id, roles=[role])

    def push_disable_user(self, tenant_id: str, user_id: str) -> None:
        admin = self._admin(tenant_id)
        admin.update_user(user_id, {"enabled": False})
```

---

## 8. curl 스크립트 {#curl}

### 멀티테넌트 Realm 프로비저닝 스크립트

```bash
#!/bin/bash
set -e

KC_HOST="http://localhost:8080"
ADMIN_CLIENT_ID="admin-service-client"
ADMIN_CLIENT_SECRET="YOUR_SECRET"

# Admin 토큰 발급 (master realm)
TOKEN=$(curl -s -X POST \
  "${KC_HOST}/realms/master/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=${ADMIN_CLIENT_ID}" \
  -d "client_secret=${ADMIN_CLIENT_SECRET}" \
  | jq -r '.access_token')

# 함수: 테넌트 Realm 생성
provision_tenant() {
  local TENANT_ID="$1"
  local DISPLAY_NAME="$2"
  local REALM_NAME="tenant-${TENANT_ID}"

  echo "Provisioning realm: ${REALM_NAME}"

  # 1. Realm 생성
  curl -s -X POST "${KC_HOST}/admin/realms" \
    -H "Authorization: Bearer ${TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{
      \"realm\": \"${REALM_NAME}\",
      \"displayName\": \"${DISPLAY_NAME}\",
      \"enabled\": true,
      \"passwordPolicy\": \"length(8) and digits(1)\"
    }"

  # 2. 기본 Role 생성
  for ROLE in "ROLE_ADMIN" "ROLE_AGENT" "ROLE_SUPERVISOR"; do
    curl -s -X POST "${KC_HOST}/admin/realms/${REALM_NAME}/roles" \
      -H "Authorization: Bearer ${TOKEN}" \
      -H "Content-Type: application/json" \
      -d "{\"name\": \"${ROLE}\"}"
    echo "Role created: ${ROLE}"
  done

  echo "Provisioning complete: ${REALM_NAME}"
}

# 사용 예시
provision_tenant "corp-abc" "Corp ABC"
provision_tenant "startup-xyz" "Startup XYZ"
```

### 테넌트별 사용자 Push 예시

```bash
#!/bin/bash
TENANT_ID="corp-abc"
REALM_NAME="tenant-${TENANT_ID}"

# 사용자 생성 Push
curl -s -X POST "${KC_HOST}/admin/realms/${REALM_NAME}/users" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john.doe",
    "email": "john@corp-abc.com",
    "firstName": "John",
    "lastName": "Doe",
    "enabled": true,
    "credentials": [{"type": "password", "value": "Temp1234!", "temporary": true}]
  }'

# 사용자 ID 조회
USER_ID=$(curl -s \
  "${KC_HOST}/admin/realms/${REALM_NAME}/users?username=john.doe&exact=true" \
  -H "Authorization: Bearer ${TOKEN}" \
  | jq -r '.[0].id')

# Role 할당 Push
ROLE_ID=$(curl -s \
  "${KC_HOST}/admin/realms/${REALM_NAME}/roles/ROLE_AGENT" \
  -H "Authorization: Bearer ${TOKEN}" \
  | jq -r '.id')

curl -s -X POST \
  "${KC_HOST}/admin/realms/${REALM_NAME}/users/${USER_ID}/role-mappings/realm" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "[{\"id\": \"${ROLE_ID}\", \"name\": \"ROLE_AGENT\"}]"

echo "User pushed to KC: realm=${REALM_NAME}, userId=${USER_ID}"
```
