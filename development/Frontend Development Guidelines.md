# Frontend Development Guidelines

## 1. 개발 목표

Frontend의 최우선 목표는 사용자가 별도의 앱 설치나 회원가입 없이 모바일 브라우저에서 링크만으로 게임에 참여하고, 카메라 촬영부터 실시간 게임 결과 확인까지 끊김 없이 진행할 수 있도록 하는 것이다.

Frontend는 게임 규칙의 최종 판단자가 아니다.

시간, 제출 성공 여부, 점수, 순위, 방장 권한, 라운드 상태 등 게임 결과에 영향을 주는 모든 정보는 Backend를 Single Source of Truth로 사용한다.

Frontend가 책임지는 주요 영역은 다음과 같다.

- 모바일 웹 UI
- 카메라 권한 및 카메라 스트림 제어
- 셀카 촬영
- 업로드 전 이미지 리사이즈 및 압축
- Backend API 호출
- 실시간 이벤트 수신 및 화면 상태 반영
- 네트워크 단절 후 재접속 UX
- 게임 상태에 따른 사용자 인터랙션 제어

---

# 2. 기본 기술 스택

## Language

- TypeScript

## Framework

- React
- Vite

## Routing

- React Router

## State

- Zustand
- TanStack Query

## Realtime

- Socket.IO Client

## Styling

- Tailwind CSS

## Animation

- Motion

## Browser APIs

- MediaDevices API
- Canvas API
- Blob / Object URL API

---

# 3. 기본 개발 원칙

## 3.1 서버 상태를 신뢰한다

다음 값은 Frontend가 직접 결정하지 않는다.

- 라운드 시작 시점
- 라운드 종료 시점
- 제출 마감 여부
- 제출 성공 여부
- 감정 점수
- 라운드 순위
- 획득 포인트
- 좋아요 보너스
- 최종 순위
- 방장 여부
- 참여 가능 여부

Frontend에서 계산한 값은 화면 표시를 위한 보조 정보일 뿐이다.

예를 들어 타이머는 서버에서 받은 절대 마감 시각을 기준으로 표현한다.

```text
deadlineAt
    ↓
현재 브라우저 시간과 비교
    ↓
남은 시간 표시
```

다음과 같이 구현하지 않는다.

```text
화면 진입
↓
30초짜리 클라이언트 타이머 시작
```

---

# 4. 카메라 구현 지침

## 4.1 MediaDevices API를 사용한다

카메라 촬영은 브라우저의 다음 API를 기본으로 한다.

```text
navigator.mediaDevices.getUserMedia()
```

별도의 파일 업로드 기반 촬영 기능을 제공하지 않는다.

사진첩이나 파일 선택을 통한 게임 참여는 허용하지 않는다.

---

## 4.2 카메라 권한은 자동 요청하지 않는다

페이지 로드 직후 `getUserMedia()`를 실행하지 않는다.

사용자에게 먼저 다음 내용을 안내한다.

- 이 게임에는 카메라가 필요하다.
- 촬영된 사진은 감정 판별에 사용된다.
- 사진은 판별 이후 영구 보관되지 않는다.

사용자가 명시적으로 `카메라 켜기` 등의 버튼을 누른 뒤 처음 권한을 요청한다.

---

## 4.3 대기실에서 카메라를 계속 켜두지 않는다

카메라 권한 확인이 끝났다면 MediaStream track을 중지한다.

```ts
stream.getTracks().forEach(track => track.stop());
```

실제 촬영 화면에 들어갈 때 다시 MediaStream을 생성한다.

목적은 다음과 같다.

- 배터리 사용 감소
- 발열 감소
- 사용자의 카메라 사용 불안감 감소

---

## 4.4 앱 전환 및 화면 잠금을 고려한다

모바일에서는 다음 상황이 일반적으로 발생할 수 있다.

- 전화 수신
- 앱 전환
- 브라우저 background 이동
- 화면 잠금
- 브라우저 복귀

화면 복귀 시 카메라 preview가 정상인지 확인하고 필요한 경우 자동 복구한다.

자동 복구가 실패하면:

```text
카메라 다시 켜기
```

버튼을 제공한다.

---

# 5. 이미지 처리 지침

## 5.1 서버에 원본 카메라 이미지를 전송하지 않는다

촬영 이미지는 Client에서 먼저 리사이즈 및 압축한다.

기준:

```text
최대 장변: 720px
Format: JPEG
Quality: 0.8
```

처리 과정:

```text
Camera
 ↓
video
 ↓
Canvas capture
 ↓
Resize
 ↓
JPEG 0.8
 ↓
Blob
 ↓
HTTP Upload
```

---

## 5.2 Canvas 기반 처리를 기본으로 한다

가능하면 이미지 압축을 위해 별도의 무거운 이미지 라이브러리를 추가하지 않는다.

브라우저 API를 우선 사용한다.

```ts
canvas.toBlob(
  callback,
  "image/jpeg",
  0.8,
);
```

이미지 비율은 유지한다.

예:

```text
3024 × 4032

↓

540 × 720
```

---

## 5.3 이미지 저장을 하지 않는다

게임에서 촬영한 이미지를 다음 저장소에 저장하지 않는다.

- localStorage
- sessionStorage
- IndexedDB
- Cache Storage

UI 표현을 위해 Blob URL을 사용하는 경우 사용이 끝나면 반드시 해제한다.

```ts
URL.revokeObjectURL(url);
```

페이지 lifecycle보다 오래 이미지가 남지 않도록 한다.

---

# 6. 이미지 업로드

이미지 업로드는 Socket.IO가 아니라 HTTP를 사용한다.

권장 형태:

```http
POST /rooms/{roomId}/rounds/{roundId}/submissions

Content-Type: multipart/form-data
```

Socket.IO에서는 이미지 자체가 아니라 다음과 같은 상태 이벤트를 처리한다.

```text
submission:status
submission:scored
round:finalized
```

업로드가 시작됐다는 이유로 Frontend에서 해당 제출을 성공 처리해서는 안 된다.

Backend 응답 또는 서버 이벤트를 기다린다.

---

# 7. 상태 관리

상태의 성격에 따라 저장 위치를 구분한다.

## Zustand

게임 중 계속 변경되는 Client 상태를 관리한다.

예:

```text
connection
room
participants
game
round
camera
submission
```

## TanStack Query

일반적인 HTTP 요청 상태에 사용한다.

예:

```text
방 생성
방 조회
초기 상태 복원
닉네임 변경
```

Socket.IO 이벤트로 지속적으로 변경되는 데이터를 TanStack Query만으로 관리하려고 하지 않는다.

---

# 8. Socket.IO 처리

Socket.IO 이벤트 처리 코드는 각 화면 컴포넌트에 흩어놓지 않는다.

권장 구조:

```text
src/
  realtime/
    socket.ts
    handlers/
      room.ts
      round.ts
      submission.ts
      like.ts
```

이벤트 수신 후 Zustand store를 변경하고 React UI는 store를 구독한다.

```text
Socket.IO
   ↓
Event Handler
   ↓
Zustand
   ↓
React
```

---

# 9. 재접속

모바일 네트워크는 안정적이지 않다는 것을 기본 전제로 한다.

다음 상황을 정상적인 사용자 행동으로 취급한다.

```text
Wi-Fi → LTE
LTE → Wi-Fi
브라우저 background
일시적 연결 끊김
페이지 reload
```

Socket 재연결만으로 현재 상태가 완전히 복구된다고 가정하지 않는다.

재접속이 성공하면 Backend에서 현재 게임 snapshot을 다시 받아 UI를 복원할 수 있어야 한다.

복원 대상에는 최소한 다음 정보가 포함된다.

- 현재 방
- 참여자
- 방장 여부
- 현재 게임 상태
- 현재 라운드
- 라운드 마감 시각
- 자신의 제출 상태
- 누적 포인트

---

# 10. 결과 정보 노출 제어

미제출자는 해당 라운드의 결과를 볼 수 없다.

Frontend에서도 적절한 화면을 보여줘야 하지만 보안을 Client UI에 의존해서는 안 된다.

Backend에서 결과 데이터를 받지 못하는 것을 정상적인 상태로 처리한다.

다음 방식은 금지한다.

```text
Backend에서 전체 결과 수신
↓
Frontend에서 display:none
```

Frontend는 Backend가 허용한 정보만 화면에 표시한다.

---

# 11. UI 구현 원칙

모바일 Portrait 화면을 기본으로 한다.

주요 액션은 한 손으로 누르기 쉬운 화면 하단에 배치한다.

촬영 화면에서는 카메라 preview가 가장 중요한 정보가 되어야 한다.

불필요한 텍스트를 최소화한다.

모든 대기 상태에는 다음 중 하나가 보여야 한다.

- 남은 시간
- 진행 상태
- 남은 사용자 수

무한 로딩 상태를 만들지 않는다.

---

# 12. 결과 정렬 애니메이션

라운드 결과에서는 제출자가 추가될 때 카드가 점수 순서로 재배치된다.

애니메이션 기준:

```text
300~500ms
```

여러 카드가 짧은 시간 안에 들어올 수 있기 때문에 애니메이션을 동시에 무제한 실행하지 않는다.

순차적으로 처리하거나 Motion의 layout animation 등을 이용해 UI가 과도하게 흔들리지 않도록 한다.

---

# 13. 브라우저 지원

최소 검증 대상:

```text
iOS Safari 최신 2개 버전
Android Chrome 최신 2개 버전
```

특히 다음 기능은 Desktop Chrome에서 동작한다고 완료 처리하지 않는다.

- Camera permission
- getUserMedia
- background → foreground 복원
- orientation
- Blob upload
- Socket reconnect

실제 모바일 브라우저에서 검증해야 한다.

---

# 14. Frontend가 하면 안 되는 것

다음 게임 로직을 Frontend에서 최종 결정하지 않는다.

```text
❌ 점수 계산
❌ 순위 계산
❌ 좋아요 포인트 계산
❌ 방장 권한 판단
❌ 제출 마감 판단
❌ 제출 성공 판단
❌ 라운드 완료 판단
```

또한 다음 기능을 추가하지 않는다.

```text
❌ 사진첩 업로드
❌ 촬영 이미지 영구 저장
❌ 이미지 다운로드 기능
❌ Backend에서 받지 않은 결과 추론
```

---

# 15. 성능 목표

Frontend는 다음 제품 목표를 염두에 두고 개발한다.

```text
초대 링크 → 대기실 렌더링
p95 2초 이내

라운드 상태 전환
Client 간 편차 500ms 이내
```

특히 초기 bundle size와 모바일 4G 환경을 고려한다.

게임 시작 전에 필요하지 않은 코드는 가능한 경우 lazy loading한다.

---

# 16. Definition of Done

Frontend 기능은 다음 조건을 만족해야 완료로 판단한다.

### 기능

- 요구사항 ID가 명확하다.
- 정상 flow가 동작한다.
- Backend 상태를 기준으로 동작한다.

### Mobile

- iOS Safari에서 검증한다.
- Android Chrome에서 검증한다.

### Camera

- 최초 권한 요청 flow가 정상이다.
- 거부 flow가 정상이다.
- 앱 전환 후 복구가 가능하다.

### Image

- 장변 720px 이하로 리사이즈된다.
- JPEG 0.8로 변환된다.
- 원본 이미지를 서버로 보내지 않는다.
- 로컬에 영구 저장하지 않는다.

### Realtime

- reconnect가 동작한다.
- reload 이후 현재 게임 상태를 복원할 수 있다.
- 서버 deadline과 UI timer가 일치한다.

### Requirement

모든 작업은 다음과 같이 요구사항 ID와 연결되어야 한다.

```text
RD-08
CP-04
RS-03
PM-13
```

기획문서의 요구사항과 연결할 수 없는 기능은 임의로 추가하지 않는다.

---

# 17. 최우선 목표

Frontend 개발에서 가장 중요하게 보는 순서는 다음과 같다.

1. 모바일 카메라가 안정적으로 동작하는가
2. 사용자가 권한 요청 과정을 이해할 수 있는가
3. 촬영부터 제출까지 빠르게 진행되는가
4. 실시간 게임 상태가 서버와 일치하는가
5. 재접속했을 때 게임으로 복귀할 수 있는가
6. 이미지가 불필요하게 저장되거나 전송되지 않는가
7. 결과 화면이 재미있고 자연스럽게 움직이는가

화려한 UI보다 게임 진행의 안정성과 모바일 카메라 UX를 우선한다.