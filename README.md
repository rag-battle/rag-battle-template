# RAG Battle Template

> **RAG 기반 변호사시험 성능 비교 프로젝트** 공통 템플릿

---

## A. 프로젝트 개요

이 템플릿은 **No-RAG vs RAG** 성능 비교 실험을 위한 공통 구조를 제공한다.

### 목적
- 동일한 평가 환경에서 RAG(Retrieval-Augmented Generation) 도입 효과를 측정한다.
- 변호사시험 과목별(민법, 형법, 공법) 성능을 비교한다.

### 공식 스코어
| 메트릭 | 설명 |
|--------|------|
| **No-RAG Accuracy** | Retrieved context 없이 모델만으로 정답률 |
| **RAG Accuracy** | Retrieved context를 포함한 정답률 |
| **ΔAccuracy** | `RAG Accuracy - No-RAG Accuracy` (RAG 도입 효과) |

---

## B. 공통 공정성 규칙

실험의 **공정성과 재현성**을 위해 아래 규칙을 반드시 준수한다.

### 필수 규칙
1. **모델 동일**: 모든 실험에서 동일한 LLM 모델을 사용한다.
2. **테스트셋 동일**: 공식 테스트셋만 사용한다. (임의 분할 금지)
3. **평가 스크립트**: 팀별 구현 (`scripts/eval.py` 권장 위치). 평가 로직(정답률 계산)은 동일해야 한다.
4. **프롬프트/하이퍼파라미터 동일**: No-RAG와 RAG 간 차이는 **retrieved context 유무만** 허용한다.
5. **데이터/키 커밋 금지**: `data/`, `.env`는 절대 커밋하지 않는다.

### No-RAG vs RAG 차이점
```
No-RAG: [System Prompt] + [Question]
RAG:    [System Prompt] + [Retrieved Context] + [Question]
```
- 위 구조 외의 차이(temperature, max_tokens 등)는 허용되지 않는다.

### 팀별 자유 (비교 대상)
다음 항목은 팀별로 자유롭게 구현한다:
- 전처리, chunking, metadata 구성
- Retrieval/Rerank 전략
- **RAG 컨텍스트 포맷팅** (조문 인용 형식, 메타데이터 포함 여부 등)
- 벡터DB/인덱스 선택
- 파이프라인 구현 디테일

> 프롬프트의 **베이스 구조**(System Prompt + Question)는 고정이지만,
> **Retrieved Context를 어떻게 구성하는지**는 팀별 자유다.

---

## C. 디렉토리 구조

```
rag-battle-template/
├── src/                  # 핵심 소스 코드 (모델, retriever 등)
├── scripts/              # 실행 스크립트 (run_*.py, eval.py)
├── configs/              # 설정 파일 (YAML, JSON)
├── report/               # 결과물 (metrics, predictions, summary)
│   ├── predictions/      # 예측 결과 JSONL 파일
│   ├── metrics.json      # 평가 메트릭
│   └── summary.md        # 최종 요약 리포트
├── data/                 # 🚫 .gitignore (로컬에서만 관리)
│   ├── testset/          # 테스트셋 JSON 파일
│   └── index/            # RAG 인덱스 파일
├── .gitignore            # Git 제외 규칙
├── .env.example          # 환경 변수 템플릿
├── requirements.txt      # Python 의존성
└── README.md             # 이 문서
```

### 폴더별 역할

| 폴더 | 역할 | Git 포함 |
|------|------|----------|
| `src/` | (선택) 공통 라이브러리 코드 위치(팀 레포에서 확장) | ✅ |
| `scripts/` | 실행/평가 entrypoints (CLI) | ✅ |
| `configs/` | 실험 설정 스냅샷/예시 (yaml/json) | ✅ |
| `report/` | 결과/리포트(커밋 대상) | ✅ |
| `data/` | 실험 데이터/인덱스/캐시 (git 추적 제외) | ❌ |

> ⚠️ **중요**: `data/` 폴더는 `.gitignore`에 포함되어 있으며, **로컬에서만 관리**한다.
> 테스트셋 정답 유출 및 라이선스 위반 방지를 위함이다.

---

## D. 데이터 준비

### 공통 Retriever 원천 데이터
법령정보 오픈 API([open.law.go.kr](https://open.law.go.kr))에서 다음 5개 대상을 **전량 수집**하여 공유한다:
| target | 설명 |
|--------|------|
| `law` | 현행법령 목록 + 상세(조문/부칙/개정문) |
| `prec` | 판례 목록 + 상세(전문/요지/판시사항) |
| `expc` | 법령해석 목록 + 상세 |
| `lstrm` | 법률용어사전 목록 + 상세(정의) |
| `detc` | 결정례 목록 + 상세 |

> 전량수집 기준 및 버전은 `manifest.json`으로 확정/공유한다 (버전 태깅 필수).

### 1. 환경 변수 설정
```bash
cp .env.example .env
# .env 파일을 편집하여 실제 값을 입력한다.
```

### 2. Hugging Face 토큰 (필요시)
- [KMMLU-Pro](https://huggingface.co/datasets/HAERAE-HUB/KMMLU-Pro)는 gated dataset일 수 있다.
- Hugging Face 웹사이트에서 라이선스 동의 후 `HF_TOKEN`을 설정한다.

### 3. 테스트셋 다운로드 예시
```python
from datasets import load_dataset
import json
import os

# HF 토큰 설정 (필요시)
# from huggingface_hub import login
# login(token=os.environ["HF_TOKEN"])

# KMMLU-Pro 로드
ds = load_dataset("HAERAE-HUB/KMMLU-Pro", split="test")

# 변호사 과목 필터링
lawyer_ds = ds.filter(lambda x: x["license_name"] == "변호사")

# 과목별 분리 및 저장
os.makedirs("data/testset", exist_ok=True)

subject_map = {
    "민법": "civil",
    "형법": "criminal",
    "공법": "public"
}

for kor_name, eng_name in subject_map.items():
    subset = lawyer_ds.filter(lambda x: x["subject"] == kor_name)
    with open(f"data/testset/{eng_name}.json", "w", encoding="utf-8") as f:
        json.dump(list(subset), f, ensure_ascii=False, indent=2)
    print(f"Saved {len(subset)} items to data/testset/{eng_name}.json")
```

> 📌 **참고**: `data/`는 repo에 포함되지 않으므로 **각 참가자가 직접 준비**해야 한다.

---

## E. 실행 방법 (예시임. 파일명을 꼭 똑같이 할 필요 x)

### 1. 의존성 설치
```bash
pip install -r requirements.txt
```
> `requirements.txt`는 최소 공통 의존성만 포함한다. 팀별로 필요한 패키지(langchain, chromadb, faiss 등)는 각자 추가한다.

### 2. No-RAG 실행
```bash
python scripts/run_no_rag.py \
    --subject civil \
    --in data/testset/civil.json \
    --out report/predictions/no_rag_civil.jsonl
```

### 3. RAG 실행
```bash
python scripts/run_rag.py \
    --subject civil \
    --in data/testset/civil.json \
    --index data/index/ \
    --out report/predictions/rag_civil.jsonl
```

### 4. 평가
```bash
python scripts/eval.py \
    --gold data/testset/civil.json \
    --pred report/predictions/rag_civil.jsonl \
    --out report/metrics.json
```

### 생성되는 파일
```
report/
├── predictions/
│   ├── no_rag_civil.jsonl    # No-RAG 예측 결과
│   ├── no_rag_criminal.jsonl
│   ├── no_rag_public.jsonl
│   ├── rag_civil.jsonl       # RAG 예측 결과
│   ├── rag_criminal.jsonl
│   └── rag_public.jsonl
├── metrics.json              # 정량 메트릭
└── summary.md                # 최종 요약
```

---

## F. 산출물 규격

### report/metrics.json 예시
```jsonc
{
  "model": "<TBD: gemini-2.0-flash 등 확정 후 기입>",
  "run_name": "baseline-v1",
  "timestamp": "2025-02-03T10:00:00+09:00",
  "subjects": {
    "civil": {
      "no_rag_accuracy": 0.72,
      "rag_accuracy": 0.81,
      "delta_accuracy": 0.09,
      "total_questions": 70   // 민사법
    },
    "criminal": {
      "no_rag_accuracy": 0.68,
      "rag_accuracy": 0.77,
      "delta_accuracy": 0.09,
      "total_questions": 40   // 형사법
    },
    "public": {
      "no_rag_accuracy": 0.65,
      "rag_accuracy": 0.74,
      "delta_accuracy": 0.09,
      "total_questions": 40   // 공법
    }
  },
  "overall": {
    "no_rag_accuracy": 0.69,
    "rag_accuracy": 0.78,
    "delta_accuracy": 0.09,
    "total_questions": 150   // 총 150문항
  }
}
```

### report/summary.md 포함 항목
1. **점수 요약**: 과목별/전체 No-RAG, RAG, ΔAccuracy
2. **실험 설정**: 모델명, 버전, 하이퍼파라미터
3. **설계 포인트** (3~5개):
   - Retriever 구조 (chunk size, overlap, top-k 등)
   - Embedding 모델
   - RAG 컨텍스트 포맷팅 (조문 인용 형식, 메타데이터 구성 등)
   - Index 구축 방법
   - 기타 최적화 기법

---

## G. .gitignore 정책

### 제외 대상 및 이유

| 대상 | 제외 이유 |
|------|----------|
| `data/` | 테스트셋 정답 유출 방지, 라이선스 준수 |
| `.env` | API 키, 토큰 등 민감 정보 보호 |
| `*.key`, `*.pem` | 비밀 키 파일 보호 |
| `__pycache__/` | 재생성 가능, 저장소 크기 절약 |
| `.DS_Store` | OS 자동 생성 파일, 불필요 |

### Public Repository 주의사항
- 이 템플릿은 **Public repo** 기준으로 설정되어 있다.
- 민감 정보나 테스트셋 정답이 **절대 커밋되지 않도록** 주의한다.
- 실수로 커밋한 경우 `git filter-branch` 등으로 히스토리에서 완전 제거해야 한다.

---

## 문의

디스코드 채널 활용
