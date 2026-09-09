# 개발 문서 안내

이 디렉터리는 **개발 가이드라인**을 관리한다. 구현 명세(spec)는 각 구현 저장소가 소유한다.

| 문서 | 위치 |
|---|---|
| Backend 개발 가이드라인 | [Backend Development Guidelines.md](./Backend%20Development%20Guidelines.md) |
| Frontend 개발 가이드라인 | [Frontend Development Guidelines.md](./Frontend%20Development%20Guidelines.md) |
| 감정 판별 PoC | [femo_web.py](./femo_web.py) |

## 구현 명세와 실행 계획

명세를 코드와 같은 저장소에 두어, 구현 변경과 명세 갱신이 하나의 PR에서 함께 리뷰되도록 한다.

| 저장소 | 문서 |
|---|---|
| emoselfie-BE | [구현 명세](../../emoselfie-BE/spec.md) · [개발 계획](../../emoselfie-BE/PLAN.md) · [TODO](../../emoselfie-BE/TODO.md) |
| emoselfie-FE | [구현 명세](../../emoselfie-FE/spec.md) · [개발 계획](../../emoselfie-FE/PLAN.md) · [TODO](../../emoselfie-FE/TODO.md) |

## 문서 위계

```
requirements.md (PRD)      무엇을 만들 것인가
  └ DECISIONS.md           PRD ↔ 디자인 충돌 12건의 결정 이력
      └ development/*.md   어떤 원칙으로 만들 것인가 (가이드라인)
          └ 각 저장소 spec.md   코드로 옮길 수 있는 계약 (구현 명세)
```

세 층이 어긋나면 **PRD > DECISIONS > 가이드라인** 순으로 따른다. 구현 명세는 상위 문서를 해석해 확정한 결과이므로, 상위 문서가 바뀌면 각 저장소의 spec을 함께 갱신한다.

갱신일: 2026-09-09. 상대 경로는 emoselfie-DOCS·emoselfie-BE·emoselfie-FE가 같은 상위 디렉터리에 있는 워크스페이스 기준이다.
