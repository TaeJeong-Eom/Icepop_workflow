# Icepop Workflow — 무인 아이스크림 매장 매출 수집 · 발주 추천 자동화

가족이 운영하는 무인 아이스크림 매장 2곳의 **매출 수집 → 판매량 집계 → 발주 추천 → 카카오톡 발송**을 n8n으로 자동화한 시스템입니다.
**2026.05부터 실제 매장 운영에 쓰고 있습니다.**

`n8n (self-hosted)` `Oracle Cloud ARM` `Docker` `Caddy(HTTPS)` `Google Sheets` `Looker Studio` `Kakao API`

<table>
<tr>
<td width="34%" align="center"><img src="images/kakao-order-recommendation.png" alt="발주 추천 카카오톡"><br><sub>매주 수요일 아침 도착하는 발주 추천 카톡</sub></td>
<td width="66%" align="center"><img src="images/dashboard-overview.png" alt="Looker Studio 대시보드"><br><sub>매일 쌓이는 매출 데이터로 만든 Looker Studio 대시보드 (금액은 가림)</sub></td>
</tr>
</table>

## Before → After

| | Before | After |
|---|---|---|
| 매출 확인 | 매일 POS 사이트에 접속해서 확인 | 매일 23:30 자동 수집 · 시트 적재 (하루 약 100건) |
| 발주 | 매주 약 1시간 동안 판매량 보고 수기 정리 | 추천 목록이 카톡으로 도착, 확인에 약 3분 |

<table>
<tr>
<td width="50%" align="center"><img src="images/legacy-pos-main.png" alt="레거시 POS"><br><sub><b>Before</b> — API 없는 레거시 JSP POS (IE 모드)</sub></td>
<td width="50%" align="center"><img src="images/n8n-execution-log.png" alt="n8n 실행 로그"><br><sub><b>After</b> — 매일 23:30 자동 실행되는 수집 워크플로우</sub></td>
</tr>
</table>

## 시스템 구조

```mermaid
flowchart LR
    POS[(레거시 POS<br/>JSP · EUC-KR)] -->|세션 로그인 + HTTP 재현| COL
    subgraph Daily["매일 23:30"]
        COL[일일 매출 수집] --> SUB1[조회 서브<br/>상품별·영수증별·시간대별]
        SUB1 --> SUB2[적재 서브<br/>유니크키 upsert]
    end
    SUB2 --> SHEET[(Google Sheets)]
    SHEET --> LOOKER[Looker Studio]
    subgraph Weekly["매주 수요일"]
        WF0[WF0 6주 판매량 갱신<br/>08:30] --> WF1[WF1 발주 대기 빌드<br/>09:00] --> WF2[WF2 발주 추천<br/>09:05] --> WF3[WF3 카톡 발송 + 큐 정리<br/>09:10]
    end
    SHEET -->|gviz 쿼리| WF0
    WF3 --> KAKAO[카카오톡]
```

## 핵심 설계 결정

### 1. DB 대신 Google Sheets
- **문제**: 데이터를 실제로 보고 고치는 사람은 개발자가 아닌 매장 운영자.
- **선택**: 시트를 저장소로 쓰고 Looker Studio를 바로 연결.
- **이유**: 운영자가 직접 열어보고 수정할 수 있어야 하고, 매장 2곳 규모에서는 DB 운영 부담이 더 큼.

### 2. 유니크키 + upsert (멱등 적재)
- **문제**: 같은 날짜를 다시 수집하면 행이 그대로 쌓임. 실제로 약 6.5만 행까지 불어났고, 정리해 보니 실제 데이터는 6,058행이었음.
- **선택**: `매출일자_매장코드_바코드` 유니크키를 만들고 `Append or Update`로 적재.
- **결과**: 같은 구간을 몇 번 다시 돌려도 중복이 생기지 않음. 백필도 안전하게 재실행 가능.

<img src="images/sheet-sales-dedup-key.png" alt="중복방지키 컬럼">
<sub>오른쪽 끝 '중복방지키' 컬럼이 upsert 기준 (매장코드는 가림)</sub>

### 3. 전체 조회 대신 gviz 쿼리
- **문제**: 판매 기록이 쌓일수록 시트 전체를 읽으면 API 호출 제한과 처리 시간 문제가 생김.
- **선택**: gviz 엔드포인트에 SQL 비슷한 쿼리를 보내서 필요한 행만 받아옴.
- **주의한 점**: 응답이 JSONP로 감싸져 있어서 정규식으로 벗겨야 하고, 모든 컬럼이 text면 첫 행이 헤더로 들어옴.

### 4. 워크플로우를 역할별로 분리 (WF0~WF3)
- **문제**: 하나로 묶으면 어느 단계에서 실패했는지 알기 어렵고, 처음부터 다시 돌려야 함.
- **선택**: 집계 → 대기열 → 추천 → 발송을 시간차를 두고 분리.
- **결과**: 실패한 단계만 다시 실행 가능. 카톡 발송에 성공한 뒤에만 큐에서 지우기 때문에 발송이 실패해도 추천이 유실되지 않음.

<img src="images/n8n-wf1-order-queue.png" alt="WF1 발주 대기 빌드">
<sub>WF1 발주_대기 빌드 — gviz로 마스터·6주 판매량·대기열을 조회한 뒤 메인 로직에서 계산</sub>

<img src="images/n8n-wf3-kakao-send.png" alt="WF3 카톡 발송 + 큐 정리">
<sub>WF3 카톡 발송 + 큐 정리 — 매장별 발송 분기 후, 발송이 끝나면 대기열을 정리</sub>

## 레거시 POS 연동

공식 API가 없어서 브라우저에서 오가는 HTTP 요청을 캡처해 분석했습니다. 세션 쿠키(JSESSIONID) 기반 로그인, EUC-KR 인코딩, 요청 순서에 의존하는 stateful JSP 구조를 n8n HTTP Request 노드로 재현했습니다.

## 대시보드

<table>
<tr>
<td width="50%"><img src="images/dashboard-store.png" alt="매장별 분석"></td>
<td width="50%"><img src="images/dashboard-product.png" alt="상품별 분석"></td>
</tr>
</table>

## 파일 안내

- `[자동] *` — 스케줄로 도는 메인 워크플로우
- `[서브] *` — 다른 워크플로우가 호출하는 재사용 모듈
- `[백필] *` — 누락 구간 복구용 수동 실행
- `[수동] *` / `[문서] *` — 최초 토큰 발급, 시스템 문서(스티키 노트)
- `images/` — README 이미지 (개인정보·매출 금액은 가림 처리)

## 그대로 가져다 쓰려면

보안을 위해 아래 값은 모두 placeholder로 바꿔두었습니다. n8n에서 Import한 뒤 직접 채우거나 Credential로 옮겨서 쓰면 됩니다.

| placeholder | 의미 |
|---|---|
| `YOUR_ICEPOP_SHEET_ID` | Google Sheets 문서 ID |
| `STORE_CODE_1`, `STORE_CODE_2` | POS 매장 코드 |
| `{{POS_USER_ID}}`, `{{POS_PASSWORD}}` | POS 로그인 정보 |
| `{{KAKAO_REST_API_KEY}}`, `{{CLIENT_SECRET}}`, `{{AUTH_CODE}}` | 카카오 API 키 · 인가코드 |

Credential 연결 정보와 실행 데이터(pinData)는 내보낼 때 제거했습니다.
