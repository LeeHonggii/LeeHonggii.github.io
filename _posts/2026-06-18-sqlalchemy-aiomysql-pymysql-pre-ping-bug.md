---
title: "pool_pre_ping을 끄지 않고 DB 연결 장애를 고친 과정: PyMySQL 1.2.0 함정"
date: 2026-06-18 17:00:00 +0900
categories: [Backend]
tags: [SQLAlchemy, PyMySQL, aiomysql]
---

# 도입 — 왜 이 글을 쓰는가

어느 날 배포 직후부터 DB 연결이 전부 실패하기 시작했다. 오류 메시지는 단 한 줄이었다.

```
TypeError: AsyncAdapt_aiomysql_connection.ping() missing 1 required positional argument: 'reconnect'
```

처음엔 `pool_pre_ping=True` 옵션이 의심스러웠다. "그냥 끄면 되는 거 아닐까?" 생각했지만, 그러면 연결 재사용 시 stale connection(오래된 커넥션) 문제가 다시 생긴다. 빠른 회피보다 정확한 원인 파악이 필요했다.

## pool_pre_ping이 뭔데 이게 문제가 됐나

`pool_pre_ping`은 SQLAlchemy의 커넥션 풀 옵션이다. 쿼리를 실행하기 전 DB 커넥션이 살아 있는지 먼저 `SELECT 1`을 보내 확인한다. 서버가 커넥션을 끊었을 때 자동으로 재연결해주는 안전장치다.

비동기 스택에서는 SQLAlchemy가 `aiomysql` 어댑터를 통해 이 ping을 수행하고, aiomysql은 내부적으로 `PyMySQL`의 코드를 일부 가져다 쓴다. 즉 세 패키지가 연쇄적으로 연결되어 있다.

```
SQLAlchemy (pool_pre_ping)
  └─ aiomysql (AsyncAdapt_aiomysql_connection)
       └─ PyMySQL (ping 시그니처)
```

## PyMySQL 1.2.0에서 무슨 일이 있었나

문제는 `PyMySQL`의 마이너 버전 업에 있었다. 1.1.x까지의 `ping()` 시그니처는 이렇게 생겼다.

```python
# PyMySQL 1.1.x
def ping(self, reconnect=True): ...
```

1.2.0에서 이 인자가 제거됐다. aiomysql의 어댑터 코드는 여전히 `ping(reconnect=False)`를 호출하고 있었고, 결과는 `TypeError`였다.

직접 명시하지 않은 의존성(transitive dependency)이 재빌드 타이밍에 최신 버전으로 올라간 것이다. `requirements.txt`에 `PyMySQL`을 적지 않았다면, `pip install aiomysql`이 최신 PyMySQL을 끌어온다.

버전 상태를 확인하는 것부터 시작했다.

```bash
pip show sqlalchemy aiomysql PyMySQL
```

결과를 보니 `PyMySQL 1.2.0`이 설치되어 있었다. 배포 이전엔 1.1.3이었다.

## 재현 스크립트로 확인한 것

"혹시 다른 이유일 수도 있다"는 의심을 걷어내기 위해 최소 재현 코드를 작성했다. `SELECT 1`을 두 번 실행해 커넥션 재사용 경로를 타게 만들고, pre_ping 시 ping이 호출되는 흐름을 재현했다.

```python
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "mysql+aiomysql://user:pass@localhost/db",
    pool_pre_ping=True,
)

async def main():
    async with engine.connect() as conn:
        await conn.execute(text("SELECT 1"))
    # 동일 풀에서 커넥션을 다시 꺼내 ping 경로를 탄다
    async with engine.connect() as conn:
        await conn.execute(text("SELECT 1"))

asyncio.run(main())
```

PyMySQL 1.2.0 환경에서는 두 번째 `execute`에서 정확히 위 `TypeError`가 재현됐다. 1.1.3으로 교체하면 깔끔하게 통과했다.

## 해결: transitive dependency 핀닝

aiomysql이 PyMySQL 1.2.0과의 호환성을 아직 맞추지 않은 상태였으므로, 당장의 수정은 버전 핀이었다.

```
# requirements.txt
PyMySQL==1.1.3
```

`pool_pre_ping=False`로 설정해 ping 자체를 끄는 방법도 있지만, 그러면 네트워크 단절 후 stale 커넥션이 풀에 남아 첫 쿼리에서 오류가 터진다. 근본 원인을 알고 있으니 회피책보다 정확한 핀이 훨씬 안전하다.

## 마치며

이번 장애에서 배운 것은 두 가지다.

첫째, **transitive dependency는 보이지 않게 올라간다.** 내가 명시한 패키지만 고정해도, 그 패키지가 의존하는 패키지는 재빌드마다 최신 버전이 들어올 수 있다. 운영 환경이라면 `pip freeze > requirements.txt`로 전체 트리를 고정하는 습관이 필요하다.

둘째, **오류 메시지에서 가장 중요한 단서는 시그니처 불일치다.** `missing 1 required positional argument`는 "호출부와 정의부의 버전이 다르다"는 신호다. 이 패턴을 보는 순간 의존성 버전 확인으로 바로 좁혀가면 시간을 아낄 수 있다.