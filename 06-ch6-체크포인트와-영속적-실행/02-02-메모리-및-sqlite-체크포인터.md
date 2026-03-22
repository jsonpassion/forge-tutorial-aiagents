# 메모리 및 SQLite 체크포인터

> InMemorySaver부터 SqliteSaver, PostgresSaver까지 — 체크포인터 구현체를 비교하고 영속적 상태 저장을 실습합니다.

## 개요

이 섹션에서는 LangGraph가 제공하는 세 가지 체크포인터 구현체를 깊이 있게 다룹니다. [이전 섹션](06-ch6-체크포인트와-영속적-실행/01-01-체크포인트-시스템-이해.md)에서 체크포인트의 아키텍처와 StateSnapshot 구조를 배웠다면, 이번에는 **어디에 저장할 것인가**라는 실전 질문에 답합니다.

**선수 지식**: 체크포인트 시스템의 기본 개념(슈퍼스텝, StateSnapshot, thread_id), StateGraph 컴파일과 `checkpointer` 파라미터

**학습 목표**:
- InMemorySaver의 내부 구조와 적합한 사용 시나리오를 설명할 수 있다
- SqliteSaver로 파일 기반 영속 체크포인트를 구현할 수 있다
- PostgresSaver의 프로덕션 설정 패턴을 이해할 수 있다
- 세 체크포인터의 성능·확장성·내구성 트레이드오프를 판단하여 적절히 선택할 수 있다

## 왜 알아야 할까?

개발 중에는 `InMemorySaver`로 충분합니다. 프로세스가 살아 있는 동안 상태가 유지되니까요. 하지만 현실 세계의 애플리케이션은 다릅니다. 서버가 재시작되면? 배포를 하면? 사용자가 다음 날 돌아오면? **메모리에만 의존하면 모든 대화 맥락이 사라집니다.**

프로덕션 에이전트는 반드시 체크포인트를 **영속 저장소**에 보관해야 합니다. SQLite는 단일 사용자 앱이나 프로토타입에 적합하고, PostgreSQL은 다중 사용자 프로덕션 환경의 표준입니다. 올바른 체크포인터를 선택하는 것은 에이전트의 **신뢰성과 사용자 경험**을 좌우하는 핵심 결정입니다.

## 핵심 개념

### 개념 1: 체크포인터 계층 구조

> 💡 **비유**: 체크포인터는 게임의 **저장 시스템**과 같습니다. InMemorySaver는 게임 중 RAM에만 저장하는 것 — 전원이 꺼지면 끝입니다. SqliteSaver는 로컬 디스크의 세이브 파일 — 내 컴퓨터에서는 안전하지만 다른 기기에서는 못 불러옵니다. PostgresSaver는 클라우드 세이브 — 어디서든, 누구든 접근할 수 있는 영속 저장소입니다.

모든 체크포인터는 `BaseCheckpointSaver` 추상 클래스를 상속합니다. 이 인터페이스는 다섯 가지 핵심 메서드를 정의하거든요:

```python
# 개념적 인터페이스 (실제 구현은 다를 수 있음)
# 실제 시그니처와 파라미터는 langgraph-checkpoint 버전에 따라 변경될 수 있습니다.
# 참고: from langgraph.checkpoint.base import BaseCheckpointSaver

class BaseCheckpointSaver:
    """체크포인터 공통 인터페이스 — 주요 메서드 개요"""
    def put(self, config, checkpoint, metadata, new_versions) -> RunnableConfig: ...
    def put_writes(self, config, writes, task_id) -> None: ...
    def get_tuple(self, config) -> CheckpointTuple | None: ...
    def list(self, config, *, filter=None, before=None, limit=None) -> Iterator: ...
    def delete_thread(self, thread_id: str) -> None: ...
    # 비동기 버전: aput, aput_writes, aget_tuple, alist, adelete_thread
```

`delete_thread`는 특정 스레드의 모든 체크포인트를 삭제하는 메서드입니다. 여기서는 인터페이스 소개 수준으로만 언급하고, 실제 활용법과 주의사항은 [03. 멀티 세션과 스레드 관리](06-ch6-체크포인트와-영속적-실행/03-03-멀티-세션과-스레드-관리.md)에서 자세히 다룹니다.

> 📊 **그림 1**: 체크포인터 클래스 계층 구조

```mermaid
classDiagram
    class BaseCheckpointSaver {
        +put(config, checkpoint, metadata)
        +get_tuple(config)
        +list(config)
        +delete_thread(thread_id)
        +aput() async
        +aget_tuple() async
    }
    class InMemorySaver {
        -storage: defaultdict
        -writes: defaultdict
        -blobs: dict
    }
    class SqliteSaver {
        -conn: sqlite3.Connection
        +from_conn_string(path)
    }
    class AsyncSqliteSaver {
        -conn: aiosqlite.Connection
        +from_conn_string(path)
    }
    class PostgresSaver {
        -conn: Connection or Pool
        +setup()
        +from_conn_string(uri)
    }
    class AsyncPostgresSaver {
        -conn: AsyncConnection
        +setup()
        +from_conn_string(uri)
    }
    BaseCheckpointSaver <|-- InMemorySaver
    BaseCheckpointSaver <|-- SqliteSaver
    BaseCheckpointSaver <|-- PostgresSaver
    SqliteSaver <|-- AsyncSqliteSaver
    PostgresSaver <|-- AsyncPostgresSaver
```

핵심은 **인터페이스가 동일**하다는 것입니다. `graph = builder.compile(checkpointer=checkpointer)` 한 줄만 바꾸면 저장소를 교체할 수 있어요. 개발 중에는 InMemorySaver, 배포 시에는 PostgresSaver로 전환하는 것이 일반적인 패턴입니다.

### 개념 2: InMemorySaver — 개발의 친구

> 💡 **비유**: InMemorySaver는 포스트잇에 메모하는 것과 같습니다. 빠르고 간편하지만, 책상을 치우면 (프로세스가 종료되면) 다 사라지죠.

InMemorySaver는 세 개의 딕셔너리로 상태를 관리합니다:

```python
from langgraph.checkpoint.memory import InMemorySaver

# InMemorySaver 내부 구조 (단순화)
# storage: thread_id → checkpoint_ns → checkpoint_id → 직렬화된 체크포인트
# writes: (thread_id, checkpoint_ns, checkpoint_id) → 쓰기 목록
# blobs: (thread_id, checkpoint_ns, channel, version) → 채널 데이터
```

사용법은 가장 간단합니다:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

# 체크포인터 생성 — 끝!
checkpointer = InMemorySaver()

# 그래프 컴파일 시 주입
graph = builder.compile(checkpointer=checkpointer)

# 실행 — thread_id로 세션 구분
config = {"configurable": {"thread_id": "user-123"}}
result = graph.invoke({"messages": [("human", "안녕!")]}, config)
```

> 📊 **그림 2**: InMemorySaver의 데이터 흐름

```mermaid
flowchart LR
    A["그래프 실행"] -->|"슈퍼스텝 완료"| B["InMemorySaver"]
    B --> C["storage 딕셔너리<br/>체크포인트 본체"]
    B --> D["writes 딕셔너리<br/>노드별 쓰기 기록"]
    B --> E["blobs 딕셔너리<br/>채널 데이터"]
    F["프로세스 종료"] -->|"모든 데이터 소멸"| C
    F -->|"모든 데이터 소멸"| D
    F -->|"모든 데이터 소멸"| E
```

InMemorySaver의 장점은 **속도**와 **단순성**입니다. 별도 설치도, 설정도 필요 없거든요. 비동기도 네이티브로 지원하기 때문에 `await graph.ainvoke(...)` 호출에도 별도의 비동기 체크포인터가 필요 없습니다. 단, 테스트와 개발 용도로만 사용해야 합니다.

> ⚠️ **흔한 오해**: "InMemorySaver는 느리다"라고 생각하는 분이 있는데, 사실은 정반대입니다. 메모리 딕셔너리 접근이므로 **가장 빠른 체크포인터**입니다. 다만 영속성이 없을 뿐이죠.

### 개념 3: SqliteSaver — 파일 하나로 영속화

> 💡 **비유**: SqliteSaver는 노트북에 메모하는 것과 같습니다. 포스트잇과 달리 노트북은 서랍에 넣어두면 다음에 꺼내 볼 수 있죠. 파일 하나에 모든 체크포인트가 저장되어, 프로세스를 재시작해도 상태가 살아 있습니다.

SqliteSaver는 별도 패키지로 제공됩니다:

```python
# 설치
# pip install langgraph-checkpoint-sqlite

from langgraph.checkpoint.sqlite import SqliteSaver

# 방법 1: from_conn_string (권장 — 컨텍스트 매니저)
with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "session-1"}}
    result = graph.invoke({"messages": [("human", "안녕!")]}, config)

# 방법 2: 직접 Connection 전달
import sqlite3
conn = sqlite3.connect("checkpoints.db")
checkpointer = SqliteSaver(conn)
```

`from_conn_string` 팩토리 메서드를 쓰면 연결 생성과 정리를 자동으로 처리해 줍니다. `":memory:"`를 전달하면 InMemorySaver처럼 메모리 SQLite를 사용할 수도 있어요.

> 📊 **그림 3**: SqliteSaver의 저장 구조

```mermaid
flowchart TD
    A["그래프 실행"] -->|"슈퍼스텝 완료"| B["SqliteSaver"]
    B -->|"직렬화 + INSERT"| C["checkpoints.db"]
    C --> D["checkpoints 테이블<br/>thread_id, checkpoint_id,<br/>parent_checkpoint_id, metadata"]
    C --> E["writes 테이블<br/>task_id, channel, type, blob"]
    C --> F["blobs 테이블<br/>channel, version, type, blob"]
    G["프로세스 재시작"] -->|"SELECT로 복원"| C
```

비동기 버전도 있습니다. FastAPI 같은 비동기 웹 프레임워크와 함께 쓸 때 유용하죠:

```python
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

async with AsyncSqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)
    result = await graph.ainvoke(
        {"messages": [("human", "비동기 실행!")]},
        {"configurable": {"thread_id": "async-1"}}
    )
```

SqliteSaver의 한계는 **동시 쓰기**입니다. SQLite는 기본적으로 단일 쓰기만 지원하기 때문에, 여러 사용자가 동시에 체크포인트를 저장하면 잠금 충돌이 발생할 수 있습니다. 단일 사용자 앱, 로컬 데모, CLI 도구에는 완벽하지만, 다중 사용자 웹 서비스에는 적합하지 않습니다.

### 개념 4: PostgresSaver — 프로덕션의 표준

> 💡 **비유**: PostgresSaver는 은행 금고에 보관하는 것과 같습니다. 단순히 내구성뿐 아니라, 여러 사람이 동시에 접근할 수 있고, 백업과 복제도 지원하는 전문적인 저장소죠.

PostgresSaver는 프로덕션 환경을 위해 설계되었습니다:

```python
# 설치
# pip install langgraph-checkpoint-postgres

from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:password@localhost:5432/mydb"

# 기본 사용법
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # 첫 실행 시 테이블 자동 생성 + 마이그레이션
    graph = builder.compile(checkpointer=checkpointer)
    result = graph.invoke(
        {"messages": [("human", "프로덕션 에이전트!")]},
        {"configurable": {"thread_id": "prod-session-1"}}
    )
```

프로덕션에서는 **커넥션 풀**을 사용하는 것이 필수입니다:

```python
from psycopg_pool import ConnectionPool
from langgraph.checkpoint.postgres import PostgresSaver

# 커넥션 풀로 동시 접근 처리
pool = ConnectionPool(
    conninfo="postgresql://user:pass@localhost:5432/mydb",
    max_size=20,       # 최대 동시 연결 수
    min_size=5,        # 최소 유지 연결 수
)
checkpointer = PostgresSaver(pool)
checkpointer.setup()

# 이제 여러 스레드/프로세스에서 안전하게 사용 가능
graph = builder.compile(checkpointer=checkpointer)
```

> 📊 **그림 4**: 프로덕션 체크포인터 아키텍처

```mermaid
flowchart LR
    U1["사용자 A"] --> S1["Worker 1"]
    U2["사용자 B"] --> S2["Worker 2"]
    U3["사용자 C"] --> S3["Worker 3"]
    S1 --> P["ConnectionPool<br/>max_size=20"]
    S2 --> P
    S3 --> P
    P --> DB["PostgreSQL<br/>체크포인트 DB"]
    DB --> R["복제본<br/>읽기 전용"]
    DB --> BK["백업<br/>장애 대비"]
```

직접 커넥션을 생성할 때는 주의할 점이 있습니다:

```python
from psycopg import Connection
from psycopg.rows import dict_row

# 수동 연결 시 필수 설정 두 가지!
conn = Connection.connect(
    DB_URI,
    autocommit=True,    # setup() 테이블 생성에 필요
    row_factory=dict_row # row["column"] 접근 방식에 필요
)
checkpointer = PostgresSaver(conn)
```

> 🔥 **실무 팁**: `from_conn_string`을 쓰면 `autocommit`과 `row_factory` 설정을 자동으로 해줍니다. 직접 커넥션을 만들 때만 이 두 옵션을 잊지 마세요. 빠뜨리면 "cannot execute in a transaction block" 또는 "tuple indices must be integers" 에러가 발생합니다.

### 개념 5: 체크포인터 비교와 선택 기준

세 체크포인터의 핵심 차이를 한눈에 비교해 봅시다:

> 📊 **그림 5**: 체크포인터 선택 의사결정 트리

```mermaid
flowchart TD
    Q1{"프로세스 재시작 후에도<br/>상태가 필요한가?"}
    Q1 -->|"아니오"| A["InMemorySaver<br/>개발/테스트용"]
    Q1 -->|"예"| Q2{"다중 사용자 또는<br/>분산 환경인가?"}
    Q2 -->|"아니오"| B["SqliteSaver<br/>단일 사용자/로컬 앱"]
    Q2 -->|"예"| Q3{"비동기 프레임워크<br/>사용 여부?"}
    Q3 -->|"동기"| C["PostgresSaver<br/>+ ConnectionPool"]
    Q3 -->|"비동기"| D["AsyncPostgresSaver<br/>+ AsyncConnectionPool"]

    style A fill:#4CAF50,color:#fff
    style B fill:#2196F3,color:#fff
    style C fill:#FF9800,color:#fff
    style D fill:#9C27B0,color:#fff
```

| 기준 | InMemorySaver | SqliteSaver | PostgresSaver |
|------|:---:|:---:|:---:|
| **영속성** | 없음 (RAM) | 파일 기반 | DB 기반 |
| **동시성** | 단일 프로세스 | 단일 쓰기 | 커넥션 풀 |
| **설치** | 기본 포함 | `pip install langgraph-checkpoint-sqlite` | `pip install langgraph-checkpoint-postgres` |
| **설정 복잡도** | 없음 | 낮음 | 중간 |
| **장애 복구** | 불가 | 파일 백업 | 복제·백업·PITR |
| **적합 환경** | 개발·테스트 | 로컬 앱·데모 | 프로덕션 |

## 실습: 직접 해보기

InMemorySaver와 SqliteSaver를 비교하는 실습입니다. 같은 그래프를 두 체크포인터로 실행하고, 프로세스 재시작 시뮬레이션을 통해 영속성의 차이를 확인합니다.

```python
"""체크포인터 비교 실습 — InMemorySaver vs SqliteSaver"""
import sqlite3
from typing import Annotated
from typing_extensions import TypedDict

from langchain_core.messages import HumanMessage, AIMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import InMemorySaver


# --- 1. 상태 스키마 정의 ---
class State(TypedDict):
    messages: Annotated[list, add_messages]  # 메시지 리듀서
    step_count: int                           # 실행 단계 카운터


# --- 2. 노드 함수 ---
def chatbot(state: State) -> dict:
    """간단한 에코 챗봇 노드"""
    last_msg = state["messages"][-1].content
    step = state.get("step_count", 0) + 1
    reply = f"[Step {step}] 입력하신 내용: '{last_msg}'"
    return {
        "messages": [AIMessage(content=reply)],
        "step_count": step,
    }


# --- 3. 그래프 빌드 ---
def build_graph(checkpointer):
    """체크포인터를 주입받아 그래프를 컴파일"""
    builder = StateGraph(State)
    builder.add_node("chatbot", chatbot)
    builder.add_edge(START, "chatbot")
    builder.add_edge("chatbot", END)
    return builder.compile(checkpointer=checkpointer)


# --- 4. InMemorySaver 테스트 ---
print("=" * 50)
print("InMemorySaver 테스트")
print("=" * 50)

memory_saver = InMemorySaver()
graph = build_graph(memory_saver)
config = {"configurable": {"thread_id": "test-thread"}}

# 첫 번째 메시지
result = graph.invoke(
    {"messages": [HumanMessage(content="안녕하세요")]},
    config,
)
print(f"응답: {result['messages'][-1].content}")
print(f"단계: {result['step_count']}")

# 두 번째 메시지 — 같은 스레드
result = graph.invoke(
    {"messages": [HumanMessage(content="두 번째 메시지")]},
    config,
)
print(f"응답: {result['messages'][-1].content}")
print(f"단계: {result['step_count']}")
print(f"총 메시지 수: {len(result['messages'])}")
```

```run:python
# 실행 결과 시뮬레이션
print("=" * 50)
print("InMemorySaver 테스트")
print("=" * 50)
print("응답: [Step 1] 입력하신 내용: '안녕하세요'")
print("단계: 1")
print("응답: [Step 2] 입력하신 내용: '두 번째 메시지'")
print("단계: 2")
print("총 메시지 수: 4")
```

```output
==================================================
InMemorySaver 테스트
==================================================
응답: [Step 1] 입력하신 내용: '안녕하세요'
단계: 1
응답: [Step 2] 입력하신 내용: '두 번째 메시지'
단계: 2
총 메시지 수: 4
```

이제 SqliteSaver로 영속성을 확인해 봅시다:

```python
"""SqliteSaver 영속성 테스트"""
from langgraph.checkpoint.sqlite import SqliteSaver

DB_PATH = "my_agent_checkpoints.db"

# --- 세션 1: 첫 번째 실행 ---
print("=" * 50)
print("SqliteSaver 세션 1 (첫 실행)")
print("=" * 50)

with SqliteSaver.from_conn_string(DB_PATH) as checkpointer:
    graph = build_graph(checkpointer)
    config = {"configurable": {"thread_id": "persistent-thread"}}

    result = graph.invoke(
        {"messages": [HumanMessage(content="SQLite에 저장해줘")]},
        config,
    )
    print(f"응답: {result['messages'][-1].content}")
    print(f"단계: {result['step_count']}")

# 여기서 컨텍스트 매니저가 종료 → 연결 닫힘
# 실제 프로덕션에서는 서버 재시작에 해당

# --- 세션 2: 재시작 후 복원 ---
print("\n" + "=" * 50)
print("SqliteSaver 세션 2 (재시작 후)")
print("=" * 50)

with SqliteSaver.from_conn_string(DB_PATH) as checkpointer:
    graph = build_graph(checkpointer)
    config = {"configurable": {"thread_id": "persistent-thread"}}

    # 이전 상태 확인
    state = graph.get_state(config)
    print(f"복원된 메시지 수: {len(state.values['messages'])}")
    print(f"복원된 단계: {state.values['step_count']}")

    # 이어서 대화 계속
    result = graph.invoke(
        {"messages": [HumanMessage(content="다시 돌아왔어!")]},
        config,
    )
    print(f"응답: {result['messages'][-1].content}")
    print(f"단계: {result['step_count']}")
    print(f"총 메시지 수: {len(result['messages'])}")
```

```run:python
# SqliteSaver 실행 결과 시뮬레이션
print("=" * 50)
print("SqliteSaver 세션 1 (첫 실행)")
print("=" * 50)
print("응답: [Step 1] 입력하신 내용: 'SQLite에 저장해줘'")
print("단계: 1")
print()
print("=" * 50)
print("SqliteSaver 세션 2 (재시작 후)")
print("=" * 50)
print("복원된 메시지 수: 2")
print("복원된 단계: 1")
print("응답: [Step 2] 입력하신 내용: '다시 돌아왔어!'")
print("단계: 2")
print("총 메시지 수: 4")
```

```output
==================================================
SqliteSaver 세션 1 (첫 실행)
==================================================
응답: [Step 1] 입력하신 내용: 'SQLite에 저장해줘'
단계: 1

==================================================
SqliteSaver 세션 2 (재시작 후)
==================================================
복원된 메시지 수: 2
복원된 단계: 1
응답: [Step 2] 입력하신 내용: '다시 돌아왔어!'
단계: 2
총 메시지 수: 4
```

핵심 포인트를 볼까요? InMemorySaver에서 `graph`를 새로 만들면 이전 상태가 사라지지만, SqliteSaver는 **같은 DB 파일만 열면 이전 대화를 그대로 이어갈 수 있습니다**. `thread_id`가 열쇠 역할을 하는 거죠.

체크포인트 히스토리를 조회하는 방법도 알아봅시다:

```python
"""체크포인트 히스토리 조회"""
with SqliteSaver.from_conn_string(DB_PATH) as checkpointer:
    graph = build_graph(checkpointer)
    config = {"configurable": {"thread_id": "persistent-thread"}}

    # 전체 히스토리 조회
    print("체크포인트 히스토리:")
    for i, state in enumerate(graph.get_state_history(config)):
        msg_count = len(state.values.get("messages", []))
        step = state.values.get("step_count", 0)
        source = state.metadata.get("source", "unknown")
        print(f"  [{i}] messages={msg_count}, step={step}, "
              f"source={source}, next={state.next}")
```

```run:python
# 히스토리 조회 결과 시뮬레이션
print("체크포인트 히스토리:")
print("  [0] messages=4, step=2, source=loop, next=()")
print("  [1] messages=3, step=1, source=input, next=('chatbot',)")
print("  [2] messages=2, step=1, source=loop, next=()")
print("  [3] messages=1, step=0, source=input, next=('chatbot',)")
```

```output
체크포인트 히스토리:
  [0] messages=4, step=2, source=loop, next=()
  [1] messages=3, step=1, source=input, next=('chatbot',)
  [2] messages=2, step=1, source=loop, next=()
  [3] messages=1, step=0, source=input, next=('chatbot',)
```

## 더 깊이 알아보기

### 체크포인터 패키지의 분리 역사

LangGraph 초기(v0.1)에는 모든 체크포인터가 `langgraph` 메인 패키지에 포함되어 있었습니다. 하지만 2024년 v0.2 릴리스에서 큰 변화가 있었는데요 — 체크포인터를 **별도 패키지로 분리**한 것입니다.

왜 분리했을까요? PostgreSQL을 사용하지 않는 개발자에게 `psycopg`를, SQLite 영속화가 필요 없는 개발자에게 `aiosqlite`를 강제 설치할 이유가 없었기 때문입니다. 이 결정으로 `langgraph` 코어 패키지는 가벼워졌고, 각 체크포인터는 독립적으로 버전 관리됩니다:

- `langgraph-checkpoint` (v4.0.1) — 기본 인터페이스 + InMemorySaver
- `langgraph-checkpoint-sqlite` (v3.0.3) — SQLite 구현체
- `langgraph-checkpoint-postgres` (v3.0.5) — PostgreSQL 구현체

이 분리 아키텍처 덕분에 커뮤니티 체크포인터도 등장했습니다. Redis(`langgraph-checkpoint-redis`), Couchbase, Amazon Bedrock AgentCore Memory 등 다양한 저장소를 지원하는 서드파티 패키지가 생태계를 넓히고 있죠.

### JsonPlusSerializer — 직렬화의 비밀

모든 체크포인터는 기본적으로 `JsonPlusSerializer`를 사용합니다. 일반적인 JSON 직렬화와 어떻게 다를까요? 이 직렬화기는 먼저 `ormsgpack`(MessagePack)으로 바이너리 직렬화를 시도하고, 실패하면 JSON으로 폴백합니다. LangChain의 `BaseMessage`, `datetime`, `Enum` 같은 복잡한 타입도 투명하게 처리하거든요.

> 💡 **알고 계셨나요?** `JsonPlusSerializer`라는 이름이 "JSON" + "Plus"인 이유는, JSON에서 지원하지 않는 Python 고유 타입(set, tuple, bytes, datetime 등)까지 "플러스"로 처리하기 때문입니다. 실제 내부에서는 JSON보다 빠른 MessagePack을 우선 사용하면서도, 호환성을 위해 JSON 폴백을 유지하는 실용적인 설계입니다.

## 흔한 오해와 팁

> ⚠️ **흔한 오해**: "SqliteSaver는 프로덕션에서 쓸 수 없다"고 단정짓는 경우가 많은데, 이는 맥락에 따라 다릅니다. 단일 사용자 데스크톱 앱이나 CLI 도구에서는 SqliteSaver가 오히려 최적입니다. 별도의 데이터베이스 서버 없이 파일 하나로 완전한 영속성을 제공하니까요. "프로덕션 부적합"은 **다중 사용자 웹 서비스** 맥락에서의 이야기입니다.

> 💡 **알고 계셨나요?**: InMemorySaver는 실제로 비동기를 네이티브로 지원합니다. 메모리 딕셔너리 접근은 I/O가 아니기 때문에, sync/async 어느 쪽으로 호출해도 동일하게 동작해요. 별도의 `AsyncInMemorySaver` 클래스가 없는 이유가 바로 이것입니다.

> 🔥 **실무 팁**: PostgresSaver를 쓸 때 `from_conn_string` 대신 직접 커넥션을 만든다면, **반드시** `autocommit=True`와 `row_factory=dict_row` 두 옵션을 설정하세요. `autocommit`이 없으면 `setup()` 호출 시 트랜잭션 에러가, `row_factory`가 없으면 행 접근 시 타입 에러가 발생합니다. 가능하면 `from_conn_string`을 쓰는 것이 안전합니다.

> 🔥 **실무 팁**: 개발 환경에서는 이런 패턴이 유용합니다 — 환경 변수로 체크포인터를 전환:
> ```python
> import os
> if os.getenv("ENV") == "production":
>     checkpointer = PostgresSaver.from_conn_string(os.getenv("DB_URI"))
> elif os.getenv("ENV") == "staging":
>     checkpointer = SqliteSaver.from_conn_string("staging.db")
> else:
>     checkpointer = InMemorySaver()
> ```

## 핵심 정리

| 개념 | 설명 |
|------|------|
| **BaseCheckpointSaver** | 모든 체크포인터의 추상 베이스 클래스. put/get_tuple/list/delete_thread 인터페이스 정의 |
| **InMemorySaver** | 메모리 딕셔너리 기반, 가장 빠르지만 프로세스 종료 시 데이터 소멸. 개발/테스트용 |
| **SqliteSaver** | 파일 기반 영속 저장. `from_conn_string("file.db")` 패턴. 단일 사용자에 적합 |
| **AsyncSqliteSaver** | SqliteSaver의 비동기 버전. `aiosqlite` 기반, FastAPI 등 비동기 프레임워크와 호환 |
| **PostgresSaver** | 프로덕션 표준. 커넥션 풀 지원, 동시 접근 안전, 장애 복구 가능 |
| **setup()** | PostgresSaver 전용. 첫 실행 시 체크포인트 테이블 생성 및 마이그레이션 수행 |
| **from_conn_string** | 팩토리 메서드. 연결 설정을 자동 처리하여 올바른 옵션 보장 |
| **커넥션 풀** | `psycopg_pool.ConnectionPool`로 다중 연결 관리. 프로덕션 필수 패턴 |
| **패키지 분리** | 체크포인터별 독립 패키지: `langgraph-checkpoint-sqlite`, `langgraph-checkpoint-postgres` |

## 다음 섹션 미리보기

체크포인터의 종류와 설정을 마스터했으니, 다음 섹션 [03. 멀티 세션과 스레드 관리](06-ch6-체크포인트와-영속적-실행/03-03-멀티-세션과-스레드-관리.md)에서는 `thread_id`를 활용한 **멀티 세션 관리** 패턴을 배웁니다. 여러 사용자가 각자의 대화를 독립적으로 유지하는 방법, 스레드 목록 조회와 삭제, 그리고 네임스페이스(`checkpoint_ns`)를 활용한 고급 세션 분류까지 다룰 예정입니다.

## 참고 자료

- [LangGraph Persistence 공식 문서](https://docs.langchain.com/oss/python/langgraph/persistence) - 체크포인터 설정과 사용법의 공식 가이드
- [langgraph-checkpoint-sqlite (PyPI)](https://pypi.org/project/langgraph-checkpoint-sqlite/) - SqliteSaver 패키지 정보와 설치 가이드
- [langgraph-checkpoint-postgres (PyPI)](https://pypi.org/project/langgraph-checkpoint-postgres/) - PostgresSaver 패키지 정보와 버전 히스토리
- [LangGraph GitHub Repository](https://github.com/langchain-ai/langgraph) - 체크포인터 소스 코드와 예제
- [Couchbase LangGraph Persistence Tutorial](https://developer.couchbase.com/tutorial-langgraph-persistence-checkpoint/) - 커스텀 체크포인터를 포함한 영속화 튜토리얼
- [LangGraph Checkpointing Best Practices](https://sparkco.ai/blog/mastering-langgraph-checkpointing-best-practices-for-2025) - 체크포인터 실전 운영 가이드

---

---
### 🔗 Related Sessions
- [checkpoint](06-ch6-체크포인트와-영속적-실행/01-01-체크포인트-시스템-이해.md) (prerequisite)
- [stategraph](04-ch4-langgraph-stategraph-기초/01-01-langgraph-아키텍처-개관.md) (prerequisite)
- [thread_id](06-ch6-체크포인트와-영속적-실행/01-01-체크포인트-시스템-이해.md) (prerequisite)
- [compile()](04-ch4-langgraph-stategraph-기초/01-01-langgraph-아키텍처-개관.md) (prerequisite)
- [super-step](06-ch6-체크포인트와-영속적-실행/01-01-체크포인트-시스템-이해.md) (prerequisite)
- [statesnapshot](06-ch6-체크포인트와-영속적-실행/01-01-체크포인트-시스템-이해.md) (prerequisite)
- [checkpoint_id](06-ch6-체크포인트와-영속적-실행/01-01-체크포인트-시스템-이해.md) (prerequisite)
