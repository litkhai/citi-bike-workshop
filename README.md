# NY Citi Bike — PostGIS meets ClickHouse

**[Documentation site](https://litkhai.github.io/citi-bike-workshop/) · [Workshop overview](workshop/workshop-overview.md) · [Start here](workshop/00-prerequisites.md) · [Instructor guide](workshop/instructor-guide.md)**

[English](#english) | [한국어](#한국어)

---

## English

A self-service workshop built on a data feed that is **actually live**. New
York's Citi Bike publishes the state of every dock as public JSON with no API
key, and within ten minutes of starting your database is filling up with it in
real time.

You will keep the geography in **ClickHouse Managed Postgres**, push the
counting down to **ClickHouse Cloud**, and prove from the execution plan —
not from a stopwatch — which engine answered each query.

### The claim you are going to test

> You do not have to choose between Postgres and ClickHouse. Keep the geography
> in Postgres, send only the counting to ClickHouse, and neither engine does
> the thing it is bad at.

That is easy to say and easy to fake. A dashboard that shows numbers cannot
tell you which engine produced them, and "it felt fast" is not evidence — a
foreign table will happily drag millions of rows across the network and count
them locally. So every query here ends with a verdict read out of the plan.

### What you get

```text
Citi Bike GBFS            public JSON · no API key · ~2,500 stations · refreshed every 60s
      │  ClickHouse refreshable MV over url(), every minute
      ▼
ClickHouse Cloud          database ny_citibike
      gbfs_status                                  ← landing
      │  ny_citibike_ch.gbfs_status  +  pg_cron, every minute
      ▼
ClickHouse Managed Postgres   schema ny_citibike
      stations              PostGIS points  · 2,500 rows      · barely changes
      station_status        snapshots       · +3.6M rows/day  · only ever counted
      │  ClickPipes (Postgres CDC)
      ▼
ClickHouse Cloud          database ny_citibike
      stations · station_status                    ← mirrored, name for name
      ▲
      │  ny_citibike_ch.*  — foreign tables, back in the Postgres session
      │
Your SQL: geometry stays local, aggregates run remotely
      │
      ▼
Dashboard (Docker) — badges every query with the engine that answered
```

Two things make that work.

**The join key is a `bigint`**, so no geometry ever has to cross.

**One namespace name on both engines** — the Postgres schema and the ClickHouse
database are both `ny_citibike`, and the mirrored tables keep their names. So
`ny_citibike.station_status` means the same thing on either side, and the only
difference between a local query and a pushed-down one is a schema prefix:

```text
ny_citibike.station_status      the real table, in Postgres
ny_citibike_ch.station_status   the same rows, answered by ClickHouse
```

When the verdict changes, the query text did not. That is what makes the
comparison evidence rather than anecdote.

### Requirements

- **Docker Desktop or Docker Engine with Compose v2** — `psql` and the dashboard run in containers; nothing else is installed. Ingestion needs no container at all: both schedulers are server-side
- **A ClickHouse Cloud account** — both services live there; a new organization starts with trial credit
- `curl`, `git`, and a browser

You do not need a Postgres client, Python, or a ClickHouse client on your
machine.

> **This costs money.** Two small managed services run for about two hours.
> On trial credit that is comfortably free; on a paid account it is small but
> not zero. The teardown is [module 08](workshop/08-wrap-up.md) — read its
> cost note before you start rather than after.

### Quick start

```bash
git clone https://github.com/litkhai/citi-bike-workshop.git
cd citi-bike-workshop

./scripts/preflight.sh          # checks Docker and the live feed — no account needed yet
```

Then follow [module 00](workshop/00-prerequisites.md). After provisioning the
two services in [module 01](workshop/01-provision.md):

```bash
./setup.sh                                    # asks for both services, writes .env

./scripts/psql.sh -f /sql/01-schema.sql             # PostGIS schema + publication
./scripts/clickhouse.sh -f /clickhouse/01-ingest-rmv.sql   # ClickHouse starts pulling
./scripts/psql.sh -f /sql/03-postgres-sync.sql -v ch_host=... -v ch_pass=...
./scripts/psql.sh -f /sql/02-verify.sql       # is it moving?

docker compose up -d --build ui               # http://localhost:8080
```

### Modules

| | Module | Time | |
|---|---|---|---|
| 00 | [Prerequisites](workshop/00-prerequisites.md) | 10 min | |
| 01 | [Provision the two services](workshop/01-provision.md) | 20 min | **console** |
| 02 | [Postgres, PostGIS and the schema](workshop/02-postgres-and-feed.md) | 10 min | |
| 03 | [The feed, with nothing on your laptop](workshop/03-the-feed.md) | 20 min | |
| 04 | [The half that cannot move](workshop/04-spatial.md) | 15 min | |
| 05 | [Replicate to ClickHouse](workshop/05-clickpipes.md) | 20 min | **console** |
| 06 | [Push the counting down](workshop/06-pushdown.md) | 25 min | |
| 07 | [The dashboard](workshop/07-dashboard.md) | 15 min | |
| 08 | [Wrap-up and teardown](workshop/08-wrap-up.md) | 10 min | |
| 09 | [Trips, generated](workshop/09-trips.md) — optional extra | 20 min | |

Two modules are **console walkthroughs** rather than scripts. Creating cloud
services and connecting a ClickPipe are tied to your own account and billing, so
the workshop clicks through them with you instead of asking for an
organization-wide API key. Everything else — including module 03's ClickHouse
statements, via `scripts/clickhouse.sh` — runs from this repository.

### What is in here

```text
workshop/       the guide — also published as the documentation site
                data-model.md  every table on both engines, and the five routes
clickhouse/     01 the refreshable MV that pulls the feed (runs on ClickHouse)
sql/            01 schema · 02 verify · 03 the Postgres side of ingestion
                03-check  can Postgres fetch https itself? (no, and why)
                10 spatial · 20 aggregates · 30 snapshot-to-events
                40 pg_clickhouse FDW
                50 the trip generator (optional, module 09)
setup.sh        asks for both services once and writes .env
scripts/        preflight · psql · clickhouse (both containerised) · explain-pushdown
ui/             the dashboard — two files, stdlib + psycopg
```

### Using a different city

Nothing is New York-specific except the map's initial centre. Any **docked**
system in the
[GBFS registry](https://github.com/MobilityData/gbfs/blob/master/systems.csv)
works — over 1,500 of them, none requiring a key.

ClickHouse is what fetches the feed, so the URL lives in the two `url()` calls
in `clickhouse/01-ingest-rmv.sql`. Resolve the discovery document once to find
them, then edit the file and re-run it:

```bash
curl -s https://gbfs.lyft.com/gbfs/2.3/dca-cabi/gbfs.json \
  | python3 -m json.tool | grep -A1 'station_status\|station_information'
```

Resolve rather than guess: the host serving the data is often not the one the
registry lists — Citi Bike registers `gbfs.citibikenyc.com` and serves from
`gbfs.lyft.com`.

### Credentials

This repository is public. `.env` is the only file holding real values and it
is gitignored. Scripts read connection details from the environment and fail
with instructions when they are missing; `scripts/psql.sh` masks the hostname
on the way out, because the hostname carries your service name and people
screenshot terminals during workshops.

### Troubleshooting

Run `./scripts/preflight.sh` first — it checks Docker, the feed and your Managed Postgres
extensions without creating anything. The failures below are the ones seen in sessions; the
[instructor guide](workshop/instructor-guide.md) has the background.

**The ClickPipe will not connect (module 05).** The pipe connects from ClickHouse Cloud's network,
not from your laptop, so the IP you allow-listed in [01](workshop/01-provision.md) does nothing for
it — the Postgres service has to accept that connection too. The pipe also rejects the source unless
`wal_level` is `logical`, publication `ny_citibike_pub` has 2 tables, and both tables have replica
identity `default`; [05](workshop/05-clickpipes.md) has the three queries.

**Nothing arrives after module 03.** Expect up to two minutes: up to one for the refreshable
materialized view, up to another for `pg_cron`. Wait three minutes before running `02-verify.sql`
again or debugging anything.

**Module 06 only ever reports `dragged`.** The small station table was not replicated, only the big
one. Mixing one local table into the join silently collapses the pushdown — no error, no warning.
Replicate both tables ([05](workshop/05-clickpipes.md), [06](workshop/06-pushdown.md)).

**The feed is down.** Point the two `url()` calls in `clickhouse/01-ingest-rmv.sql` at another GBFS
feed (Capital Bikeshare works); nothing else changes. See [Using a different city](#using-a-different-city).

**A cancelled backfill still holds locks (module 09).** Cancelling `psql` does not cancel the
backend. Find it with `SELECT pid, state, query FROM pg_stat_activity WHERE query LIKE '%sim_%';`
and stop it with `SELECT pg_terminate_backend(<pid>);` ([09](workshop/09-trips.md)).

**Costs keep running after the session.** Both schedulers are server-side; closing your laptop does
not stop collection. Tear down with [08 — Wrap-up](workshop/08-wrap-up.md).

### Verification status

Every claim here was run. This section says on what.

**Verified against the real products on 2026-08-15** — ClickHouse Managed
Postgres (PostgreSQL 18.4, PostGIS 3.6.4, pg_cron 1.6, pg_clickhouse 0.3) and
ClickHouse Cloud 26.4.1:

- ClickHouse fetching the live GBFS feed with `url()` — 1,073,635 bytes, parsed to 2,509 stations
- a refreshable materialized view with `REFRESH EVERY 1 MINUTE APPEND` accumulating snapshots unattended
- `pg_clickhouse` importing the landing tables and Postgres reading them
- `pg_cron` syncing forward every minute, unattended for seven hours: **1.38M rows, 0 duplicate `(station_key, polled_at)` pairs**
- the schema, the publication, and all five query files

**Verified end to end on 2026-08-16**, same pair of services, with ClickPipes
connected and module 09's generated trip table replicated:

- **the pushdown, from the plan.** One query text, no schema prefix, resolved through `search_path`: a single `Foreign Scan` whose `Remote SQL` carries the join, the `GROUP BY` and the aggregates
- **the counter-example.** The same text with one table pinned local — `dragged`, `Remote SQL` selecting columns only, no error and no warning
- **timings that mean something**, over 9.8M trips: trips-by-hour **10,404 ms** local against **465 ms** pushed; a four-relation join 8,234 against 1,606
- **the dialect translation** — `extract(hour FROM …)` leaving as `toHour(…)`
- all ten dashboard checkpoints green

**Verified on a local PostgreSQL 17 + PostGIS 3.6.4 container:** the plan shape
of the window query — which turned out to be index-covered rather than sorting,
so module 06 makes the narrower argument that survives scrutiny.

**Three claims this repository made and measurement disproved**, all now
corrected in place rather than quietly dropped:

- `geom` does cross to ClickHouse. It arrives as `text`; `sql/40-fdw-clickhouse.sql` drops it from the foreign table afterwards so that reaching for it fails loudly instead of casting per row without an index
- deriving departures from snapshot deltas **over**-counts, not under-counts — 244k/day implied against a published 100–150k, because reporting noise moves counts down as often as riders do
- the dimension upsert rewrote all 2,509 stations every minute whether or not anything changed: 1,289,626 updates over 514 runs, on the table the workshop calls the half that barely changes

**A negative result, verified and kept:** Postgres cannot fetch the feed
itself. There is no `http` or `pg_net` extension in the catalogue; `plperlu`
installs cleanly but the server's Perl has no `IO::Socket::SSL` or
`Net::SSLeay`, so every https fetch dies. `sql/03-check-in-db-http.sql` re-runs
that check on your own service, because it is the image rather than a
permission and a platform update could change it.

**Still written from the product documentation rather than run:** the console
click-throughs in [01](workshop/01-provision.md) (provisioning) and
[05](workshop/05-clickpipes.md) (creating the ClickPipe). The pipe itself has
been connected and everything downstream of it measured — what is unverified is
the sequence of screens, which is exactly the part that drifts. Both modules
describe what you are looking for alongside the current labels.

If a step does not match what you see, that is worth an
[issue](https://github.com/litkhai/citi-bike-workshop/issues).

### License

[MIT](LICENSE).

Citi Bike system data is published by Lyft Bikes and Scooters, LLC under the
[GBFS](https://gbfs.org) specification and is fetched at run time, not
redistributed here. MapLibre GL JS is loaded from a CDN under its own licence.
ClickHouse is a registered trademark of ClickHouse, Inc.; this is an
independent educational workshop and not an official ClickHouse product.

---

## 한국어

실제로 살아있는 데이터 피드를 기반으로 한 셀프 서비스 워크숍입니다. 뉴욕의
Citi Bike는 API 키 없이 모든 도크의 상태를 공개 JSON으로 제공하며, 시작한 지
10분 이내에 여러분의 데이터베이스는 실시간으로 데이터를 채우기 시작합니다.

지리 정보는 **ClickHouse Managed Postgres**에 유지하고, 집계(카운팅)는
**ClickHouse Cloud**로 내려보낸 뒤, 스톱워치가 아니라 실행 계획(execution
plan)을 근거로 각 쿼리에 어떤 엔진이 답했는지 증명합니다.

### 확인할 주장

> Postgres와 ClickHouse 중 하나를 선택할 필요는 없습니다. 지리 정보는
> Postgres에 두고 집계만 ClickHouse로 보내면, 두 엔진 모두 자신이 잘 못하는
> 일을 하지 않게 됩니다.

말하기는 쉽고 속이기도 쉬운 주장입니다. 숫자를 보여주는 대시보드는 어떤
엔진이 그 숫자를 만들어냈는지 알려주지 않으며, "빠르게 느껴졌다"는 증거가
되지 못합니다 — 외부 테이블(foreign table)은 아무렇지 않게 수백만 행을
네트워크 너머로 끌어와 로컬에서 집계할 수 있습니다. 그래서 여기 나오는 모든
쿼리는 실행 계획에서 읽어낸 판정(verdict)으로 끝납니다.

### 얻게 되는 것

```text
Citi Bike GBFS            public JSON · no API key · ~2,500 stations · refreshed every 60s
      │  ClickHouse refreshable MV over url(), every minute
      ▼
ClickHouse Cloud          database ny_citibike
      gbfs_status                                  ← landing
      │  ny_citibike_ch.gbfs_status  +  pg_cron, every minute
      ▼
ClickHouse Managed Postgres   schema ny_citibike
      stations              PostGIS points  · 2,500 rows      · barely changes
      station_status        snapshots       · +3.6M rows/day  · only ever counted
      │  ClickPipes (Postgres CDC)
      ▼
ClickHouse Cloud          database ny_citibike
      stations · station_status                    ← mirrored, name for name
      ▲
      │  ny_citibike_ch.*  — foreign tables, back in the Postgres session
      │
Your SQL: geometry stays local, aggregates run remotely
      │
      ▼
Dashboard (Docker) — badges every query with the engine that answered
```

이것이 가능한 이유는 두 가지입니다.

**조인 키는 `bigint`**이므로, 지오메트리(geometry)가 넘어갈 일이 전혀
없습니다.

**두 엔진에서 동일한 네임스페이스 이름을 사용합니다** — Postgres 스키마와
ClickHouse 데이터베이스는 둘 다 `ny_citibike`이며, 미러링된 테이블도 이름을
그대로 유지합니다. 따라서 `ny_citibike.station_status`는 양쪽에서 같은 것을
의미하며, 로컬 쿼리와 푸시다운된 쿼리의 유일한 차이는 스키마 접두사(prefix)
뿐입니다:

```text
ny_citibike.station_status      the real table, in Postgres
ny_citibike_ch.station_status   the same rows, answered by ClickHouse
```

판정은 바뀌어도 쿼리 텍스트는 바뀌지 않습니다. 바로 이 점이 비교를 일화가
아니라 증거로 만들어줍니다.

### 요구 사항

- **Compose v2를 지원하는 Docker Desktop 또는 Docker Engine** — `psql`과 대시보드는 컨테이너에서 실행되며, 그 외에는 아무것도 설치하지 않습니다. 수집(ingestion)에는 컨테이너가 전혀 필요 없습니다: 두 스케줄러 모두 서버 측에서 동작합니다
- **ClickHouse Cloud 계정** — 두 서비스 모두 여기에 있으며, 새 조직(organization)은 체험 크레딧으로 시작합니다
- `curl`, `git`, 그리고 브라우저

여러분의 컴퓨터에는 Postgres 클라이언트, Python, ClickHouse 클라이언트가
필요하지 않습니다.

> **비용이 발생합니다.** 두 개의 작은 매니지드 서비스가 약 두 시간 동안
> 실행됩니다. 체험 크레딧으로는 충분히 무료 범위이며, 유료 계정에서는 적지만
> 0은 아닙니다. 종료(teardown) 방법은 [module 08](workshop/08-wrap-up.md)에
> 있습니다 — 시작하기 전에 비용 안내를 미리 읽어두세요.

### 빠른 시작

```bash
git clone https://github.com/litkhai/citi-bike-workshop.git
cd citi-bike-workshop

./scripts/preflight.sh          # checks Docker and the live feed — no account needed yet
```

그 다음 [module 00](workshop/00-prerequisites.md)을 따라가세요.
[module 01](workshop/01-provision.md)에서 두 서비스를 프로비저닝한 후:

```bash
./setup.sh                                    # asks for both services, writes .env

./scripts/psql.sh -f /sql/01-schema.sql             # PostGIS schema + publication
./scripts/clickhouse.sh -f /clickhouse/01-ingest-rmv.sql   # ClickHouse starts pulling
./scripts/psql.sh -f /sql/03-postgres-sync.sql -v ch_host=... -v ch_pass=...
./scripts/psql.sh -f /sql/02-verify.sql       # is it moving?

docker compose up -d --build ui               # http://localhost:8080
```

### 모듈

| | 모듈 | 시간 | |
|---|---|---|---|
| 00 | [사전 준비](workshop/00-prerequisites.md) | 10 min | |
| 01 | [두 서비스 프로비저닝](workshop/01-provision.md) | 20 min | **콘솔** |
| 02 | [Postgres, PostGIS와 스키마](workshop/02-postgres-and-feed.md) | 10 min | |
| 03 | [로컬에 아무것도 없이 받는 피드](workshop/03-the-feed.md) | 20 min | |
| 04 | [옮길 수 없는 절반](workshop/04-spatial.md) | 15 min | |
| 05 | [ClickHouse로 복제하기](workshop/05-clickpipes.md) | 20 min | **콘솔** |
| 06 | [집계를 아래로 내리기](workshop/06-pushdown.md) | 25 min | |
| 07 | [대시보드](workshop/07-dashboard.md) | 15 min | |
| 08 | [마무리와 종료](workshop/08-wrap-up.md) | 10 min | |
| 09 | [생성된 트립(Trips) 데이터](workshop/09-trips.md) — 선택적 추가 | 20 min | |

두 모듈은 스크립트가 아니라 **콘솔 실습(console walkthroughs)**입니다.
클라우드 서비스를 생성하고 ClickPipe를 연결하는 작업은 여러분 자신의 계정 및
결제와 연결되어 있으므로, 조직 전체에 적용되는 API 키를 요구하는 대신
워크숍이 여러분과 함께 화면을 클릭해 나갑니다. 그 외의 모든 것 — module
03의 ClickHouse 문(statement)을 포함해, `scripts/clickhouse.sh`를 통해 —
은 이 저장소 안에서 실행됩니다.

### 저장소 구성

```text
workshop/       the guide — also published as the documentation site
                data-model.md  every table on both engines, and the five routes
clickhouse/     01 the refreshable MV that pulls the feed (runs on ClickHouse)
sql/            01 schema · 02 verify · 03 the Postgres side of ingestion
                03-check  can Postgres fetch https itself? (no, and why)
                10 spatial · 20 aggregates · 30 snapshot-to-events
                40 pg_clickhouse FDW
                50 the trip generator (optional, module 09)
setup.sh        asks for both services once and writes .env
scripts/        preflight · psql · clickhouse (both containerised) · explain-pushdown
ui/             the dashboard — two files, stdlib + psycopg
```

### 다른 도시 사용하기

지도의 초기 중심 좌표를 제외하면 뉴욕에 특화된 부분은 전혀 없습니다.
[GBFS 레지스트리](https://github.com/MobilityData/gbfs/blob/master/systems.csv)에
있는 **도크 방식(docked)** 시스템이라면 무엇이든 동작합니다 — 1,500개가
넘는 시스템 중 키가 필요한 곳은 하나도 없습니다.

피드를 가져오는 주체는 ClickHouse이므로, URL은
`clickhouse/01-ingest-rmv.sql`의 두 `url()` 호출 안에 있습니다. discovery
문서를 한 번 확인해 URL을 찾은 다음, 파일을 수정하고 다시 실행하세요:

```bash
curl -s https://gbfs.lyft.com/gbfs/2.3/dca-cabi/gbfs.json \
  | python3 -m json.tool | grep -A1 'station_status\|station_information'
```

추측하지 말고 직접 확인하세요: 실제로 데이터를 제공하는 호스트는
레지스트리에 적힌 것과 다른 경우가 많습니다 — Citi Bike는
`gbfs.citibikenyc.com`으로 등록되어 있지만 실제로는 `gbfs.lyft.com`에서
서비스합니다.

### 자격 증명

이 저장소는 공개(public)되어 있습니다. 실제 값을 담고 있는 파일은 `.env`
뿐이며 gitignore 처리되어 있습니다. 스크립트는 환경 변수에서 연결 정보를
읽고, 값이 없으면 안내와 함께 실패합니다; `scripts/psql.sh`는 출력할 때
호스트명을 마스킹하는데, 호스트명에 여러분의 서비스 이름이 담겨 있고
워크숍 중에는 사람들이 터미널을 스크린샷으로 찍는 경우가 많기 때문입니다.

### 문제 해결

먼저 `./scripts/preflight.sh`를 실행하세요 — 아무것도 생성하지 않고 Docker,
피드, Managed Postgres 확장(extension)을 점검합니다. 아래 실패 사례들은
실제 세션에서 확인된 것들이며, 배경 설명은
[instructor guide](workshop/instructor-guide.md)에 있습니다.

**ClickPipe가 연결되지 않습니다 (module 05).** 파이프는 여러분의 노트북이
아니라 ClickHouse Cloud의 네트워크에서 연결을 시도하므로,
[01](workshop/01-provision.md)에서 허용 목록(allow-list)에 등록한 IP는
여기에 아무 소용이 없습니다 — Postgres 서비스가 그 연결도 별도로 허용해야
합니다. 또한 `wal_level`이 `logical`이 아니거나, publication
`ny_citibike_pub`에 테이블이 2개가 아니거나, 두 테이블의 replica identity가
`default`가 아니면 파이프가 소스를 거부합니다;
[05](workshop/05-clickpipes.md)에 이 세 가지를 확인하는 쿼리가 있습니다.

**module 03 이후 아무것도 도착하지 않습니다.** 최대 2분 정도 걸릴 수
있습니다: refreshable materialized view에 최대 1분, `pg_cron`에 다시 최대
1분. `02-verify.sql`을 다시 실행하거나 디버깅을 시작하기 전에 3분을
기다리세요.

**Module 06이 항상 `dragged`만 보고합니다.** 작은 station 테이블이
복제되지 않고, 큰 테이블만 복제된 상태입니다. 로컬 테이블 하나가 조인에
섞여 들어가면 오류나 경고 없이 조용히 푸시다운이 무너집니다. 두 테이블을
모두 복제하세요 ([05](workshop/05-clickpipes.md),
[06](workshop/06-pushdown.md)).

**피드가 다운되었습니다.** `clickhouse/01-ingest-rmv.sql`의 두 `url()`
호출을 다른 GBFS 피드로 돌리세요 (Capital Bikeshare로도 동작합니다); 그
외에는 아무것도 바뀌지 않습니다. [다른 도시 사용하기](#다른-도시-사용하기)
참고.

**취소한 백필(backfill)이 여전히 락을 잡고 있습니다 (module 09).**
`psql`을 취소해도 백엔드 프로세스는 취소되지 않습니다. `SELECT pid, state,
query FROM pg_stat_activity WHERE query LIKE '%sim_%';`로 찾아서 `SELECT
pg_terminate_backend(<pid>);`로 중지하세요 ([09](workshop/09-trips.md)).

**세션이 끝나도 비용이 계속 발생합니다.** 두 스케줄러 모두 서버 측에서
동작하므로, 노트북을 닫아도 수집이 멈추지 않습니다.
[08 — 마무리](workshop/08-wrap-up.md)로 종료하세요.

### 검증 상태

이 문서의 모든 주장은 실제로 실행해본 것입니다. 이 섹션은 무엇을 기준으로
실행했는지 설명합니다.

**2026-08-15에 실제 제품을 대상으로 검증함** — ClickHouse Managed Postgres
(PostgreSQL 18.4, PostGIS 3.6.4, pg_cron 1.6, pg_clickhouse 0.3) 및
ClickHouse Cloud 26.4.1:

- `url()`로 실시간 GBFS 피드를 가져오는 ClickHouse — 1,073,635바이트, 2,509개 station으로 파싱됨
- `REFRESH EVERY 1 MINUTE APPEND`로 동작하는 refreshable materialized view가 사람 개입 없이 스냅샷을 누적함
- `pg_clickhouse`가 랜딩 테이블을 임포트하고 Postgres가 이를 읽음
- `pg_cron`이 1분마다 동기화하며 7시간 동안 무인으로 동작함: **1.38M개의 행, 중복된 `(station_key, polled_at)` 쌍 0개**
- 스키마, publication, 그리고 다섯 개의 쿼리 파일 전체

**2026-08-16에 엔드투엔드로 검증함**, 동일한 두 서비스 조합에서
ClickPipes를 연결하고 module 09에서 생성한 trip 테이블을 복제한 상태로:

- **푸시다운, 실행 계획으로 확인.** 쿼리 텍스트 하나, 스키마 접두사 없이 `search_path`로 해석됨: 단일 `Foreign Scan`이 있고 그 `Remote SQL`에 조인, `GROUP BY`, 집계 함수가 모두 담김
- **반례(counter-example).** 동일한 쿼리 텍스트에서 테이블 하나를 로컬로 고정하면 — `dragged`, `Remote SQL`은 컬럼만 선택하며, 오류도 경고도 없음
- **의미 있는 실측 시간**, 9.8M건의 trip 기준: 시간별 trip 집계는 로컬 **10,404 ms** 대 푸시다운 **465 ms**; 4개 릴레이션 조인은 8,234 대 1,606
- **방언 변환(dialect translation)** — `extract(hour FROM …)`이 `toHour(…)`로 나감
- 대시보드 체크포인트 10개 전부 green

**로컬 PostgreSQL 17 + PostGIS 3.6.4 컨테이너에서 검증함:** window 쿼리의
실행 계획 형태 — 정렬(sorting)이 아니라 인덱스로 커버되는 것으로 확인되어,
module 06은 검증을 견디는 더 좁은 범위의 주장을 합니다.

**이 저장소가 내세웠지만 실측으로 반박된 세 가지 주장**은 조용히 삭제하지
않고 모두 본문에서 바로잡았습니다:

- `geom`은 실제로 ClickHouse로 넘어갑니다. `text`로 도착하며, `sql/40-fdw-clickhouse.sql`이 이후 foreign table에서 이를 제거하여, 인덱스 없이 행마다 캐스팅하는 대신 접근 시 눈에 띄게 실패하도록 만듭니다
- 스냅샷 델타로 출발(departure) 건수를 유도하면 과소가 아니라 **과다**집계됩니다 — 공식 발표치 100–150k에 비해 244k/일이 추정되는데, 리포팅 노이즈가 실제 라이더만큼이나 자주 카운트를 낮추는 방향으로도 움직이기 때문입니다
- dimension upsert는 실제로 변경이 있었는지와 무관하게 매분 2,509개 station 전체를 다시 씁니다: 워크숍이 "거의 바뀌지 않는 절반"이라고 부르는 바로 그 테이블에서, 514번의 실행에 걸쳐 1,289,626건의 업데이트가 발생했습니다

**검증되어 유지되는 부정적 결과:** Postgres는 피드를 직접 가져올 수
없습니다. 카탈로그에 `http`나 `pg_net` 확장이 없으며; `plperlu`는 문제없이
설치되지만 서버의 Perl에는 `IO::Socket::SSL`이나 `Net::SSLeay`가 없어서
모든 https 요청이 실패합니다. `sql/03-check-in-db-http.sql`은 여러분
자신의 서비스에서 이 점검을 다시 실행하는데, 이는 권한이 아니라
이미지(image) 자체의 문제이며 플랫폼 업데이트로 바뀔 수 있기 때문입니다.

**아직 실행이 아니라 제품 문서를 바탕으로 작성된 부분:**
[01](workshop/01-provision.md)(프로비저닝)과
[05](workshop/05-clickpipes.md)(ClickPipe 생성)의 콘솔 클릭 절차입니다.
파이프 자체는 실제로 연결되었고 그 이후의 모든 것은 측정되었습니다 —
검증되지 않은 부분은 화면의 순서이며, 바로 이 부분이 가장 자주 바뀝니다.
두 모듈 모두 현재 라벨과 함께 무엇을 찾아야 하는지 설명합니다.

화면에 보이는 것과 단계 설명이 맞지 않는다면,
[issue](https://github.com/litkhai/citi-bike-workshop/issues)로 남길
가치가 있습니다.

### 라이선스

[MIT](LICENSE).

Citi Bike 시스템 데이터는 Lyft Bikes and Scooters, LLC가
[GBFS](https://gbfs.org) 규격에 따라 공개하는 것이며, 실행 시점에 가져올
뿐 이 저장소에 재배포되지 않습니다. MapLibre GL JS는 자체 라이선스 하에
CDN에서 로드됩니다. ClickHouse는 ClickHouse, Inc.의 등록 상표이며, 이
워크숍은 독립적인 교육용 워크숍으로 공식 ClickHouse 제품이 아닙니다.
