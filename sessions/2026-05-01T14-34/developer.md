# 💻 Developer — MVP의 핵심 기능 구현을 위한 기술 스택(Frontend/Backend)을 선정하고, 데이터베이스 스키마 및 초기 API 구조에 대한 기술적 청사진을 작성합니다.

💻 Developer: 작업 시작합니다.

## 🛠️ 기술 청사진: Collect 플랫폼 MVP 구현 방안

CEO님의 지시에 따라 MVP의 핵심 기능(모금 및 아카이빙)을 구현하기 위한 기술 스택 선정, 데이터베이스 스키마, 초기 API 구조에 대한 기술적 청사진을 제시합니다.

### 1. 기술 스택 선정 (Technology Stack Selection)

관계 기반의 데이터 무결성과 빠른 개발 속도를 고려하여 다음과 같은 스택을 제안합니다.

*   **Backend (서버):** **Node.js (NestJS)**
    *   *선정 이유:* 비동기 처리 능력 우수, TypeScript 기반으로 안정적인 타입 시스템 제공, API 개발에 용이하며, 향후 확장성(Microservices)에 유리함.
*   **Frontend (클라이언트):** **React (Next.js)**
    *   *선정 이유:* 컴포넌트 기반으로 복잡한 UI/UX 구현에 적합하며, 대규모 SPA(Single Page Application) 구축에 용이함. Next.js를 사용하여 서버 사이드 렌더링(SSR)을 통해 초기 로딩 속도와 SEO를 확보합니다.
*   **Database (데이터베이스):** **PostgreSQL**
    *   *선정 이유:* 관계형 데이터(Relationship, Funding)의 무결성을 보장하는 데 최적화되어 있으며, 복잡한 관계 그래프(Relationship Graph)를 관리하는 데 강력합니다.

---

### 2. 데이터베이스 스키마 설계 (Database Schema Design)

핵심 엔티티인 `User`, `Collection`, `Contribution`, `Archive` 간의 관계를 중심으로 스키마를 설계합니다. (PostgreSQL 기준)

#### 2.1. `users` 테이블 (사용자 정보)
| 필드명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `user_id` | UUID | PK | 사용자 고유 ID |
| `username` | VARCHAR(50) | UNIQUE, NOT NULL | 사용자 이름 |
| `email` | VARCHAR(100) | UNIQUE, NOT NULL | 이메일 주소 |
| `profile_data` | JSONB | | 프로필 상세 정보 (선택적) |

#### 2.2. `collections` 테이블 (모금 캠페인 정보)
| 필드명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `collection_id` | UUID | PK | 캠페인 고유 ID |
| `creator_id` | UUID | FK (`users.user_id`) | 캠페인 생성자 |
| `title` | VARCHAR(255) | NOT NULL | 캠페인 제목 |
| `goal_amount` | NUMERIC(10, 2) | NOT NULL | 목표 금액 |
| `start_date` | TIMESTAMP | NOT NULL | 캠페인 시작일 |
| `end_date` | TIMESTAMP | NOT NULL | 마감일 |
| `status` | ENUM | NOT NULL | (Draft, Active, Completed, Archived) |

#### 2.3. `contributions` 테이블 (기여 내역)
| 필드명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `contribution_id` | UUID | PK | 기여 고유 ID |
| `collection_id` | UUID | FK (`collections.collection_id`) | 연결된 캠페인 |
| `contributor_id` | UUID | FK (`users.user_id`) | 기여자 |
| `amount` | NUMERIC(10, 2) | NOT NULL | 기여 금액 |
| `contribution_type` | VARCHAR(50) | | 기여 유형 (금액, 아이템, 경험) |
| `contribution_date` | TIMESTAMP | NOT NULL | 기여 시점 |
| `status` | ENUM | NOT NULL | (Pending, Approved, Rejected) |

#### 2.4. `archives` 테이블 (추억 아카이빙)
| 필드명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `archive_id` | UUID | PK | 아카이빙 기록 고유 ID |
| `collection_id` | UUID | FK (`collections.collection_id`) | 연결된 캠페인 |
| `story_title` | VARCHAR(255) | NOT NULL | 아카이빙 스토리 제목 |
| `content` | TEXT | NOT NULL | 상세 추억 기록 (관계 중심 스토리) |
| `related_ids` | JSONB | | 관련 기여자/이벤트 ID 목록 (관계 그래프) |
| `archived_at` | TIMESTAMP | DEFAULT NOW() | 아카이빙 완료 시점 |

---

### 3. 초기 API 구조 (Initial API Structure)

RESTful API 디자인을 기반으로 핵심 기능에 필요한 엔드포인트를 정의합니다. (Base URL: `/api/v1`)

#### 3.1. 사용자 인증 및 프로필 (Authentication & Profile)

| Method | Endpoint | 설명 | 권한 |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/register` | 신규 사용자 등록 | Public |
| `POST` | `/auth/login` | 사용자 로그인 및 JWT 발급 | Public |
| `GET` | `/users/me` | 현재 로그인된 사용자 정보 조회 | Private |
| `PUT` | `/users/me` | 사용자 프로필 업데이트 | Private |

#### 3.2. 모금 기능 (Funding Module Endpoints)

| Method | Endpoint | 설명 | 권한 |
| :--- | :--- | :--- | :--- |
| `POST` | `/collections` | 새로운 모금 캠페인 생성 (F-01) | Private (Creator) |
| `GET` | `/collections/{id}` | 특정 캠페인 상세 정보 조회 | Public |
| `POST` | `/collections/{id}/contribute` | 특정 캠페인에 기여 내역 추가 (F-02) | Private (Contributor) |
| `GET` | `/collections/{id}/contributions` | 특정 캠페인의 모든 기여 내역 조회 | Private |

#### 3.3. 아카이빙 기능 (Archiving Module Endpoints)

| Method | Endpoint | 설명 |
