# 아키텍처

이 문서의 프로세스·상태·저장 구조는 권장 설계이며 아직 구현되지 않았다. 확정 요구는 [요구사항](requirements.md)을 따른다.

## 1. 배치와 신뢰 경계

```text
[사용자 / 사내 메신저]
         │ 인증된 수신 이벤트
         ▼
[Messenger Gateway] ───── 발신 API ────> [원래의 사용자 대화방]
         ▲
         │ Runner가 시작한 인증 연결 (개발: loopback / 운영: TLS)
         ▼
[Local Runner / 사내 PC]
         │ loopback HTTP API + SSE, OpenCode 인증
         ▼
[OpenCode 1.18.33 / 사내 PC] ───> [사내 모델·허용된 파일·도구]
```

개발 중 Gateway와 Runner를 같은 PC에서 실행한다. VM 이전 후에도 Runner와 OpenCode의 위치는 바뀌지 않는다. 브라우저 제어를 추가할 때는 OpenCode가 기존 agent-browser 도구를 사용하도록 별도 검증하며, Gateway가 Chrome에 직접 접근하지 않는다.

사내 메신저가 WSL 내부의 주소를 바로 호출할 수 있다고 가정하지 않는다. 필요한 경우 승인된 Windows 수신 계층이나 네트워크 설정을 별도로 준비한다. 이는 초기 API·네트워크 확인 항목이다.

## 2. 컴포넌트 책임

| 컴포넌트 | 책임 | 금지할 책임 |
| --- | --- | --- |
| MessengerAdapter | 업체별 인증·수신 정규화·발신·오류 해석 | OpenCode API 호출·작업 실행 |
| Gateway | 허용 사용자 검증, inbox·전달 대기열·outbox, Runner 연결 | PC 파일 접근·LLM 호출·임의 API 프록시 |
| Runner | 세션 매핑, 실행 장부, 질문·승인·중단, 결과 outbox | 메신저 토큰 관리·임의 수신자 선택 |
| OpenCodeAdapter | 1.18.33 HTTP/SSE 해석, 상태 조회·오류 정규화 | 메신저 업체별 필드 처리 |
| OpenCode | 실제 추론·도구 실행 | 메신저 전달 성공 여부 결정 |

운영 정책·명령 파서·프로토콜 타입은 얇게 공통화한다. 일반 플러그인 프레임워크나 추가 작업 브로커는 만들지 않는다.

## 3. 제안 디렉터리 구조

아래는 구현 단계의 목표이며 현재 실제 코드 경로가 아니다.

```text
apps/gateway/                  메신저 엔드포인트·발신·Runner 연결
apps/runner/                   PC 실행 제어·전용 세션·복구
packages/protocol/             메시지 스키마·버전·검증
packages/messenger-adapter/    합성 및 사내 어댑터 경계
packages/opencode-v1/          대상 설치본 API 어댑터
packages/core/                 최소 공통 상태·명령·정책
tests/                        단위·합성 통합·사내 전용 테스트 분리
 docs/                        설계·운영·검증 문서
```

처음에는 실제 사용하는 모듈만 만든다. `packages/core`가 필요 없으면 생략할 수 있다. 개발 실행기는 두 프로세스를 함께 시작할 수 있으나 저장 경로나 메모리를 공유하는 것을 연결 계약으로 삼지 않는다.

## 4. 데이터 소유권

| 데이터 | 권위 있는 소유자 | 설명 |
| --- | --- | --- |
| 수신 이벤트·중복 키 | Gateway | 인증된 입력과 수신 순서를 영속 저장한다. |
| 아직 실행되지 않은 일반 요청 대기열 | Gateway | 한 Runner에 한 작업을 배정한다. 재전달은 같은 jobId를 사용한다. |
| OpenCode 세션·진행·승인·결과 | Runner | PC 실행 상태의 원본이다. Gateway는 표시용 상태만 복제한다. |
| 실제 대화와 도구 실행 이력 | OpenCode | Runner는 필요한 ID·상태·결과만 저장한다. |
| Runner→Gateway 결과 미전달 | Runner outbox | Gateway의 영속 저장 ACK 전까지 유지한다. |
| 메신저 답변 전송 상태 | Gateway outbox | 실행 성공과 발신 성공을 분리한다. |
| PC 경로·OpenCode 비밀값 | Runner 로컬 설정 | Gateway로 전송하지 않는다. |
| 메신저 인증·허용 사용자·대화 매핑 | Gateway 로컬 설정 | Runner에는 검증된 불투명 주체 ID만 전달한다. |

SQLite 파일은 각 프로세스가 자신의 로컬 디스크에 둔다. VM과 PC가 하나의 DB 파일을 네트워크 공유하지 않는다. Gateway가 Runner 상태를 재구성할 때 새 작업을 임의 생성하지 않는다.

초기 테이블 후보는 Gateway의 `inbox`, `queued_jobs`, `runner_events`, `outbox`와 Runner의 `conversation_bindings`, `executions`, `pending_interactions`, `outbox`다. 실제 열과 마이그레이션은 M1에서 테스트와 함께 확정한다.

## 5. 정상 요청 흐름

1. Gateway가 수신 원문에서 업체 계약에 따른 서명·인증을 검증한다.
2. 허용 사용자, 등록된 1:1 대화, 봇 자신이 아닌 입력, 지원 이벤트인지 확인한다.
3. inbox 중복 키와 요청 기록을 트랜잭션으로 저장하고 업체가 요구하는 ACK를 반환한다. 작업 완료까지 HTTP 요청을 유지하지 않는다.
4. 로컬 제어 명령은 우선 처리하고, 일반 요청은 대기열에 넣는다. 온라인·실행 가능 상태인 Runner에 안정적인 jobId로 전달한다.
5. Runner는 중복 장부와 세션 매핑을 확인하고 요청 전송 직전 상태를 영속 저장한다.
6. OpenCode에 요청하고 SSE·조회로 현재 작업에 해당하는 질문·권한·진행·종료를 확인한다.
7. Runner가 결과를 outbox에 기록하고 Gateway로 보낸다. Gateway는 저장 후 ACK한다.
8. Gateway가 정해진 원래 대화방에 답변을 보내고 전달 상태를 기록한다.

HTTP 접수 ACK, Runner 수신 ACK, Gateway 결과 저장 ACK, 메신저 발신 완료는 각각 다른 확인이다. 어느 한 단계 성공을 전체 성공으로 표시하지 않는다.

## 6. 작업·대화 정책

권장 매핑 키는 `(bot, principal, conversation, runner, workspaceBinding)`이다. 초기에는 등록된 작업 디렉터리 하나와 전용 세션 하나를 사용한다. Gateway가 자유 문자열 경로를 받아 Runner의 실행 디렉터리로 쓰지 않도록 한다.

OpenCode 서버를 다른 클라이언트와 공유할 수는 있지만 메신저 소유 세션만 조회·전달한다. 메신저 세션에 다른 클라이언트가 동시에 입력하면 실행 상관관계를 재확인하고 모호한 상태를 보류한다. CLI UI 동기화는 구현하지 않는다.

일반 작업은 직렬화한다. 대기열은 크기·유효기간 상한을 두며 초과·만료를 안내한다. 승인·질문·중단·상태는 현재 작업의 일반 대기열 뒤에 넣지 않는다. `새대화`는 실행 중일 때 조용히 세션을 바꾸지 않고 중단 완료 후 다시 요청하도록 안내한다.

## 7. 상태와 불확실성

전달 상태와 실행 상태는 분리한다.

```text
Gateway 요청: queued → dispatched → runner_accepted
                         └─ 재전달: 같은 jobId, 새 실행 생성 금지
오프라인/만료: held_offline / expired

Runner 실행: received → submitting → running
                                   ↔ waiting_question / waiting_permission
                                   → stopping → cancelled
                                   → succeeded / failed
상태 불명: execution_unknown (조회·운영자 확인 전 재실행 금지)

발신: pending → sending → delivered
                    └→ retryable / delivery_unknown / failed
```

`submitting` 기록 후 OpenCode가 요청을 받았는지 알 수 없게 된 경우는 `execution_unknown`이다. upstream이 idempotency나 조회 가능한 안정적인 요청 식별자를 실제로 보장하지 않는다면 end-to-end exactly-once를 주장하지 않는다.

중단 API 호출 성공은 부작용 롤백을 뜻하지 않는다. 실행 종료 확인 후 취소 상태를 기록하되 이미 변경된 파일·외부 동작을 원상복구했다고 말하지 않는다.

## 8. 연결 단절·동시 실행 방지

Runner는 동일 데이터 디렉터리에 대한 단일 인스턴스 잠금을 사용한다. Gateway는 등록된 Runner의 활성 연결 세대를 관리하고 오래된 연결의 메시지를 거절한다. 실행 장부·jobId 중복 검사를 함께 사용한다. 연결 세대나 lease만으로 이미 OpenCode에 전달된 실행이 취소된다고 가정하지 않는다.

연결이 끊겨도 이미 승인된 작업은 PC에서 계속될 수 있다. Gateway는 `연결 끊김·완료 여부 확인 중`으로 표시한다. 새 승인을 추측하거나 자동 부여하지 않고, 정책상 대기 유효기간이 끝나면 거절·중단 경로로 처리한다. 재연결 후 상태와 outbox를 조정한 뒤 새 작업을 배정한다.

SSE가 끊어지면 세션·메시지·pending 목록을 다시 조회한다. 서버 이벤트 ID와 재생 지원을 검증하기 전에는 Last-Event-ID에 복구를 의존하지 않는다.

## 9. 작은 구현을 유지하는 기준

Redis/Kafka, 서비스 검색, 관리 웹 UI, 클라우드 모델, 다중 Runner 스케줄러는 필요하지 않다. SQLite 장부와 두 컴포넌트의 제한된 계약으로 시작한다. 단, 인증·중복·승인·오류 상태를 제거해서 작게 만들지는 않는다.
