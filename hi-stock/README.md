# 테마찾기 - 테마주 예측 & 오늘의 상승 테마

## 📱 다운로드

<a href="https://apps.apple.com/kr/app/%ED%85%8C%EB%A7%88%EC%B0%BE%EA%B8%B0/id1497642441">
  <img src="https://img.shields.io/badge/App_Store-0D96F6?style=for-the-badge&logo=app-store&logoColor=white" />
</a>
<a href="https://play.google.com/store/apps/details?id=com.woojin.hi_stock_flutter">
  <img src="https://img.shields.io/badge/Google_Play-414141?style=for-the-badge&logo=google-play&logoColor=white" />
</a>

---

## 🛠 기술 스택

### Client (Flutter App)
| 분류 | 기술 |
|------|------|
| **Framework** | Flutter 3.6, Dart |
| **상태관리** | GetX |
| **광고** | Google AdMob |
| **분석** | Firebase Analytics |

### Server (Data Pipeline)
| 분류 | 기술 |
|------|------|
| **Runtime** | Python 3.9 |
| **Infra** | AWS EC2 t2.micro (Seoul) |
| **Database** | Supabase (PostgreSQL + RLS) |
| **API** | KIS Open API (한국투자증권) |

---

## 🏗 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                      Flutter App                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   GetX      │  │    Hive     │  │  Keychain   │      │
│  │ Controller  │  │ (로컬 캐시)   │  │ (기기 ID)     │     │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                     Supabase                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ PostgreSQL  │  │     RLS     │  │  Real-time  │     │
│  │ (stock DB)  │  │  (보안정책)   │  │   (순위)     │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────▲────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────┐
│              AWS EC2 (ap-northeast-2)                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Python Scheduler (systemd)            │   │
│  │  ┌─────────────┐      ┌─────────────────────┐   │   │
│  │  │ KIS Open API│ ───▶ │ 10분마다 시세 수집      │   │   │
│  │  │   (한국투자)  │      │                     │   │   │
│  │  └─────────────┘      └─────────────────────┘   │   │
│  │  • 장 시간: 09:00~16:00 KST (평일)                 │   │
│  │  • 공휴일 자동 휴장                                 │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## ✨ 주요 기능

### 1. 테마 예측 게임
- 매일 12:30(한국시간) 전까지 상승할 테마 선택
- 16:00 이후 결과 확인 및 순위 산정
- 연속 적중 시 스트릭 보상 (5연속 적중 시 기프티콘)

### 2. 실시간 순위
- 전체 유저 예측 순위 확인
- 오늘의 적중자 목록

### 3. 테마/종목 정보
- 오늘의 상승/하락 테마 목록
- 테마별 종목 구성 및 등락률
- KOSPI/KOSDAQ 종목 지연 시세

---

## 🔐 기술적 특징

### 아키텍처 의사결정

**왜 Firebase가 아닌 Supabase인가?**
- RLS(Row Level Security)로 서버 시간 기반 마감 검증을 DB 레벨에서 강제
- Firebase Security Rules는 클라이언트 타임스탬프 의존 → 조작 가능
- PostgreSQL의 `now()` 함수로 서버 시간 기준 INSERT 제한

**왜 실시간 WebSocket이 아닌 10분 폴링인가?**
- Supabase Free Tier 동시 연결 제한 (200 connections)
- 주식 시세는 실시간성보다 정확성이 중요 (장중 변동폭 제한)
- 트래픽 패턴: 점심 마감(12:30) 직전 피크 → 폴링으로 충분

---

### 보안: 클라이언트 시간 조작 방어

**공격 벡터**: 기기 시간을 12:29로 조작 → 마감 후에도 예측 제출 시도

**방어 전략 (이중 검증)**
```
[Client]                    [Server]
    │                           │
    ├─ 1. 로컬 시간 체크   ────────┤  (UX용, 빠른 피드백)
    │   if (now > 12:30)        │
    │     return "마감됨"        │
    │                           │
    ├─ 2. INSERT 요청  ─────────→│
    │                           ├─ RLS Policy 검증
    │                           │  now() AT TIME ZONE 'Asia/Seoul'
    │                           │  < '12:30:00'
    │                           │
    │←─ 3a. 성공  ───────────────┤
    │←─ 3b. RLS 위반 에러   ──────┤  (시간 조작 감지)
```

```sql
-- Supabase RLS Policy (서버 시간 기준)
CREATE POLICY "prediction_insert_policy" ON predictions
FOR INSERT WITH CHECK (
  (now() AT TIME ZONE 'Asia/Seoul')::time < '12:30:00'::time
);
```

**기기 ID 스푸핑 한계 인지**
- iOS Keychain: 앱 삭제 후에도 유지, 탈옥 기기에서 복제 가능
- Android SharedPreferences: 앱 삭제 시 리셋 → 다중 계정 가능
- **수용한 트레이드오프**: 무료 서비스에서 완벽한 기기 식별은 불가능, 어뷰징은 수동 검토로 대응

---

### 데이터 파이프라인 운영

**2,500+ 종목 배치 처리 최적화**
```python
# 문제: KIS API 초당 20회 제한
# 해결: 청크 단위 처리 + 지수 백오프

for chunk in chunks(stock_codes, size=20):
    try:
        fetch_prices(chunk)
    except RateLimitError:
        time.sleep(exponential_backoff())
        retry()
```

**장애 대응 전략**
| 장애 유형 | 대응 |
|----------|------|
| KIS 토큰 만료 | 자동 갱신 (만료 1시간 전 refresh) |
| API 일시 장애 | 3회 재시도 + 지수 백오프 (1s → 2s → 4s) |
| EC2 재시작 | systemd 자동 복구 + 마지막 성공 시점부터 재개 |
| 공휴일/휴장 | 한국거래소 휴장일 캘린더 연동 |

**모니터링 (무료 인프라 제약 내)**
- CloudWatch 기본 메트릭 (CPU, 메모리)
- 커스텀 로그: 수집 성공/실패 카운트 → 일일 리포트

---

### 클라이언트 상태 동기화

**문제**: 앱 재설치 시 로컬 예측 기록 유실 → 비교 섹션 렌더링 불가

**해결**: PredictionRecord에 비교용 데이터 스냅샷 저장
```dart
class PredictionRecord {
  final String topThemeName;      // 1위 테마명 (스냅샷)
  final double topThemeChangePct; // 1위 등락률 (스냅샷)
  // ...
}

// 결과 확정 시 스냅샷 저장 → 앱 재설치 후에도 "내 선택 vs 정답" 표시 가능
```

---

## 📁 프로젝트 구조

### Flutter App
```
lib/app/
├── core/services/          # 비즈니스 로직
│   ├── prediction_service.dart
│   ├── supabase_service.dart
│   └── device_id_service.dart
├── data/models/            # 데이터 모델
├── ui/pages/               # 화면 UI
├── ad/                     # 광고 (Native, Rewarded)
└── routes/                 # 라우팅
```

### Python Server
```
hi-stock-server/
├── scheduler.py            # 메인 스케줄러 (systemd)
├── update_stocks.py        # 전 종목 시세 업데이트
├── update_markets.py       # KOSPI/KOSDAQ 지수 업데이트
└── kis_auth.py             # KIS API 인증 (토큰 관리)
```

---

## 📊 수집 데이터

### stock 테이블 (53개 필드)

**기본 정보**
| 필드 | 설명 |
|------|------|
| code | 종목코드 |
| name | 종목명 |
| market | 시장구분 (KOSPI/KOSDAQ) |
| updated_at | 마지막 업데이트 시각 |

**시세 정보**
| 필드 | 설명 |
|------|------|
| current_price | 현재가 |
| prev_close | 전일 종가 |
| change_abs / change_pct | 전일 대비 (절대값/%) |
| open_price / high_price / low_price | 당일 시가/고가/저가 |
| volume / trade_amount | 거래량/거래대금 |
| bid_price / ask_price | 매수/매도 호가 |

**투자지표**
| 필드 | 설명 |
|------|------|
| market_cap | 시가총액 |
| per / pbr / eps / bps | 투자지표 |
| turnover_rate | 회전율 |
| shares_outstanding | 상장주식수 |
| foreign_ratio / foreign_net_buy | 외국인 보유율/순매수 |

**가격 범위**
| 필드 | 설명 |
|------|------|
| week52_high / week52_low | 52주 최고/최저가 |
| d250_high / d250_low | 250일 최고/최저가 |
| year_high / year_low | 연중 최고/최저가 |

**종목 상태**
| 필드 | 설명 |
|------|------|
| sector | 업종 |
| credit_able / short_able | 신용/공매도 가능 여부 |
| is_etf / is_etn / is_elw | ETF/ETN/ELW 여부 |

### prediction 테이블 (유저 예측)
| 필드 | 설명 |
|------|------|
| device_id | 기기 고유 ID |
| prediction_date | 예측 날짜 |
| theme_id | 선택한 테마 |
| status | 결과 (success/fail) |
| result_rank | 최종 순위 |

---

## 📫 Contact

- Email: woojin1900@gmail.com
