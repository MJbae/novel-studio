# 설정문서 매핑 — Rewrite 에이전트용

rewrite 에이전트가 참조하는 설정문서는 프로젝트별 `novel-config.md`에 정의한다.
polish 스킬과 동일한 설정 파일을 공유하며, rewrite 전용 키가 추가된다.

공통 매핑 가이드: `${CLAUDE_PLUGIN_ROOT}/skills/polish/references/design-document-map.md` 참조.
프로젝트 설정 작성법: `${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md` 참조.

---

## Rewrite 전용 설정 키

novel-config.md의 공통 문서(character_core, character_detail, bootstrap)에 더해,
rewrite 스킬은 아래 전용 키를 추가로 사용한다.

### 1. writing_rules (공통 문서 섹션)

| 문서 키 | 기본값 | 용도 | 참조 에이전트 |
|---------|--------|------|-------------|
| writing_rules | CLAUDE.md | 집필 규칙 바이블 — 문체, 시점, 에피소드 구성 원칙 | revision-analyst, episode-rewriter, quality-verifier |

novel-config.md에 이 키가 없으면 프로젝트 루트의 `CLAUDE.md`를 기본값으로 사용한다.
프로젝트별로 집필 규칙이 다르면 별도 파일 경로를 지정한다.

### 2. character_dialogue_dna (Rewrite 전용 설정 섹션)

| 문서 키 | 기본값 | 용도 | 참조 에이전트 |
|---------|--------|------|-------------|
| character_dialogue_dna | (character_detail로 대체) | 캐릭터별 대화 DNA — 사고 패턴, 정보 처리, 설득 방식, 상황별 변주 | character-sculptor, episode-rewriter |

이 파일이 없으면 character_detail만으로 대화 진단을 수행한다.
대화 DNA를 별도로 관리하면 캐릭터 대사의 교체 불가성 검증이 강화된다.

### 3. rewrite_work_dir (Rewrite 전용 설정 섹션)

| 키 | 기본값 | 용도 |
|----|--------|------|
| rewrite_work_dir | (polish의 work_dir 사용) | rewrite-plan.md, rewrite-log.md 저장 위치 |

polish와 rewrite의 작업 파일을 분리하고 싶으면 별도 경로를 지정한다.
같은 디렉토리를 공유해도 파일명이 다르므로(fix_plan.md vs rewrite-plan.md) 충돌하지 않는다.

### 4. Rewrite 보존 가드레일 (Rewrite 전용 설정 섹션)

polish의 "보존 가드레일"에 추가되는 rewrite 전용 가드레일.
재작성 시 캐릭터 경악 방식 교차오염 방지, 주인공 동기 표현 규칙 등을 정의한다.

---

## 에이전트별 문서 참조

### revision-analyst
| 참조 목적 | 문서 키 |
|----------|---------|
| 플롯 비트·확정 수치 | plot_by_ep (EP 범위별) |
| 세계관·수치 규칙 | bootstrap |
| 캐릭터 핵심 정의 | character_core |
| 집필 규칙 | writing_rules |
| 커스텀 축 | custom_axis (있을 경우) |

### character-sculptor
| 참조 목적 | 문서 키 |
|----------|---------|
| 보이스표·비언어·관계 변화 | character_detail |
| 캐릭터 핵심 정의 | character_core |
| 대화 DNA | character_dialogue_dna (없으면 character_detail) |
| 관계 아크 현황 | alive-tracker.md (work_dir) |

### episode-rewriter
| 참조 목적 | 문서 키 |
|----------|---------|
| 집필 규칙 | writing_rules |
| 플롯 비트·확정 수치 | plot_by_ep |
| 캐릭터 핵심 | character_core |
| 보이스표·비언어 | character_detail |
| 세계관 | bootstrap |
| 대화 DNA | character_dialogue_dna |
| 관계 아크 현황 | alive-tracker.md (work_dir) |

### quality-verifier (REWRITE 모드)
| 참조 목적 | 문서 키 |
|----------|---------|
| 집필 규칙 | writing_rules |
| 플롯 비트·확정 수치 | plot_by_ep |
| 캐릭터 핵심 | character_core |
| 보이스표·비언어 | character_detail |
