# OpenCode 호환성 이식 계획

> 브랜치: `feat/opencode-compat`
> 작성일: 2026-04-26
> 목표: openclone을 Claude Code 전용에서 OpenCode까지 지원하는 멀티 호스트 스킬로 확장

---

## 1. 배경

openclone은 현재 Claude Code의 Skill/Hook/Statusline 아키텍처에 깊이 결합되어 있다. OpenCode(anomalyco/opencode)는 Claude Code와 유사하지만 **Plugin 기반 TypeScript 아키텍처**를 사용하며, 다행히 `~/.claude/skills/*/SKILL.md` 경로를 그대로 스캔하는 하위 호환성을 갖추고 있다.

이 문서는 Claude Code 기능을 유지하면서 OpenCode 네이티브 지원을 추가하는 작업 계획이다.

---

## 2. 아키텍처 매핑

openclone의 각 Claude Code 기능이 OpenCode에서 어떻게 대체되는지 정리.

### 2.1 Skill 인식

| 항목 | Claude Code | OpenCode | 호환성 |
|------|-------------|----------|--------|
| Skill 경로 | `~/.claude/skills/openclone/SKILL.md` | 동일 경로 스캔 + `~/.config/opencode/skills/` | ✅ 변경 불요 |
| Frontmatter 필수 | `name`, `description`, `allowed-tools` | `name`, `description` (나머지 무시) | ✅ 변경 불요 |
| `$ARGUMENTS` 확장 | 슬래시 커맨드 인자 자동 주입 | 동일 | ✅ 변경 불요 |
| `${CLAUDE_SKILL_DIR}` | Claude Code가 런타임 확장 | OpenCode도 동일하게 확장 | ✅ 변경 불요 |
| Natural-language 매칭 | `description` 필드로 자동 매칭 | 동일 | ✅ 변경 불요 |

**결론**: SKILL.md 자체는 수정 없이 양쪽에서 동작한다.

### 2.2 Hook → Plugin 변환 (핵심 작업)

| Claude Code Hook | 역할 | OpenCode 대체 |
|-----------------|------|--------------|
| `UserPromptSubmit` → `inject-active-clone.sh` | 매 턴 페르소나 주입 | `experimental.chat.system.transform` 플러그인 훅 |
| `SessionStart` → `session-update.sh` | 세션 시작 시 자동 업데이트 | `event: session.created` 플러그인 훅 |
| `statusLine` → `statusline.sh` | 상태줄 표시 | TUI 토스트 또는 생략 |

### 2.3 설정 등록

| 항목 | Claude Code | OpenCode |
|------|-------------|----------|
| Hook 등록 | `~/.claude/settings.json` (python3 편집) | `opencode.json`의 `plugin` 배열 |
| 설치 스크립트 | `./setup` (bash) | `./setup-opencode` (bash, 설정 파일 생성) |
| 설치 경로 | `~/.claude/skills/openclone/` | 동일 (OpenCode도 이 경로를 스캔) |

---

## 3. 작업 항목

### Phase 1: OpenCode Plugin 작성 (핵심)

#### 3.1 페르소나 주입 플러그인

**파일**: `.opencode/plugins/openclone-inject.ts` (또는 `plugins/openclone-inject.ts`)

`hooks/inject-active-clone.sh`의 로직을 TypeScript로 이식:

```
입력: ~ 매 사용자 메시지
처리:
  1. ~/.openclone/force-push-detected 존재 → 업그레이드 배너 주입
  2. ~/.openclone/room 존재 + 비어있지 않음 → Room 모드
     - 전체 멤버 페르소나 읽기 (user 우선, built-in 차선)
     - 라우팅 규칙 + 인용 계약 주입
  3. ~/.openclone/active-clone 존재 → Single-clone 모드
     - 페르소나 + 지식 디렉토리 경로 + 인용 계약 주입
  4. 둘 다 없음 → no-op
출력: system prompt에 <openclone-*> 블록 추가
```

**사용할 OpenCode 훅**: `experimental.chat.system.transform`
- `output.system`에 페르소나 컨텍스트를 append
- 기존 `inject-active-clone.sh`의 heredoc 내용을 그대로 재사용

**slug 검증**: bash의 `is_valid_slug()`를 TypeScript 정규식으로 이식

**편의 함수**: `resolveClone(name)` — user 경로 우선, built-in 차선 조회

#### 3.2 자동 업데이트 플러그인

**파일**: `.opencode/plugins/openclone-update.ts`

`scripts/session-update.sh`의 로직을 TypeScript로 이식:

```
이벤트: session.created
처리:
  1. ~/.openclone/no-auto-update 존재 → 스킵
  2. last-update-check mtime으로 1시간 쓰로틀
  3. git fetch + merge --ff-only (Bun.$ 사용)
  4. fast-forward 실패 시 force-push-detected 마커 작성
  5. 로그를 ~/.openclone/last-update.log에 기록
```

**참고**: Bun의 `$` API로 bash 명령을 직접 실행 가능하므로, 기존 git 명령어를 그대로 재사용.

#### 3.3 상태줄 (선택)

OpenCode는 TUI 토스트(`tui.toast.show`)를 지원하지만, Claude Code의 statusLine과 동일한 개념은 아니다.

**옵션**:
- A) 상태줄 기능 생략 (MVP)
- B) 활성 클론 정보를 `tui.toast.show`로 세션 시작 시 1회 표시
- C) 세션 제목에 클론명 포함

**권장**: A (MVP에서 생략, 추후 추가)

---

### Phase 2: 설치 스크립트

#### 3.4 `setup-opencode` 스크립트

**파일**: `setup-opencode` (루트, 실행 가능한 bash 스크립트)

```
동작:
  1. 설치 위치 확인 (~/.claude/skills/openclone 또는 ~/.config/opencode/skills/openclone)
  2. ~/.config/opencode/opencode.json 존재 확인, 없으면 생성
  3. opencode.json에 플러그인 경로 등록:
     - "plugin" 배열에 "./plugins/openclone-inject.ts" 추가
     - 이미 있으면 스킵
  4. ~/.openclone/ 디렉토리 생성
  5. 완료 메시지 출력
```

**참고**: Claude Code의 `./setup`과 독립적으로 동작. 기존 Claude Code 설치에 영향 없음.

#### 3.5 `uninstall-opencode` 스크립트

**파일**: `uninstall-opencode` (루트)

```
동작:
  1. opencode.json에서 openclone 플러그인 항목 제거
  2. 플러그인 파일은 스킬 디렉토리 내에 있으므로 삭제 불요
  3. ~/.openclone/ 데이터는 보존 (Claude Code와 공유)
  4. 완료 메시지 출력
```

---

### Phase 3: Chrome MCP 대체 (선택)

#### 3.6 인터뷰/ingest/update 워크플로의 Chrome MCP 의존 제거

현재 `interview-workflow.md`, `refine-workflow.md`, `update-workflow.md`이 `claude-in-chrome` MCP를 사용한다. OpenCode에서는 이 MCP가 없으므로:

**옵션**:
- A) OpenCode에서는 WebFetch/WebSearch으로 fallback (기능은 동일, 속도만 약간 느림)
- B) Playwright MCP를 OpenCode에서 설정 가이드 제공
- C) Chrome MCP 의존을 선택적으로 만들기 (MCP 있으면 사용, 없으면 WebSearch)

**권장**: C — MCP 유무를 런타임에 감지하여 자동 분기. 이렇게 하면 Claude Code에서는 기존대로 동작하고 OpenCode에서는 WebSearch/WebFetch로 fallback.

**수정 파일**:
- `references/interview-workflow.md` — Chrome MCP preflight를 조건부로 변경
- `references/refine-workflow.md` — 동일
- `references/update-workflow.md` — 동일

---

### Phase 4: 문서 및 CI

#### 3.7 README.md 업데이트

설치 섹션에 OpenCode 설치 가이드 추가:

```markdown
### OpenCode

```bash
git clone --filter=blob:none --sparse --depth=1 \
  https://github.com/open-clone/openclone.git \
  ~/.claude/skills/openclone \
  && cd ~/.claude/skills/openclone \
  && git sparse-checkout set --no-cone '/*' '!/clones/*/knowledge/' \
  && ./setup-opencode
```
```

플랫폼 지원 표에 OpenCode 행 추가.

#### 3.8 CI 검증 확장

- `.github/scripts/validate-opencode-plugin.ts` — Plugin 파일 존재 및 TypeScript 문법 검증
- `.github/workflows/validate.yml` — `tsc --noEmit` 스텝 추가 (Plugin용)

#### 3.9 CLAUDE.md 업데이트

프로젝트 구조 및 아키텍처 섹션에 OpenCode Plugin 경로 및 설치 스크립트 정보 추가.

---

## 4. 파일 변경 요약

### 신규 파일

| 파일 | 설명 |
|------|------|
| `plugins/openclone-inject.ts` | 페르소나 주입 Plugin (experimental.chat.system.transform) |
| `plugins/openclone-update.ts` | 자동 업데이트 Plugin (event: session.created) |
| `setup-opencode` | OpenCode용 설치 스크립트 |
| `uninstall-opencode` | OpenCode용 제거 스크립트 |

### 수정 파일

| 파일 | 변경 내용 |
|------|-----------|
| `README.md` | OpenCode 설치 섹션, 플랫폼 지원 표 |
| `CLAUDE.md` | 프로젝트 구조에 Plugin 경로 추가 |
| `references/interview-workflow.md` | Chrome MCP 조건부 처리 |
| `references/refine-workflow.md` | Chrome MCP 조건부 처리 |
| `references/update-workflow.md` | Chrome MCP 조건부 처리 |
| `.github/workflows/validate.yml` | Plugin TypeScript 검증 스텝 |

### 변경 없는 파일

| 파일 | 이유 |
|------|------|
| `SKILL.md` | OpenCode가 동일하게 인식 |
| `hooks/inject-active-clone.sh` | Claude Code 전용, 유지 |
| `setup` / `uninstall` | Claude Code 전용, 유지 |
| `scripts/*.sh` | 순수 bash, 변경 불요 |
| `references/clone-schema.md` | 순수 마크다운 스키마 |
| `references/categories.md` | 순수 마크다운 |
| `references/home-workflow.md` | 순수 워크플로 |
| `references/panel-workflow.md` | 순수 워크플로 |
| `references/room-workflow.md` | 순수 워크플로 |
| `clones/*/persona.md` | 데이터 파일 |
| `assets/clone-template.md` | 템플릿 |

---

## 5. 실행 순서

```
Phase 1 (핵심)
  ├── 3.1 openclone-inject.ts 작성
  ├── 3.2 openclone-update.ts 작성
  └── 3.3 상태줄 결정 (MVP: 생략)

Phase 2 (설치)
  ├── 3.4 setup-opencode 작성
  └── 3.5 uninstall-opencode 작성

Phase 3 (선택)
  └── 3.6 Chrome MCP 조건부 처리

Phase 4 (문서/CI)
  ├── 3.7 README.md 업데이트
  ├── 3.8 CI 검증 확장
  └── 3.9 CLAUDE.md 업데이트
```

---

## 6. 리스크 및 주의사항

### 6.1 `experimental.chat.system.transform` 안정성

이 훅 이름에 `experimental` 접두사가 붙어 있다. OpenCode 버전업에 따라 API가 변경될 수 있음.

**대응**: Plugin에서 OpenCode 버전을 체크하거나, `chat.message` 훅을 폴백으로 준비.

### 6.2 양쪽 호스트 동시 사용

한 시스템에 Claude Code와 OpenCode가 모두 설치된 경우:
- `~/.openclone/` 상태는 양쪽이 공유 → 문제 없음
- Claude Code는 `./setup`으로, OpenCode는 `./setup-opencode`로 각각 독립 설정
- 동시 실행 시 active-clone/room 충돌 가능성 → 파일 기반이므로 last-write-wins (실질적 문제 낮음)

### 6.3 `$ARGUMENTS` 및 `${CLAUDE_SKILL_DIR}` 확장

OpenCode가 Claude Code와 정확히 동일한 방식으로 이 변수들을 확장하는지 검증 필요. 공식 문서상 동일 경로를 스캔하므로 동작해야 하지만, 런타임 검증이 필수.

### 6.4 Plugin 배포 방식

초기: 스킬 저장소 내에 `.opencode/plugins/` 디렉토리로 포함
추후: npm 패키지로 분리 배포 고려 (`opencode-plugin-openclone`)

---

## 7. 검증 체크리스트

- [ ] `SKILL.md`가 OpenCode에서 `/openclone`으로 인식되는지 확인
- [ ] Plugin이 활성 클론 페르소나를 시스템 프롬프트에 주입하는지 확인
- [ ] Room 모드가 정상 동작하는지 확인
- [ ] `/openclone stop`으로 상태가 정리되는지 확인
- [ ] 자동 업데이트가 세션 시작 시 실행되는지 확인
- [ ] `setup-opencode`가 idempotent한지 확인
- [ ] Claude Code에서 기존 기능이 회귀하지 않는지 확인
- [ ] Chrome MCP 없는 환경에서 interview/ingest/update가 fallback 동작하는지 확인
