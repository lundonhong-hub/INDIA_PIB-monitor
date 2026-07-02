# PIB Monitor — 인도 정부 보도자료(PIB) 실시간 감시기

인도 **Press Information Bureau(PIB)**의 National 보도자료 RSS를 주기적으로 확인해서,
지정 키워드(`copper`, `QCO`, `anti-dumping` 등)가 **제목**에 걸린 신규 발표가 뜨면
**Gmail로 알림 메일**을 보낸다.

동관(copper tube) 제조사 관점의 인도 규제 감시 체계에서 **"1차 그물"** 역할.
(참고: egazette monitor = "3차 그물(법적 확정 관보)". 이 저장소는 그것과 **완전 별개**로 운영.)

---

## 왜 PIB인가 (3중 그물 개념)

| 단계 | 소스 | 성격 | 타이밍 |
|---|---|---|---|
| 1차 그물 | **PIB (이 저장소)** | 정부 보도자료 | 가장 빠름 (신문보다 앞섬) |
| 2차 그물 | DGTR/DGFT (예정) | 무역 실무 공고 | 중간 |
| 3차 그물 | eGazette (별도 저장소) | 법적 확정·발효 | 가장 늦음 |

PIB는 인도 중앙정부의 공식 보도자료 허브다. 상공부·재무부·소비자부(BIS 주무) 등
대부분 부처가 PIB를 통해 발표하므로, copper·무역규제 관련 발표를 **관보보다 먼저** 잡을 수 있다.

**한계:** PIB RSS에는 본문(description)이 없다 → **제목에 키워드가 있어야만** 잡힌다.
본문에만 copper가 있는 발표는 놓칠 수 있어, 키워드를 넓게 잡아두는 게 실전 전략.

---

## 파일 구성

```
pib_monitor/
├── pib_monitor.py                # 메인 로직 (RSS 수신 → 파싱 → 매칭 → 메일)
├── requirements.txt              # (비어있음. 표준 라이브러리만 사용)
├── state.json                    # 이미 알린 PRID 목록 (자동 관리)
├── README.md                     # 이 문서
└── .github/workflows/monitor.yml # 15분 주기 자동 실행 정의
```

---

## 설정 (pib_monitor.py 상단 CONFIG)

- **`RSS_URL`** — ★가장 중요★
  사이트에서 Region을 **"National"**로 고른 뒤 Press Releases RSS 링크의
  **실제 URL**을 확인해서 그 값으로 맞출 것.
  (예: `https://pib.gov.in/RssMain.aspx?ModId=6&Lang=1&Regid=<National번호>`)
  현재 코드엔 임시로 `Regid=3`이 들어있으니 **반드시 본인 확인값으로 교체.**

- **`KEYWORDS`** — 감시 키워드(제목에서 소문자 매칭).
  기본: copper, hindustan copper, quality control order, qco, bis,
  anti-dumping, dgtr, safeguard duty, countervailing 등.
  줄 추가/삭제로 조절. 처음엔 넓게 → 메일 받아보며 노이즈 줄이기.

- **실행 주기** — `.github/workflows/monitor.yml`의 `cron`.
  기본 15분(`*/15 * * * *`). RSS라 가벼워 자주 돌려도 무방.

---

## 배포 (5분)

1. **GitHub에 새 저장소 생성** (Public 권장 — Actions 무제한 무료)
2. 이 폴더의 파일 전부 업로드
3. **Settings → Secrets and variables → Actions**에서 Secret 3개 등록:
   - `GMAIL_USER` — 발신 Gmail 주소
   - `GMAIL_APP_PASSWORD` — Gmail **앱 비밀번호** 16자리 (일반 비번 아님)
   - `ALERT_TO` — 수신 주소 (콤마로 여러 명 가능)
4. **`RSS_URL`을 National 확인값으로 교체** (위 설정 참고)
5. **Actions 탭 → PIB Monitor → Run workflow**로 수동 1회 실행해 동작 확인

> Gmail 앱 비밀번호는 Google 계정 → 보안 → 2단계 인증 → 앱 비밀번호에서 발급.

---

## 동작 방식

1. National 보도자료 RSS(XML)를 `urllib`로 수신 (브라우저 UA 헤더 사용)
2. `<item>`에서 title, link 추출 → link의 `PRID=` 뒤 숫자를 고유 ID로 사용
3. `state.json`과 대조해 **신규만** 추림
4. 신규 제목에 키워드가 걸리면 → Gmail 알림 (제목 + 키워드 + PRID + 링크)
5. 이번에 본 신규 PRID 전부를 `state.json`에 기록 (매칭 안 된 것도 재알림 방지)

---

## 운영 팁 / 주의

- **컨테이너/일부 클라우드 IP에서 403**: PIB가 일부 데이터센터 IP를 막는다.
  GitHub Actions의 IP 대역에서는 대개 통과하지만, 만약 Actions에서도 지속 403이면
  → 헤더 보강 또는 다른 실행 환경 고려. (`fetch_rss`는 3회 재시도한다.)
- **기억 초기화**: `state.json`을 `[]`로 비우면 현재 피드의 매칭분을 다시 받는다.
- **개별 실행 실패 허용**: 인도 서버 순단으로 한 회차 실패해도, 15분 뒤 재시도.
  발표가 피드에 남아있는 한 다음 회차에서 잡는다.
- **메일 실패 시**: 그 회차 신규를 기억에 넣지 않고 exit 1 → 다음 회차에 재알림 시도.

---

## 향후 (2차 그물)

DGTR(반덤핑 조사 개시)·DGFT(수입정책)는 PIB로 안 나가는 실무 공고가 많다.
이들은 RSS가 없어 HTML 파싱이 필요하므로, 별도 모듈로 추가 예정.
