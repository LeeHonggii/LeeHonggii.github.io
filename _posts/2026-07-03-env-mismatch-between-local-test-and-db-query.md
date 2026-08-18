---
title: "로컬은 운영 DB에 쓰고, 나는 개발 DB를 보고 있었다"
date: 2026-07-03 17:00:00 +0900
categories: [DevOps]
tags: [database, debugging, environment, ssh-tunnel]
---

"7월 이후로 추천 로그가 안 쌓이는 것 같은데요."

메신저로 이 한 줄이 왔을 때, 나는 당연히 코드를 의심했다. 최근에 추천 파이프라인을 몇 번 건드렸고, 로그 저장부도 리팩터한 적이 있었다. 어딘가에서 조용히 INSERT가 빠졌겠거니 했다.

두 시간 뒤에 알게 됐다. 코드는 죄가 없었다. 내가 잘못된 DB를 들여다보고 있었을 뿐이다.

## 먼저 한 일: 스키마부터 의심

가장 먼저 개발 DB에 붙어서 테이블을 열어봤다. 마지막 레코드가 며칠 전 날짜에 멈춰 있었다. 그 사이 앱은 멀쩡히 돌고 있었고, 에러 로그도 없었다.

머릿속에 떠오른 후보들.

- 코드 변경으로 로그 저장 로직이 조건 안쪽으로 밀려 들어갔나
- 스키마가 바뀌면서 INSERT가 조용히 실패하나
- Auto Increment(기본 키 자동 증가)가 한계에 걸렸나

```sql
SHOW CREATE TABLE recommendation_log;
SELECT MAX(id), COUNT(*) FROM recommendation_log;
```

스키마 정상. 최근 카운트만 죽어 있음. 실패 흔적 없음.

이 시점에 나는 "그럼 코드가 로그를 안 부르고 있는 거네" 라고 넘겨짚었다. 그리고 저장소를 뒤지기 시작했다. 이게 첫 번째 실수였다.

## 코드에는 없었다

레포지토리 계층부터 훑었다. 커넥션을 여는 지점은 이렇게 생겼다.

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

`config.DB_HOST` 를 쓴다. `config` 는 환경변수와 `.env` 를 병합해서 만든 모듈이다. 여기까지는 흔한 구조다.

풀을 로드하는 쪽도 봤다. 텐넌트 단위로 캐시가 걸려 있고, `load_company_pool` 은 콘텐츠 테이블을 조인해서 딕셔너리를 반환한다.

```python
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _company_pool(tenant_id: str) -> dict:
    return load_company_pool(tenant_id)
```

로그를 남기는 지점은 이 풀을 쓴 뒤 추천 결과를 뱉기 직전이었다. 코드 흐름상 로그 호출이 스킵될 조건은 없었다. `if` 분기도 없고, try/except 로 삼키지도 않았다. 이상하다.

git log 를 며칠치 훑었다. 로그 저장 함수를 건드린 커밋도 없었다. **코드는 안 바뀌었다.**

이 지점에서 나는 잠깐 멍했다. 코드는 그대로, 스키마도 그대로, 그런데 데이터가 안 쌓인다.

## 뒤집힌 순간

혹시 몰라서 앱이 실제로 어떤 host 에 붙는지 프린트해봤다.

```python
print(config.DB_HOST, config.GATEWAY_SCHEMA)
```

출력된 host 는 운영 DB 쪽이었다.

내 로컬 `.env` 안의 `DATASOURCE_URL` 이 운영을 가리키고 있었다. 언제 이렇게 됐는지는 정확히 기억이 안 난다. 아마 며칠 전 어떤 이슈 재현하려고 잠깐 바꿔놓고 그대로 뒀던 것 같다.

즉 상황은 이랬다.

- 로컬에서 돌린 앱: **운영 DB** 를 향해 INSERT 시도
- 내가 열어본 DB 클라이언트: **개발 DB**
- 운영 DB 는 VDI(가상 데스크톱 인프라) 내부에서만 열려 있음 → 로컬에서는 애초에 커넥션이 안 열림
- `connect_timeout` 걸리고 조용히 실패했을 가능성이 크다

`connect_timeout=config.DB_CONNECT_TIMEOUT` 이 코드에 박혀 있는 게 이 시점에서 다르게 읽혔다. 커넥션이 안 열리면 위쪽에서 어떻게 처리되지? 로그 저장 실패가 사용자 응답을 막지 않게 상단에서 예외를 삼키는 구간이 있었다. 그러니 앱은 겉으로 멀쩡했다.

## 다시 개발 DB 로

SSH 터널(SSH Tunnel, 로컬 포트를 원격 서버 포트로 중계) 을 열고 개발 DB 에 직접 붙었다.

```bash
ssh -L 13306:<dev-db-host>:3306 <user>@<bastion-host>
```

개발 DB 의 로그 테이블은 **7월 이후로도 정상적으로 쌓여 있었다.** 애초에 없어진 적이 없다. 다른 개발자가 개발 환경에서 돌린 결과가 거기 다 있었다.

없어진 게 아니라, 내가 만든 로그가 다른 DB(정확히는 아무 데도 안 감) 로 사라진 거였다.

## 뒤늦게 정리한 체크리스트

이 실수 이후로 로그가 안 쌓인다는 신고를 받으면 스키마·코드보다 먼저 보는 것들.

| 확인 항목 | 확인 방법 |
|---|---|
| 앱이 지금 붙는 DB host | 프로세스 안에서 `config.DB_HOST` 프린트 |
| 내가 조회하는 DB host | 클라이언트 연결 정보 재확인 |
| 두 host 가 같은가 | 눈으로 비교. 도메인 한 글자 다른 경우 많음 |
| 커넥션이 실제로 열렸는가 | timeout / refused 로그를 상단에서 삼키지 않는지 |

특히 마지막 줄. 로그 저장을 "실패해도 서비스는 계속" 이라는 이유로 예외를 삼키는 관행은 필요하긴 한데, 삼킨 흔적조차 안 남기면 이번처럼 몇 시간을 태운다. 최소한 실패 카운터 하나는 남겨야 한다.

## 남은 것

로컬 `.env` 에 운영 값이 들어있는 상태 자체를 없애야 한다. 이번엔 운영 DB 가 VDI 밖에서 안 열려서 "그냥 실패" 로 끝났지만, 만약 방화벽이 열려 있었다면 로컬 테스트 결과가 그대로 운영에 쌓였을 것이다. 생각만 해도 서늘하다.

`.env.example` 에 운영 host 를 아예 못 적게 하는 방식(placeholder 만 허용) 을 팀에 제안할 참인데, 이건 아직 얘기 안 꺼냈다. 다음 주에 꺼내 볼 것.

그리고 커넥션 실패를 삼키는 저 상단 예외 처리. 아직 안 고쳤다. 고칠 때 실패 카운터를 어디에 남길지가 애매하다 — 어차피 DB 에 못 붙는 상황이라 DB 로그로 남길 수는 없고, 파일로 남기면 컨테이너 재시작 때 날아간다. 표준 에러로 뱉고 로그 수집기가 걷어가게 하는 게 맞을 것 같은데, 지금 수집기 설정을 정확히 모른다. 이건 확인이 필요하다.