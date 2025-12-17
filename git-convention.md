# Git 컨벤션 가이드 

팀 협업을 위한 Git 사용 규칙입니다. 통일된 방식으로 깔끔하고 협업하기 쉬운 Git 히스토리를 만들어갑시다!

---

## 1️⃣ 브랜치 전략

| 브랜치 | 용도 |
|--------|------|
| `main` | 배포 및 릴리즈용 |
| `dev`  | 기능 통합 브랜치 |
| `feat/*` | 새로운 기능 개발(백) |
| `design/*` | CSS 등 사용자 UI 디자인 변경(프론트) |
| `fix/*`  | 버그 수정 |
| `docs/*` | 문서 수정 |
| `refactor/*` | 리팩토링 |
| `test/*` | 테스트 코드 |
| `chore/*` | 설정, 기타 작업 |

### 브랜치 네이밍 예시
feat/login<br>
fix/signup-validation<br>
refactor/matching-service<br>
---

## 2️⃣ 커밋 컨벤션

| 타입 | 설명 |
|------|------|
| `feat` | 새로운 기능 추가 |
| `fix` | 버그 수정 |
| `docs` | 문서 수정 (README 등) |
| `style` | 코드 포맷팅, 세미콜론 누락 등 |
| `design` | CSS 등 사용자 UI 디자인 변경 |
| `refactor` | 코드 리팩토링 |
| `test` | 테스트 코드 추가/수정 |
| `chore` | 기타 설정, 빌드 파일 수정 등 |

###  커밋 예시
feat: 회원가입 API 구현<br>
fix: 로그인 시 비밀번호 검증 오류 수정<br>
refactor: UserService 리팩토링<br>
---

## 3️⃣ Pull Request 규칙

- PR 제목: `[타입] 작업 내용`
- PR 템플릿 사용 (자동 적용됨)
- 기능/버그 별 작은 단위로 PR 분리
- 팀원 리뷰 요청 → 승인 후 머지

###  PR 제목 예시
[feat] 회원가입 API 구현<br>
[fix] 로그인 validation 오류 해결<br>

---

## 4️⃣ GitHub Issues 관리

- 이슈는 작업 전 선제적으로 생성
- 반드시 하나의 이슈에 집중
- 작업 후 PR에 `Close #이슈번호` 연결
- 템플릿 활용 권장

### 기본본 라벨
| 라벨 | 용도 |
|------|------|
| `feature` | 신규 기능 |
| `bug` | 버그 수정 |
| `refactor` | 리팩토링 |
| `documentation` | 문서 작업 |
| `discussion` | 논의 사항 |
| `help wanted` | 도움 필요 |
| `design` | 디자인 |
| `good first issue` | 입문자용 이슈 |

---

## 마무리 체크리스트

- [ ] 기능 작업 전 이슈 등록
- [ ] 브랜치 규칙 준수
- [ ] 커밋 메시지 규칙 준수
- [ ] PR 제목/본문 충실히 작성
- [ ] PR에서 관련 이슈 링크 (`Close #1` 등)

---

컨벤션을 지켜서 더 빠르고, 더 멋진 협업을 해봐요!
