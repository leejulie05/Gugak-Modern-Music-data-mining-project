# Gugak × Modern Music — 데이터 마이닝으로 본 국악과 K-pop의 정서적 유사성

> 한국 전통 음악(국악)의 정서적 특성을 정량 점수로 정의하고, 멜론 차트 26,000곡과 비교해 *어느 가수·곡이 국악과 가장 닮았는가*를 데이터 마이닝으로 찾는 프로젝트.

---

## 개요

| 항목 | 내용 |
|------|------|
| 프로젝트 유형 | 서강대학교 25-1학기 빅데이터 마이닝 수업 final project (3인 팀) |
| 본인 담당 | 주제 발상 · **Soul Score 설계** · **정서 점수 가중치 공식** · 유사도 분석 코드 · 결과 해석 |
| 팀원 담당 | 데이터 정리 (재원: JSON 파싱·가사 결합) · 전처리 (다혜: sentiment 측정·Pykospacing) |
| 데이터 | 멜론 차트 ~26,000곡 + AI Hub 국악 악보 및 음원 데이터 + KNU 감성사전 |
| 핵심 기법 | 도메인 지식 기반 custom metric 설계 + multi-source 정서 점수 통합 |
| 결과물 | 국악-현대 음악 유사도 분석, 가장 국악스러운 K-pop 가수·곡 Top 3 |

---

## Highlights

- **Soul Score 설계** — 국악의 정서적 밀도를 *시김새(sigimsae) 빈도 / 곡 길이*로 정의한 custom metric. 도메인 지식(국악 음악학)을 정량 신호로 변환
- **Multi-weight 정서 점수** — 가사 sentiment + Soul Score + 악기·장단 가중치를 결합한 통합 정서 점수 설계
- **흥미로운 발견** — Jennifer Lopez가 한국 가수 유승우 다음으로 국악과 정서적으로 가장 가까운 아티스트로 도출됨 (예상 밖 결과)
- **트렌드 분석** — 2000~2010년대 대비 2020년대 K-pop이 국악과 정서적으로 더 가까워지는 경향 확인
- **장르별 분석** — Electronica가 가장 국악과 유사. Rap/힙합·인디음악은 오히려 거리가 멀음 (negative sentiment 영향)

---

## Architecture

```
                ┌────────────────────────────────────────┐
                │       Multi-source Raw Data            │
                │  (Melon ~26K · Gugak ~10K · KNU dict)  │
                └────────────────────┬───────────────────┘
                                     │
                ┌────────────────────┴────────────────────┐
                ▼                                         ▼
    ┌──────────────────────────┐             ┌──────────────────────────┐
    │  Modern Music Pipeline   │             │     Gugak Pipeline       │
    │  (Melon Top 100 chart)   │             │   (AI Hub 5 genres)      │
    └────────────┬─────────────┘             └────────────┬─────────────┘
                 │                                        │
                 ▼                                        ▼
        Combined Polarity                     Composite Gugak Score
    (KNU 한국어 + VADER 영어)                (가사 sentiment +
                 │                            Soul Score +
                 │                            Instrument·Rhythm weight)
                 │                                        │
                 └──────────────────┬─────────────────────┘
                                    ▼
                  artist_diff = |artist_sentiment − gugak_avg|
                                    │
                                    ▼
                    유사 가수·곡 Top-K + 장르별·연도별 분석
```

---

## 핵심 의사결정

### Decision 1 — Soul Score: 정성적 음악 특성을 정량 메트릭으로

국악의 *정서적 밀도*를 어떻게 수치화할 것인가가 첫 번째 도전이었다.

**도메인 관찰**:
- *시김새(sigimsae)*는 국악에서 멜로디나 리듬을 자연스럽게 하거나 장식하기 위한 다양한 표현 기법
- 시김새가 자주 등장할수록 곡의 정서적 밀도가 높다는 것이 음악학적 가정
- 단, *절대 빈도*만 보면 곡 길이에 영향 받음 → 정규화 필요

**설계**:
```
Soul_Score = sigimsae_count / duration
```

곡 길이로 정규화함으로써 짧은 곡과 긴 곡을 공정하게 비교 가능. AI Hub 국악 데이터의 annotation 정보(start_time, end_time, annotation_category)에서 시김새 카운트와 duration을 직접 추출.

> **방법론적 교훈**: 도메인 전문 개념을 데이터로 측정 가능한 형태로 변환할 때, *단순 카운트가 아니라 정규화 기준*까지 함께 설계해야 비교 가능한 신호가 된다.

---

### Decision 2 — 정서 점수 통합 설계: 가사 + Soul Score + 악기 가중치

국악의 정서를 단일 신호(가사 sentiment)만으로 측정하면 음악적 표현이 누락된다. 세 신호를 결합:

| 신호 | 측정 | 비중 |
|------|------|------|
| **가사 sentiment** | KNU 한국어 감성 사전 (SentiWord_info) 기반 polarity | 기본값 |
| **Soul Score** | 시김새 빈도 정규화 | 가중치로 추가 |
| **악기·장단** | 슬픔/흥분 계열: 1.5 / 기본: 1.2 / 디폴트: 1.0 | 곱셈 가중치 |

**왜 가중치 곱셈 방식?**
- 가사가 슬픈데 악기가 흥겨우면 정서 신호가 모호해짐
- 악기·장단은 *증폭기* 역할로 설계 → 기본 sentiment 위에 곱하는 형태
- 가중치 값은 도메인 자료 + 팀 내 정성 평가로 결정

> **교훈**: 다중 신호 결합 시 각 신호가 *어떤 역할*을 하는지 명확히 정의하고 그에 맞는 결합 방식을 골라야 한다. 단순 평균은 신호 간 의미가 동등할 때만 유효.

---

### Decision 3 — 데이터 전처리 (팀원 담당)

본인이 설계한 Soul Score와 정서 점수 통합 공식을 입력으로 받기 위해, 팀원 **이다혜**가 멜론·국악 데이터의 전처리를 담당.

**처리 내용 요약** (상세는 다혜 작업물 참조):
- 멜론 가사 sentiment 측정 — 한국어는 KNU 감성 사전, 영어는 NLTK VADER, 두 점수의 평균으로 `Combined_Avg_Polarity` 산출
- `Lexical_Diversity` (TTR), `Is_Korean_Song` 플래그 등 보조 지표 산출
- 국악 가사 띄어쓰기 보정 (Pykospacing) → KNU 사전 매칭 정확도 향상
- 5개 국악 장르 JSON → 단일 dataframe 통합

> 본 README의 Decision 1·2 (Soul Score 설계, 정서 점수 통합 공식)는 본인 담당, Decision 3 (전처리)은 팀원 담당으로 명확히 분리.

---

### Decision 4 — 유사도 측정: 절대값 거리

가수별·곡별 유사도를 어떻게 측정할 것인가.

**선택**: 통합 정서 점수의 **절대값 거리**

```
artist_diff = |artist_sentiment - weighted_gugak_avg|
```

**왜 거리인가**:
- 코사인 유사도는 *방향*만 보고 *크기*를 무시 → 이번 분석에선 sentiment의 절대 위치가 중요
- 점수 0.3471(국악 평균) 근처에 가까이 있을수록 정서적으로 유사
- 해석이 직관적: "0.0001 차이"가 "0.5 차이"보다 명확히 더 가까움

> **교훈**: 유사도 메트릭은 분석 목적에 따라 골라야 한다. 단어 임베딩처럼 *의미 방향*이 중요할 때는 코사인, 정서 점수처럼 *위치 자체*가 의미일 때는 거리.

---

## 주요 결과

### 결과 1 — 연도별 트렌드

2000-2010년대 K-pop은 sentiment 0.05~0.15 구간에서 진동.
**2020년 이후 sharp increase**, 2023년 ~0.25 도달.
국악 baseline 0.3471에 점차 수렴하는 경향.

→ 해석: 2020년대 K-pop이 정서적으로 국악에 더 가까워지고 있음. (이유 추정: 글로벌 시장에서 한국적 정체성 강조 트렌드)

### 결과 2 — 장르별 유사성

| 장르 | Sentiment | Gugak과의 거리 |
|------|-----------|----------------|
| Electronica (일렉트로니카) | 가장 가까움 | 최소 |
| R&B/Soul · 인디음악 | 중간 | 중간 |
| Rap/힙합 · 인디음악 | 음수 (negative) | 큼 |

흥미롭게도 **전통적 정서 표현이 강할 것 같은 발라드보다 Electronica가 국악에 더 가깝게 측정됨**. (가설: 일렉트로니카의 음향적 다양성이 국악 시김새의 표현적 다양성과 통계적 유사성을 만듦)

### 결과 3 — 가수 Top 3 (가장 국악 정서에 가까운)

| 순위 | 아티스트 | Score | 차이 |
|------|---------|-------|------|
| 1 | 유승우 | 0.3474 | 0.00021 |
| 2 | **Jennifer Lopez** | 0.3477 | 0.00052 |
| 3 | 김형중 | 0.3464 | 0.00078 |

**Jennifer Lopez가 2위**라는 점이 가장 흥미로운 발견. 한국 전통 정서와 통계적으로 가까운 해외 아티스트가 존재한다는 신호.

### 결과 4 — 곡 단위 매칭

- 유승우 *유후(U Who?)* ↔ 최호성 *판소리 춘향가* (diff: 0.013)
- Jennifer Lopez *Brave* ↔ 박복남 *자장가4* (diff: 0.0004)
- 김형중 *그랬나봐* ↔ 박복남 *자장가4* (diff: 0.0000)

---

## Tech Stack

| 영역 | 도구 |
|------|------|
| 언어/환경 | Python 3, Jupyter (Colab) |
| 데이터 처리 | pandas, numpy |
| 한국어 sentiment | KNU Korean Sentiment Dictionary (SentiWord_info.json) |
| 영어 sentiment | NLTK VADER |
| 한국어 띄어쓰기 보정 | Pykospacing |
| 통계·시각화 | scipy, matplotlib |
| 데이터 출처 | 멜론 Top 100 차트 (GitHub), AI Hub 국악 음원 데이터 |

---

## Lessons Learned

### 1. 도메인 개념 → 정량 메트릭 변환의 정규화 단계
시김새 빈도 자체로는 곡 길이에 따라 의미가 달라짐. *분모 설정(duration)* 이 도메인 메트릭의 비교 가능성을 좌우.

### 2. 다중 신호 결합 시 역할 정의
가사·Soul Score·악기 가중치는 각각 다른 역할(기본 신호 / 추가 신호 / 증폭기). 단순 평균이 아니라 *결합 방식 자체*가 설계 변수.

### 3. 다국어 corpus는 언어별 특화 도구로
KNU 사전과 VADER를 분리해 사용. 단일 도구로 두 언어를 처리하려는 욕심이 정확도 손실로 이어짐.

### 4. 유사도 메트릭은 분석 목적에 종속
방향(코사인) vs 거리(절대값)는 *측정하려는 것*에 따라 갈림. 메트릭 선택을 사후가 아니라 분석 설계 단계에서.

### 5. 예상 밖 결과를 *버그*가 아니라 *신호*로 해석
Jennifer Lopez가 국악 정서 Top 2로 나온 결과를 데이터 오류로 판단하기 쉬움. 그러나 가설을 검증한 결과 *실제 통계적 패턴*이었고, 이게 가장 흥미로운 발견이 됨.

---

## 향후 개선

- 시김새 외 추가 음악적 특성(장단, 농현 등) 정량화
- 가중치 값을 도메인 전문가 인터뷰로 보정
- 더 많은 해외 아티스트 데이터 추가 (현재 멜론 차트 위주라 K-pop bias)
- 음원 자체 분석(스펙트럼·MFCC) 추가로 가사·메타데이터 중심 한계 보완

---

## How to Run

```
.
├── README.md
├── melon_chart_code_final.ipynb     # 멜론 차트 sentiment 처리
├── STS2019_AIstein_Final_Project.ipynb  # 메인 분석 노트북
├── data/
│   ├── final_melon_chart_analysis.csv
│   ├── 국악_통합데이터_10개압축분.csv
│   └── SentiWord_info.json
```

### 환경
- Google Colab (또는 로컬 Jupyter)
- `pip install pandas numpy nltk pykospacing matplotlib`
- VADER lexicon: `nltk.download('vader_lexicon')`

### 실행 순서
1. `melon_chart_code_final.ipynb` 실행 — 멜론 sentiment 산출
2. `STS2019_AIstein_Final_Project.ipynb` 실행 — 국악 처리 + 비교 분석

---
## Data Sources & Acknowledgements

본 프로젝트는 다음 외부 자원을 활용했습니다.

- **Kpop Lyric Dataset** — [EX3exp/Kpop-lyric-datasets](https://github.com/EX3exp/Kpop-lyric-datasets)
  - 2000~2023년 K-pop 25,696곡의 가사·메타데이터 JSON 데이터셋
  - 멜론 차트 정서 분석의 입력 데이터로 사용
- **AI Hub 국악 음원 데이터** — [https://aihub.or.kr/aihubdata/data/view.do?dataSetSn=71470]
  - 5개 장르의 시김새 annotation, 가사, 악기·장단 메타데이터
- **KNU Korean Sentiment Dictionary (SentiWord_info.json)** — 한국어 가사 sentiment polarity 측정
- **NLTK VADER** — 영어 가사 sentiment compound score 측정
- **Pykospacing** — 국악 가사 띄어쓰기 자동 보정

---
## About this project

서강대학교 25년 1학기 빅데이터 마이닝 수업 final project로 진행한 3인 팀 프로젝트입니다.

### 팀 및 역할

데이터 파이프라인 순서로 정리하면 **재원 (raw → table) → 다혜 (table → 정서 신호) → 본인 (정서 신호 → 유사도 분석·해석)**.

- **이재원** — Raw data 정리
  - AI Hub 국악 데이터의 nested JSON 구조 파싱 및 dataframe 변환
  - 음절 단위로 분리된 가사를 곡 단위로 재결합
  - 분석 가능한 표 형태로 정리 (이후 다혜·본인 작업의 기반)
- **이다혜** — 정서 신호 산출
  - 멜론 sentiment 측정 (`melon_chart_code_final.ipynb`)
  - 한국어 KNU 감성 사전 + 영어 NLTK VADER 결합, `Combined_Avg_Polarity` 산출
  - `Lexical_Diversity` (TTR), `Is_Korean_Song` 플래그 등 보조 지표
  - 국악 가사 Pykospacing 보정으로 사전 매칭 정확도 향상
- **이승연** (본인) ([@leejulie05](https://github.com/leejulie05)) — 메트릭 설계 + 유사도 분석
  - 주제 발상
  - **Soul Score 메트릭 설계** (`sigimsae_count / duration`)
  - **정서 점수 가중치 공식 설계** (악기·장단별 1.5/1.2/1.0)
  - 유사도 분석 코드 (`STS2019_AIstein_Final_Project.ipynb`)
  - 결과 해석 (연도별 트렌드, 장르별 분석, Top 가수 도출)

본 저장소는 본인이 작성한 분석 코드 중심으로 정리되며, 팀원의 작업물(데이터 정리·전처리 노트북 등)은 동의하에 일부 포함됩니다.

---

*Last updated: 2026-04*
