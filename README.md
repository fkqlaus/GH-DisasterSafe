# 재난안전 통합 플랫폼 (GH-DisasterSafe)

> 고흥군 재난·안전 데이터를 통합 관제하는 공공 플랫폼입니다.
> 초기 프로젝트 설정, Jenkins CI/CD 파이프라인 구축, 공통 모듈 전체 설계, 다수 도메인 기능 개발을 담당했습니다.

**개발 기간**: 2024.11 ~ 진행 중
**개발 규모**: 팀 프로젝트
**운영 여부**: 개발 중

---

## 기술 스택

| 구분 | 기술 |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.x, Spring Security, Spring WebSocket |
| ORM | MyBatis |
| DB | PostgreSQL (멀티 데이터소스) |
| Infra | Jenkins CI/CD, Linux, Synology NAS (내부 Git 서버) |
| 리포팅 | OZ Report 9.0 |
| 엑셀 파싱 | Apache POI |
| 빌드 | Gradle |

---

## 담당 역할 및 기여 범위

- **초기 프로젝트 구성** 전체 (멀티 데이터소스, 공통 모듈, 패키지 구조 설계)
- **Jenkins CI/CD 파이프라인 구축** (내부 Git 서버 → SSH 배포 자동화)
- **공통 모듈 전체 설계 및 구현** (AOP, Config, Events, Exceptions, Security, Utilities)
- **OZ Report 개발 서버 구축 및 웹 연동** (환경 구성 ~ 실 서비스 연동 전 과정)
- **보건관리 엑셀 파싱 → DB 적재** (Apache POI 기반 데이터 일괄 처리)
- 계정관리 / 로그인·로그아웃 / 대시보드
- 위험도 평가 / 전담기관 관리 / 건강관리 / 근로자 의견수렴 / 아카이브
- **관리자 페이지 / 마이페이지 / 중대재해 관리 현황 페이지**

---

## 기술적 도전과 해결 과정

---

### 1. Jenkins CI/CD — 내부 Git 서버 기반 배포 자동화

**배경**

회사 방침상 외부 GitHub을 쓸 수 없어 Synology NAS에 내부 Git 서버가 구성되어 있었고, 배포 대상은 별도 Windows 개발 서버였습니다.
기존에는 빌드 파일을 직접 복사하는 수동 배포 방식이어서 배포 누락, 구버전 잔존 문제가 반복됐습니다.

**구성**

```
Synology NAS (내부 Git 서버)
↓  push 이벤트
Jenkins (빌드 서버)
↓  Gradle 빌드 → JAR 생성
↓  SSH 전송
Windows 배포 서버
↓  구버전 프로세스 종료 (WMIC)
↓  신버전 JAR 실행
```

**결과**

Git push 하나로 빌드·배포가 완료되는 구조로 전환했고, 배포 오류 발생 시 Jenkins 로그에서 바로 추적할 수 있게 됐습니다.

---

### 2. 공통 모듈 설계 — 팀 협업을 위한 코드 표준화

**배경**

팀 프로젝트에서 각자 기능을 개발하면 예외 처리, 로그 형식, 인증 로직이 제각각이 됩니다.
초기 설정 담당으로 이 문제를 사전에 막기 위해 공통 모듈을 설계했습니다.

**구성**

```
common
├── aop         # 공통 로깅 (AccessLogAspect, LoggingAspect)
├── config      # Security, Web, Async, Redis, WebClient 설정
├── event       # Spring 이벤트 기반 비동기 처리 (접근로그, 파일 삭제)
├── exception   # BusinessException + ErrorCode + GlobalExceptionHandler
├── excel       # 엑셀 파싱 공통 엔진 (Strategy 패턴)
├── mybatis     # TypeHandler (Role, ApprovalStatus Enum 매핑)
├── response    # 공통 응답 구조 (Result<T>, PageResponse)
├── security    # CustomUserDetails, SecurityConfig, 커스텀 PasswordEncoder
└── util        # FileUtil, FieldCryptoUtil, FileCryptoUtil
```

---

### 3. OZ Report 개발 서버 구축 및 웹 연동

**배경**

프로젝트에서 보고서 출력을 외부 OZ Report를 사용했습니다.
단순 API 호출이 아니라 OZ Report 서버 구성부터 Spring Boot 연동까지 전 과정을 직접 담당했습니다.

**진행한 작업**

```
1. OZ Report 9.0 개발 서버 설치 및 구성
2. PostgreSQL JDBC 드라이버 연동 (vendor=user 방식)
3. OZ Report Designer로 보고서 양식(.ozr) 설계
4. Spring Boot ↔ OZ Report 서버 연동 (파라미터 전달, 보고서 렌더링)
5. 웹 화면에서 보고서 조회/출력 기능 구현
```

**트러블슈팅 — JDBC 드라이버 인식 문제**

OZ Report가 PostgreSQL JDBC 드라이버를 자동 인식하지 못했습니다.
`vendor=user` 방식으로 드라이버 클래스와 URL을 직접 명시해 해결했습니다.
OZ Report 레퍼런스가 거의 없어 공식 매뉴얼과 직접 디버깅으로만 해결해야 했습니다.

---

### 4. 보건관리 엑셀 파싱 → DB 일괄 적재

**배경**

보건 데이터가 엑셀로 관리되고 있었고 수작업 입력은 오류 가능성이 높아, 업로드 → 파싱 → DB 저장까지 자동화했습니다.
엑셀 양식이 시트별로 달랐고, 이후 다른 도메인에서도 엑셀 업로드 기능이 추가될 가능성이 있어
단순 for 루프가 아닌 확장 가능한 구조로 설계했습니다.

**구조 — Strategy 패턴 기반 엑셀 파싱 엔진**

```
ExcelImportEngine          # 진입점: 파일을 열고 SheetImporter에 위임
SheetImporter (interface)  # supports(sheetName), importSheet(sheet, fileId)
AbstractSheetImporter      # 공통 유틸 메서드 (병합셀, 헤더 탐색, 배치 저장)
└── HealthSheetImporter    # 보건 시트 파싱 구체 구현체
└── (향후 다른 시트 추가 가능)
```

`ExcelImportEngine`이 SheetImporter 목록을 주입받아 시트명 또는 타입 기준으로 적합한 구현체를 찾아 처리합니다.
새 양식이 추가될 때 `AbstractSheetImporter`를 상속한 구현체만 추가하면 엔진 코드는 변경이 없습니다.

```java
@Transactional
public void importExcel(MultipartFile file, Long excelFileId, String type, String password) {
    // 타입 기반(페이지별 업로드) 또는 시트명 기반(기존 방식) 자동 분기
    SheetImporter importer = (type != null)
            ? findImporterByType(type)
            : findImporterBySheetName(sheetName);

    importer.importSheet(sheet, excelFileId);
}
```

---

### 5. GS 인증 결함 대응 — 보안 취약점 수정 및 HTTPS 구성

**배경**

TTA(한국정보통신기술협회) GS 인증 심사 과정에서 결함 리포트를 수령하고,
해당 항목들을 직접 수정해 재심사를 통과시켰습니다.

**결함 항목 및 해결 과정**

---

**① 비밀번호 평문 전송**

로그인 시 비밀번호가 평문으로 네트워크에 전송되어 와이어샤크 등 패킷 분석 도구로 탈취 가능한 상태였습니다.

서버 기동 시 RSA-2048 키쌍을 `@PostConstruct`로 1회 생성하고 공개키를 클라이언트에 제공,
클라이언트는 Web Crypto API로 비밀번호를 RSA-OAEP(SHA-256)로 암호화해 전송하는 구조로 해결했습니다.

```java
// 서버 기동 시 RSA-2048 키쌍 1회 생성
@PostConstruct
public void init() {
    KeyPairGenerator kpg = KeyPairGenerator.getInstance("RSA");
    kpg.initialize(2048, new SecureRandom());
    KeyPair keyPair = kpg.generateKeyPair();
    this.privateKey = keyPair.getPrivate();
    this.publicKeyBase64 = Base64.getEncoder().encodeToString(keyPair.getPublic().getEncoded());
}
```

`RsaDecryptionFilter`가 Spring Security 인증 흐름 앞에서 `encryptedPassword` 파라미터를 복호화해 `password`로 교체하고,
암호화되지 않은 평문 로그인 시도는 필터 단계에서 차단합니다.

```java
// 암호화 파라미터 없으면 즉시 차단
if (encryptedPassword == null || encryptedPassword.isBlank()) {
    response.sendRedirect("/login?error=true");
    return;
}
```

---

**② 비밀번호 암호화 알고리즘 — Argon2 → SHA-256 + Salt**

기존에 Argon2를 사용하고 있었는데, TTA 심사 기준인 KISA 소프트웨어 개발보안 가이드에서
허용하는 단방향 알고리즘(SHA-256, SHA-512 등)이 아니라는 결함이 접수됐습니다.

Spring Security의 `PasswordEncoder` 인터페이스를 구현해 SHA-256 + SecureRandom Salt 방식의 인코더를 직접 작성했습니다.
저장 형식은 `{salt}:{hash}`로 분리해 검증 시 동일한 Salt로 재계산 후 비교합니다.

```java
// 저장 형식: {Base64(salt)}:{Base64(SHA-256(salt + password))}
@Override
public String encode(CharSequence rawPassword) {
    byte[] salt = new byte[16];
    RANDOM.nextBytes(salt);
    byte[] hash = digest(salt, rawPassword.toString());
    return Base64.getEncoder().encodeToString(salt) + ":" + Base64.getEncoder().encodeToString(hash);
}

@Override
public boolean matches(CharSequence rawPassword, String encodedPassword) {
    String[] parts = encodedPassword.split(":", 2);
    byte[] salt = Base64.getDecoder().decode(parts[0]);
    byte[] expectedHash = Base64.getDecoder().decode(parts[1]);
    byte[] actualHash = digest(salt, rawPassword.toString());
    return MessageDigest.isEqual(expectedHash, actualHash); // 타이밍 공격 방지
}
```

---

**③ GET 방식 비밀번호 전달**

URL 파라미터에 비밀번호가 포함되는 구조를 POST + RSA 암호화 전송 방식으로 전환해 해결했습니다.

---

**④ 필수 입력 항목 표시 미제공**

위험성평가 등록 화면에서 필수 항목임에도 별표(`*`) 등의 시각적 표시가 없던 문제를 UI에서 수정했습니다.

---

**HTTPS 구성**

결함 대응과 함께 HTTPS 환경도 직접 구성했습니다.

```properties
# TLS 1.2 / 1.3만 허용, ECDHE 기반 암호화 스위트 적용
server.ssl.enabled=true
server.ssl.key-store=file:./ssl/keystore.p12
server.ssl.key-store-type=PKCS12
server.ssl.enabled-protocols=TLSv1.2,TLSv1.3
server.ssl.ciphers=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,...

# 세션 쿠키 보안
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.http-only=true
```

프로덕션 환경에서는 키스토어 비밀번호를 환경변수(`${SSL_KEYSTORE_PASSWORD}`)로 분리해 소스코드에 노출되지 않도록 했습니다.

---

