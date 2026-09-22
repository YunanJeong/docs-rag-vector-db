# docs-rag-vector-db

마크다운 문서를 벡터 DB 에 넣고 자연어로 관련 내용을 찾는, 최소 구성의 **RAG 파이프라인**.

RAG(Retrieval-Augmented Generation)는 LLM 에게 질문을 던질 때 관련 문서를 함께 넣어주는
방식이다. 모델이 모르는 사내 문서를 근거로 답하게 만들 수 있다. 이때 "어떤 문서가 관련
있는지" 찾아내는 앞단이 이 프로젝트다.

키워드가 아니라 의미로 찾기 때문에 문서에 없는 표현으로 물어도 걸린다.
예를 들어 `워커 늘리는 법` 으로 검색하면 `노드그룹 desired 조정` 문서가 나온다.

찾은 문서 조각을 그대로 돌려준다. LLM 호출은 하지 않는다.

## 필요한 것

- [uv](https://docs.astral.sh/uv/)
- Docker
- 디스크 3GB (임베딩 모델 bge-m3 를 최초 실행 시 내려받는다)

## 사용법

```bash
# 1. Qdrant 기동
docker run -d --name rag-qdrant -p 6333:6333 -v qdrant_storage:/qdrant/storage qdrant/qdrant

# 2. 의존성 설치
uv sync

# 3. 색인
DOCS_DIR=tests/fixtures uv run index.py

# 4. 검색
uv run search.py "워커 늘리는 법"
uv run search.py "kr-mum" --top-k 3
```

`DOCS_DIR` 을 자기 문서 디렉터리로 바꾸면 그대로 쓸 수 있다.
색인은 매번 전량 다시 넣는다.

### 정리

```bash
docker stop rag-qdrant           # 중지 (데이터 유지, docker start 로 재개)
docker rm -f rag-qdrant          # 컨테이너 삭제 (데이터는 볼륨에 남음)
docker volume rm qdrant_storage  # 색인 데이터까지 삭제
```

## 환경변수

| 변수 | 기본값 | 용도 |
|---|---|---|
| `DOCS_DIR` | (필수) | 색인할 md 디렉터리 |
| `QDRANT_URL` | `http://localhost:6333` | 원격 Qdrant 를 쓸 때 이 값만 바꾼다 |
| `QDRANT_COLLECTION` | `docs` | |
| `EMBED_MODEL` | `BAAI/bge-m3` | |
| `TOP_K` | `5` | 검색 결과 개수 |

## 구조

```
md 파일 → 청킹 → 임베딩 → Qdrant 저장 → 질의 임베딩 → 검색 → 관련 조각
```

| 파일 | 하는 일 |
|---|---|
| `rag/chunk.py` | md 를 헤더 단위로 자른다. 조각 앞에 `[파일 > 헤딩]` 한 줄을 붙여 맥락을 남긴다 |
| `rag/embed.py` | bge-m3 로 텍스트를 벡터로 바꾼다 |
| `rag/store.py` | Qdrant 컬렉션 생성 / 저장 / 검색 |
| `index.py` | `DOCS_DIR` 아래 md 를 전부 읽어 색인한다 |
| `search.py` | 질의하고 결과를 출력한다 |

검색은 **dense**(의미가 비슷하면 찾음)와 **sparse**(글자가 정확히 맞아야 찾음) 두 방식을
각각 돌려 순위를 합친다. dense 는 `kr-mum` 과 `kr-sum` 을 구별하지 못하고, sparse 는
표현이 다르면 못 찾는다. 약점이 반대라서 둘 다 쓴다.

## 테스트

```bash
uv run pytest -q
```

청킹 테스트만 있다. 모델과 Qdrant 없이 돈다.
