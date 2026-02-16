---
title: "벡터서치 도구 7종 비교 — 사이드 프로젝트에서 grep 너머로"
date: 2026-02-17T14:00:00+09:00
draft: false
tags: [vector-db, vector-search, hybrid-search, lancedb, qdrant, weaviate, turbopuffer, pageindex, rag, ai-agent]
categories: [research]
summary: "grep보다 나은 검색이 필요했다. 벡터 + 풀텍스트 하이브리드 서치도 뭔가 아쉬웠다. 2026년 기준 LanceDB, Qdrant, Turbopuffer, Weaviate, PageIndex 등 7종을 CPU 사용량, 비용, 프로덕션 사례 기준으로 비교했다."
description: "2026년 벡터서치 도구 비교. LanceDB, Qdrant, Turbopuffer, Weaviate, PageIndex, sqlite-vec, Chroma를 CPU 사용량, 비용, Mac Mini 셀프호스팅 관점에서 분석한 실전 가이드."
keywords: ["벡터서치", "vector search", "벡터 데이터베이스", "LanceDB", "Qdrant", "Turbopuffer", "하이브리드 서치", "RAG", "사이드 프로젝트", "AI agent"]
cover:
  image: ""
  alt: "벡터서치 도구 비교"
  hidden: true
---

## grep으로는 부족한 순간

사이드 프로젝트를 하다 보면 검색이 필요해지는 순간이 온다. 노트, 코드 조각, 아카이브된 컨텍스트... 처음엔 `grep`으로 충분하다. 그런데 데이터가 쌓이고, "이 개념과 비슷한 내용"을 찾고 싶을 때 grep은 무력하다. 정확한 키워드를 알아야만 찾을 수 있으니까.

그래서 기존에 쓰고 있던 MongoDB의 벡터 검색 + 풀텍스트 검색을 합쳐서 하이브리드 서치를 구성해봤다. 써보니 또 다른 아쉬움이 생겼다.

**단순 vector + full-text search의 한계:**
- 벡터 유사도가 높다고 해서 실제로 관련 있는 건 아니다 ("유사성 != 관련성")
- 리랭킹 없이 두 결과를 합치면 노이즈가 많다
- 파이프라인 구성이 번거롭고, 튜닝할 여지가 적다

거기에 **개인 사이드 프로젝트에서 쓰기엔 비용 부담도 있다**. 벡터 인덱스를 제대로 쓰려면 유료 클러스터가 필요하고, 검색 하나 때문에 월 수만 원을 쓰기엔 과하다.

그래서 처음부터 다시 조사해봤다. 2026년 2월 기준, 벡터 검색 도구 생태계가 어떻게 돌아가고 있는지.

---

## 전체 지형도

벡터 검색 도구는 크게 세 카테고리로 나뉜다.

### Serverless (서버 관리 X)

| 도구 | 특징 | 가격대 |
|------|------|--------|
| **Turbopuffer** | Object storage 기반, 10x 저렴 | ~$64/월 최소 |
| **Pinecone Serverless** | 가장 쉬운 셋업 | Free~$50/월~ |
| **Qdrant Cloud** | 하이브리드 서치 강점 | ~$27/월 (최적화 시) |
| **LanceDB Cloud** | S3 기반, scale-to-zero | Private beta |

### Self-hosted (로컬/클라우드 서버)

| 도구 | 특징 | 언어 |
|------|------|------|
| **Qdrant** | Docker 한 줄, 양자화 지원 | Rust |
| **Weaviate** | 올인원 풀스택, GraphQL | Go |
| **Milvus** | 대규모 분산 처리 | Go/C++ |

### Embedded (서버 없이 앱에 내장)

| 도구 | 특징 | 사용감 |
|------|------|--------|
| **LanceDB** | 벡터계의 SQLite | 라이브러리 import |
| **sqlite-vec** | 의존성 제로, 어디서든 실행 | SQLite 확장 |
| **Chroma** | Python 원라이너 셋업 | 프로토타이핑용 |

사이드 프로젝트 관점에서 보면 Serverless는 비용이, Self-hosted는 리소스가, Embedded는 기능이 각각 트레이드오프다. 하나씩 파보자.

---

## 각 도구 심층 분석

### LanceDB - "벡터계의 SQLite"

가장 인상적이었던 도구. SQLite처럼 앱에 임베드하는 벡터 DB다.

**핵심:**
- 서버 프로세스가 없다. 라이브러리를 import하면 끝
- 아이들 CPU/RAM = 0. 쿼리할 때만 리소스를 쓴다
- 하이브리드 서치(벡터 + FTS + 리랭킹)가 네이티브로 내장

```python
import lancedb
from lancedb.rerankers import RRFReranker

db = lancedb.connect("./my-vectors")
table = db.open_table("notes")

# 하이브리드 서치 한 줄
results = (
    table.search("query", query_type="hybrid")
    .rerank(RRFReranker())
    .limit(10)
    .to_list()
)
```

기존에 벡터 검색과 풀텍스트 검색을 파이프라인으로 복잡하게 엮던 걸, 한 줄로 처리한다. 리랭킹(RRF)도 내장이라 "벡터 결과와 키워드 결과를 어떻게 합칠까"를 고민할 필요가 없다. 임베딩 자동화도 된다:

```python
from lancedb.embeddings import get_registry
from lancedb.pydantic import LanceModel, Vector

openai_embed = get_registry().get("openai").create(name="text-embedding-3-small")

class Note(LanceModel):
    text: str = openai_embed.SourceField()
    vector: Vector(1536) = openai_embed.VectorField()
    tags: list[str]
```

텍스트만 넣으면 벡터가 자동 생성된다. 검색도 텍스트로 바로 가능.

**CPU 사용량 (실측 기반):**

| 상태 | CPU | 성능 |
|------|-----|------|
| 아이들 | 0% (프로세스 없음) | — |
| 벡터 검색 | 순간 스파이크 | P50 135ms, 97 QPS |
| FTS | 순간 스파이크 | P50 10ms, 1,534 QPS |
| 인덱싱 (1M vectors) | **CPU 풀 사용, ~60초** | 디스크 기반 |

**주의점:**
- 쿼리 recall ~88% (Qdrant의 ~95% 대비 낮음)
- FastAPI 래퍼로 장시간 운용 시 메모리 누수 보고됨 (커넥션 관리 필요)
- Apple Silicon MPS 가속 인덱싱에 크래시 이슈 ([GitHub #2212](https://github.com/lancedb/lancedb/issues/2212))
- 700M vectors 프로덕션 사례에서 128GB RAM으로도 인덱싱 OOM 발생

사이드 프로젝트 규모(수만~수십만 벡터)에서는 이런 문제를 만날 일이 없다.

---

### Qdrant - 전문 벡터 DB의 왕

Rust로 만든 고성능 벡터 DB. 기능과 성능 면에서 가장 성숙하다.

**핵심:**
- HNSW 인덱스 → recall ~95%, 쿼리 20-30ms
- Scalar/Binary Quantization으로 메모리 4~32x 압축
- Docker 한 줄로 로컬 실행

```bash
docker run -p 6333:6333 qdrant/qdrant
```

**하지만 CPU 이슈가 있다:**
- GitHub에 CPU 관련 이슈 다수 ([#4128](https://github.com/qdrant/qdrant/issues/4128), [#4446](https://github.com/qdrant/qdrant/issues/4446))
- HNSW 인덱스 빌드 시 CPU 폭주
- 필터링 쿼리에서 수천 벡터 스캔 시 CPU 스파이크
- 백그라운드 세그먼트 최적화가 상시 돌아감

프로덕션급 성능이 필요하면 최고의 선택이다. 하지만 사이드 프로젝트용 Mac Mini에서 다른 서비스와 함께 돌리기엔, **서버 프로세스가 항상 떠 있어야 한다**는 점이 부담이다.

---

### Turbopuffer - 서버리스의 끝판왕

Notion AI가 채택한 서버리스 벡터 검색 엔진. Object storage($0.02/GB) 위에 구축해서 비용이 극단적으로 싸다.

**실적:**
- Notion: 10B+ 벡터, "수백만 달러" 절감
- 2.5T+ 문서 처리, 100B+ 벡터 단일 인덱스
- p50 <10ms, p99 200ms

**문제:** 셀프호스팅 불가. SaaS only. 최소 $64/월. 데이터가 외부에 저장된다.

대규모 서비스에서는 최강이지만, 사이드 프로젝트에 월 $64를 쓸 이유가 없다.

---

### Weaviate - 올인원이지만 무겁다

Go로 만든 풀스택 벡터 DB. 임베딩 생성 모듈, GraphQL API, RAG, 리랭킹이 전부 내장된 "배터리 포함" 철학.

**강점:**
- LangChain/LlamaIndex에서 1st-class 지원
- 멀티테넌시 아키텍처 (B2B SaaS에 적합)
- 벡터화 모듈 내장 (OpenAI, Cohere 등)

**약점:**
- **리소스 소비가 심하다.** 커뮤니티 리포트: 100K records에 35GB RAM
- 메모리 = 2 x (전체 벡터 메모리). 1M vectors(1536dim) → ~12GB
- Mac Mini 16GB에서는 사실상 소규모만 가능

벡터 DB계의 Supabase 같은 존재. "풀스택"이 필요한 B2B SaaS가 아니면 오버킬이다.

---

### PageIndex - 벡터 없는 RAG, 그런데...

벡터 DB가 아닌 완전히 다른 접근법. 문서를 트리 구조로 만들고 LLM이 추론하며 탐색한다.

```
벡터 DB:    문서 → 청킹 → 임베딩 → 유사도 검색
PageIndex:  문서 → 트리 구조 → LLM 추론 탐색
```

FinanceBench에서 98.7% 정확도를 달성했다 (벡터 RAG는 50-80%). "유사성 != 관련성" 문제를 정면으로 해결하는 흥미로운 아이디어다.

**하지만 실제로 파보면:**
- 프로덕션 사례 **0건** 확인됨 (Mafin 2.5는 자체 벤치마크)
- pip 설치 이슈 (pyproject.toml 누락), async 미지원, 보안 취약점
- 멀티문서 검색 **미완성** (워크어라운드만 3가지)
- 매 쿼리마다 LLM API 호출 → 비용 + 레이턴시 (초~분 단위)

결론: **벡터 DB의 대안이 아니라 보완재**. 벡터 DB로 문서를 추린 다음, 단일 긴 문서를 정밀 분석할 때 쓸 만하다. 하지만 지금 당장 사이드 프로젝트에 넣기엔 성숙도가 부족하다.

---

## 사이드 프로젝트에서 뭘 쓸까

결국 중요한 건 세 가지다:
1. **비용**: 개인 프로젝트에 월 수만 원은 과하다
2. **리소스**: Mac Mini 같은 제한된 환경에서 돌아가야 한다
3. **검색 품질**: grep보다 나아야 한다 (당연히)

이 기준으로 정리하면:

| 용도 | 1순위 | 2순위 | 이유 |
|------|-------|-------|------|
| 사이드 프로젝트 검색 (로컬) | **LanceDB** | sqlite-vec | 무료, 서버 불필요, 하이브리드 서치 |
| AI agent 컨텍스트 검색 | **LanceDB** | Qdrant | 아이들 리소스 0 |
| 프로덕션 서비스 (서버리스) | **Turbopuffer** | Qdrant Cloud | 비용 효율 |
| B2B SaaS 멀티테넌트 | **Weaviate** | Qdrant | 풀스택 DX |
| 단일 긴 문서 정밀 분석 | **PageIndex** | — | 추론 기반 (보조용) |

---

## 리소스 사용량 비교

리소스가 제한된 환경에서 가장 중요한 건 **아이들 시 부담**이다.

| 도구 | 아이들 CPU | 아이들 RAM | 쿼리 시 |
|------|-----------|-----------|---------|
| **LanceDB** | 0% | ~수 MB | 순간 스파이크 후 반환 |
| **sqlite-vec** | 0% | ~수 MB | 순간 스파이크 후 반환 |
| **Chroma** | 낮음 | ~200MB | 중간 |
| **Qdrant** | 낮음~중간 | ~500MB+ | 높음 + 백그라운드 |
| **Weaviate** | 중간 | ~1GB+ | 높음 |

서버 프로세스를 상시 띄우는 것 자체가 부담이라면, **임베디드 방식(LanceDB, sqlite-vec)이 구조적으로 유리하다.**

---

## 마무리

grep → 풀텍스트 검색 → 벡터 검색 → 하이브리드 서치. 검색의 진화는 결국 "내가 뭘 찾는지 정확히 몰라도 찾을 수 있게" 하는 방향이다.

이번 리서치를 통해 각 도구의 특성은 파악했다. 개인적으로는 관리해야 할 문서가 더 쌓이면 LanceDB부터 써볼 계획이다. 서버 없이 `pip install lancedb` 하나로 하이브리드 서치 + 리랭킹까지 쓸 수 있다는 게 사이드 프로젝트에서는 가장 끌리는 포인트다.

실제로 도입해보면 recall 88%가 체감상 어느 정도인지, 인덱싱 시간이 프로젝트 규모에서 문제가 되는지 등 리서치만으로는 알 수 없는 것들이 보일 것 같다. 써보고 나서 후속 글로 정리해볼 생각이다.
