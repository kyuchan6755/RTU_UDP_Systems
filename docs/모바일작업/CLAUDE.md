# CLAUDE.md — 모바일작업/모바일코드연습 프로젝트

이 파일은 Claude Code가 이 프로젝트 디렉토리에서 작업할 때 따라야 할 규칙입니다.

## 프로젝트 개요

- **목적**: Claude Code 모바일 앱으로 클라우드 컨테이너에서 코드 개발 후 Google Drive로 결과물 보관
- **개발 환경**: Anthropic 클라우드 Linux 컨테이너 (일회성, ephemeral)
- **보관소**: Google Drive `모바일코드/모바일작업/모바일코드연습/`
- **소유자**: kyuchan6755@gmail.com

## 디렉토리 매핑 (마운트)

| 위치 | 경로 | ID/링크 |
|------|------|---------|
| 컨테이너 로컬 | `/home/user/모바일작업/모바일코드연습/` | (세션별 임시) |
| Google Drive | `내 드라이브/모바일코드/모바일작업/모바일코드연습/` | `1wj8DqEB87Nkepejv5pxJQKZbkA4FS_pK` |

상위 Drive 폴더 ID:
- 모바일코드: `1dA5rgPPm6867t4KDAM5LNxDB6qMY7Em3`
- 모바일작업: `190JP_fU5Bb2RL3TCJ48sIIebblpiD0O2`
- 모바일코드연습: `1wj8DqEB87Nkepejv5pxJQKZbkA4FS_pK`

## 작업 규칙

### 파일 생성/수정
- 모든 파일은 **로컬 컨테이너** (`/home/user/모바일작업/모바일코드연습/`)에 작성
- Claude는 사용자 승인 없이 즉시 파일 생성/수정/삭제 가능
- 한국어 파일명/디렉토리명 사용 허용

### Drive 업로드 트리거
사용자가 다음과 같이 말하면 **즉시 Drive로 업로드** 실행:
- "올려줘" / "업로드해줘" / "Drive에 올려" → 직전 작업 파일 업로드
- "전부 올려줘" / "변경된 파일 다 올려줘" → 디렉토리 전체 업로드
- "○○ 올려줘" → 특정 파일만 업로드

업로드 시 MCP 도구 사용:
- `mcp__453deae7-75a4-430f-a183-82c4014499d1__create_file`
- `parentId`: 위 매핑 테이블의 Drive 폴더 ID
- `contentMimeType`: 텍스트는 `text/markdown` / `text/plain`, 그 외 적절히
- `disableConversionToGoogleType`: `true` (Google Docs 변환 방지)
- 한글 포함 텍스트는 `base64Content`로 업로드 (UTF-8 인코딩 후 base64)

### 하위 디렉토리 처리
- 로컬에 하위 디렉토리 생성 시 Drive에도 동일 구조 자동 생성
- Drive 폴더 ID는 생성 후 응답에서 추출하여 사용

### 보고 형식
업로드 완료 시 다음 정보 전달:
- 로컬 경로
- Drive 경로
- Drive 직접 링크 (`https://drive.google.com/file/d/<ID>`)

## 절대 하지 말 것

- 핸드폰 내장 저장소 직접 접근 시도 (불가능 — 클라우드 컨테이너에서 동작)
- rclone/FUSE 마운트 시도 (OAuth 토큰 필요, 비효율)
- git 저장소 (`/home/user/RTU_UDP_Systems/`)와 혼동 — 이 프로젝트는 별개
- 사용자가 "올려줘" 말하기 전에 임의로 Drive 업로드
- Drive 폴더 ID를 추측하거나 새로 만들기 (위 매핑 사용)

## 참고

- 이 컨테이너는 세션 종료 후 사라짐 → 로컬 파일도 함께 사라짐
- 영구 보관은 Drive 업로드로만 가능
- 다음 세션에서 이전 작업 이어가려면 Drive에서 파일을 다시 컨테이너로 복원 필요
  - `mcp__...__search_files`로 검색 → `download_file_content`로 다운로드
