# demo-cubrid — 서브모듈 자동 PR 생성 A/B안 데모 (부모 저장소)

서브모듈 SHA 자동 갱신을 **A안(Dependabot)** 과 **B안(자체 구축)** 두 방식으로 나란히 시연하기 위한 부모 저장소입니다.
결정권자에게 "실제로 열리는 PR의 모습"과 "각 방식의 운영 차이"를 보여주는 것이 목적입니다.

## 구성

| 파일 | 방식 | 역할 |
|------|------|------|
| `.github/dependabot.yml` | A안 | 스케줄마다 서브모듈 SHA 확인 → PR 자동 생성 |
| `.github/CODEOWNERS` | A안 | PR에 리뷰어 자동 요청 |
| `.github/workflows/pr-style.yml` | 공용 | PR 제목 규칙 필수 체크 (실제 cubrid 복제) |
| `.github/workflows/bump-receiver.yml` | B안 | dispatch 수신 → bump → PR 생성 (peter-evans) |
| 서브모듈 `demo-jdbc/` | 공용 | 자동 갱신 대상 |

> 준비(저장소 생성·PAT·브랜치 보호)는 상위 폴더의 **`SETUP.md`** 참고. `setup.sh` 로 한 번에 처리 가능.

---

## 🎬 시연 각본 (약 10분)

### 사전 상태
- `pr-style` 이 필수 체크로 등록돼 있음 → 제목 규칙 위반 시 머지 버튼이 막힘
- (선택) 저장소 변수 `DEMO_REVIEWER` = 동료 GitHub ID

### 장면 ① 공통 — "서브모듈에 변경이 생겼다"
```bash
# demo-jdbc 저장소에서
echo "fix $(date)" >> CHANGES.md
git commit -am "[DEMO-123] NPE 수정"
git push
```

### 장면 ② A안 (Dependabot)
1. `demo-cubrid` → **Insights → Dependency graph → Dependabot → "Check for updates"** (수동 트리거)
2. 잠시 후 PR 자동 생성 확인:
   - 제목: `Bump demo-jdbc from <old> to <new>`  ← **[번호] 형식이 아님**
   - CODEOWNERS 로 리뷰어 자동 요청됨
   - ❌ **`pr-style` 빨간불 → 머지 차단**
3. **예외 적용 (before → after):** Settings → Secrets and variables → Actions → Variables →
   `SKIP_PRSTYLE_FOR_BOT = true` 설정 후, PR의 실패한 `pr-style` 체크를 **Re-run**
   → job이 skip 되어 ✅ **성공 처리 → 머지 가능** (커밋 없이 변수만으로)

> 메시지: "설정 파일 1개면 매일 자동으로 PR이 열린다. 단 제목이 우리 규칙과 달라 **봇 예외 한 줄**이 필요하다."

### 장면 ③ B안 (자체 구축)
- 장면 ①의 push 직후(1~2분 내) `bump/demo-jdbc` PR이 이미 열려 있음:
  - 제목: `[DEMO-123] Update demo-jdbc submodule`  ← **이슈번호 상속, pr-style 자연 통과 ✅**
  - Assignee = push한 사람, Reviewer = 동료 자동 지정
- (수동 실행도 가능) Actions → **Submodule bump (receiver)** → Run workflow

> 메시지: "1~2분 만에, 제목·담당자·리뷰어가 우리 규칙 그대로. 대신 **이 워크플로들을 우리가 소유·운영**한다."

### 장면 ④ 공통 — PR은 쌓이지 않는다
```bash
# demo-jdbc 에서 연속으로 2번 push
echo "a" >> CHANGES.md && git commit -am "[DEMO-124] 변경 A" && git push
echo "b" >> CHANGES.md && git commit -am "[DEMO-125] 변경 B" && git push
```
→ 두 방식 모두 **열린 PR은 1개**, 최신 SHA 로 갱신될 뿐 새 PR이 쌓이지 않음.

### 장면 ⑤ 구조 차이
- `demo-jdbc` 저장소: A안 관점에선 **아무 워크플로도 없음**(무접촉). B안에선 `notify-parent.yml` 이 들어가 있음.
- → "A안은 서브모듈에 손대지 않고 하루 1회, B안은 서브모듈마다 알림을 심고 실시간."

---

## 데모의 한계 (실전과 다른 점 — 각주로 안내)
- **CircleCI · TC 게이트(Check TC PRs)는 재현하지 않음** → 비교 보고서 5장이 담당. 여기선 `pr-style` 하나만 복제.
- **B안 토큰은 PAT 로 대체** → 실전은 GitHub App 설치 토큰 권장.
- **서브모듈 1개** 로 축약 → 실전은 3개(jdbc·cci·manager). 동작 원리는 동일.
