---
title: "로그가 사라진 줄 알았는데: 커밋, 스키마, Auto Increment로 추적한 기록"
date: 2026-07-03 17:00:00 +0900
categories: [Backend]
tags: [debugging, mysql, postmortem]
---

"특정 날짜 이후로 추천 로그가 안 쌓이는 것 같아요."

이 문장 하나로 오후가 날아갔다. 그리고 결론부터 말하면, 로그는 사라진 적이 없었다. 내가 엉뚱한 곳을 보고 있었을 뿐이다.

## 처음 든 생각은 틀렸다

들었을 때 반사적으로 커밋 로그를 열었다. "언제부터 안 쌓인다고 하지?" 물어봐서 받은 날짜를 기준으로 `git log` 를 훑는다. 그 근처에 머지된 PR이 하나 있다. 추천 파이프라인 쪽을 만진 게 맞다. 시간대도 맞물린다.

이 시점에서 이미 뇌 속에서는 시나리오가 완성돼 있었다. "그 PR이 INSERT 경로를 건드렸고, 어떤 조건에서 로그가 안 찍히게 됐다." 코드를 열어 diff 를 보기 시작했다.

여기서 30분을 썼다. 아무것도 안 나왔다. 로그를 쓰는 경로는 그 PR 이전과 동일했다. try/except 로 삼켜지는 지점도 없었다.

시간이 겹치는 것과 원인인 것은 다르다. 머리로는 알지만, 급하면 그 둘을 자꾸 붙여 놓는다.

## 코드 대신 DB를 열어야 했다

방향을 바꿨다. 코드가 멀쩡해 보이면, 코드가 하려던 일이 DB 쪽에서 조용히 실패하고 있을 수 있다. 가장 흔한 시나리오는 `NOT NULL` 컬럼에 값이 안 들어가서 INSERT 가 예외로 튕기는 것. 그런데 애플리케이션 로거가 워낙 시끄러워서, 특정 예외가 다른 로그에 묻혀 안 보였을 수도 있다.

스키마부터 확인한다.

```sql
SHOW CREATE TABLE recommend_logs;
```

`NOT NULL` 인데 `DEFAULT` 가 없는 컬럼이 최근에 추가됐다면 그게 범인일 확률이 높다. 이번엔 그런 컬럼이 없었다. 스키마는 지난 배포 기준 그대로였고, 애플리케이션이 채워 넣는 필드 목록과도 맞았다.

스키마가 깨끗하다는 것만으로는 아직 아무것도 확정할 수 없다. INSERT 가 실제로 일어나고 있는지, 아니면 시도조차 없는지를 봐야 한다.

## AUTO_INCREMENT 가 남기는 흔적

여기서 MySQL 의 특이한 성질을 쓴다. `AUTO_INCREMENT` 카운터는 트랜잭션이 롤백돼도 되돌아오지 않는다. 즉 **INSERT 를 시도했다가 실패한 흔적**은 카운터에 남는다. `MAX(id)` 보다 `AUTO_INCREMENT` 값이 훨씬 크면, 실패한 시도가 그 사이 어딘가에 있었다는 뜻이다.

먼저 테이블의 전체 상태를 본다.

```sql
SELECT COUNT(*)         AS cnt,
       MIN(id)          AS min_id,
       MAX(id)          AS max_id,
       MAX(created_at)  AS last_at
FROM recommend_logs;
```

그리고 `information_schema` 로 카운터 값을 뽑는다.

```sql
SELECT AUTO_INCREMENT
FROM information_schema.tables
WHERE table_schema = DATABASE()
  AND table_name   = 'recommend_logs';
```

여기서 예상했던 그림은 두 가지 중 하나였다.
- `MAX(id)` 는 며칠 전에서 멈춰 있는데 `AUTO_INCREMENT` 는 계속 올라가 있음 → 실패하는 INSERT 가 있다.
- 둘 다 며칠 전에서 멈춰 있음 → 아예 시도조차 없다. 애플리케이션이 그 경로를 안 타고 있다.

날짜별로도 함께 본다. 어느 시점부터 곡선이 꺾였는지가 그림으로 보이니까.

```sql
SELECT DATE(created_at) AS d, COUNT(*) AS c
FROM recommend_logs
GROUP BY DATE(created_at)
ORDER BY d DESC
LIMIT 14;
```

## 예상과 다른 결과

쿼리를 돌렸다. `MAX(created_at)` 이 오늘이었다. 어제도, 그제도 정상적으로 찍혀 있었다. 곡선이 꺾인 적이 없다.

여기서 잠깐 멍했다. "안 쌓인다"는 신고가 들어온 게 사실인데, 눈앞의 DB 는 정상이다. 데이터가 없는 게 아니라 **내가 보고 있는 DB 가 신고자가 보고 있는 DB 와 다르다**는 뜻이었다.

## 진짜 원인은 접속 경로

리포지토리를 다시 열었다. 연결부는 이렇게 생겨 있다.

```python
def get_connection() -> pymysql.connections.Connection:
    return pymysql.connect(
        host=config.DB_HOST,
        user=config.DB_USER,
        password=config.DB_PASSWORD,
        database=config.GATEWAY_SCHEMA,
        port=config.DB_PORT,
        cursorclass=pymysql.cursors.DictCursor,
        charset='utf8mb4',
        connect_timeout=config.DB_CONNECT_TIMEOUT,
    )
```

`config.DB_HOST` 는 결국 환경변수에서 온다. 로컬 `.env`, 배포 환경의 시크릿, VDI 안에서 쓰는 SSH 터널 설정 — 이게 다 각자 다른 호스트를 가리킬 수 있다. 그리고 실제로 그랬다. 신고자는 스테이징 게이트웨이를 통해 접근한 DB 를 "운영"이라고 부르고 있었고, 내가 로컬에서 붙은 DB 는 완전히 다른 인스턴스였다.

풀 로딩 로그도 다시 봤다.

```python
logger.info('pool loaded: agent=%s size=%d (labeled=%d)', tenant_id, len(pool), labeled)
```

이 한 줄이 진작에 힌트를 주고 있었다. 내가 로컬에서 서비스를 띄웠을 때 나오는 pool size 와, 신고자가 붙은 환경에서 나온 pool size 가 눈에 띄게 달랐다. 처음엔 "테넌트별로 다르니까 그런가보다" 하고 넘겼는데, 사실은 **다른 DB 를 보고 있다는 신호**였다.

로그를 못 쌓고 있던 게 아니다. 내가 쌓인 곳을 못 찾고 있었다.

## 서비스 캐시가 상황을 더 흐렸다

한 가지 더 나를 헷갈리게 한 게 있었다. 추천 서비스는 tenant 별 풀을 프로세스 메모리에 캐시한다.

```python
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _company_pool(tenant_id: str) -> dict:
    return load_company_pool(tenant_id)
```

이 `lru_cache` 때문에, 프로세스를 재시작하기 전까지는 DB 를 새로 안 읽는다. 로컬에서 접속 정보를 바꿔가며 테스트하는 동안, 어떤 요청은 캐시된 옛 풀을 그대로 쓰고 있었다. "분명 DB 를 바꿨는데 왜 응답이 그대로지?" 하는 순간이 몇 번 있었고, 그때마다 캐시를 잊고 있었다.

디버깅 중에 `lru_cache` 가 있는 경로를 만지려면, 프로세스를 다시 띄우거나 `_company_pool.cache_clear()` 를 명시적으로 부르는 게 맞다. 이걸 몸으로 배웠다.

## 남은 것

문제 자체는 접속 정보를 맞추는 걸로 끝났지만, 뒤에 남은 게 두 개 있다.

하나는 신고를 받을 때 "어느 환경에서 보고 계세요?" 를 먼저 묻지 않은 것. 이게 없으니 내가 30분 동안 다른 DB 를 놓고 커밋을 뒤진 거다. 다음에는 신고 접수 템플릿에 환경 이름 칸 하나를 강제로 넣기로 했다.

다른 하나는 pool size 를 찍는 그 `logger.info` 를 요청 로그에도 실어두는 게 낫겠다는 것. 응답에 "이 응답은 tenant=X, pool=Y 로 만든 것" 정도의 태그가 붙어 있으면, 이런 종류의 착각은 처음부터 안 생긴다. 아직 안 넣었다. 다음 스프린트에 볼 것.

커밋을 먼저 열지 않았더라면 30분은 안 썼을 거다. 그런데 다음번에도 나는 아마 커밋을 먼저 열 것 같다. 그게 무섭다.