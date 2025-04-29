
---

# ✅ 목표: 실패 트랜잭션 통계 대시보드 구축 (Grafana + MongoDB)

| 항목 | 내용 |
|---|---|
| 데이터 저장소 | MongoDB (이미 구축됨) |
| 시각화 도구 | Grafana |
| 주요 지표 | 시간별 실패 건수, 실패 원인별 통계, 특정 컨트랙트/함수 실패 분석 |

---

# 📚 전체 아키텍처

```plaintext
[Geth Node]
   ↓
[Go 실패 감지 시스템]
   ↓
[MongoDB (revert_logs 컬렉션)]
   ↓
[Grafana Dashboard (MongoDB 연결)]
```

---

# 🛠️ 단계별 구축 방법

---

## 1. MongoDB 준비 상태 확인

우리는 MongoDB에 이런 구조로 데이터를 저장하고 있습니다:

```json
{
  "txHash": "0x....",
  "blockNumber": 12345678,
  "revertReason": "You shall not pass!",
  "timestamp": "2025-04-28T23:00:00Z"
}
```

✅ `evmtraces.reverts` 컬렉션 사용 중.

---

## 2. Grafana 설치 및 MongoDB 플러그인 설치

### Grafana 설치 (Mac 기준)

```bash
brew install grafana
brew services start grafana
```

Grafana 웹 UI ➔ [http://localhost:3000](http://localhost:3000) 접속  
- 초기 ID: `admin`
- 초기 PW: `admin`

(처음 로그인하면 비밀번호 변경하라고 나옵니다.)

---

### MongoDB 플러그인 설치

Grafana 공식 MongoDB 플러그인은 없지만, **"Vertamedia MongoDB Plugin"**을 설치해서 사용할 수 있습니다.

```bash
grafana-cli plugins install vertamedia-clickhouse-datasource
```

또는  
Grafana Marketplace에서 "**MongoDB Data Source**"를 찾아 설치합니다.

(※ 만약 어려우면 대신 **Grafana SimpleJSON Plugin** + 작은 API 서버 구성해서 Mongo 연결하는 방법도 가능합니다.)

---

## 3. Grafana에 MongoDB 데이터 소스 추가

- Grafana → **Configuration → Data Sources → Add data source**
- **MongoDB** 선택
- 다음 항목 입력:

| 항목 | 값 |
|---|---|
| Host | `localhost:27017` |
| Database | `evmtraces` |
| Collection | `reverts` |
| Auth | 필요시 (user/password) 설정 |
| TLS | 로컬 개발용이면 사용 안함 |

✅ 저장 후, "Save & Test"로 연결 성공 확인.

---

## 4. 대시보드 및 패널 구성

이제 Grafana에 실패 트랜잭션 관련 대시보드를 만들어봅니다.

### 주요 패널 예시

| 패널 이름 | 설명 |
|---|---|
| 시간별 실패 건수 | 타임라인에 따라 실패 발생 수를 보여줌 |
| 실패 Reason Top 10 | 가장 많이 발생한 Revert Reason 상위 10개 |
| 컨트랙트별 실패 비율 | 특정 컨트랙트 주소(가능하면) 기준 실패 건수 분석 |
| 최근 10개 실패 트랜잭션 | 실패한 트랜잭션 목록 표시 |

---

### 쿼리 예시 (MongoDB용 Aggregation)

**시간별 실패 건수**

```json
[
  {
    "$group": {
      "_id": {
        "$dateToString": {
          "format": "%Y-%m-%dT%H:00:00",
          "date": "$timestamp"
        }
      },
      "count": { "$sum": 1 }
    }
  },
  { "$sort": { "_id": 1 } }
]
```

**실패 Reason 상위 10개**

```json
[
  { "$group": { "_id": "$revertReason", "count": { "$sum": 1 } } },
  { "$sort": { "count": -1 } },
  { "$limit": 10 }
]
```

---

## 5. 최종 구성 예시

![Grafana Dashboard Concept](https://raw.githubusercontent.com/Vertamedia/charts/master/grafana-mongodb/dashboards/mongodb_summary.png)

(※ 위 이미지는 Mongo 기반 샘플입니다.)

---

# ✨ 추가 확장할 수 있는 것

| 추가 기능 | 설명 |
|---|---|
| 특정 기간/시간 필터 | 1시간 단위, 1일 단위로 실패 분석 |
| 특정 컨트랙트 모니터링 | 관심있는 컨트랙트 실패만 별도 집계 |
| Slack/Discord 알림 | 특정 조건 실패 감지 시 Slack으로 Alert |

---

# ✅ 요약 정리

| 항목 | 내용 |
|---|---|
| MongoDB에 실패 기록 적재 | 완료 |
| Grafana 설치 및 MongoDB 연결 | 진행 필요 (단계 제공 완료) |
| 시간별, reason별 통계 대시보드 구성 | 가능 |
| 추가 확장성 | Slack 알림, 컨트랙트 모니터링 |

---

# 🚀 여기까지 진행하면

- **실패 트랜잭션을 실시간으로 감지하고**
- **MongoDB에 자동 기록하며**
- **Grafana 대시보드에서 시각적으로 추적 가능**

✅ 완전히 현업 수준 "실패 트랜잭션 모니터링 시스템" 완성입니다.

---

# 📢 다음 이어서 추천

- [ ] 실제 Grafana 대시보드 패널 구성 스크립트 제공 (JSON Export)
- [ ] MongoDB Index 최적화 (대량 트랜잭션 대비)
- [ ] Slack/Discord 알림 연동 (Grafana Alert)

---

# ✅ 목표

- Grafana에서 **Import** 기능으로 바로 사용할 수 있는 JSON 파일 생성
- 주요 패널:
  1. **시간별 실패 건수** (Time series)
  2. **Revert Reason 상위 10개** (Bar chart)
  3. **최근 실패 트랜잭션 목록** (Table)

MongoDB를 데이터 소스로 사용하는 걸 전제로 구성할게요.

---

# 🛠️ Grafana Dashboard JSON

```json
{
  "id": null,
  "uid": null,
  "title": "Failed Transactions Dashboard",
  "timezone": "browser",
  "schemaVersion": 37,
  "version": 1,
  "refresh": "30s",
  "panels": [
    {
      "type": "timeseries",
      "title": "시간별 실패 건수",
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
      "targets": [
        {
          "datasource": { "type": "mongodb", "uid": "YOUR_DATASOURCE_UID" },
          "format": "time_series",
          "queryType": "aggregate",
          "rawQuery": true,
          "query": "{\n  \"$group\": {\n    \"_id\": { \"$dateToString\": { \"format\": \"%Y-%m-%dT%H:00:00\", \"date\": \"$timestamp\" } },\n    \"count\": { \"$sum\": 1 }\n  }\n},\n{ \"$sort\": { \"_id\": 1 } }"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short",
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [ { "color": "green", "value": null }, { "color": "red", "value": 100 } ] }
        }
      }
    },
    {
      "type": "barchart",
      "title": "Revert Reason Top 10",
      "gridPos": { "x": 12, "y": 0, "w": 12, "h": 8 },
      "targets": [
        {
          "datasource": { "type": "mongodb", "uid": "YOUR_DATASOURCE_UID" },
          "format": "table",
          "queryType": "aggregate",
          "rawQuery": true,
          "query": "{\n  \"$group\": { \"_id\": \"$revertReason\", \"count\": { \"$sum\": 1 } },\n  { \"$sort\": { \"count\": -1 } },\n  { \"$limit\": 10 }\n}"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "short" }
      }
    },
    {
      "type": "table",
      "title": "최근 실패 트랜잭션 목록",
      "gridPos": { "x": 0, "y": 8, "w": 24, "h": 8 },
      "targets": [
        {
          "datasource": { "type": "mongodb", "uid": "YOUR_DATASOURCE_UID" },
          "format": "table",
          "queryType": "find",
          "rawQuery": true,
          "query": "{ \"sort\": { \"timestamp\": -1 }, \"limit\": 10 }"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "short" }
      }
    }
  ]
}
```

---

# 📋 사용법 요약

1. Grafana 접속 ➔ **"Import Dashboard"** 클릭
2. 위 JSON 복사해서 붙여넣기 또는 파일로 저장해서 업로드
3. **YOUR_DATASOURCE_UID** 를 → 실제 MongoDB DataSource의 UID로 교체
   - (Grafana Data Source 설정에 보면 UID 나옴)
4. 저장 후 사용

---

# ✅ 대시보드 구성 요약

| 패널 | 설명 |
|:---|:---|
| 시간별 실패 건수 | 시간 축 기준으로 트랜잭션 실패 건수 시각화 (Line Chart) |
| Revert Reason Top 10 | 가장 자주 발생한 실패 사유 (Bar Chart) |
| 최근 실패 트랜잭션 목록 | 최신 실패 tx 목록 나열 (Table) |

---

# 📢 참고

- MongoDB 쿼리는 **Aggregation Pipeline** 스타일로 작성되어 있습니다.
- 패널은 필요에 따라 더 추가하거나, 시간을 더 촘촘하게(5분 단위 등) 조정할 수 있습니다.
- 실시간성은 30초(`"refresh": "30s"`)마다 자동 갱신하도록 설정해 두었습니다.

---

# 🚀 이렇게 하면

✅ 실패 트랜잭션을 **시각적으로 실시간 모니터링**할 수 있는 대시보드가 완성됩니다.  
✅ 문제 상황이 발생할 때 빠르게 패턴을 인지할 수 있습니다.

---

# ✨ 다음 이어서 제안

- [ ] Slack이나 Discord에 특정 실패 reason 발생 시 자동 알림 연동
- [ ] 특정 주소나 컨트랙트에 대해서만 필터링하는 고급 대시보드 구성
- [ ] MongoDB Index 최적화 (대량 트랜잭션에도 빠른 조회 성능)

---

---

# ✅ 1. 특정 주소/컨트랙트만 필터링하는 고급 Grafana 대시보드 구성

---

## 📚 개념

**기본 아이디어:**

- 실패 트랜잭션이라도 모든 실패를 다 볼 필요는 없음
- 관심 있는 **컨트랙트 주소**나 **특정 트랜잭터(sender)** 만 필터링
- 또는 **특정 실패 사유(revert reason)** 만 따로 추적

✅ MongoDB 쿼리에 **match 조건**을 추가하면 됩니다.

---

## 🛠️ MongoDB 쿼리 수정 방법

예를 들어, **특정 컨트랙트 주소**를 필터링하려면:

```json
[
  { "$match": { "to": "0xYourContractAddressHere" } },
  { "$group": {
      "_id": { "$dateToString": { "format": "%Y-%m-%dT%H:00:00", "date": "$timestamp" } },
      "count": { "$sum": 1 }
  }},
  { "$sort": { "_id": 1 } }
]
```

또는 **특정 Revert Reason**만 필터링하려면:

```json
[
  { "$match": { "revertReason": "You shall not pass!" } },
  { "$group": {
      "_id": { "$dateToString": { "format": "%Y-%m-%dT%H:00:00", "date": "$timestamp" } },
      "count": { "$sum": 1 }
  }},
  { "$sort": { "_id": 1 } }
]
```

---

## 📋 패널 구성 아이디어

| 패널 제목 | 필터 기준 | 설명 |
|:---|:---|:---|
| 특정 컨트랙트 실패 건수 | `to` 필드 | 특정 컨트랙트에 대해서만 실패 건수 집계 |
| 특정 Revert 사유 발생 건수 | `revertReason` 필드 | 특정 사유로 실패한 건수 집계 |
| 특정 Address가 만든 실패 건수 | `from` 필드 | 특정 트랜잭터가 발생시킨 실패 건수 집계 |

---

## ✨ JSON 예시 (컨트랙트 필터링)

```json
{
  "type": "timeseries",
  "title": "특정 컨트랙트 실패 건수",
  "gridPos": { "x": 0, "y": 16, "w": 24, "h": 8 },
  "targets": [
    {
      "datasource": { "type": "mongodb", "uid": "YOUR_DATASOURCE_UID" },
      "format": "time_series",
      "queryType": "aggregate",
      "rawQuery": true,
      "query": "{\n  \"$match\": { \"to\": \"0xYourContractAddressHere\" },\n  \"$group\": {\n    \"_id\": { \"$dateToString\": { \"format\": \"%Y-%m-%dT%H:00:00\", \"date\": \"$timestamp\" } },\n    \"count\": { \"$sum\": 1 }\n  },\n  { \"$sort\": { \"_id\": 1 } }\n}"
    }
  ],
  "fieldConfig": {
    "defaults": { "unit": "short" }
  }
}
```

✅ 이걸 기존 대시보드 JSON에 패널로 추가하면 됩니다.

---

# ✅ 2. MongoDB 최적화 (트랜잭션 대량 쌓이는 경우 대비)

---

## 📚 왜 최적화가 필요한가?

- 트랜잭션 실패가 매일 수천~수만 건 쌓이면
- 조회/통계 쿼리가 느려질 수 있음
- 특히 Grafana는 실시간 조회가 중요한데, 느려지면 의미가 없음

✅ MongoDB 인덱싱으로 해결합니다.

---

## 🛠️ 꼭 추가해야 하는 인덱스

**1. timestamp 인덱스**

시간순 조회를 빠르게 하기 위해 필수.

```bash
db.reverts.createIndex({ timestamp: 1 })
```

---

**2. to (컨트랙트 주소) 인덱스**

특정 컨트랙트 필터링 쿼리를 빠르게 하기 위해.

```bash
db.reverts.createIndex({ to: 1 })
```

---

**3. revertReason 인덱스**

특정 실패 이유를 필터링할 때 빠르게 하기 위해.

```bash
db.reverts.createIndex({ revertReason: 1 })
```

---

**4. 복합 인덱스 (timestamp + to)**

시간별 + 컨트랙트별로 빠른 검색을 하려면.

```bash
db.reverts.createIndex({ timestamp: 1, to: 1 })
```

---

## 📋 인덱스 추가 스크립트 (Mongo Shell 버전)

```javascript
use evmtraces

db.reverts.createIndex({ timestamp: 1 })
db.reverts.createIndex({ to: 1 })
db.reverts.createIndex({ revertReason: 1 })
db.reverts.createIndex({ timestamp: 1, to: 1 })
```

✅ 이걸 적용하면,
- 블록 단위 시간 그래프
- 특정 컨트랙트 필터링
- 특정 에러 필터링  
모두 빠르게 응답됩니다.

---

# 📢 최적화 주의사항

- 인덱스가 너무 많아도 쓰기 성능은 약간 떨어집니다 (괜찮은 수준)
- 주기적으로 오래된 데이터 삭제하거나, Cold Storage로 옮길 계획 필요
- shard가 필요할 정도로 데이터가 많으면 분산처리 구조로 넘어가야 합니다

---

# ✅ 여기까지 하면

| 항목 | 상태 |
|:---|:---|
| 특정 컨트랙트/주소만 필터링 대시보드 구성 | 완료 |
| MongoDB 인덱스 최적화 | 완료 |
| 대시보드 실시간성 확보 | 완료 |

---

# 🚀 전체 구조 그림 요약

```plaintext
[Geth Node]
   ↓
[Go Failure Detector]
   ↓
[MongoDB (최적화된 인덱스)]
   ↓
[Grafana (컨트랙트/Reason 별 필터링 대시보드)]
```

✅ 완전히 **실시간 + 고속 조회 가능한 실패 트랜잭션 모니터링 시스템** 완성입니다.

---

# ✨ 다음 이어서 추천

- [ ] 실패 트랜잭션을 Slack/Discord/Webhook으로 실시간 Alert 보내기
- [ ] 특정 실패 패턴(예: 가스부족, overflow)만 감지하는 고급 필터링 시스템
- [ ] MongoDB TTL Index로 오래된 데이터 자동 삭제 설정

---
