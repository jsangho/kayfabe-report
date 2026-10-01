---
layout: report
title: 시스템 설계
permalink: /03-architecture/
chapter: 3
nav_order: 3
---

## 1) 시스템 구성

두 개의 규칙이 동시에 걸린다. 앱 **안쪽**은 선형(클린 아키텍처)이고, 앱 **사이**는 비선형(스타 토폴로지)이다.

### 앱 안쪽 — 헥사고날 레이어

```
domain/            순수 파이썬. 프레임워크 import 금지
app/               UseCase(Protocol) · DTO · Interactor
adapter/inbound/   FastAPI 라우터 · Pydantic 스키마
adapter/outbound/  SQLAlchemy 리포지토리 · 외부 API 클라이언트
dependencies/      FastAPI Depends 팩토리
```

의존성은 항상 안쪽을 향한다(`adapter → app → domain`). `kayfabe`의 파일 분포는 `app` 81 · `adapter` 64 · `domain` 16 · `dependencies` 12다. **도메인이 가장 얇다** — 순수 계산만 남기고 나머지를 밖으로 밀어낸 결과다. 합성 산식(`prediction_synthesis`)과 결과 판정(`result_adjudication`)이 거기 있고, DB도 LLM도 모르므로 고정 픽스처만으로 시험된다.

### 앱 사이 — 스타 토폴로지

```
   [kayfabe] [admin] [heyman] [lion_king] [soccer] [auth]   ← SPOKE
        \        |       |        |          /
                    [ontology]                              ← HUB
```

- **허브(`ontology`)** 는 공유 커널이다. Gemini 호출, 시맨틱 라우팅, 크롤·스크랩, 분류기가 여기 있다. **스포크를 import하지 않는다** — 최하위라 위를 볼 수 없다.
- **스포크 ↔ 스포크 직접 import는 금지**다. 두 앱에 공통 로직이 필요하면 허브로 **올려서** 푼다.
- **`auth`는 아무도 import할 수 없다.** 허브조차 금지다. 검증이 필요하면 `jsangho.core.security`를 쓴다.

이 규칙은 문서가 아니라 **계약으로 강제된다.** `import-linter` 계약 넷이 pre-commit 훅과 GitHub Actions 양쪽에서 돌고, 위반하면 커밋이 막힌다.

| 계약 | 내용 |
|---|---|
| `no_spoke_to_spoke` | 스포크끼리 직접 import 금지 |
| `star_topology_hub_only` | 스포크는 허브만, 허브는 스포크를 못 본다 |
| `clean_architecture_layers` | 앱 안의 레이어 순서 |
| `auth_isolation` | `auth`는 누구도 import 불가 |

> **새 앱을 만들면 계약에 이름을 넣어야 한다.** 빠뜨리면 그 앱은 **검사되지 않은 채 초록으로 통과한다.**

## 2) 기술 스택

| 층 | 기술 |
|---|---|
| 프론트 | Next.js 16 (App Router) · React 19 · TypeScript strict · Tailwind CSS · Radix UI · Recharts 2.15 |
| 백엔드 | FastAPI 0.136 · Uvicorn · Python 3.13 |
| ORM · 마이그레이션 | SQLAlchemy 2.0 async · SQLModel · Alembic |
| 관계형 DB | PostgreSQL (Supabase) · `pgvector` 확장 |
| 그래프 DB | Neo4j 6 |
| 캐시 | Redis |
| LLM | Google Gemini (`google-genai`) · 온프레미스 경로로 Ollama 어댑터 |
| 임베딩 | `BAAI/bge-m3` (Transformers) |
| 형태소 | Kiwi (`kiwipiepy`) |
| 배포 | EC2 + Docker Compose (운영) · k3s (로컬) · Vercel (프론트) · Cloudflare Tunnel |

**차트 라이브러리를 새로 고르지 않았다.** Recharts는 이미 저장소에 있었고 `/admin`이 쓰고 있었다 — 새 의존성을 더하지 않는다는 결정의 결과다. React 19를 peer로 선언하고 SVG로 렌더하므로 CSS 변수 토큰을 `fill`·`stroke`에 그대로 넣을 수 있다.

**의존성은 줄이는 방향으로 관리한다.** 2026-09-29에 코드 참조가 0건이던 `lightgbm`·`xgboost`·`ultralytics`를 뺐다. `xgboost` 하나가 자기 99MB에 더해 `nvidia-nccl-cu12` **290MB**를 끌고 오고 있었다 — torch는 CPU 휠인데 그쪽으로 CUDA가 들어오던 경로다.

## 3) 데이터 모델

`kayfabe`가 소유한 테이블은 열셋이고, 네 묶음으로 갈린다.

### 대회와 경기

| 테이블 | 내용 |
|---|---|
| `ple_events` | 대회. `slug` 유일 · `start_date` · `status` |
| `ple_matches` | 경기. `card_json`에 대진 원본, `point_value`에 배점 |
| `ple_predictions` | 사용자 픽 |

`ple_matches`의 유일 제약은 **`(event_id, match_key)`** 다. 경기 id는 대회 안에서만 유일하면 되며, 실제로 SummerSlam과 Survivor Series가 `ss26` 접두사를 나눠 쓴다. 고치려고 한쪽을 바꾸면 이미 저장된 예측이 그 경기를 잃는다.

### AI 예측과 그 계보

| 테이블 | 내용 |
|---|---|
| `ple_agent_predictions` | 최종 pick · 승률 · 확신도 · `knowledge_query` · `synthesis_version` |
| `ple_agent_reports` | 에이전트별 의견 · 가중치 · 근거 문장 |
| `ple_prediction_retrievals` | 그때 읽은 청크의 **본문 스냅샷**과 개정본 시각 |
| `ple_knowledge_chunks` | RAG 코퍼스. 임베딩 벡터 포함 |

**계보가 별도 테이블인 이유**는 코퍼스가 판본을 하나만 갖기 때문이다. 같은 URL을 다시 수집하면 옛 청크를 지우므로, 그때 읽은 원문은 `ple_prediction_retrievals`에만 남는다. 해시는 대조만 되고 복원은 안 된다.

리포트와 계보는 예측에 `cascade="all, delete-orphan"`으로 묶인다 — 예측 하나를 지우면 딸린 행이 함께 지워진다.

### 기록과 챔피언십

`wrestlers` · `championship_titles` · `title_acquisitions` — 타이틀 이력은 위키 챔피언 보드에서 동기화한다.

### 포인트와 상점

`point_ledger_entries` · `shop_items` · `user_shop_items` — 채점 결과가 원장에 쌓이고 상점에서 쓰인다. 배점이 5의 배수인 것은 상점 가격을 소수점 없이 다루기 위해서다.

## 4) 정보 구조

화면은 세 갈래이고, 제 1 장 3절의 세 축과 그대로 대응한다.

```
/ple · /ple/[slug] · /results · /rankings · /records · /shop     예측·랭킹
/data-center  ├ /ple  ├ /wrestlers  ├ /matches                   데이터 센터
              ├ /championships  └ /analytics
/ai-lab  ├ /predictions  ├ /agents  ├ /performance               AI LAB
         ├ /knowledge  ├ /readiness  ├ /leakage
         └ /audit/[eventSlug]/[matchKey]
```

**AI LAB 안에서도 시선의 방향이 갈린다.** 여섯 화면이 뒤를 보고(이미 만들어진 예측을 판정·재현·귀속) `readiness` 하나만 앞을 본다. 그 여섯이 전부 같은 결론에 닿기 때문이다 — **예측을 만들기 전에 코퍼스를 손봤어야 했다.** 누수는 판정이 아니라 수집에서 생긴다.

`audit/[eventSlug]/[matchKey]`가 가장 깊은 화면이고, 예측 한 건의 계보 전체(질의 · 읽은 청크 · 리포트 · 자격 판정 · 재현 결과)를 한 페이지에 세운다.
