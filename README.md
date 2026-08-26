# 📖 StoryMate — 소설 캐릭터 RAG 챗봇

공개된 소설 텍스트를 기반으로 소설 속 캐릭터와 대화하고,
작품 관련 퀴즈를 풀 수 있는 RAG 기반 챗봇입니다.

---

## 📂 프로젝트 구조

```
storymate/
├── app.py              # Flask API
├── chatbot_sql.py      # 챗봇 로직 (MariaDB 연동)
├── chatbot.py          # 챗봇 로직 (로컬 테스트용)
├── utils.py            # Chroma DB, Retriever, LLM 설정
├── template.py         # 캐릭터별 프롬프트
├── character.py        # 캐릭터 및 퀴즈 데이터
├── quiz.py             # 퀴즈 출제·채점
├── requirements.txt
├── Dockerfile
└── [책 제목]/
    └── [캐릭터명]/
        └── data/
            ├── 전문.txt
            ├── 인물평가.txt
            ├── 인물특성.txt
            ├── 예상질문.txt
            └── embedding/    # Chroma Vector DB
```

챗봇 로직과 API는 최상위 파일에서 관리하고,
책·캐릭터별 텍스트와 Vector DB는 하위 디렉터리에 분리해 저장했습니다.

---

## 🗂️ 오프라인 인덱싱

소설 관련 데이터를 전문 / 인물평가 / 인물특성 / 예상질문 네 종류로 나눠 관리했습니다.
각 텍스트를 `chunk_size=100`으로 분할하고 OpenAI Embeddings로 임베딩한 뒤,
책×캐릭터별로 4개의 Chroma Vector DB를 생성했습니다.

```
┌──────────┬──────────┬──────────┬──────────┐
│   전문   │ 인물평가 │ 인물특성 │ 예상질문 │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┘
     └──────────┴──────────┴──────────┘
                     │
              chunk_size = 100
                     │
              OpenAI Embeddings
                     │
             Chroma Vector DB 4종
           (책 × 캐릭터별 persist)
```

---

## 🤖 런타임 RAG 응답 생성

사용자 요청에는 `session_id`, `query`, `character_name`이 포함됩니다.
질의가 들어오면 전문·인물평가·인물특성·예상질문 4개의 Vector DB에서
각각 관련 문서를 검색합니다. Retriever에는 `similarity_score_threshold`를
사용하고, score 0.7 미만의 문서는 제외합니다.

이후 MariaDB의 `conversations`에 저장된 이전 대화를 조회하고 LLM으로 요약합니다.
최종적으로 4개 Vector DB의 검색 결과, 캐릭터 설정, 이전 대화 요약을 하나의
프롬프트에 구성해 GPT-4o에 전달합니다. 생성된 답변은 MariaDB에 저장한 뒤
API 응답으로 반환합니다.

```
              사용자 질문
     (session_id, query, character_name)
                     │
    ┌────────┬────────┬────────┬────────┐
    │  전문  │인물평가│인물특성│예상질문│
    └───┬────┴───┬────┴───┬────┴───┬────┘
        └────────┴────────┴────────┘
                     │
       각 Vector DB에서 관련 문서 검색
       similarity_score_threshold ≥ 0.7
                     │
              이전 대화 조회 및 요약
                (MariaDB conversations)
                     │
             검색 결과 + 캐릭터 설정
                + 이전 대화 요약
                     │
                  GPT-4o
                     │
               MariaDB 저장
                     │
                 응답 반환
```

---

## ✅ 구현 내용

- 전문·인물평가·인물특성·예상질문을 각각 별도의 Chroma Vector DB로 구성
- OpenAI Embeddings를 사용한 텍스트 임베딩
- `similarity_score_threshold` 기반 Retriever 사용, score 0.7 미만 문서 제외
- 4개 Vector DB에서 관련 문서를 각각 검색
- 검색 결과와 캐릭터 설정을 GPT-4o 프롬프트에 포함
- MariaDB에 세션별 대화 이력 저장
- 이전 대화를 요약해 후속 응답에 포함
- Flask API를 통한 세션 기반 멀티턴 대화
- OX·객관식·서술형 퀴즈 출제 및 채점 기능 구현
