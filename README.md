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
| ORM | MyBatis + JPA (혼용) |
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

공공 SI 특성상 외부 GitHub을 쓸 수 없어 Synology NAS에 내부 Git 서버를 구성했고, 배포 대상은 별도 Windows 운영 서버였습니다.
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

**핵심 문제: Windows 환경에서 프로세스 격리**

Linux와 달리 Windows에서는 PID 파일 방식이 불안정해 구버전 JAR를 정확히 종료하기 어려웠습니다.
WMIC로 JAR 파일명 기준으로 프로세스를 특정해 종료하는 방식으로 해결했습니다.

```bash
# 구버전 프로세스 종료
wmic process where "commandline like '%GH-DisasterSafe%'" delete

# 신버전 실행
start javaw -jar GH-DisasterSafe.jar
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

**예외 처리 표준화**

`ErrorCode` enum으로 에러 코드와 HTTP 상태를 한 곳에서 관리하고,
`@RestControllerAdvice`에서 예외 종류별로 일괄 처리하는 구조를 잡았습니다.
팀원들은 `BusinessException`만 throw하면 일관된 응답이 내려갑니다.

```java
// 비즈니스 예외는 ErrorCode와 함께 throw
@ExceptionHandler(BusinessException.class)
public ResponseEntity<Result<Void>> handleBusiness(BusinessException e, HttpServletRequest request) {
    return build(e.getCode(), e.getMessage(), e, request);
}

// API vs 웹 요청 구분 처리 — 404도 요청 유형에 따라 다르게 응답
@ExceptionHandler(NoResourceFoundException.class)
public ResponseEntity<Result<Void>> handleNoResourceFound(NoResourceFoundException e, HttpServletRequest request) throws NoResourceFoundException {
    if (isApiRequest(request)) {
        return build(ErrorCode.NOT_FOUND, "요청한 페이지를 찾을 수 없습니다", e, request);
    }
    throw e; // 웹 요청은 Spring Boot 기본 에러 페이지로 위임
}
```

API 요청과 일반 웹 요청을 URL 패턴과 Accept 헤더로 구분해,
REST 응답과 HTML 에러 페이지를 각각 내려주도록 처리한 점이 포인트였습니다.

---

### 4. OZ Report 개발 서버 구축 및 웹 연동

**배경**

공공 프로젝트에서 보고서 출력은 OZ Report가 표준입니다.
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

### 5. 보건관리 엑셀 파싱 → DB 일괄 적재

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

**병합 셀 처리**

실무 엑셀에는 병합 셀이 많아 단순 `row.getCell(col)` 로는 빈 값으로 읽히는 문제가 있었습니다.
`getMergedAwareCell()`로 병합 영역을 검사해 첫 번째 셀의 값을 읽도록 처리했습니다.

**한계**

전체 시트를 메모리에 올려 파싱하는 구조라 수만 건 이상이면 OOM 위험이 있습니다.
대용량 처리가 필요하면 Apache POI의 SAX 방식(streaming) 또는 Spring Batch 청크 처리로 교체해야 합니다.

---

## 배운 점 / 개선하고 싶은 것

초기 공통 모듈을 잡지 않았다면 팀원마다 예외 처리와 로그 형식이 제각각인 코드가 됐을 것입니다.
규칙을 초반에 정해두는 것이 나중에 합칠 때 충돌을 얼마나 줄여주는지 직접 경험했습니다.

Jenkins 구성 전 수동 배포 중 구버전이 남은 채로 서비스가 뜨는 사고를 겪었습니다.
그 뒤로 배포 프로세스를 코드로 관리하는 것이 편의가 아니라 신뢰성의 문제라는 걸 알게 됐습니다.

| 항목 | 현재 | 개선 방향 |
|---|---|---|
| 배포 환경 | Windows + WMIC | Linux + Docker |
| 배포 방식 | JAR 직접 실행 | Blue-Green 무중단 배포 |
| 분산 트랜잭션 | 미적용 | JTA (Atomikos) |
| 테스트 | 미작성 | JUnit5 + Mockito |

Windows + WMIC 방식은 동작은 하지만 처음부터 Linux + Docker 환경이었다면 훨씬 안정적인 구조가 됐을 것 같습니다.
다음 프로젝트에서는 컨테이너 환경을 전제로 설계하고 싶습니다.
