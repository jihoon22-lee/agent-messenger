# 내부 프로토콜과 텍스트 UX

**설계 초안이다. 사내 메신저 API나 OpenCode API의 실제 형식이 아니다.** 사내 계약은 각 어댑터가 이 내부 형식으로 변환한다. 필드·명령은 M1에서 스키마와 테스트로 고정한다.

## 1. 연결 계약

Runner가 Gateway에 outbound 연결한다. 운영에서는 TLS, 인증서 검증, 별도 Runner 자격증명, 허용 Runner 매핑이 필요하다. 인증을 완료하기 전에는 작업·결과 프레임을 처리하지 않는다. 토큰을 URL query나 로그에 기록하지 않는다.

연결 시 `protocolVersion`, `runnerId`, 실행 가능 상태, 지원 기능을 교환한다. Gateway가 connection epoch를 부여하고 이전 세대에서 온 제어 프레임을 거절한다. 지원하지 않는 버전·메시지는 명시적으로 거절한다. heartbeat·재연결 간격·크기 제한은 설정값이며 메신저 ACK 제한 시간과 혼동하지 않는다.

## 2. 공통 envelope 예시

모든 값은 합성 예제다. `jobId`는 실제 실행 ID, `id`는 전달 메시지 ID다. 재전달할 때 둘 다 유지하며 업무 payload와 주체·대화·작업 식별자는 변경하지 않는다. `connectionEpoch`는 전송 시점의 연결 메타데이터이므로 재연결 후 현재 세대로 바꾼다. 중복 검사는 이 연결 메타데이터를 제외한 불변 업무 내용으로 한다.

```json
{
  "protocolVersion": 1,
  "id": "wire-synthetic-001",
  "type": "job.submit",
  "runnerId": "runner-synthetic",
  "connectionEpoch": "epoch-synthetic",
  "jobId": "job-synthetic-001",
  "principalId": "principal-synthetic",
  "conversationId": "conversation-synthetic",
  "createdAt": "2026-10-15T00:00:00Z",
  "payload": {
    "bindingId": "workspace-synthetic",
    "text": "합성 테스트 파일의 제목을 알려줘."
  }
}
```

주체·대화 식별자는 메신저 인증 결과와 사전 매핑에서 생성한다. 사용자 메시지에 적힌 `principalId`, `runnerId`, 경로를 신뢰하지 않는다. 실제 대화방 ID와 발신 대상은 Gateway가 로컬에서 해결한다. Runner도 허용된 binding만 받아들인다.

## 3. 메시지 종류

| 방향 | 종류 | 용도 |
| --- | --- | --- |
| 양방향 | `hello`, `heartbeat`, `ack`, `error` | 버전·연결 상태·영속 저장 확인 |
| Gateway → Runner | `job.submit` | 일반 요청을 기존 jobId로 전달 |
| Gateway → Runner | `job.status.request` | 실행 여부·결과 재확인 |
| Gateway → Runner | `job.abort` | 실행 중단 요청 |
| Gateway → Runner | `interaction.reply`, `interaction.reject` | 승인·질문에 대한 검증된 사용자 응답 |
| Gateway → Runner | `conversation.reset` | 실행 중이 아닐 때 새 세션 연결 요청 |
| Runner → Gateway | `job.accepted`, `job.state` | PC 수신 및 실행 상태 갱신 |
| Runner → Gateway | `interaction.request` | 질문·승인 내용과 요청 식별 정보 |
| Runner → Gateway | `job.result` | 현재 작업의 최종 결과·실패·취소 요약 |
| Runner → Gateway | `conversation.reset.completed` | 새 세션 연결 확인 |

ACK는 수신 처리와 저장이 끝난 업무 프레임의 `id`를 지정한다. ACK 자체와 heartbeat에는 다시 영속 ACK를 보내지 않는다. 입력 오류·인증 실패·stale·충돌·일시 오류를 구분한다. unknown 메시지를 범용 OpenCode API 호출이나 셸 명령으로 처리하지 않는다.

`job.state`에는 단조 증가 상태 revision을 두고 Gateway는 오래된 revision을 무시한다. 업무 텍스트와 진단 상태를 분리하며 OpenCode 내부 추론·원시 도구 이벤트 전체를 전송하지 않는다.

## 4. 중복·전달 보장

- Gateway 수신 키는 업체가 보장하는 `(bot, eventId)` 또는 문서화된 범위의 메시지·이벤트 조합을 사용한다. 안정적인 키가 없으면 M0에서 대안을 합의하며 본문 해시만으로 처리하지 않는다.
- Gateway→Runner 재전달은 동일 jobId를 사용한다. Runner는 이미 접수·실행·완료한 jobId에 대해 현재 상태를 반환하고 새 모델 요청을 만들지 않는다.
- OpenCode에 보낸 요청의 HTTP 결과가 유실되면 upstream 반영 여부를 조사한다. 조회 가능한 messageId가 지원되어도 재사용의 멱등성이 검증된 것은 별도다.
- Runner 결과는 안정적인 결과 eventId와 outbox로 전달한다. Gateway에 같은 결과가 반복 도착해도 새 답변을 반복 생성하지 않는다.
- 발신 분할 키는 `(resultEventId, chunkIndex)`로 관리한다. 이미 완료된 chunk는 재전송하지 않는다.
- 메신저가 발신 멱등 키나 발신 조회를 지원하지 않는 상황에서 timeout이 발생하면 `delivery_unknown`으로 둔다. 무제한 자동 재시도로 중복 메시지를 만들지 않는다.

이 설계는 at-least-once 전달에 대한 중복 억제를 목표로 한다. 임의 PC 도구 실행과 메신저 표시까지의 exactly-once를 약속하지 않는다.

## 5. 텍스트 명령 초안

명령은 Bridge 로컬 규약이며 메신저 자체의 슬래시 기능을 요구하지 않는다. 일반 문장 안의 단어를 명령으로 오인하지 않도록 전체 메시지 문법을 검사한다.

| 입력 | 동작 |
| --- | --- |
| `도움말` | 가능한 명령을 표시한다. 모델 호출 없음. |
| `상태` 또는 `상태 J12` | 현재 또는 지정 작업 상태를 조회한다. |
| `중단` 또는 `중단 J12` | 유일한 현재 작업 또는 지정 작업의 중단을 요청한다. |
| `새대화` | 실행·대기 요청과 충돌 여부를 확인하고 새 세션을 연결한다. |
| `승인 A17` / `거절 A17` | 특정 권한 요청에 일회 승인 또는 거절한다. |
| `답변 Q28 2` | 특정 선택 질문에 답한다. |
| `답변 Q29 자유 입력 내용` | 특정 자유 입력 질문에 답한다. |
| `질문취소 Q28` | 질문을 거절하는 OpenCode 경로로 처리한다. |
| `재개 J12` | 오프라인 보류 요청을 확인하고 대기열로 되돌린다. 불명확한 실행을 무조건 재실행하는 명령이 아니다. |

`J12`, `A17`, `Q28`은 가독성용 별칭이다. 내부의 예측 불가능한 ID와 주체·대화·세션·jobId·generation에 연결한다. 별칭만 알고 있다고 승인 권한이 생기지 않는다. 과거 별칭을 새 요청에 재사용하지 않는다.

`중단 J12`의 대상이 아직 실행되지 않은 대기·오프라인 보류 요청이면 Gateway에서 취소하고 OpenCode에는 중단 요청을 보내지 않는다. 이미 Runner에 전달됐을 가능성이 있으면 Runner 상태를 먼저 확인해 실행 중단 경로와 구분한다.

대기 작업이 있는 상태의 `새대화`는 자동으로 그 작업들을 새 세션에 옮기지 않는다. 초기에는 대기 작업을 처리·취소한 후 다시 요청하도록 안내하는 단순 정책을 권장한다.

## 6. 승인 처리

```text
[승인 A17]
대상 작업: J12
요청: 합성 결과 파일 새로 생성
대상: 허용된 작업 폴더의 result.md
허용하려면 '승인 A17', 거절하려면 '거절 A17'을 보내주세요.
```

Runner는 OpenCode의 실제 요청 ID, 도구·대상·범위, sessionId, jobId, 사용자·대화, 실행 generation, 만료 시각, 처리 여부를 보관한다. 표시용 설명은 실제 요청과 일치해야 하며 모호하게 축약해 위험 범위를 감추지 않는다.

응답 처리 직전에 pending 상태를 재확인한다. 승인 대상·내용이 바뀌었거나 새 세션으로 바뀌었거나 만료됐으면 거절한다. 상태 변경과 중복 응답 방지는 원자적으로 처리한다. 초기에는 `always`·전체 자동승인을 제공하지 않는다.

승인 API timeout 후에는 pending/실행 상태를 확인한다. 성공 여부가 불명확한 동일 승인을 맹목 재전송하지 않는다. 이미 처리된 404를 새 요청 승인으로 바꾸지 않는다.

권한 UI가 파일 생성 수준으로 항상 나온다고 가정하지 않는다. 실제 OpenCode가 제공하는 도구·패턴·대상 정보로 표현하며 지원하지 않는 안전성은 약속하지 않는다.

## 7. 질문 처리

질문이 정확히 하나이고 단일 선택형이면 숫자만 답하는 편의를 제공할 수 있다. 권한 승인은 이 편의를 적용하지 않는다. 여러 질문·여러 대기 작업이 있으면 Q ID를 요구한다.

복수 선택, 여러 질문, 직접 입력을 어댑터가 구분한다. 필요한 경우 Q28-1, Q28-2처럼 순차 안내하고 모든 필수 답이 준비된 뒤 대상 버전의 배열 형식으로 한 번 제출한다. 일부 질문만 조용히 누락시키거나 첫 답을 전체 답으로 사용하지 않는다. 미지원 형식은 명시 안내 후 거절·중단 경로를 제공한다.

질문 대기 중에 들어온 일반 문장을 모두 답으로 삼지 않는다. `답변 Q…` 문법 또는 유일한 대기 질문의 명시적인 응답 문맥을 사용한다. 새 업무 요청은 일반 대기열에 둔다.

## 8. 결과·상태 표현

평소에는 한국어 자연어로 답하고, 내부 ID는 질문·승인·오류·여러 대기 작업 구별에 필요할 때만 노출한다. 토큰마다 메시지를 발송하지 않는다. 접수, 의미 있는 장기 작업 진행, 사용자 입력 요청, 최종 결과를 중심으로 전송한다.

메신저 크기 제한의 단위가 바이트인지 문자 수인지 확인한다. 한글·이모지·코드 블록을 깨뜨리지 않고 문단 중심으로 나눈다. 지원되는 Markdown만 사용하며 멘션·HTML·제어 문자열을 안전하게 처리한다. 기본으로 reasoning, 환경변수, 인증정보, 도구 원시 로그를 전달하지 않는다.

완료 답변을 꾸미기 위해 별도 외부 LLM 호출을 추가하지 않는다. 전용 에이전트의 응답 지침과 결정적인 포맷팅을 사용한다. 민감한 결과는 로컬에서 열도록 안내하되 초기 파일 첨부 기능을 암묵적으로 만들지 않는다.

## 9. 제안 오류 분류

`unauthorized`, `unsupported_event`, `invalid_command`, `runner_offline`, `queue_full`, `expired`, `stale_interaction`, `opencode_unavailable`, `execution_unknown`, `delivery_unknown`, `policy_denied`를 분리한다. 사용자에게는 안전한 한국어 안내와 조치만 제공하고 내부 응답 원문·토큰을 노출하지 않는다.
