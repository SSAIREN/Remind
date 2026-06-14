# SSAIREN 코드 분석 & 회고

> SSAIREN 조직의 4개 코드 레포(AI · Backend · Frontend · DashBoard)를 전수 분석한 결과입니다.
> 해커톤(약 1주) 결과물 기준으로 **잘한 점 / 부족한 점 / 다음에 하면 좋은 것 / 이렇게 하면 안 되는 것**을 정리했습니다.

---

## 1. 레포별 요약

| 레포 | 스택 | 규모 | 테스트 | 한줄 평 |
| --- | --- | --- | --- | --- |
| **AI** | FastAPI · LangGraph · LangChain | ~1,900 LOC | 0 | LangGraph 그래프 + 3단계 방어로직이 핵심. 설계는 좋으나 테스트 0 |
| **Backend** | Spring Boot · JPA · WebSocket · FCM | ~6,600 LOC | ~1,100 LOC (13개) | 도메인 주도 패키징 + 실제 단위테스트까지. 4개 중 가장 단단함 |
| **Frontend** | Flutter · Dio · Whisper · FCM | ~8,100 LOC | 2개 | feature-first 구조, mock 토글 잘 됨. STT 위치가 설계와 다름 |
| **DashBoard** | Vue 3 · Pinia · Vite | ~2,800 LOC | 0 | 가볍고 깔끔. 인증이 클라이언트 목업 |

### 전체 아키텍처 (실제 코드 기준)

```
Flutter ──REST(평시) / WebSocket(위험 시)──▶ Spring ──REST──▶ FastAPI(LangGraph)
  │ Whisper STT(서버 API 호출)               │ 게이트키퍼/1차 룰     │ situation_detector
  ▼                                          ├─ PostgreSQL          │ → scenario_analyzer
FCM 푸시                                      └─ Dashboard WS 브로드캐스트   → tool_executor → 콜백 5종
```

- Spring이 **게이트키퍼**, FastAPI가 **AI 추론·도구 실행**, 둘 사이는 내부 REST. 역할 분리는 코드로도 일관되게 지켜짐.
- AI의 tool 5종(`check_family_gps`, `send_family_sms_alert`, `notify_police`, `save_evidence`, `show_warning_banner`)이 Spring의 `/ai/use/**` 콜백과 1:1 매핑됨.

---

## 2. 잘한 점 👍

**아키텍처 / 설계**
- **역할 분리가 코드까지 일관됨.** Spring=클라이언트 facing + 1차 룰, FastAPI=AI 추론. 발표 자료의 2-layer 구조가 실제 코드와 일치한다.
- **AI의 3단계 방어 로직.** `nodes.py`의 `_validate_and_sanitize_result` → `_run_rule_based_fallback` → LLM 실패 시 룰베이스로 graceful degradation. LLM이 깨진 JSON을 뱉어도 (`invoke_json`에서 코드블록·trailing comma 제거 + 정규식 추출) 서비스가 죽지 않는다. **해커톤에서 데모 안정성을 코드로 확보한 좋은 판단.**
- **레이턴시 최적화 결정.** `scenario_analyzer`를 별도 LLM 호출이 아닌 passthrough(가중 평균만 계산)로 합쳐 LLM 호출을 1회로 줄임. 실시간 통화 분석에 맞는 트레이드오프.
- **`execute_tools` 드라이런 가드.** `can_call_external()`로 키·URL 없으면 자동 dry-run. 외부 부수효과(경찰 신고 등)를 안전하게 토글.

**Backend 품질**
- **도메인 주도 패키징.** `callsession`, `casefile`, `guardianreply`, `notification`, `pairing` 등 도메인별로 controller/service/repository/dto/entity가 깔끔히 분리됨.
- **실제 단위테스트 ~1,100줄.** 서비스·컨트롤러·WebSocket 핸들러까지 커버. 해커톤 결과물에서 보기 드문 수준.
- **추상화 게이트웨이 패턴.** `FcmPushGateway`의 구현을 `FirebaseFcmPushGateway`/`LoggingFcmPushGateway`로 분리 → FCM 미설정 환경에서도 로그로 대체 동작. `TranscriptAnalysisGateway`도 mock/FastAPI 교체 가능.
- **전역 예외 처리 + 표준 에러 응답**(`GlobalExceptionHandler`, `ErrorCode`, `ErrorResponse`).

**Frontend / DashBoard**
- **mock API 토글**(`USE_MOCK_API`)로 백엔드 없이도 UI 개발/시연 가능. 병렬 개발에 유리했던 선택.
- Flutter **feature-first** 구조(`features/<도메인>/presentation·widgets`)와 `core/theme`·`core/widgets` 디자인 시스템 분리가 일관적.
- DashBoard는 Pinia store로 websocket/auth/상태를 분리, http 래퍼가 단순명료.

**협업 인프라**
- 4개 레포 모두 `.env`를 `.gitignore` 처리 → **레포에 라이브 시크릿 커밋 없음.**
- GitHub Actions CI/CD(GHCR 빌드 → EC2 SSH 배포)가 AI·Backend에 구축됨.
- Backend `docs/`에 Flutter 연동/WebSocket 가이드 문서화.

---

## 3. 부족한 점 👎

**보안 (해커톤 범위 감안하되, 그대로 운영하면 위험)**
- **인증이 사실상 없음.** Spring에 Spring Security 의존성 자체가 없고, DashBoard 로그인은 `admin/1234` 하드코딩 + `localStorage` 클라이언트 검증. 누구나 API 직접 호출 가능.
- **내부 콜백 미검증.** AI는 `X-Internal-Key`를 보내지만 Backend `/ai/use/**` 콜백 컨트롤러는 키를 **검증하지 않는다.** 외부에서 경찰 신고·가족 알림 트리거 가능.
- **FastAPI CORS가 `allow_origins=["*"]` + `allow_credentials=True`** 조합 — 사양상 무효이자 위험한 설정.
- **Whisper API 키가 클라이언트 번들에 포함.** `whisper_stt_service.dart`가 dotenv의 OPENAI 키로 GMS Whisper를 **앱에서 직접 호출** → 앱 디컴파일 시 키 노출. (카톡에서도 키가 평문 공유됨 → 즉시 rotate 권장.)

**테스트 / 검증**
- AI·DashBoard 테스트 **0개.** 서비스의 핵심인 AI 위험도 판정 로직(`_run_rule_based_fallback`, 점수 구간)에 회귀 테스트가 없어 프롬프트/가중치 바꾸면 무엇이 깨지는지 알 수 없다.
- Backend CI에 **테스트 게이트가 없을 가능성**(deploy 워크플로만 존재). 테스트는 있는데 PR에서 안 돌면 절반의 가치.

**일관성 / 정직성**
- **"온디바이스 STT" 서술과 실제 구현 불일치.** 발표/회고는 온디바이스 Whisper를 말하지만 코드는 GMS Whisper **API 호출**(네트워크)이다. 데모로는 충분하나 발표에서 과장하면 Q&A에서 약점.
- 발표상 "situation_detector → scenario_analyzer 2단계 LLM 분석"이지만 실제론 1회 LLM + passthrough. (엔지니어링상 좋은 결정이나, 설명과 코드가 다르면 심사 때 혼선)
- `ddl-auto: update` (운영 DB에 위험), 로깅 레벨 `DEBUG` 고정 등 개발 설정이 그대로.

**기타**
- `nodes.py`에 `_keyword_detector`/`_fallback_analysis`/`_run_rule_based_fallback`처럼 **거의 같은 룰베이스 로직이 3벌** 존재(중복). 유지보수 시 한 곳만 고치면 어긋남.
- DashBoard `http.js`에 에러 바디 파싱·재시도·인증 헤더 주입 지점이 없음(403 디버깅을 카톡에서 수동으로 했던 흔적).

---

## 4. 다음에 하면 좋은 것들 ✅

1. **최소 인증부터 깔고 시작.** 내부 호출엔 공유 시크릿 헤더 검증(필터 1개), 외부엔 토큰 1개라도. "나중에" 붙이면 거의 안 붙는다.
2. **시크릿은 처음부터 환경변수 + 시크릿 매니저.** 키를 채팅/이슈/코드에 붙이지 않는 규칙을 1일차에 합의. 클라이언트가 외부 AI API를 직접 부르지 말고 **백엔드 프록시** 경유(키를 서버에만 둠).
3. **핵심 판정 로직에 golden test.** "이 대화 → 위험도 HIGH, tool=notify_police" 같은 입출력 케이스 10~20개를 AI 레포에 고정. 프롬프트 튜닝의 안전망.
4. **CI에 test 게이트.** push/PR에서 `./gradlew test`, `pytest`, `flutter test`가 돌고 실패 시 머지 차단.
5. **룰베이스 로직 1벌로 통합.** 키워드·점수 산정을 단일 함수/모듈로 추출해 LLM·fallback이 같은 소스를 공유.
6. **환경 분리.** `application-{local,prod}.yaml`, `ddl-auto`는 prod에서 `validate` + Flyway 마이그레이션, 로깅은 prod `INFO`.
7. **계약 문서를 단일 소스로.** 이미 있는 `docs/FLUTTER_BACKEND_API_SPEC.md`를 OpenAPI 스키마와 자동 동기화(또는 codegen)해 카톡에서 enum 물어보는 일을 없앤다.

---

## 5. 이렇게 하면 안 되는 것들 ⛔

- ❌ **API 키·FCM 토큰·DB 비밀번호를 단톡방/평문으로 공유.** (이번에 OPENAI·GMS·LangChain 키, DB 비번, FCM 토큰이 그대로 노출됨 → **전부 rotate 필요.** 카톡 export 파일도 레포에 커밋 금지.)
- ❌ **클라이언트 앱에 외부 유료 API 키를 박아 배포.** 디컴파일 한 번이면 끝. 비용 폭탄·키 탈취 경로.
- ❌ **인증 없는 부수효과 엔드포인트.** "경찰 신고/가족 알림"처럼 실제 행동을 일으키는 API를 무인증으로 열어두기.
- ❌ **`allow_origins=["*"]` + `allow_credentials=True`** 동시 사용.
- ❌ **운영에서 `ddl-auto: update`.** 컬럼 하나 잘못 건드리면 데이터 손상. 마이그레이션 도구로.
- ❌ **발표 서술과 코드의 불일치를 방치.** "온디바이스/2단계 LLM"처럼 실제와 다른 설명은 심사 Q&A에서 신뢰를 깎는다. 코드가 실제로 하는 것을 그대로 말하는 게 더 강하다.
- ❌ **데모 안정성을 위한 더미/fallback을 끝까지 안 걷어내기.** 어디까지가 진짜 동작이고 어디가 mock인지 README에 명시하지 않으면, 다음 사람이 "되는 줄 알고" 그 위에 쌓는다.

---

## 6. 총평

해커톤 1주 결과물로서 **설계 일관성과 안정장치(3단계 방어, 게이트웨이 추상화, mock 토글, 드라이런)** 측면은 평균을 크게 웃돈다. 특히 Backend는 도메인 구조와 테스트까지 갖춰 그대로 확장 가능한 토대다. 반면 **보안(인증·시크릿 관리)과 검증(AI 테스트·CI 게이트)** 은 시간에 밀린 전형적 부채로 남았다. 다음 사이클에서 *인증·시크릿·핵심 로직 테스트* 세 가지만 1일차에 깔고 가면, 같은 팀 역량으로 "데모용"이 아니라 "운영 가능한" 결과물이 나온다.
