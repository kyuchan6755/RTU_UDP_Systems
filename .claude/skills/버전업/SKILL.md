---
name: 버전업
description: "버전을 patch +1 증가시키고 변경사항을 자동 메시지로 git commit (push/PR/merge는 별도)"
disable-model-invocation: true
allowed-tools: Bash Read Edit
---

# 버전업 — 버전 +1 + 자동 commit

`/버전업` 호출 시 아래 순서로 수행. **push, PR, merge는 절대 수행하지 않음.**

## 1단계: 상태 확인

병렬로 실행:
- `git status --short`
- `git branch --show-current`
- `git diff --stat HEAD`

이후 다음을 결정:
- **A 케이스 (변경 있음)**: 사용자가 작업 후 호출 → 변경사항 + 버전bump 한 번에 commit
- **B 케이스 (clean)**: 버전 마일스톤만 찍는 용도 → 버전bump 만 commit

`HEAD`가 없는 빈 저장소면 중단하고 안내.

## 2단계: 민감 파일 차단

`git status --short` 결과에서 다음 패턴 발견 시 사용자에게 경고 후 중단:
- `.env`, `.env.*`
- `*credentials*`, `*secret*`, `*password*`
- `*.key`, `*.pem`, `id_rsa*`

사용자가 "그래도 진행해" 명시할 때만 계속. 그 외엔 멈춤.

## 3단계: 버전 읽기 및 bump

`common/protocol_constants.py` 읽기. 다음 라인 찾기:

```python
PROGRAM_VERSION = "X.Y.Z"
```

- X.Y.Z 파싱 → Z를 +1
- `Edit` 도구로 해당 라인만 교체
  - old_string: `PROGRAM_VERSION = "X.Y.Z"`
  - new_string: `PROGRAM_VERSION = "X.Y.(Z+1)"`
- 기록: `<old_version>` → `<new_version>`

**중요**:
- `PROTOCOL_VERSION` (V3.0.1 등)은 절대 건드리지 않음 — 와이어 프로토콜 사양 버전이라 별개
- major(X)/minor(Y)는 자동 증가 안 함 — patch(Z)만
- 파일이 없거나 PROGRAM_VERSION 라인이 없으면 중단 후 사용자에게 알림

## 4단계: 변경 분석 (commit 메시지용)

`git diff HEAD` 또는 staged + unstaged 합쳐 분석:
- 어떤 디렉토리/파일이 바뀌었나?
- 새 파일인지 수정인지 삭제인지?
- 변경 성격이 무엇인지 (기능 추가, 버그 수정, 문서, 리팩터링, 설정 등)

## 5단계: 커밋 메시지 자동 생성

형식:
```
<type>: <한국어 한 줄 요약> (v<new_version>)

<상세 설명 1 - 어떤 파일에 뭐가 바뀌었는지>
<상세 설명 2 - 왜>
<상세 설명 3 - 영향 범위>
```

`<type>` 선택 규칙:
- `feat`: 새 기능/모듈 추가
- `fix`: 버그 수정
- `docs`: 문서/주석만 변경
- `refactor`: 동작 동일, 구조 변경
- `test`: 테스트 추가/수정
- `chore`: 빌드/설정/잡일
- `style`: 포맷팅/공백
- B 케이스(버전bump만)면 `chore: bump version to vX.Y.Z`

세션 ID 푸터는 붙이지 않음 (CLAUDE.md 규칙에서 명시한 경우만 추가).

## 6단계: 스테이지 + 커밋

```bash
git add -A
git commit -m "$(cat <<'EOF'
<생성한 메시지>
EOF
)"
```

HEREDOC 사용 필수 (개행/특수문자 안전).

`--no-verify` 절대 사용 안 함. 훅 실패 시 원인 보고 후 사용자 판단 대기.

## 7단계: 결과 보고

다음 형식으로 출력:

```
버전업 완료

  버전:    <old> → <new>
  커밋:    <short_hash> "<commit_title>"
  파일:    <changed_count>개 변경
  브랜치:  <current_branch>

다음 단계 (필요 시 직접 실행):
  push:  git push -u origin <current_branch>
  PR:    "PR 만들어줘" 요청
  merge: "main에 머지해줘" 요청
```

## 절대 금지

- `git push` 실행 금지
- PR 생성 금지
- main 브랜치 merge 금지
- `git commit --amend` 금지 (사용자 명시적 요청 시만)
- major/minor 버전 증가 (사용자 명시적 요청 시만)
- `--no-verify`, `--no-gpg-sign` 등 검증 우회 플래그 금지
- 민감 파일 commit (3단계 차단됨)

## 보조 케이스

### 충돌하는 PROGRAM_VERSION 정의가 여러 개일 때
1차: `common/protocol_constants.py` 만 bump
2차: 다른 위치(`equipment_simulator/common/protocol_constants.py` 등) 발견 시 사용자에게 보고. 자동 동기화 여부 질문.

### 다른 디렉토리에서 호출되었을 때
`git rev-parse --show-toplevel`로 저장소 루트 찾고, 거기 기준으로 동작.

### 인자로 메시지 전달 받았을 때
무시. 항상 자동 생성 (사용자 설정: 자동 생성).
