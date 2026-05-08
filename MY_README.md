# 🛠 담당 기능 상세 (My Contributions)

> **SilverLink-BE** 프로젝트에서 담당한 백엔드 기능을 정리한 문서입니다.
> 기간: 2026.01.16 ~ 2026.02.01

---

## 목차

1. [공지사항 기능 (Notice Management)](#1-공지사항-기능-notice-management)
2. [복지시설 맵 기능 (Welfare Facility Map)](#2-복지시설-맵-기능-welfare-facility-map)
3. [파일 관리 기능 (File Management)](#3-파일-관리-기능-file-management)
4. [도메인 구조도](#4-도메인-구조도)

---

## 1. 공지사항 기능 (Notice Management)

### 개요

관리자가 공지사항을 등록·수정·삭제하고, 각 역할(관리자/상담사/보호자/어르신)에 맞는 공지만 조회할 수 있도록 구현한 기능입니다. 중요 공지 상단 고정, 팝업 공지, 필독 확인(읽음 처리), 파일 첨부 등을 지원합니다.

### 주요 기능

- **역할 기반 공지 대상 설정**: 전체 대상(`ALL`) 또는 특정 역할 조합(`ROLE_SET`) 선택 가능
- **중요 공지 우선 정렬**: `isPriority` 플래그로 중요 공지를 상단 고정
- **팝업 공지**: 지정 기간 동안 메인 화면에 팝업으로 노출
- **필독 확인 (읽음 처리)**: 사용자가 "확인" 버튼 클릭 시 읽음 로그 기록
- **Soft Delete**: 삭제 시 물리 삭제가 아닌 상태값(`DELETED`) 변경
- **첨부파일 지원**: 공지사항에 PDF 등 파일 첨부 가능
- **키워드 검색**: 제목/내용 기반 공지사항 검색
- **이전글/다음글 네비게이션**: 상세 조회 시 이전/다음 공지 ID 제공
- **카테고리 분류**: 공지(`NOTICE`), 이벤트(`EVENT`), 뉴스(`NEWS`), 시스템(`SYSTEM`)

### API 엔드포인트

#### 관리자 API (`/api/admin/notices`)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| `GET` | `/api/admin/notices` | 공지사항 목록 조회 (페이징, 최신순 정렬) | 관리자 |
| `POST` | `/api/admin/notices` | 공지사항 등록 | 관리자 |
| `GET` | `/api/admin/notices/{id}` | 공지사항 상세 조회 | 관리자 |
| `PUT` | `/api/admin/notices/{id}` | 공지사항 수정 | 관리자 |
| `DELETE` | `/api/admin/notices/{id}` | 공지사항 삭제 (Soft Delete) | 관리자 |

#### 사용자 API (`/api/notices`)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| `GET` | `/api/notices` | 내 권한에 맞는 공지사항 목록 조회 (키워드 검색 지원) | 로그인 사용자 |
| `GET` | `/api/notices/popups` | 현재 활성 팝업 공지 조회 | 로그인 사용자 |
| `GET` | `/api/notices/{id}` | 공지사항 상세 조회 (이전/다음글 ID 포함) | 로그인 사용자 |
| `POST` | `/api/notices/{id}/read` | 공지사항 필독 확인 (읽음 처리) | 로그인 사용자 |
| `POST` | `/api/notices/{id}/confirm` | 공지사항 필독 확인 (프론트엔드 호환 별칭) | 로그인 사용자 |

### 설계 포인트

- **엔티티 불변성**: Notice 엔티티는 `@Builder` 패턴을 사용하여 수정/삭제 시 새 객체를 반환하는 방식으로 불변성 유지
- **대상 역할 다대다 관계**: `NoticeTargetRole` 중간 테이블을 통해 하나의 공지에 여러 역할을 지정 가능
- **읽음 추적**: `NoticeReadLog` 테이블로 사용자별 읽음 여부 관리
- **Validation**: `@NotBlank`, `@NotNull`, `@Size` 등 Bean Validation 적용

### 관련 파일

```
domain/notice/
├── controller/
│   ├── AdminNoticeController.java    — 관리자용 CRUD API
│   └── NoticeController.java         — 사용자용 조회/읽음확인 API
├── dto/
│   ├── NoticeRequest.java            — 등록/수정 요청 DTO
│   └── NoticeResponse.java           — 응답 DTO (첨부파일, 읽음 여부 포함)
├── entity/
│   ├── Notice.java                   — 공지사항 엔티티 (Soft Delete, 팝업 설정)
│   ├── NoticeCategory.java           — 카테고리 Enum (공지/이벤트/뉴스/시스템)
│   ├── NoticeAttachment.java         — 첨부파일 엔티티
│   ├── NoticeTargetRole.java         — 대상 역할 매핑 엔티티
│   ├── NoticeTargetRoleId.java       — 복합키 클래스
│   └── NoticeReadLog.java            — 읽음 로그 엔티티
├── repository/
│   ├── NoticeRepository.java         — 역할별 조회, 팝업 조회, 이전/다음글 쿼리
│   ├── NoticeAttachmentRepository.java
│   ├── NoticeReadLogRepository.java
│   └── NoticeTargetRoleRepository.java
└── service/
    └── NoticeService.java            — 비즈니스 로직 (323 lines)
```

---

## 2. 복지시설 맵 기능 (Welfare Facility Map)

### 개요

공공데이터포털(국립사회보장정보원)의 전국사회복지시설 데이터를 서울 지역 기준으로 동기화하고, 사용자의 현재 위치(GPS) 기반으로 반경 내 복지시설을 검색하여 지도에 표시하는 기능입니다.

### 주요 기능

- **공공데이터 API 자동 동기화**: 앱 시작 시 DB가 비어있으면 자동으로 공공데이터포털 API를 호출하여 서울 지역 복지시설 데이터 동기화
- **GPS 반경 검색**: 사용자 현재 위치의 위도/경도 기반으로 반경(km) 내 시설 조회 (MySQL `ST_Distance_Sphere` 함수 활용)
- **시설 유형별 분류**: 노인복지관, 장애인복지관, 아동복지관, 종합사회복지관, 경로당, 주간보호센터, 재가복지서비스 등 7가지 유형
- **관리자 시설 관리**: 수동으로 시설 등록/수정/삭제 가능
- **좌표 유효성 검증**: 위도(-90~90), 경도(-180~180) 범위 검증

### API 엔드포인트

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| `GET` | `/api/map/facilities/nearby?lat={위도}&lon={경도}&radius={반경km}` | 반경 내 복지시설 조회 (기본 1km) | 전체 |
| `GET` | `/api/map/facilities/{id}` | 시설 상세 조회 | 전체 |
| `POST` | `/api/map/facilities/sync` | 공공데이터 수동 동기화 | 전체 |
| `GET` | `/api/map/facilities` | 모든 시설 목록 조회 | 관리자(`ADMIN`) |
| `POST` | `/api/map/facilities` | 시설 등록 | 관리자(`ADMIN`) |
| `PUT` | `/api/map/facilities/{id}` | 시설 수정 | 관리자(`ADMIN`) |
| `DELETE` | `/api/map/facilities/{id}` | 시설 삭제 | 관리자(`ADMIN`) |

### 시설 유형 (WelfareFacilityType)

| Enum 값 | 설명 |
|---------|------|
| `ELDERLY_WELFARE_CENTER` | 노인복지관 |
| `DISABLED_WELFARE_CENTER` | 장애인복지관 |
| `CHILD_WELFARE_CENTER` | 아동복지관 |
| `COMMUNITY_WELFARE_CENTER` | 종합사회복지관 |
| `SENIOR_CENTER` | 경로당 |
| `DAYCARE_CENTER` | 주간보호센터 |
| `HOME_CARE_SERVICE` | 재가복지서비스 |

### 설계 포인트

- **`@PostConstruct` 자동 동기화**: 서버 시작 시 DB가 비어있으면 공공데이터를 자동으로 가져와 초기 데이터를 세팅
- **MySQL 공간 함수 활용**: `ST_Distance_Sphere(POINT(), POINT())` 네이티브 쿼리로 정확한 거리 계산
- **Fallback 더미 데이터**: API 키가 없거나 응답이 비어있을 경우 테스트용 더미 데이터를 자동 삽입
- **서울 지역 필터링**: 시도명(`ctprvnNm`), 지번주소(`lnmadr`), 도로명주소(`rdnmadr`)에서 "서울" 키워드로 필터링
- **시설 유형 자동 매핑**: API 응답의 시설종류명(`fcltyKndNm`)을 Enum으로 자동 매핑 (예: "노인" 포함 → `ELDERLY_WELFARE_CENTER`)
- **`@PreAuthorize`**: 관리자 전용 API에 메서드 수준 권한 제어 적용

### 관련 파일

```
domain/map/
├── controller/
│   └── WelfareFacilityController.java — 사용자/관리자 API (반경 검색, CRUD)
├── dto/
│   ├── WelfareFacilityRequest.java    — 등록/수정 요청 DTO (좌표 Validation 포함)
│   ├── WelfareFacilityResponse.java   — 응답 DTO (시설 유형 설명 포함)
│   └── PublicDataResponse.java        — 공공데이터포털 API 응답 매핑 DTO
├── entity/
│   ├── WelfareFacility.java           — 복지시설 엔티티
│   └── WelfareFacilityType.java       — 시설 유형 Enum (7가지)
├── repository/
│   └── WelfareFacilityRepository.java — ST_Distance_Sphere 반경 검색 쿼리
└── service/
    ├── WelfareFacilityService.java    — 시설 CRUD 비즈니스 로직
    └── WelfareDataSyncService.java    — 공공데이터 API 동기화 서비스
```

---

## 3. 파일 관리 기능 (File Management)

### 개요

공지사항 첨부파일 등을 지원하기 위한 범용 파일 업로드/다운로드 서비스입니다. AWS S3와 로컬 파일 시스템을 모두 지원하며, S3 설정 여부에 따라 자동으로 저장소를 전환합니다.

### 주요 기능

- **듀얼 스토리지**: S3 설정 시 AWS S3에 업로드, 미설정 시 로컬 파일 시스템에 자동 전환
- **단일/다중 파일 업로드**: 하나 또는 여러 개의 파일을 동시에 업로드
- **Pre-signed URL 생성**: S3 파일에 대한 1시간 유효 임시 접근 URL 생성
- **파일 다운로드**: S3 또는 로컬에서 파일을 바이트 배열로 다운로드 (한글 파일명 인코딩 지원)
- **파일 삭제**: S3 또는 로컬에서 파일 삭제
- **UUID 기반 파일명**: 업로드 시 UUID로 파일명을 변환하여 충돌 방지, 원본 파일명은 별도 보관
- **Swagger 문서화**: `@Tag`, `@Operation` 어노테이션으로 API 명세 자동 생성

### API 엔드포인트

| Method | Endpoint | 설명 | Content-Type |
|--------|----------|------|-------------|
| `POST` | `/api/files/upload` | 단일 파일 업로드 | `multipart/form-data` |
| `POST` | `/api/files/upload-multiple` | 다중 파일 업로드 | `multipart/form-data` |
| `DELETE` | `/api/files?filePath={경로}` | 파일 삭제 | - |
| `GET` | `/api/files/url?filePath={경로}` | 파일 URL 조회 | - |
| `GET` | `/api/files/download?filePath={경로}&originalFileName={파일명}` | 파일 다운로드 | - |

### 설계 포인트

- **전략 패턴 (Strategy)**: `s3Enabled` 플래그에 따라 S3/로컬 저장소를 자동 전환하는 구조
- **`@Autowired(required = false)`**: S3Client/S3Presigner가 없어도 애플리케이션이 정상 기동되도록 선택적 주입
- **한글 파일명 처리**: `URLEncoder`로 파일명을 인코딩하여 다운로드 시 한글 깨짐 방지
- **S3 URL 파싱**: `s3://bucket/key` 형식과 key 직접 전달 형식 모두 지원

### 관련 파일

```
domain/file/
├── controller/
│   └── FileController.java         — 파일 업로드/다운로드/삭제 API
├── dto/
│   └── FileUploadResponse.java     — 업로드 결과 응답 DTO
└── service/
    └── FileService.java            — S3/로컬 듀얼 스토리지 서비스 (335 lines)
```

---

## 4. 도메인 구조도

### 공지사항 도메인 ERD

```
┌─────────────────────────────┐
│         notices             │
├─────────────────────────────┤
│ notice_id (PK)              │
│ created_by_admin_user_id(FK)│──→ Admin
│ title                       │
│ content (TEXT)               │
│ category (ENUM)              │     NOTICE | EVENT | NEWS | SYSTEM
│ target_mode (ENUM)           │     ALL | ROLE_SET
│ status (ENUM)                │     DRAFT | PUBLISHED | ARCHIVED | DELETED
│ is_priority                  │
│ is_popup                     │
│ popup_start_at               │
│ popup_end_at                 │
│ created_at                   │
│ updated_at                   │
│ deleted_at                   │     ← Soft Delete용
└──────────┬──────────────────┘
           │ 1:N
    ┌──────┴───────┐──────────────────┐
    ▼              ▼                  ▼
┌──────────┐ ┌───────────────┐ ┌──────────────┐
│ notice_  │ │ notice_       │ │ notice_      │
│ target_  │ │ attachments   │ │ read_logs    │
│ roles    │ │               │ │              │
├──────────┤ ├───────────────┤ ├──────────────┤
│notice_id │ │attachment_id  │ │ notice_id    │
│target_   │ │notice_id (FK) │ │ user_id      │
│  role    │ │file_name      │ │ read_at      │
│(복합PK)  │ │original_file_ │ └──────────────┘
└──────────┘ │  name         │
             │file_path      │
             │file_size      │
             └───────────────┘
```

### 복지시설 도메인 ERD

```
┌─────────────────────────────────┐
│       welfare_facilities        │
├─────────────────────────────────┤
│ facility_id (PK)                │
│ name                            │
│ address                         │
│ latitude (Double)               │     ← GPS 위도
│ longitude (Double)              │     ← GPS 경도
│ type (ENUM)                     │     ELDERLY_WELFARE_CENTER
│                                 │     DISABLED_WELFARE_CENTER
│                                 │     CHILD_WELFARE_CENTER
│                                 │     COMMUNITY_WELFARE_CENTER
│                                 │     SENIOR_CENTER
│                                 │     DAYCARE_CENTER
│                                 │     HOME_CARE_SERVICE
│ phone                           │
│ operating_hours                 │
└─────────────────────────────────┘

         ↑ 데이터 출처
         │
┌────────┴────────────────────────┐
│  공공데이터포털 API              │
│  (전국사회복지시설표준데이터)     │
│  → 서울 지역만 필터링 후 저장    │
└─────────────────────────────────┘
```

---

## 관련 브랜치 및 PR

| 기능 | 브랜치 | PR |
|------|--------|-----|
| 공지사항 초기 구현 | `feature/2-공지사항` | [#50](../../pull/50) |
| 공지사항 추가 기능 | `feature/60-공지사항-기능-추가` | [#61](../../pull/61), [#63](../../pull/63) |
| 복지시설 맵 | `feature/95-맵기능-추가` | [#100](../../pull/100), [#101](../../pull/101) |

---

## 기술 스택 (담당 기능 한정)

| 분류 | 기술 |
|------|------|
| Framework | Spring Boot 3.x, Spring Security, Spring Data JPA |
| Database | MySQL 8.x (`ST_Distance_Sphere` 공간 함수) |
| Storage | AWS S3 SDK v2 (`S3Client`, `S3Presigner`) |
| 외부 API | 공공데이터포털 REST API (`RestTemplate`) |
| Validation | Jakarta Bean Validation (`@NotBlank`, `@NotNull`, `@Min`, `@Max`) |
| 문서화 | Springdoc OpenAPI (Swagger UI) |
| 테스트 | JUnit 5, Spring Boot Test |
