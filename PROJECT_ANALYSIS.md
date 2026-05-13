# GreenPick - 친환경 상품 추천 플랫폼 백엔드

> 2024 인하대학교 · 상명대학교 · 인제대학교 공동 주최 Net Zero 해커톤 **장려상 수상작**

---

## 프로젝트 개요

**GreenPick**은 소비자가 친환경 상품을 쉽게 발견하고 선택할 수 있도록 돕는 상품 추천 플랫폼의 백엔드 서버입니다.  
각 상품에 `greenScore` 지표를 부여하여 친환경성을 정량화하고, 카테고리별 랜덤 추천 API를 제공합니다.

- **수상**: 장려상 (인하대학교 · 상명대학교 · 인제대학교 공동 주최 Net Zero 해커톤, 2024)
- **기간**: 2024년 (해커톤)
- **역할**: 백엔드 개발 (NestJS API 서버 설계 및 구현, 배포 파이프라인 구축)
- **저장소**: https://github.com/inha-swuniv-2024-netzero-hackathon/greenpick

---

## 기술 스택

| 분류 | 기술 |
|------|------|
| **언어** | TypeScript |
| **프레임워크** | NestJS 10 |
| **ORM** | TypeORM 0.3 |
| **데이터베이스** | PostgreSQL |
| **API 문서화** | Swagger (OpenAPI) |
| **배포** | GitHub Actions + SSH + PM2 |
| **유효성 검사** | class-validator, class-transformer |

---

## 주요 기능

### 1. 친환경 상품(Goods) CRUD API
- `POST /api/goods` — 상품 등록
- `GET /api/goods/:count` — 전체 랜덤 상품 10개 조회
- `GET /api/goods/category/:category` — 카테고리별 랜덤 상품 10개 조회

### 2. greenScore 기반 친환경 지표
`Goods` 엔티티에 `greenScore` 컬럼을 정의하여 각 상품의 친환경 점수를 저장·제공합니다.

```
id | name | price | market | marketUrl | category
imageUrl | greenScore | deliveryDate | reviewCount | rate
```

### 3. 통일된 응답 형식 (ResponseEntity)
모든 API 응답을 `{ statusCode, message, data }` 형태로 규격화하는 제네릭 `ResponseEntity<T>` 클래스를 직접 설계했습니다.

```typescript
static OK<T>(data: T): ResponseEntity<T>
static ERROR(message: string): ResponseEntity<null>
static fromStatusCode<T>(statusCode, message, data): ResponseEntity<T>
```

### 4. 전역 예외 처리 (Global Exception Filter)
`@Catch()` 데코레이터를 활용한 `AllExceptionsFilter`로 모든 예외를 중앙에서 처리합니다.
- `HttpException` → 적절한 HTTP 상태코드 반환
- `QueryFailedError` (TypeORM) → 400 Bad Request로 변환
- 그 외 → 500 Internal Server Error + Logger 기록

### 5. CI/CD 파이프라인 (GitHub Actions)
`main` 브랜치 push 시 자동으로 원격 서버에 배포됩니다.

```
push to main → GitHub Actions → SSH 접속 → git pull → yarn build → pm2 restart
```

---

## 아키텍처

```
src/
├── main.ts                          # 앱 진입점, Swagger 설정, GlobalPipe 등록
├── app.module.ts                    # 루트 모듈
├── infrastructure/
│   └── database.module.ts           # TypeORM PostgreSQL 연결 (환경변수 기반)
├── domains/
│   └── goods/
│       ├── goods.controller.ts      # 라우팅 레이어
│       ├── goods.service.ts         # 비즈니스 로직
│       ├── goods.module.ts          # 도메인 모듈
│       ├── entities/goods.entity.ts # DB 엔티티
│       └── dtos/create-goods.dto.ts # 입력 유효성 검사
└── common/
    ├── response-entity.ts           # 공통 응답 래퍼
    ├── filters/
    │   └── all-exceptions-filter.ts # 전역 예외 필터
    └── decorators/
        └── api-operation.decorator.ts
```

계층형 아키텍처(Controller → Service → Repository)를 준수하고, 도메인 단위로 모듈을 분리했습니다.

---

## 기술적 의사결정

### TypeORM QueryBuilder로 랜덤 조회 구현
`RANDOM()` 내장 함수를 QueryBuilder에 직접 적용하여 매 요청마다 다른 상품 세트를 반환합니다.  
단순 `findMany` 대신 QueryBuilder를 선택한 이유는 DB 레벨 랜덤 정렬이 애플리케이션 레벨보다 효율적이기 때문입니다.

```typescript
return this.goodsRepository
  .createQueryBuilder('goods')
  .where('goods.category = :category', { category })
  .orderBy('RANDOM()')
  .limit(10)
  .getMany();
```

### SnakeNamingStrategy 적용
TypeORM 기본값은 camelCase 컬럼명을 그대로 사용합니다.  
`typeorm-naming-strategies`의 `SnakeNamingStrategy`를 적용하여 TypeScript 컨벤션(camelCase)과 DB 컨벤션(snake_case)을 자동으로 매핑했습니다.

### 환경변수 기반 설정 분리
`@nestjs/config`의 `ConfigModule`과 `ConfigService`를 활용하여 DB 접속 정보를 코드에서 완전히 분리했습니다. `.develop.env` 파일로 로컬 환경을 관리합니다.

---

## 실행 방법

```bash
# 의존성 설치
yarn install

# 환경변수 설정 (.develop.env)
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=your_username
DB_PASSWORD=your_password
DB_DATABASE=greenpick

# 개발 서버 실행
yarn start:dev

# Swagger 문서 확인
http://localhost:3000/api/docs
```

---

## 배운 점 / 성과

- NestJS의 **모듈 시스템**과 **의존성 주입(DI)**을 활용한 계층형 아키텍처 설계 경험
- **TypeORM** + PostgreSQL 연동 및 엔티티 설계, QueryBuilder 활용
- **Swagger** 자동 문서화로 프론트엔드 팀과의 API 협업 효율 향상
- **GitHub Actions**를 이용한 SSH 기반 무중단 CI/CD 파이프라인 구축
- 제네릭 `ResponseEntity<T>` 및 전역 예외 필터로 일관된 API 응답 체계 수립
