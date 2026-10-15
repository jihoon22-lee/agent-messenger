# OpenCode 1.18.33 호환성 계획

## 1. 확인 수준

**1.18.33은 사용자가 사내 PC의 설치 버전으로 확인한 값이다. 이 저장소에서 실제 1.18.33 서버·사내 모델을 호출한 결과는 아직 없다.** 아래 API는 검증 후보이지 모두 동작한다고 보증하는 계약이 아니다.

2026-10-15에 agent-browser의 호환성 문서와 HTTP/SSE 클라이언트를 참고했다. 참고 저장소의 기준은 1.18.29이며, 공개 OpenCode 문서의 최신 설명과도 일부 표현·경로·기본값이 다르므로 세 가지를 구분해야 한다.

검증 우선순위는 **사내 설치본의 실제 응답·동작 → 해당 설치본 명세·소스 → 다른 버전의 참고 구현·공개 문서**다. 명세에 있어도 실제 모델·도구 동작까지 검증한 것은 아니다.

## 2. 출처

- [OpenCode 공식 서버 문서](https://opencode.ai/docs/server/): HTTP 서버·OpenAPI·세션·SSE의 개념 참고. 2026-10-15 확인.
- [OpenCode 공식 권한 문서](https://opencode.ai/docs/permissions/): 허용·질문·거절 설정의 개념 참고. 대상 버전의 실제 설정 구문은 별도 검증.
- [agent-browser 호환성 기록](https://github.com/jihoon22-lee/agent-browser/blob/b8169859246a882eddcfcf958a11adc8aab6fcfc/docs/opencode-v1-compatibility.md): 1.18.29 실측·합성 테스트·미검증 범위를 구분한 참고 자료.
- [agent-browser HTTP/SSE 클라이언트](https://github.com/jihoon22-lee/agent-browser/blob/b8169859246a882eddcfcf958a11adc8aab6fcfc/extension/src/agents/opencode-v1/client.ts): 재사용 검토 대상. 참조 커밋 `b8169859246a882eddcfcf958a11adc8aab6fcfc`.

참고 코드의 API·파서 구조를 읽었지만 서버용 추출·빌드·1.18.33 동작 검증은 수행하지 않았다. 다른 저장소의 검증 결과를 agent-messenger의 통과 결과로 옮기지 않는다.

## 3. 사내 계약 확인표

모든 행의 초기 상태는 **미검증**이다.

| 기능 | 후보 경로·항목 | 반드시 확인할 것 |
| --- | --- | --- |
| 버전·기동 | `opencode --version`, `opencode serve --help` | 1.18.33, 실제 실행 위치·인자·포트·작업 디렉터리 |
| 건강 상태 | `GET /global/health` | healthy/version, 미인증 요청 처리 |
| 명세 | `GET /doc` | 설치본 스키마·응답 content-type·실제 경로 |
| 인증 | Basic 또는 설치본이 제공하는 서버 인증 | 401/200, SSE 인증, secret 저장 방식 |
| 세션 생성·조회 | `POST /session`, `GET /session/{id}` | 세션 ID, directory/workspace 전달, 전용 세션 식별 |
| 일반 요청 | `POST /session/{id}/prompt_async` | body, agent/model, 응답 코드, messageId 지원·상관관계 |
| 메시지 조회 | `GET /session/{id}/message`, `GET /session/{id}/message/{mid}` | user/assistant, parent 관계, 최종 응답·완료·오류 필드 |
| 상태 | `GET /session/status` | idle/busy/retry 등 실제 의미와 현재 요청의 관계 |
| 이벤트 | `GET /event` | 인증 SSE, 유형·필드, 세션 필터, chunk·완료·오류·질문·권한 |
| 중단 | `POST /session/{id}/abort` | 접수·실제 종료의 차이, pending 처리, 이미 실행된 부작용 |
| 질문 | `GET /question`, `POST /question/{rid}/reply`, `POST /question/{rid}/reject` | 단일·복수·자유 입력·취소, stale, 응답 배열의 구조 |
| 권한 | `GET /permission`, `POST /permission/{rid}/reply` | once/reject, 실제 요청 범위, stale·중복·응답 유실 |
| 모델·에이전트 | `GET /provider`, `GET /agent` | 사내 모델 연결, 지정 에이전트의 실제 로드·설정 격리 |

참고 1.18.29에는 질문 `answers: string[][]`, 권한 `reply: once|always|reject` 형태가 기록되어 있다. **1.18.33에서도 동일하다고 가정하지 않는다.** MVP 권한 UX는 일회 승인·거절만 제공한다. 공개 문서의 구형 permission 경로로 조용히 fallback하지 않는다.

현재 단계에서 요청 JSON이나 설치 명령을 제품의 확정 사용법으로 제공하지 않는다. 확인한 계약을 바탕으로 최소 타입·스키마와 오류 처리를 작성한다.

## 4. 전용 연결·실행 환경

기존 사내 모델 설정을 읽는 명시적인 `serve` 주소에 연결한다. 새 `serve`가 기존 CLI 서버와 동일한 프로세스라고 가정하지 않는다. 전용 세션을 사용하되 기존 전역 설정·provider·plugin·스킬을 임의 수정하지 않는다.

Runner와 OpenCode가 모두 WSL이면 같은 WSL 안에서 통신한다. Windows 네이티브 실행이면 해당 네이티브 환경의 경로를 쓴다. Windows 경로를 Linux 경로처럼 전달하거나 요청마다 `wsl.exe`를 실행하지 않는다. 실제 배포 대상부터 검증하고 양쪽 동시 지원을 첫 실연동의 선행 조건으로 만들지 않는다.

작업 디렉터리, 메신저 전용 agent, provider/model은 Runner 로컬 설정으로 고정한다. 사용자 텍스트를 OpenCode 설정 경로·CLI 인자·셸 명령으로 직접 이어 붙이지 않는다.

## 5. 이벤트 해석·완료 판정

현재 작업의 sessionId와 사용자 요청 식별자를 저장하고, 응답의 parent/message 관계를 추적한다. 단순히 “마지막 assistant 메시지”를 결과로 택하지 않는다. 이전 요청이나 다른 세션의 메시지가 섞이지 않는지 테스트한다.

SSE는 힌트와 실시간 표시 수단이며 최종 상태 확인을 대신하지 않는다. 재연결하면 세션·현재 요청의 메시지·질문·권한을 조회한다. 이벤트 replay·Last-Event-ID·중복 이벤트 ID의 안정성은 검증 전 의존하지 않는다.

완료 판정은 대상 요청과 연결된 응답의 완료·오류 정보, 세션 상태, 대기 interaction, 필요한 도구·하위 실행 상태를 종합한다. 정확한 predicate는 1.18.33 실측으로 고정한다. HTTP 204, text delta, session idle 하나만으로 성공 처리하지 않는다. 출력이 없는 실패도 사용자에게 종료 상태를 알려야 한다.

원시 reasoning·부분 도구 로그는 메신저 답변으로 내보내지 않는다. 전역 이벤트를 사용해야 한다면 등록된 세션 allowlist를 먼저 적용하고, 관련 없는 데이터는 저장도 하지 않는다.

## 6. 변경 요청의 재시도

GET 조회는 제한된 backoff로 재시도할 수 있다. prompt, 세션 생성, 승인, abort 등 변경 요청은 자동 재시도 규칙을 별도 검증한다. POST 실패라는 이유만으로 같은 작업을 다시 제출하지 않는다.

upstream이 요청 식별자를 허용하는 것과 중복 요청을 안전하게 무시하는 것은 다르다. 요청 저장 후 timeout·프로세스 종료를 주입해 확인한다. 반영 여부를 결정할 수 없으면 실행을 보류하고 `execution_unknown`으로 보고한다.

## 7. 재사용 범위

HTTP 요청, 인증 헤더, SSE 파서, 오류 타입, 작업 디렉터리 전달을 우선 검토한다. Node.js에서 사용하는 `fetch`, 스트림, AbortSignal 등 실제 지원 여부를 사내 런타임으로 확인한다. 필요 기능만 추출하고 브라우저 파일 탐색·UI·토큰 대시보드·설치 관리까지 가져오지 않는다.

참고 코드의 타입 단언을 런타임 검증으로 착각하지 않는다. 외부 응답은 필요한 필드를 검사한다. 파서의 크기 제한·UTF-8 분할·CRLF·불완전 이벤트·중단·재연결도 독립 테스트한다.

## 8. 증거 저장

사내 증거에는 버전, 명세 hash, 테스트 항목, 결과, 검증 날짜, 실행 OS를 기록한다. 명세 원문·로그·모델 이름·경로·인증값은 승인된 사내 저장소에 둔다. 외부에는 정책상 허용된 비식별 PASS/FAIL/미검증 요약만 남긴다.
