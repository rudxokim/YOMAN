# Causality Tracking Algorithms - Implementation Guide

> **목적**: YOMAN에 적용 가능한 causality tracking 알고리즘 상세 구현

---

## 1. Vector Clock 상세 구현

### 1.1 Python 구현 (Production-Ready)

```python
from typing import Dict, Optional, Set
from dataclasses import dataclass, field
from copy import deepcopy
import json

@dataclass
class VectorClock:
    """
    Vector Clock implementation for causal ordering.

    Properties:
    - Tracks logical time per node
    - Detects concurrent operations
    - Merges clocks from different nodes
    """
    clocks: Dict[str, int] = field(default_factory=dict)

    def increment(self, node_id: str) -> None:
        """로컬 노드의 clock 증가"""
        self.clocks[node_id] = self.clocks.get(node_id, 0) + 1

    def merge(self, other: 'VectorClock') -> None:
        """다른 vector clock과 병합 (element-wise max)"""
        all_nodes = set(self.clocks.keys()) | set(other.clocks.keys())
        for node in all_nodes:
            self.clocks[node] = max(
                self.clocks.get(node, 0),
                other.clocks.get(node, 0)
            )

    def compare(self, other: 'VectorClock') -> str:
        """
        다른 vector clock과 비교

        Returns:
        - 'BEFORE': self < other (self가 인과적으로 이전)
        - 'AFTER': self > other (self가 인과적으로 이후)
        - 'CONCURRENT': 동시 발생 (causal 관계 없음)
        - 'EQUAL': 동일
        """
        all_nodes = set(self.clocks.keys()) | set(other.clocks.keys())

        less_or_equal = True
        greater_or_equal = True

        for node in all_nodes:
            self_val = self.clocks.get(node, 0)
            other_val = other.clocks.get(node, 0)

            if self_val > other_val:
                less_or_equal = False
            if self_val < other_val:
                greater_or_equal = False

        if less_or_equal and greater_or_equal:
            return 'EQUAL'
        elif less_or_equal:
            return 'BEFORE'
        elif greater_or_equal:
            return 'AFTER'
        else:
            return 'CONCURRENT'

    def copy(self) -> 'VectorClock':
        """Deep copy"""
        return VectorClock(clocks=deepcopy(self.clocks))

    def to_json(self) -> str:
        """JSON 직렬화"""
        return json.dumps(self.clocks)

    @classmethod
    def from_json(cls, json_str: str) -> 'VectorClock':
        """JSON 역직렬화"""
        clocks = json.loads(json_str)
        return cls(clocks=clocks)

    def __repr__(self) -> str:
        return f"VectorClock({self.clocks})"
```

### 1.2 YOMAN Operation 클래스

```python
from uuid import uuid4
from datetime import datetime
from enum import Enum

class BucketType(Enum):
    IDEA = "IDEA"
    RESEARCH = "RESEARCH"
    TODO = "TODO"
    STUDY = "STUDY"
    PR = "PR"

@dataclass
class Dependency:
    """Operation 의존성"""
    op_id: str
    version: int
    bucket: BucketType

@dataclass
class Operation:
    """
    YOMAN의 기본 작업 단위

    Causal consistency를 위한 메타데이터 포함
    """
    id: str = field(default_factory=lambda: str(uuid4()))
    project_id: str = ""
    bucket: BucketType = BucketType.IDEA
    version: int = 1
    vector_clock: VectorClock = field(default_factory=VectorClock)
    dependencies: list[Dependency] = field(default_factory=list)
    content: dict = field(default_factory=dict)
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    node_id: str = "default_node"  # AI 모델 또는 처리 노드

    def depends_on(self, other: 'Operation') -> None:
        """
        다른 Operation에 대한 의존성 추가

        이 메서드는:
        1. 의존성 리스트에 추가
        2. Vector clock 병합
        3. 자신의 clock 증가
        """
        # 의존성 추가
        dep = Dependency(
            op_id=other.id,
            version=other.version,
            bucket=other.bucket
        )
        self.dependencies.append(dep)

        # Vector clock 업데이트
        self.vector_clock.merge(other.vector_clock)
        self.vector_clock.increment(self.node_id)

        self.updated_at = datetime.now()

    def can_apply(self, state: 'OperationState') -> bool:
        """
        모든 의존성이 충족되었는지 확인

        Args:
            state: 현재 시스템 상태

        Returns:
            True if all dependencies are satisfied
        """
        for dep in self.dependencies:
            if not state.has_version(dep.op_id, dep.version):
                return False
        return True

    def to_dict(self) -> dict:
        """Supabase 저장용 딕셔너리 변환"""
        return {
            "id": self.id,
            "project_id": self.project_id,
            "bucket": self.bucket.value,
            "version": self.version,
            "vector_clock": self.vector_clock.to_json(),
            "dependencies": [
                {"op_id": d.op_id, "version": d.version, "bucket": d.bucket.value}
                for d in self.dependencies
            ],
            "content": self.content,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat(),
            "node_id": self.node_id
        }
```

### 1.3 OperationState (시스템 상태 관리)

```python
class OperationState:
    """
    전체 시스템 상태 관리

    - 적용된 operations 추적
    - 의존성 충족 여부 확인
    - Causal ordering 보장
    """

    def __init__(self):
        # op_id → {version: Operation}
        self.operations: Dict[str, Dict[int, Operation]] = {}

        # 대기 중인 operations (의존성 미충족)
        self.pending: Set[Operation] = set()

        # 현재 시스템의 vector clock
        self.system_clock = VectorClock()

    def has_version(self, op_id: str, version: int) -> bool:
        """특정 버전의 operation이 적용되었는지 확인"""
        return (
            op_id in self.operations and
            version in self.operations[op_id]
        )

    def apply(self, op: Operation) -> bool:
        """
        Operation 적용 시도

        Returns:
            True if applied, False if pending (dependencies not met)
        """
        # 의존성 확인
        if not op.can_apply(self):
            self.pending.add(op)
            return False

        # 적용
        if op.id not in self.operations:
            self.operations[op.id] = {}

        self.operations[op.id][op.version] = op

        # 시스템 clock 업데이트
        self.system_clock.merge(op.vector_clock)

        # 대기 중인 operations 재시도
        self._retry_pending()

        return True

    def _retry_pending(self) -> None:
        """대기 중인 operations 중 적용 가능한 것 처리"""
        applied = set()
        for op in self.pending:
            if op.can_apply(self):
                self.apply(op)
                applied.add(op)

        self.pending -= applied

    def get_dependencies_graph(self, op_id: str) -> Dict[str, Set[str]]:
        """Operation의 전체 의존성 그래프 반환"""
        graph = {}
        visited = set()

        def dfs(current_id: str):
            if current_id in visited:
                return
            visited.add(current_id)

            if current_id not in self.operations:
                return

            # 최신 버전 가져오기
            versions = self.operations[current_id]
            latest = max(versions.keys())
            op = versions[latest]

            deps = {dep.op_id for dep in op.dependencies}
            graph[current_id] = deps

            for dep_id in deps:
                dfs(dep_id)

        dfs(op_id)
        return graph
```

---

## 2. COPS-style Dependency Tracking

### 2.1 Causal+ Consistency 구현

```python
@dataclass
class CausalVersion:
    """COPS-style versioned value with dependencies"""
    key: str
    value: any
    version: int
    vector_clock: VectorClock
    dependencies: list[Dependency]
    timestamp: datetime = field(default_factory=datetime.now)

class CausalStore:
    """
    COPS-inspired causal+ consistency store

    Guarantees:
    - Causal consistency (happens-before relations)
    - Read-your-writes
    - Monotonic reads
    """

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.store: Dict[str, list[CausalVersion]] = {}
        self.my_clock = VectorClock()

        # Read-your-writes를 위한 클라이언트별 클럭
        self.client_clocks: Dict[str, VectorClock] = {}

    def write(
        self,
        key: str,
        value: any,
        client_id: str,
        deps: Optional[list[Dependency]] = None
    ) -> CausalVersion:
        """
        Write operation with causal dependencies

        Args:
            key: 키
            value: 값
            client_id: 클라이언트 식별자
            deps: 명시적 의존성 (선택)

        Returns:
            생성된 CausalVersion
        """
        # 클라이언트 clock 업데이트
        if client_id not in self.client_clocks:
            self.client_clocks[client_id] = VectorClock()

        client_clock = self.client_clocks[client_id]
        client_clock.increment(self.node_id)

        # 시스템 clock 업데이트
        self.my_clock.merge(client_clock)
        self.my_clock.increment(self.node_id)

        # 버전 생성
        if key not in self.store:
            self.store[key] = []

        version_num = len(self.store[key]) + 1

        causal_version = CausalVersion(
            key=key,
            value=value,
            version=version_num,
            vector_clock=self.my_clock.copy(),
            dependencies=deps or []
        )

        self.store[key].append(causal_version)

        return causal_version

    def read(
        self,
        key: str,
        client_id: str,
        version: Optional[int] = None
    ) -> Optional[any]:
        """
        Read operation with causal consistency

        Args:
            key: 키
            client_id: 클라이언트 식별자
            version: 특정 버전 (없으면 최신)

        Returns:
            값 (없으면 None)
        """
        if key not in self.store:
            return None

        versions = self.store[key]

        if version is None:
            # 최신 버전
            target = versions[-1]
        else:
            # 특정 버전
            target = next((v for v in versions if v.version == version), None)
            if target is None:
                return None

        # 의존성 먼저 충족 (COPS 핵심)
        self._ensure_dependencies(target.dependencies)

        # 클라이언트 clock 업데이트 (Read-your-writes)
        if client_id not in self.client_clocks:
            self.client_clocks[client_id] = VectorClock()

        self.client_clocks[client_id].merge(target.vector_clock)

        return target.value

    def _ensure_dependencies(self, deps: list[Dependency]) -> None:
        """
        의존성 충족 대기

        실제 구현에서는:
        - 네트워크 통신으로 다른 노드에서 가져오기
        - 타임아웃 설정
        - 재시도 로직
        """
        for dep in deps:
            # 의존성 버전이 로컬에 있는지 확인
            if dep.op_id not in self.store:
                # 실제로는 remote fetch
                self._fetch_remote(dep.op_id, dep.version)
                continue

            versions = self.store[dep.op_id]
            has_version = any(v.version == dep.version for v in versions)

            if not has_version:
                self._fetch_remote(dep.op_id, dep.version)

    def _fetch_remote(self, op_id: str, version: int) -> None:
        """
        다른 노드에서 operation 가져오기

        실제 구현 시:
        - HTTP/gRPC 요청
        - 재시도 + 타임아웃
        - 캐싱
        """
        # Placeholder for network fetch
        pass
```

### 2.2 YOMAN Bucket에 적용

```python
class BucketCausalStore:
    """
    YOMAN 버킷 시스템에 causal consistency 적용

    버킷 간 이동:
    - IDEA → RESEARCH: IDEA가 dependency
    - RESEARCH → TODO: RESEARCH가 dependency
    - TODO → PR: TODO가 dependency
    """

    def __init__(self):
        self.store = CausalStore(node_id="yoman_server")
        self.state = OperationState()

    def create_idea(
        self,
        project_id: str,
        content: dict,
        client_id: str
    ) -> Operation:
        """IDEA 버킷에 새 아이디어 생성"""
        op = Operation(
            project_id=project_id,
            bucket=BucketType.IDEA,
            content=content,
            node_id=client_id
        )

        # Vector clock 초기화
        op.vector_clock.increment(client_id)

        # Store에 쓰기
        self.store.write(
            key=op.id,
            value=op.to_dict(),
            client_id=client_id,
            deps=[]  # IDEA는 의존성 없음
        )

        # State 업데이트
        self.state.apply(op)

        return op

    def create_research(
        self,
        idea_id: str,
        content: dict,
        client_id: str
    ) -> Operation:
        """RESEARCH 버킷 생성 (IDEA에 의존)"""
        # IDEA operation 읽기
        idea_dict = self.store.read(idea_id, client_id)
        if not idea_dict:
            raise ValueError(f"IDEA {idea_id} not found")

        # IDEA operation 재구성
        idea_op = self._dict_to_operation(idea_dict)

        # RESEARCH operation 생성
        research_op = Operation(
            project_id=idea_op.project_id,
            bucket=BucketType.RESEARCH,
            content=content,
            node_id=client_id
        )

        # 의존성 추가 (IDEA)
        research_op.depends_on(idea_op)

        # Store에 쓰기
        dep = Dependency(
            op_id=idea_op.id,
            version=idea_op.version,
            bucket=idea_op.bucket
        )

        self.store.write(
            key=research_op.id,
            value=research_op.to_dict(),
            client_id=client_id,
            deps=[dep]
        )

        # State 업데이트
        self.state.apply(research_op)

        return research_op

    def create_todo(
        self,
        research_id: str,
        content: dict,
        prerequisites: list[str],
        client_id: str
    ) -> Operation:
        """TODO 버킷 생성 (RESEARCH에 의존)"""
        # RESEARCH operation 읽기
        research_dict = self.store.read(research_id, client_id)
        if not research_dict:
            raise ValueError(f"RESEARCH {research_id} not found")

        research_op = self._dict_to_operation(research_dict)

        # TODO operation 생성
        todo_op = Operation(
            project_id=research_op.project_id,
            bucket=BucketType.TODO,
            content={
                **content,
                "prerequisites": prerequisites,
                "acceptance_criteria": content.get("acceptance_criteria", [])
            },
            node_id=client_id
        )

        # 의존성 추가
        todo_op.depends_on(research_op)

        # Prerequisites도 의존성으로 추가 (다른 TODO일 수 있음)
        for prereq_id in prerequisites:
            prereq_dict = self.store.read(prereq_id, client_id)
            if prereq_dict:
                prereq_op = self._dict_to_operation(prereq_dict)
                todo_op.depends_on(prereq_op)

        # Store에 쓰기
        deps = [
            Dependency(
                op_id=research_op.id,
                version=research_op.version,
                bucket=research_op.bucket
            )
        ]

        self.store.write(
            key=todo_op.id,
            value=todo_op.to_dict(),
            client_id=client_id,
            deps=deps
        )

        self.state.apply(todo_op)

        return todo_op

    def _dict_to_operation(self, data: dict) -> Operation:
        """딕셔너리를 Operation 객체로 변환"""
        return Operation(
            id=data["id"],
            project_id=data["project_id"],
            bucket=BucketType(data["bucket"]),
            version=data["version"],
            vector_clock=VectorClock.from_json(data["vector_clock"]),
            dependencies=[
                Dependency(**d) for d in data.get("dependencies", [])
            ],
            content=data["content"],
            node_id=data["node_id"]
        )
```

---

## 3. CALM-aware Operations

### 3.1 Monotonic Operations 식별

```python
from abc import ABC, abstractmethod

class MonotonicOperation(ABC):
    """
    단조로운 operation 추상 클래스

    CALM theorem에 의해 coordination-free
    """

    @abstractmethod
    def is_monotonic(self) -> bool:
        """단조성 체크"""
        pass

    @abstractmethod
    def merge(self, other: 'MonotonicOperation') -> 'MonotonicOperation':
        """두 operation 병합"""
        pass

class AddIdeaOperation(MonotonicOperation):
    """IDEA 추가 (monotonic)"""

    def __init__(self, idea_id: str, content: dict):
        self.idea_id = idea_id
        self.content = content

    def is_monotonic(self) -> bool:
        return True  # 추가만 하므로 monotonic

    def merge(self, other: 'AddIdeaOperation') -> 'AddIdeaOperation':
        # 두 IDEA를 모두 유지 (집합 union)
        return self  # 각각 독립적

class UpdateTodoOperation(MonotonicOperation):
    """TODO 업데이트 (버전 관리로 monotonic 만듦)"""

    def __init__(self, todo_id: str, version: int, content: dict):
        self.todo_id = todo_id
        self.version = version
        self.content = content

    def is_monotonic(self) -> bool:
        # 새 버전 추가는 monotonic (이전 버전 삭제 안 함)
        return True

    def merge(self, other: 'UpdateTodoOperation') -> 'UpdateTodoOperation':
        # 더 높은 버전 선택
        if self.version > other.version:
            return self
        return other

class CALMChecker:
    """
    Operation이 CALM theorem을 만족하는지 확인

    Monotonic operations는 coordination 없이 실행 가능
    """

    @staticmethod
    def requires_coordination(op_type: str) -> bool:
        """
        Operation이 coordination을 필요로 하는지 판단

        Returns:
            True if coordination needed (non-monotonic)
        """
        # Monotonic operations (coordination-free)
        monotonic_ops = {
            "add_idea",
            "add_research",
            "add_study",
            "append_log",
            "increment_counter"
        }

        # Non-monotonic operations (coordination required)
        non_monotonic_ops = {
            "delete_todo",  # 삭제
            "cancel_pr",    # 취소
            "aggregate_stats"  # 집계
        }

        if op_type in monotonic_ops:
            return False

        if op_type in non_monotonic_ops:
            return True

        # 기본값: 안전하게 coordination 필요로 간주
        return True
```

### 3.2 Coordination-Free Execution

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class CoordinationFreeExecutor:
    """
    Monotonic operations를 coordination 없이 병렬 실행

    CALM theorem 활용:
    - Monotonic ops → 병렬 실행
    - Non-monotonic ops → 순차 실행 또는 locking
    """

    def __init__(self, max_workers: int = 10):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.checker = CALMChecker()

    async def execute_batch(
        self,
        operations: list[tuple[str, callable]]
    ) -> list[any]:
        """
        Operation 배치 실행

        Args:
            operations: [(op_type, callable), ...]

        Returns:
            각 operation의 결과 리스트
        """
        monotonic_ops = []
        non_monotonic_ops = []

        # Operation 분류
        for op_type, func in operations:
            if self.checker.requires_coordination(op_type):
                non_monotonic_ops.append((op_type, func))
            else:
                monotonic_ops.append((op_type, func))

        # Monotonic ops 병렬 실행 (coordination-free)
        loop = asyncio.get_event_loop()
        monotonic_results = await asyncio.gather(*[
            loop.run_in_executor(self.executor, func)
            for _, func in monotonic_ops
        ])

        # Non-monotonic ops 순차 실행
        non_monotonic_results = []
        for _, func in non_monotonic_ops:
            result = await loop.run_in_executor(self.executor, func)
            non_monotonic_results.append(result)

        return monotonic_results + non_monotonic_results

# 사용 예시
async def example_batch_execution():
    executor = CoordinationFreeExecutor()

    # 여러 IDEA 동시 생성 (monotonic)
    operations = [
        ("add_idea", lambda: create_idea("AI agent")),
        ("add_idea", lambda: create_idea("Blockchain integration")),
        ("add_idea", lambda: create_idea("Mobile app")),
        # 삭제는 coordination 필요
        ("delete_todo", lambda: delete_todo("TODO-123"))
    ]

    results = await executor.execute_batch(operations)
    return results
```

---

## 4. Supabase Integration

### 4.1 Schema Definition

```sql
-- Vector clock 저장을 위한 JSON 타입
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Operations 테이블
CREATE TABLE operations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL,
    bucket TEXT NOT NULL CHECK (bucket IN ('IDEA', 'RESEARCH', 'TODO', 'STUDY', 'PR')),
    version INTEGER NOT NULL DEFAULT 1,

    -- Causal consistency 메타데이터
    vector_clock JSONB NOT NULL,
    dependencies JSONB DEFAULT '[]'::jsonb,

    -- 실제 내용
    content JSONB NOT NULL,

    -- 메타데이터
    node_id TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    -- 복합 인덱스
    UNIQUE(id, version)
);

-- 인덱스
CREATE INDEX idx_ops_project ON operations(project_id);
CREATE INDEX idx_ops_bucket ON operations(bucket);
CREATE INDEX idx_ops_vector_clock ON operations USING GIN (vector_clock);
CREATE INDEX idx_ops_dependencies ON operations USING GIN (dependencies);
CREATE INDEX idx_ops_created ON operations(created_at DESC);

-- Dependencies 추적용 별도 테이블 (optional, 쿼리 최적화)
CREATE TABLE operation_dependencies (
    operation_id UUID NOT NULL REFERENCES operations(id),
    depends_on_id UUID NOT NULL,
    depends_on_version INTEGER NOT NULL,
    depends_on_bucket TEXT NOT NULL,

    PRIMARY KEY (operation_id, depends_on_id)
);

CREATE INDEX idx_deps_depends_on ON operation_dependencies(depends_on_id);
```

### 4.2 Python Supabase Client

```python
from supabase import create_client, Client
import os

class YomanCausalDB:
    """
    Supabase를 사용한 causal consistency DB
    """

    def __init__(self):
        url = os.environ.get("SUPABASE_URL")
        key = os.environ.get("SUPABASE_KEY")
        self.client: Client = create_client(url, key)

    async def insert_operation(self, op: Operation) -> None:
        """Operation 삽입"""
        data = op.to_dict()

        # operations 테이블에 삽입
        result = self.client.table("operations").insert(data).execute()

        # dependencies 테이블에도 삽입 (쿼리 최적화)
        if op.dependencies:
            deps_data = [
                {
                    "operation_id": op.id,
                    "depends_on_id": dep.op_id,
                    "depends_on_version": dep.version,
                    "depends_on_bucket": dep.bucket.value
                }
                for dep in op.dependencies
            ]
            self.client.table("operation_dependencies").insert(deps_data).execute()

    async def get_operation(
        self,
        op_id: str,
        version: Optional[int] = None
    ) -> Optional[Operation]:
        """Operation 조회"""
        query = self.client.table("operations").select("*").eq("id", op_id)

        if version is not None:
            query = query.eq("version", version)
        else:
            # 최신 버전
            query = query.order("version", desc=True).limit(1)

        result = query.execute()

        if not result.data:
            return None

        data = result.data[0]

        # Operation 객체 재구성
        return Operation(
            id=data["id"],
            project_id=data["project_id"],
            bucket=BucketType(data["bucket"]),
            version=data["version"],
            vector_clock=VectorClock.from_json(data["vector_clock"]),
            dependencies=[Dependency(**d) for d in data["dependencies"]],
            content=data["content"],
            node_id=data["node_id"]
        )

    async def get_dependency_graph(self, op_id: str) -> dict:
        """
        Operation의 전체 의존성 그래프 조회

        Returns:
            {op_id: [dep_id1, dep_id2, ...]}
        """
        # Recursive CTE로 전체 그래프 조회
        query = """
        WITH RECURSIVE dep_tree AS (
            -- Base case: 주어진 operation
            SELECT
                id,
                dependencies
            FROM operations
            WHERE id = :op_id

            UNION

            -- Recursive case: 의존하는 operations
            SELECT
                o.id,
                o.dependencies
            FROM operations o
            INNER JOIN dep_tree dt ON
                o.id IN (
                    SELECT jsonb_array_elements(dt.dependencies)->>'op_id'
                )
        )
        SELECT * FROM dep_tree;
        """

        result = self.client.rpc("get_dependency_graph", {"op_id": op_id}).execute()

        # 그래프 구성
        graph = {}
        for row in result.data:
            deps = [d["op_id"] for d in row["dependencies"]]
            graph[row["id"]] = deps

        return graph
```

---

## 5. 성능 최적화

### 5.1 Vector Clock 압축

```python
class CompressedVectorClock:
    """
    Vector clock 압축

    문제: 노드가 많아지면 clock 크기 증가
    해결:
    - 0인 엔트리 제거
    - Delta encoding
    - Pruning (오래된 노드 제거)
    """

    @staticmethod
    def compress(vc: VectorClock) -> dict:
        """0인 엔트리 제거"""
        return {k: v for k, v in vc.clocks.items() if v > 0}

    @staticmethod
    def prune(vc: VectorClock, active_nodes: set[str]) -> VectorClock:
        """
        비활성 노드 제거

        Args:
            vc: Vector clock
            active_nodes: 현재 활성화된 노드 집합

        Returns:
            Pruned vector clock
        """
        pruned = {k: v for k, v in vc.clocks.items() if k in active_nodes}
        return VectorClock(clocks=pruned)
```

### 5.2 의존성 캐싱

```python
from functools import lru_cache
from typing import Tuple

class DependencyCache:
    """
    의존성 조회 캐싱

    자주 조회되는 operations는 캐시
    """

    def __init__(self, max_size: int = 1000):
        self.cache = {}
        self.max_size = max_size

    def get(self, op_id: str, version: int) -> Optional[Operation]:
        """캐시에서 operation 조회"""
        key = (op_id, version)
        return self.cache.get(key)

    def put(self, op: Operation) -> None:
        """캐시에 operation 저장"""
        key = (op.id, op.version)

        # LRU eviction
        if len(self.cache) >= self.max_size:
            # 가장 오래된 항목 제거
            oldest = min(self.cache.keys(), key=lambda k: self.cache[k].updated_at)
            del self.cache[oldest]

        self.cache[key] = op
```

---

## 6. 테스트 시나리오

### 6.1 Concurrent Operations Test

```python
import pytest
import asyncio

@pytest.mark.asyncio
async def test_concurrent_idea_creation():
    """여러 IDEA 동시 생성 (concurrent, causal 관계 없음)"""
    store = BucketCausalStore()

    # 3개 IDEA 동시 생성
    tasks = [
        store.create_idea("proj1", {"title": f"Idea {i}"}, f"client_{i}")
        for i in range(3)
    ]

    ideas = await asyncio.gather(*tasks)

    # Vector clocks 비교
    for i in range(len(ideas)):
        for j in range(i + 1, len(ideas)):
            relation = ideas[i].vector_clock.compare(ideas[j].vector_clock)
            assert relation == 'CONCURRENT', "IDEAs should be concurrent"

@pytest.mark.asyncio
async def test_causal_chain():
    """IDEA → RESEARCH → TODO 인과관계 체크"""
    store = BucketCausalStore()

    # 1. IDEA 생성
    idea = store.create_idea("proj1", {"title": "AI Agent"}, "client1")

    # 2. RESEARCH 생성 (IDEA에 의존)
    research = store.create_research(idea.id, {"findings": "..."}, "client1")

    # 3. TODO 생성 (RESEARCH에 의존)
    todo = store.create_todo(
        research.id,
        {"task": "Implement"}, [],
        "client1"
    )

    # Vector clock 관계 확인
    assert idea.vector_clock.compare(research.vector_clock) == 'BEFORE'
    assert research.vector_clock.compare(todo.vector_clock) == 'BEFORE'
    assert idea.vector_clock.compare(todo.vector_clock) == 'BEFORE'

@pytest.mark.asyncio
async def test_dependency_blocking():
    """의존성 미충족 시 blocking"""
    store = BucketCausalStore()
    state = OperationState()

    # TODO 생성 (RESEARCH 없이)
    todo = Operation(
        project_id="proj1",
        bucket=BucketType.TODO,
        content={"task": "Implement"}
    )

    # 존재하지 않는 RESEARCH에 의존
    fake_research = Operation(
        id="fake-research-id",
        project_id="proj1",
        bucket=BucketType.RESEARCH
    )
    todo.depends_on(fake_research)

    # 적용 시도 → 실패 (pending)
    result = state.apply(todo)
    assert result == False, "TODO should be pending (dependency not met)"
    assert todo in state.pending, "TODO should be in pending set"
```

---

**문서 작성**: 2026-01-01
**분석 도구**: Claude Sonnet 4.5, Context7 MCP
**다음 문서**: [02-yoman-application.md](./02-yoman-application.md)
