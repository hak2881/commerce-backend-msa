# Commerce Backend Microservices

Backend architecture case studies from commerce platforms built on Shopify Plus — where the storefront is hosted, but everything behind it (membership, rewards, ERP identity, fulfillment) is a service you own and operate.

These are write-ups, not source. Client code, credentials, and infrastructure identifiers are deliberately absent.

## Why this exists

Hosted commerce platforms handle catalog, cart, and checkout well. They stop being enough the moment a business needs rules the platform has no opinion about: a referral tree that decides who earns what, a legacy ERP that owns customer identity, a fulfillment flow that spans two carriers and three warehouses.

That gap is where these systems live. Each case below is one instance of it, with the constraints that shaped the design.

## Shape of the systems

```mermaid
flowchart LR
    S[Shopify Plus storefront] -->|webhooks<br/>HMAC verified| API
    S -->|storefront JS<br/>Bearer JWT| API
    A[Embedded admin app] -->|session token| API

    subgraph API [Owned backend]
        direction TB
        M[member / identity]
        R[reward / points]
        O[order]
        AD[admin]
    end

    API --> DB[(Postgres<br/>one logical DB per service)]
    API --> EXT[External systems<br/>ERP · carriers · payments]

    style API fill:#f6f8fa,stroke:#57606a
```

Common decisions across all three systems:

| Decision | Rationale |
|---|---|
| One logical database per service, no cross-service reads | Service boundaries survive even when deployment is consolidated |
| Webhook idempotency by platform-supplied delivery ID | Commerce webhooks retry; at-least-once delivery is the contract |
| Outbox table for cross-service writes | A write and its downstream effect commit together or not at all |
| Own JWT for storefront auth, not platform app-proxy | Storefront needs identity the platform's customer model can't express |
| Money as integers, never floats | See [loyalty-ledger-systems](https://github.com/hak2881/loyalty-ledger-systems) for what happens otherwise |

## Cases

| # | Case | Domain | Core problem |
|---|---|---|---|
| 01 | [Referral reward backend](cases/01-referral-reward-backend.md) | US beauty-device brand — B2B + B2C on one store | Six service boundaries, one deployable, zero double-payouts |
| 02 | [Unified membership over a legacy ERP](cases/02-unified-membership-erp.md) | Korean outdoor gear retailer | The ERP owns customer identity; the storefront can't see it |
| 03 | [Global D2C fulfillment services](cases/03-global-d2c-fulfillment.md) | Korean dashcam manufacturer | One catalog, many countries, carrier-specific fulfillment |

## Stack

`Go` (chi · pgx · sqlc · goose) · `Python` (FastAPI · Django REST) · `PostgreSQL` · `Kubernetes` (EKS) · `Docker` · `AWS Lambda`

---

## 한국어 요약

Shopify Plus 위에 올린 커머스 플랫폼의 **백엔드 아키텍처 케이스 스터디** 모음입니다. 소스코드가 아니라 설계 기록이고, 클라이언트 코드와 인증정보, 인프라 식별자는 빼고 썼습니다.

호스팅형 커머스 플랫폼은 카탈로그부터 장바구니, 결제까지는 잘 해줍니다. 문제는 플랫폼이 아무 의견도 갖고 있지 않은 규칙이 필요해지는 순간입니다. 누가 얼마를 적립받는지 정하는 추천 트리, 고객 신원을 쥐고 있는 레거시 ERP, 택배사 두 곳과 창고 세 곳을 넘나드는 배송 흐름 같은 것들이죠. 여기서부터는 직접 만들어 운영하는 백엔드가 필요합니다. 아래 세 건이 그런 사례입니다.

**공통 설계 원칙**

- 서비스마다 논리 DB를 나누고 교차 조회는 막았습니다. 배포를 하나로 합쳐도 경계는 남습니다.
- 웹훅 멱등성은 플랫폼이 주는 전달 ID로 잡습니다. 재전송은 예외 상황이 아니라 원래 계약입니다.
- 서비스 간 쓰기는 outbox를 거칩니다. 쓰기와 후속 효과가 같이 커밋되거나 같이 안 됩니다.
- 스토어프론트 인증은 플랫폼 App Proxy 대신 자체 JWT를 씁니다. 플랫폼 고객 모델로는 담기지 않는 신원이 필요했습니다.
- 금액은 정수로만 저장합니다. 부동소수는 쓰지 않습니다.

| # | 케이스 | 도메인 | 핵심 문제 |
|---|---|---|---|
| 01 | 추천 보상 백엔드 | 미국 뷰티 디바이스 브랜드 (B2B·B2C 단일 스토어) | 서비스 경계 6개, 배포 1개, 중복 지급 0 |
| 02 | 레거시 ERP 위의 통합회원 | 국내 아웃도어 기어 리테일러 | 고객 신원은 ERP가 쥐고 있는데 스토어프론트에서는 그게 안 보임 |
| 03 | 글로벌 D2C 주문·배송 서비스 | 국내 대시캠 제조사 | 카탈로그는 하나, 국가는 여럿, 배송은 택배사마다 다름 |
