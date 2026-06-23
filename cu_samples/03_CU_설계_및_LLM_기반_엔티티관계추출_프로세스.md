# 구술채록 온톨로지 구축을 위한 CU(Content Unit) 분할 및 LLM 추출 프로세스

## 1. 개요

본 연구의 목적은 구술채록 데이터를 기반으로 엔티티(Entity) 및 관계(Relation)를 자동 추출하여 RDF/OWL 기반 온톨로지 및 지식그래프(Knowledge Graph)를 구축하는 것이다.

구술채록은 일반 문서와 달리 장시간 인터뷰 형식으로 구성되어 있으며, 인물·사건·장소·작품 등이 여러 페이지에 걸쳐 반복적으로 등장한다.

따라서 단순 페이지 단위 분석이 아닌, 문맥(Context)을 유지할 수 있는 구조적 단위(Content Unit, CU) 설계가 필요하다.

---

# 2. CU(Content Unit) 분할의 필요성

## 2.1 페이지 단위 처리의 문제점

구술채록은 특정 인물 또는 사건에 대한 설명이 수십 페이지에 걸쳐 지속되는 경우가 많다.

예시

### Page 10

> 윤이상을 처음 만났다.

### Page 20

> 그분이 독일 유학을 권유했다.

### Page 35

> 그 영향으로 독일 유학을 가게 되었다.

페이지 단위로 LLM에 입력할 경우:

* 윤이상이 누구인지 알 수 없음
* 유학 권유와 유학 결정의 관계를 연결하지 못함
* 영향 관계 추출 실패 가능성 증가

---

## 2.2 CU 기반 처리의 장점

동일한 주제 내에서 문맥을 유지할 수 있다.

예시

```text
강석희
 └ 1회차
     ├ CU01 어린 시절
     ├ CU02 가족
     ├ CU03 학창 시절
     ├ CU04 윤이상과의 만남
     ├ CU05 독일 유학
```

CU 단위에서는 관련 문단들이 함께 존재하므로:

* 엔티티 연결 가능
* 관계 추론 가능
* 장기 문맥 유지 가능

---

# 3. OCR 원본 데이터의 문제점

현재 OCR 결과

```text
page001.txt
page002.txt
page003.txt
...
page047.txt
```

또는

```json
{
  "page": 1,
  "content": "..."
}
```

형태로 존재한다.

문제점:

* 주제가 페이지 중간에 변경됨
* 인물 설명이 여러 페이지에 분산됨
* 관계 추출 시 근거 문장이 분리됨

따라서 OCR 페이지 단위를 그대로 LLM 입력으로 사용하는 것은 적절하지 않다.

---

# 4. CU 분할 기준

## 4.1 페이지 기준 분할

예시

```text
1~10 페이지
11~20 페이지
21~30 페이지
```

### 장점

* 구현이 단순함

### 단점

* 주제가 중간에 끊어짐
* 관계 추출 성능 저하

### 결론

비추천

---

## 4.2 주제 기준 분할

예시

```text
어린 시절
가족
학창 시절
서울대학교
윤이상
독일 유학
```

### 장점

* 문맥 유지
* 엔티티 연결 용이
* 관계 추출 성능 향상

### 결론

추천

---

# 5. 최종 CU 구조

```json
{
  "cu_id": "CU01_04",
  "title": "윤이상과의 만남",
  "start_page": 35,
  "end_page": 42,
  "summary": "윤이상과의 첫 만남과 음악적 영향",
  "previous_cu": "CU01_03",
  "next_cu": "CU01_05",
  "paragraphs": []
}
```

---

# 6. 데이터 구조 설계

## 6.1 페이지 JSON

```json
{
  "page": 3,
  "speakers_detected": [
    "강",
    "서"
  ],
  "content": "..."
}
```

역할

* 원본 보존
* 페이지 정보 유지

---

## 6.2 발화 단위 JSON

```json
{
  "utterance_id": "p003_u001",
  "speaker": "강석희",
  "text": "..."
}
```

역할

* 화자 구분
* 인터뷰 흐름 유지

---

## 6.3 의미 문단 JSON

```json
{
  "paragraph_id": "P0014",
  "speaker": "강석희",
  "start_page": 3,
  "end_page": 4,
  "page_span": [3,4],
  "text": "..."
}
```

역할

* 의미 단위 구성
* 엔티티 및 관계 추출의 기본 단위

---

## 6.4 CU JSON

```json
{
  "cu_id": "CU01_04",
  "title": "윤이상과의 만남",
  "paragraphs": [
    "P010",
    "P011",
    "P012"
  ]
}
```

역할

* 주제 단위 문맥 유지
* LLM 입력 단위

---

# 7. LLM 입력 전략

## 7.1 문단 단독 입력 문제

예시

### P003

```text
어머니가 책을 노래처럼 읽어주었다.
```

### P007

```text
그 영향으로 작곡을 하게 되었다.
```

각 문단을 독립적으로 분석하면:

* 영향의 원인을 파악하기 어려움
* 관계 추출 실패 가능성 증가

---

## 7.2 Context Window 적용

추천 방식

```text
현재 문단
+ 이전 2개 문단
+ 이후 2개 문단
```

예시

```text
P001
P002
P003 ← 분석 대상
P004
P005
```

---

# 8. 엔티티 추출 프로세스

## 입력

CU + Context Window

## 출력

```json
{
  "entities": [
    {
      "id": "person_윤이상",
      "type": "Person",
      "label": "윤이상"
    }
  ]
}
```

## 주요 엔티티 유형

* Person
* Organization
* Event
* Place
* Work
* Concept

---

# 9. 관계 추출 프로세스

## 입력

엔티티 추출 결과 + 문맥

## 출력

```json
{
  "subject": "person_강석희",
  "predicate": "influencedBy",
  "object": "person_윤이상"
}
```

---

## 관계 예시

```text
influencedBy
metWith
studiedAt
workedAt
participatedIn
created
memberOf
```

---

# 10. 엔티티 레졸루션(Entity Resolution)

동일 인물 통합

예시

```text
윤이상
↓
person_윤이상
```

```text
나까무라
나카무라
↓
person_나카무라
```

---

# 11. RDF/OWL 변환

## 입력

```json
Entity
Relation
```

## 출력

```ttl
:강석희
    :influencedBy
    :윤이상 .
```

---

# 12. 최종 파이프라인

```text
PNG/PDF
↓
OCR
↓
페이지 JSON
↓
발화자 JSON
↓
의미 문단 JSON
↓
주제별 CU 생성
↓
Context Window 생성
↓
Entity Extraction
↓
Relation Extraction
↓
Entity Resolution
↓
RDF/OWL 생성
↓
Knowledge Graph 저장
```

---

# 13. 권장 구현 구조

```text
data/
├── raw
│   ├── png
│   ├── pdf
│   └── txt
│
├── page_json
│
├── utterance_json
│
├── paragraph_json
│
├── cu_json
│
├── entity_json
│
├── relation_json
│
├── rdf
│
└── graph
```

---

# 14. 결론

구술채록 데이터의 엔티티 및 관계 추출 품질을 높이기 위해서는 다음과 같은 계층적 구조가 필요하다.

```text
페이지
↓
발화자
↓
의미 문단
↓
CU(주제)
↓
Context Window
↓
Entity Extraction
↓
Relation Extraction
↓
Entity Resolution
↓
RDF/OWL
↓
Knowledge Graph
```

특히 **페이지 → 발화자 → 의미문단 → CU** 의 4단계 구조는 구술채록 데이터의 문맥을 유지하면서 LLM의 이해도와 추출 성능을 극대화할 수 있는 핵심 구조이다.
