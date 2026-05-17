---
name: 버전업
description: "버전을 patch +1 증가시키고 변경사항을 자동 메시지로 git commit. 버전 파일 자동 탐지 (pyproject.toml, package.json, setup.py, VERSION, __init__.py, 또는 프로젝트 커스텀). push/PR/merge는 별도."
disable-model-invocation: true
allowed-tools: Bash Read Edit Glob Grep
---

# 버전업 — 버전 +1 + 자동 commit (Global)

`/버전업` 호출 시 아래 순서로 수행. **push, PR, merge는 절대 수행하지 않음.**

## 1단계: 저장소 컨텍스트 확인

병렬로 실행:
- `git rev-parse --show-toplevel` → 저장소 루트 경로 (`<repo_root>`)
- `git status --short`
- `git branch --show-current`
- `git diff --stat HEAD`

git 저장소가 아니면 중단.

이후 판정:
- **A 케이스 (변경 있음)**: 변경사항 + 버전bump 한 번에 commit
- **B 케이스 (clean)**: 버전 마일스톤만 찍는 용도 → 버전bump 만 commit

## 2단계: 버전 파일 자동 탐지

다음 순서로 검색. 첫 번째로 찾은 위치 사용.

### 우선순위 0: 프로젝트 커스텀 (CLAUDE.md 힌트)
`<repo_root>/CLAUDE.md` 또는 `<repo_root>/.claude/version-config.md` 에 다음 형식 있으면 사용:
```
VERSION_FILE: <path>
VERSION_PATTERN: <regex 또는 변수명>
```

### 우선순위 1: RTU UDP Systems 패턴 (이 프로젝트 인식)
`<repo_root>/common/protocol_constants.py` 존재 + `PROGRAM_VERSION = "X.Y.Z"` 라인 있으면 사용.
(이 프로젝트 전용 빠른 경로. `PROTOCOL_VERSION`은 절대 건드리지 않음)

### 우선순위 2: pyproject.toml (Python 표준)
`<repo_root>/pyproject.toml` 에서:
```toml
[project]
version = "X.Y.Z"
```
또는 `[tool.poetry] version = "X.Y.Z"`

### 우선순위 3: package.json (Node.js)
`<repo_root>/package.json` 에서 `"version": "X.Y.Z"` 라인.

### 우선순위 4: setup.py
`<repo_root>/setup.py` 에서 `version="X.Y.Z"` 또는 `version='X.Y.Z'`.

### 우선순위 5: VERSION 파일
`<repo_root>/VERSION` 또는 `<repo_root>/VERSION.txt` (순수 텍스트 `X.Y.Z`).

### 우선순위 6: __init__.py 의 __version__
`Glob`으로 `**/__init__.py` 찾기. 그 중 `__version__ = "X.Y.Z"` 라인 있는 파일.
복수 존재 시 가장 얕은 깊이의 파일 우선. depth가 같으면 사용자에게 질문.

### 우선순위 7: 못 찾음
사용자에게 질문:
- 버전 파일 경로
- 버전 라인 패턴 (예: `version = "..."`, `VERSION = "..."`)

대답을 받으면 `<repo_root>/.claude/version-config.md` 에 저장하여 다음 호출 시 자동 사용:
```
VERSION_FILE: <answered_path>
VERSION_PATTERN: <answered_pattern>
```

## 3단계: 민감 파일 차단

`git status --short` 결과 검사. 다음 패턴 발견 시 사용자에게 경고 후 중단:
- `.env`, `.env.*`
- `*credentials*`, `*secret*`, `*password*`
- `*.key`, `*.pem`, `id_rsa*`, `*.pfx`

사용자가 명시적으로 "그래도 진행해" 라고 할 때만 계속.

## 4단계: 버전 읽기 및 bump

찾은 파일에서 `X.Y.Z` 추출:
- `X.Y.Z` 형식 (각 자리 숫자)
- 앞에 `v` 또는 `V` 접두사가 있으면 (예: `v1.2.3`) 보존
- 더 긴 형식(`X.Y.Z-rc.1`, `X.Y.Z+build.1`)은 패치 숫자만 +1, 나머지 유지

bump 규칙:
- **patch(Z) 만 +1** (예: `2.2.0` → `2.2.1`)
- major(X)/minor(Y)는 자동 증가 안 함

`Edit` 도구로 해당 라인 교체 (전체 파일 재작성 X, 라인 단위 교체).

기록: `<old_version>` → `<new_version>`

**별도 프로토콜 버전 보호**: 파일에 `PROTOCOL_VERSION`, `API_VERSION`, `WIRE_VERSION` 같은 별도 상수가 있어도 절대 건드리지 않음.

## 5단계: 변경 분석 (commit 메시지용)

`git diff HEAD --stat` + `git status --short` 분석:
- 어떤 디렉토리/파일이 바뀌었는지
- 새 파일 / 수정 / 삭제 비율
- 변경 성격 (기능 / 버그 / 문서 / 리팩터링 / 설정)

큰 diff면 주요 변경 3-5개만 추출.

## 6단계: 커밋 메시지 자동 생성

형식 (한국어):
```
<type>: <한 줄 요약> (v<new_version>)

- <상세 변경 1>
- <상세 변경 2>
- <상세 변경 3>
```

`<type>` 선택:
- `feat`: 새 기능/모듈 추가
- `fix`: 버그 수정
- `docs`: 문서/주석만 변경
- `refactor`: 동작 동일, 구조 변경
- `test`: 테스트 추가/수정
- `chore`: 빌드/설정/잡일 (B 케이스 기본값)
- `style`: 포맷팅/공백
- `perf`: 성능 개선

B 케이스(버전bump만):
```
chore: bump version to v<new_version>
```

세션 ID 푸터는 붙이지 않음 (프로젝트 CLAUDE.md에서 명시한 경우만).

## 7단계: 스테이지 + 커밋

```bash
git add -A
git commit -m "$(cat <<'EOF'
<생성한 메시지>
EOF
)"
```

HEREDOC 필수 (개행/특수문자 안전).
`--no-verify`, `--no-gpg-sign` 등 검증 우회 플래그 절대 금지.
훅 실패 시 원인 보고 후 사용자 판단 대기.

## 8단계: 결과 보고

```
버전업 완료

  버전:    <old> → <new>
  파일:    <version_file_path>
  커밋:    <short_hash> "<commit_title>"
  변경:    <changed_count>개 파일, +<insertions>/-<deletions>줄
  브랜치:  <current_branch>

다음 단계 (필요 시 직접 실행):
  push:   git push -u origin <current_branch>
  PR:     "PR 만들어줘" 요청
  merge:  "main에 머지해줘" 요청
```

## 절대 금지

- `git push` 실행 금지
- PR 생성 금지
- main 브랜치 merge 금지
- `git commit --amend` 금지 (사용자 명시적 요청 시만)
- major/minor 버전 증가 (사용자 명시적 요청 시만)
- `--no-verify`, `--no-gpg-sign` 등 검증 우회 플래그 금지
- 민감 파일 commit (3단계 차단됨)
- 별도 프로토콜/API 버전 상수 수정

## 특수 케이스

### 동일 버전이 여러 파일에 있을 때
1차로 우선순위 1순위 파일만 bump.
다른 파일에 같은 버전 문자열이 있으면 사용자에게 보고:
- "같은 버전이 X, Y 파일에도 있습니다. 함께 bump할까요?"

### 잘못된 버전 형식
`X.Y.Z` 패턴 매칭 실패 시:
- 발견한 라인 사용자에게 보여주고
- 어떻게 처리할지 (수동 입력, 건너뛰기) 질문

### 인자로 메시지 전달 받았을 때
무시. 항상 자동 생성 (사용자 설정: 자동 생성).

### 다른 디렉토리에서 호출되었을 때
`git rev-parse --show-toplevel`로 저장소 루트 찾고, 거기 기준으로 동작.
