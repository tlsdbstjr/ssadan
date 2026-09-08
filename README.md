# SSADAN - SSAFY 식단 알림 봇 🍽️

매일 Mattermost와 Discord webhook으로 점심 식단을 알려주는 자동화 봇입니다.

## 프로젝트 구조

```
.
├── .github/
│   └── workflows/          # GitHub Actions를 활용한 자동화 스케줄러
│       ├── daily_notify.yml   # 매일 오전 9시 10분 점심 알림
│       └── weekly_crawl.yml   # 매주 월요일 웰스토리 API 식단 자동 크롤링
├── db/                     # 식단 Markdown 파일 저장소 (yyyy-mm-dd.md)
│   └── .gitkeep
├── src/                    # 핵심 실행 로직 (Python)
│   ├── main.py             # 전체 프로세스 제어 (Entry point)
│   ├── welstory_crawler.py # 웰스토리 API 식단 크롤링 및 Markdown 변환
│   ├── mm_sender.py        # Mattermost 웹훅 발송 로직
│   ├── discord_sender.py   # Discord 웹훅 발송 로직
│   └── notification_sender.py  # 통합 알림 발송 (Mattermost + Discord)
├── requirements.txt        # 필요 라이브러리
├── README.md               # 프로젝트 설명
└── .env.example            # 환경변수 샘플
```

## 작동 방법

1. **식단 크롤링**: 매주 월요일 [welplan.pmh.codes](https://welplan.pmh.codes)에서 멀티캠퍼스 한 주(월~금) 식단을 가져와 `db/yyyy-mm-dd.md` 파일로 저장
   - 20층 식단: welplan.pmh.codes API 사용
   - 10층 식단: Mattermost 채널에서 식단 이미지를 자동 수집한 뒤 Gemini API로 파싱해 병합 (실패 시 placeholder 유지)
2. **일일 알림**: 매일 오전 9시 10분~9시 40분(KST)에 그 날 점심 식단을 Mattermost와 Discord로 전송 (GitHub Actions 스케줄링 지연 최대 30분 발생 가능)

## 설치

### 1. 의존성 설치

```bash
pip install -r requirements.txt
```

### 2. 환경 설정

```bash
cp .env.example .env
```

`.env` 파일 설정:
```bash
# Mattermost Webhook URL (식단 알림용)
MATTERMOST_WEBHOOK_URL=https://your-mattermost-server.com/hooks/xxx

# Discord Webhook URL (식단 알림용, 선택사항)
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/xxx/yyy

# 식당 검색어 (기본값: 멀티캠퍼스, 변경 필요 시만 설정)
# WELSTORY_RESTAURANT_QUERY=멀티캠퍼스

# 10층 식단 이미지 자동 수집 (선택사항, 미설정 시 placeholder 유지)
MATTERMOST_BASE_URL=https://your-mattermost-server.com
MATTERMOST_CHANNEL_ID=your-channel-id
MM_LOGIN_JSON={"login_id": "your-email@example.com", "password": "your-password"}
GEMINI_API_KEY=your-gemini-api-key
```

**참고**: `MATTERMOST_WEBHOOK_URL` 또는 `DISCORD_WEBHOOK_URL` 중 최소 하나는 설정되어야 합니다.

## 사용법

### 로컬 실행

#### 1. 웰스토리 API로 이번 주 식단 크롤링
```bash
cd src
python main.py crawl --db ../db
```

#### 2. 오늘 점심 식단 전송
```bash
cd src
python main.py daily --db ../db
```

#### 3. 특정 날짜 점심 식단 전송
```bash
cd src
python main.py daily --date 2026-01-15 --db ../db
```

### GitHub Actions 자동화

#### 1. GitHub Secrets 설정

Repository Settings > Secrets and variables > Actions에서 다음 Secret 추가:
- `MATTERMOST_WEBHOOK_URL`: Mattermost Incoming Webhook URL (식단 알림용, 선택사항)
- `DISCORD_WEBHOOK_URL`: Discord Webhook URL (식단 알림용, 선택사항)
- `MATTERMOST_BASE_URL`: Mattermost 서버 URL (10층 이미지 수집용, 선택사항)
- `MATTERMOST_CHANNEL_ID`: 10층 식단 이미지가 올라오는 채널 ID (선택사항)
- `MM_LOGIN_JSON`: Mattermost 로그인 정보 JSON, 예: `{"login_id":"user@example.com","password":"pass"}` (선택사항)
- `GEMINI_API_KEY`: Google Gemini API 키 (10층 이미지 파싱용, 선택사항)

> **참고**: 10층 관련 Secret이 없으면 10층 식단은 placeholder로 표시됩니다. 식단 크롤링은 [welplan.pmh.codes](https://welplan.pmh.codes)를 통해 인증 없이 진행됩니다.

#### 2. 주간 식단 크롤링

매주 월요일 오전 8시(KST)에 자동으로 실행됩니다.
수동 실행: Actions > "Weekly Menu Crawl (Wellstory API)" > "Run workflow"

#### 3. 일일 알림

매일 오전 9시 10분~9시 40분(KST)에 자동으로 실행됩니다 (GitHub Actions 지연 발생 가능).
수동 실행: Actions > "Daily Lunch Notification" > "Run workflow"

### Firebase Functions 자동화

GitHub Actions 예약 지연을 피하기 위한 Firebase Functions 배포 구성이 포함되어 있습니다. 기존 Python 로직을 그대로 호출하며, 주간 메뉴는 작업 환경의 로컬 파일 대신 Cloud Storage에 저장합니다.

1. Firebase 프로젝트에서 **Cloud Functions**와 **Cloud Storage**를 활성화하고 Blaze 요금제를 사용합니다.
2. Firebase CLI 로그인 후 프로젝트를 선택합니다.

```bash
firebase login
firebase use --add
```

3. `src/.env.example`을 `src/.env`로 복사하고 `MENU_STORAGE_BUCKET`에 Firebase Storage 버킷 이름(예: `your-project-id.firebasestorage.app`)을 입력합니다.
4. GitHub Secrets의 값을 Firebase Secret Manager로 등록합니다.

```bash
firebase functions:secrets:set MATTERMOST_WEBHOOK_URL
firebase functions:secrets:set DISCORD_WEBHOOK_URL
firebase functions:secrets:set WELSTORY_USERNAME
firebase functions:secrets:set WELSTORY_PASSWORD
firebase functions:secrets:set MATTERMOST_BASE_URL
firebase functions:secrets:set MATTERMOST_CHANNEL_ID
firebase functions:secrets:set MM_LOGIN_JSON
firebase functions:secrets:set GEMINI_API_KEY
```

각 Secret은 선택 기능을 쓰지 않아도 등록해야 하며, 사용하지 않는 값은 빈 값으로 입력할 수 있습니다. 웹훅은 최소 하나에 실제 값을 입력해야 합니다.

5. 배포합니다.

```bash
firebase deploy --only functions
```

배포 시 Cloud Scheduler 작업이 자동 생성됩니다.

- `crawl_weekly_menu`: 매주 월요일 09:20 (Asia/Seoul), 메뉴를 `menus/YYYY-MM-DD.md`에 저장
- `send_daily_lunch_notification`: 평일 09:30 (Asia/Seoul), Storage에서 메뉴를 읽어 웹훅 발송

먼저 크롤링 함수를 수동 실행해 Storage 파일 생성을 확인한 후 알림 함수를 시험하세요. 정상 확인 전에는 GitHub Actions 스케줄을 끄지 마세요. 확인 후에는 두 스케줄을 비활성화하거나 제거해야 중복 전송되지 않습니다.

> Scheduler는 드물게 중복 호출될 수 있습니다. 두 함수는 동시 실행을 한 인스턴스로 제한합니다.

## 출력 형식 예시

```markdown
## 🍴 SSAFY 주간메뉴표 (03월 02일 ~ 03월 06일)

| 구분 | 03월 02일 (월) | 03월 03일 (화) | 03월 04일 (수) | 03월 05일 (목) | 03월 06일 (금) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **20F 일반식 (A. 한식)** | 부대찌개, 현미밥, ... | ... | ... | ... | ... |
| **20F 일반식 (B. 일품)** | 카레라이스 | ... | ... | ... | ... |
```

## Mattermost Webhook 설정

1. Mattermost > Integrations > Incoming Webhooks
2. "Add Incoming Webhook" 클릭
3. 채널 선택 및 설정
4. Webhook URL 복사
5. `.env` 파일 또는 GitHub Secrets에 추가

## Discord Webhook 설정

1. Discord 서버에서 알림을 받을 채널 선택
2. 채널 설정(⚙️) > 연동 > 웹후크
3. "새 웹후크" 클릭
4. 웹후크 이름 설정 (예: "식단봇")
5. "웹후크 URL 복사" 클릭
6. `.env` 파일 또는 GitHub Secrets에 추가

## 커스터마이징

다른 식당의 식단을 사용하려면 `WELSTORY_RESTAURANT_QUERY` 환경변수를 설정하세요 (기본값: `멀티캠퍼스`).

welplan.pmh.codes API 응답의 코너명(`menuCourseName`)이 원하는 형식과 다를 경우, `src/welstory_crawler.py`의 `fetch_weekly_meal_data()` 메서드에서 처리 부분을 수정하세요.

## 라이선스

MIT License
