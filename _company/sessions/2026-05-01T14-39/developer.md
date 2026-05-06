# 💻 Developer — Designer와 Writer가 확정한 와이어프레임과 콘텐츠를 바탕으로, 웹사이트의 초기 기술 스택(Front-end/Back-end)을 최종 확정하고, API 엔드포인트(Endpoints)를 구체적으로 정의하십시오. 특히, 관계 데이터(Relationship Graph)를 처리하기 위한 데이터 파이프라인의 초기 구조와 필수 DB 테이블 관계를 설계하여 개발 착수 준비를 완료하십시오.

💻 Developer: 작업 시작합니다.

CEO님의 지시사항에 따라, '관계의 온도를 아카이빙'하는 Collect 플랫폼의 초기 기술 스택을 확정하고, 관계 그래프 중심의 데이터 모델링 및 API 청사진을 완성했습니다. 이 설계는 MVP 개발에 필요한 모든 백본(Backbone)을 제공하며, 개발팀이 즉시 코딩을 시작할 수 있는 수준입니다.

---

## 🏗️ 1. 기술 스택 확정 및 설계 근거

| 영역 | 기술 스택 | 선택 이유 |
| :--- | :--- | :--- |
| **Backend (API)** | Python + FastAPI | 비동기 처리와 빠른 API 개발에 최적화되어 있습니다. 데이터 처리 로직(특히 관계 그래프 분석) 구현에 필요한 방대한 라이브러리 생태계가 강력합니다. |
| **Database** | PostgreSQL | 높은 안정성과 복잡한 쿼리 처리 능력을 갖추고 있습니다. 특히 `JSONB` 타입과 **PostGIS (지리정보)**, 그리고 **재귀 CTE (Common Table Expression)**를 활용하여 관계 그래프 모델링의 복잡성을 효과적으로 관리할 수 있습니다. |
| **Frontend** | React (Next.js) | 서버 사이드 렌더링(SSR)을 통해 초기 로딩 성능을 확보하고, 컴포넌트 기반 아키텍처로 복잡한 대시보드와 인터랙티브 그래프 구현에 가장 적합합니다. |
| **Deployment** | Docker / AWS ECS | 개발 환경과 운영 환경의 격리 및 일관성을 보장하여 배포 과정의 리스크를 최소화합니다. |

---

## 💾 2. 데이터베이스 스키마 설계 (PostgreSQL 중심)

가장 핵심적인 '관계 데이터'를 중심으로 테이블을 설계했습니다. 관계는 단순한 FK 연결이 아닌, **속성(Attributes)**을 가진 엔티티로 취급하여 '온도', '맥락', '심리적 가치' 등의 데이터를 기록합니다.

### 📌 핵심 테이블 구조 (Schema Diagram)

```sql
-- 1. User: 사용자 정보
CREATE TABLE User (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    nickname VARCHAR(100) NOT NULL,
    profile_data JSONB, -- 생년월일, 지역 등 추가 정보 저장
    created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Collection: 콜렉(행사/이벤트) 정보 (예: 생일, 결혼)
CREATE TABLE Collection (
    collection_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    host_id UUID REFERENCES User(user_id) ON DELETE CASCADE, -- 콜렉 주최자
    title VARCHAR(255) NOT NULL,
    event_date DATE NOT NULL,
    goal_amount DECIMAL(15, 2) DEFAULT 0.00, -- 모금 목표액
    description TEXT,
    status VARCHAR(50) DEFAULT 'ACTIVE' -- ACTIVE, COMPLETED, PAUSED
);

-- 3. Relationship: 관계 데이터 (가장 중요)
CREATE TABLE Relationship (
    relationship_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_user_id UUID REFERENCES User(user_id) ON DELETE CASCADE, -- 관계 시작점
    target_user_id UUID REFERENCES User(user_id) ON DELETE CASCADE, -- 관계 대상점
    collection_id UUID REFERENCES Collection(collection_id) ON DELETE CASCADE, -- 어떤 콜렉에서 발생했는지
    
    -- 관계의 속성 (Relationship Attributes)
    sentiment VARCHAR(50) NOT NULL, -- 'Warm', 'Formal', 'Deep' 등
    intensity DECIMAL(3, 2) NOT NULL, -- 0.00 ~ 1.00 (관계의 깊이)
    context_detail TEXT, -- 관계를 정의하는 구체적인 맥락 (예: "고등학교 시절 가장 친했던 친구")
    assigned_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE (source_user_id, target_user_id, collection_id) -- 같은 콜렉에서 중복 관계 방지
);

-- 4. Memory: 추억 아카이빙 (미디어 및 텍스트)
CREATE TABLE Memory (
    memory_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES User(user_id) ON DELETE CASCADE, -- 메모 업로더
    collection_id UUID REFERENCES Collection(collection_id) ON DELETE CASCADE,
    
    media_type VARCHAR(50) NOT NULL, -- 'IMAGE', 'VIDEO', 'TEXT'
    media_url VARCHAR(512), -- AWS S3 등의 저장 경로
    caption TEXT, -- 사진/영상에 대한 캡션
    related_relationship_id UUID REFERENCES Relationship(relationship_id) ON DELETE SET NULL -- 이 메모가 어느 관계에 속하는지 연결
);

-- 5. Donation: 기여금 기록
CREATE TABLE Donation (
    donation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id UUID REFERENCES Collection(collection_id) ON DELETE CASCADE,
    donor_user_id UUID REFERENCES User(user_id) ON DELETE RESTRICT,
    amount DECIMAL(15, 2) NOT NULL,
    donated_at TIMESTAMP DEFAULT NOW()
);
```

---

## 🌐 3. 핵심 API 엔드포인트 정의 (FastAPI / RESTful)

모든 API는 인증(Authentication) 토큰을 요구하며, 주요 기능별
