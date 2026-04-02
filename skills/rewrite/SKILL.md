---
name: rewrite
description: "에피소드 재작성(rewrite) 스킬. 설정문서(캐릭터시트, 플롯가이드, 부트스트랩) 변경에 따라 기존 에피소드를 분석하고 재작성한다. '/rewrite', '/revise', '/rewrite {프로젝트명}', '/rewrite EP051', '/rewrite {프로젝트명} EP001-EP010' 으로 실행. '에피소드 재작성', '리바이즈', '리라이트', 'rewrite', 'revise' 모두 트리거. 설정문서가 바뀌어서 에피소드를 다시 써야 할 때, 캐릭터 입체성이 부족해서 대폭 수정이 필요할 때 사용."
---

# Rewrite — 에피소드 재작성 팀 오케스트레이터

설정문서(캐릭터시트·플롯가이드·부트스트랩) 변경에 따라 기존 에피소드를 **분석→재작성→검증**하는 3-Phase 팀.

**프로젝트 독립적**: polish 스킬과 동일한 `novel-config.md`를 읽어 설정문서·가드레일·커스텀 축을 자동 적용한다.
다른 소설 프로젝트에서도 novel-config.md만 작성하면 동일하게 사용 가능.

기존 polish(윤문)와의 차이: 윤문은 문장·표현 레벨의 교정이고, rewrite는 **장면 구성·캐릭터 행동·플롯 비트 레벨의 재작성**이다. 장면을 새로 쓰거나, 캐릭터의 대사·행동을 근본적으로 바꾸거나, 플롯 비트를 재배치하는 작업이다.

---

## 팀 구조: 생성-검증(Producer-Reviewer) 패턴

```
Phase 1 (분석 — Agent 2개 병렬 호출)
         ┌→ [revision-analyst]    ── 설정문서↔에피소드 괴리 분석 ─┐
[에피소드] ┤                                                       ├→ [괴리 보고서 + 재작성 설계서]
         └→ [character-sculptor]  ── 캐릭터 입체성 진단           ─┘

Phase 2 (재작성 — 순차)
[괴리 보고서 + 재작성 설계서] → [episode-rewriter] → [재작성된 에피소드]

Phase 3 (검증 — 순차)
[재작성된 에피소드] → [quality-verifier] → PASS / REWRITE → (REWRITE시 Phase 2 재실행)
```

| 에이전트 | 역할 | Phase |
|---------|------|-------|
| revision-analyst | 설정문서↔에피소드 괴리 분석. 플롯 비트·수치·시간·자금 흐름·커스텀 축 정합성 | 1 (병렬) |
| character-sculptor | 캐릭터 입체성 진단. 경악 방식·비언어·내면·관계망·고유 긴장점 반영도 | 1 (병렬) |
| episode-rewriter | 2개 보고서를 통합하여 에피소드를 재작성. CLAUDE.md 집필 규칙 준수 | 2 (순차) |
| quality-verifier | 재작성 결과를 설정문서·CLAUDE.md 규칙과 대조 검증 (REWRITE 모드) | 3 (순차) |

---

## 실행 방법

```
/rewrite                    ← 기본 프로젝트의 다음 미완료 에피소드부터
/rewrite EP042              ← EP042부터 시작
/rewrite start              ← EP001부터 전체 시작
/rewrite EP027-EP076        ← 특정 범위
/rewrite {프로젝트명}        ← 특정 프로젝트 지정 (예: /rewrite 36억평)
/rewrite {프로젝트명} EP042  ← 특정 프로젝트 + 특정 에피소드
```

---

## 자기 반복 루프 (Self-Loop Protocol)

### 루프 구조

```
LOOP:
  1. rewrite-plan.md에서 다음 미완료 에피소드 결정
  2. 해당 에피소드에 대해 Phase 1→2→3 실행
  3. rewrite-plan.md 업데이트
  4. rewrite-log.md 업데이트 (주요 변경 기록)
  5. 전체 완료 확인
     - 미완료 있음 → LOOP 반복
     - 전체 완료 → "REWRITE_COMPLETE" 출력하고 아래 후처리 실행:
       1. **fix_plan.md 부분 리셋**: rewrite된 에피소드들의 polish 상태를 리셋한다.
          `{WORK_DIR}/fix_plan.md`가 존재하면, rewrite-plan.md에서 `[x]`로 완료된 EP 목록을 추출하여
          fix_plan.md의 해당 EP를 `[ ] — rewrite 후 재윤문 필요`로 변경한다.
          fix_plan.md가 존재하지 않으면 rewrite된 EP 목록으로 새로 생성한다.
          ```
          예시:
          rewrite-plan.md에서 EP027~EP076이 완료됨
          → fix_plan.md에서 EP027~EP076을 [ ]로 리셋
          → 나머지 EP는 기존 상태 유지
          ```
       2. **완료 안내**:
          ```
          rewrite 완료. `/polish`를 실행하면 rewrite된 에피소드의 문장 품질을 교정합니다.
          fix_plan.md에서 EP{시작}~EP{끝}이 재윤문 대상으로 리셋되었습니다.
          rewrite → polish 순서를 권장합니다.
          ```
```

### 반복 지속 규칙

- **멈추지 않는다**: 에피소드 하나를 끝내면 즉시 다음으로 진행한다.
- **사용자 입력을 기다리지 않는다**: 자동으로 계속한다.
- **종료 조건**: rewrite-plan.md의 모든 항목이 `[x]`일 때만 멈춘다.
- **중단 시 재개**: `/rewrite`를 다시 치면 마지막 미완료 에피소드부터 이어간다.

---

## 설정문서 변경 영향도 자동 분석 (Rewrite Trigger)

`/rewrite`를 실행하면 Step 0 이후에 **설정문서 변경 분석**을 자동 수행한다.
이전 rewrite 또는 create 시점의 설정문서와 현재 설정문서를 비교하여 rewrite 대상 EP를 자동 결정한다.

### 변경 분류 기준

| 등급 | 변경 유형 | 예시 | 영향 범위 |
|------|----------|------|----------|
| **CRITICAL** | 캐릭터 핵심 설정 변경 | 주인공 동기/배경/나이/전문 분야, 빌런 정체성, 주요 관계 설정 | 해당 캐릭터 등장 전 EP부터 전체 |
| **CRITICAL** | 수치/타임라인 변경 | 자금 규모, 면적, 시간대, 핵심 날짜 변경 | 해당 수치 첫 언급 EP부터 전체 |
| **CRITICAL** | 플롯 비트 재배치 | 아크 구조 변경, EP별 비트 순서 변경 | 변경된 아크의 전체 EP |
| **MAJOR** | 캐릭터 상세 설정 변경 | 보이스표 변경, 비언어 태그 변경, 호칭 규칙 변경 | 해당 캐릭터 등장 EP |
| **MAJOR** | 가드레일 추가/변경 | 새 가드레일 항목 추가, 기존 항목 수정 | 전체 EP 재검증 |
| **MINOR** | 문구 다듬기 | 설정 문서의 표현 수정, 오탈자, 포맷 변경 | rewrite 불필요 |
| **MINOR** | 보조 참조 변경 | verification 문서 업데이트, 가이드 문구 수정 | rewrite 불필요 |

### 자동 분석 프로세스

```
1. 설정문서 변경 감지:
   - `{DESIGN_DIR}/` 내 설정문서의 변경 확인 (아래 우선순위):
     1. git repo인 경우: `git diff` 사용
     2. git repo가 아닌 경우: `{REWRITE_WORK_DIR}/.design-hashes` 파일과 현재 파일의 SHA256 해시 비교
     3. `.design-hashes` 파일이 없으면: 모든 설정문서를 "변경됨"으로 간주하고 사용자에게 확인
   - 변경 감지 후 `.design-hashes`를 현재 해시로 갱신
   - 변경이 없으면: "설정문서 변경 없음. 수동 rewrite 모드로 진행합니다." 안내 후 Step 1로

2. 변경 내용 분류:
   - 각 변경 항목을 CRITICAL/MAJOR/MINOR로 분류
   - MINOR만 있으면: "설정문서 변경이 경미하여 rewrite가 불필요합니다. 그래도 진행하시겠습니까?" 확인
   - CRITICAL/MAJOR 있으면: 영향 받는 EP 범위를 자동 산출

3. 사용자에게 분석 결과 제시:
   ```
   ## 설정문서 변경 분석

   ### CRITICAL 변경 (N건)
   - {변경 내용}: EP{시작}-EP{끝} 영향

   ### MAJOR 변경 (N건)
   - {변경 내용}: EP{시작}-EP{끝} 영향

   ### MINOR 변경 (N건) — rewrite 불필요
   - {변경 내용}

   ### 권장 rewrite 범위: EP{시작}-EP{끝}
   이 범위로 진행하시겠습니까? (범위 조정 가능)
   ```

4. 사용자 확인 후 rewrite-plan.md를 생성하고 Step 1로 진행
```

---

## 워크플로우 상세

### Step 0: 초기화 (최초 1회)

```
1. 프로젝트 결정
   - $ARGUMENTS에서 프로젝트명 추출 (예: "36억평")
   - 프로젝트명 지정 없으면: 프로젝트 디렉토리 탐색하여 novel-config.md가 있는 디렉토리 자동 감지
   - novel-config.md가 여러 개면: 사용자에게 선택 요청
   - novel-config.md가 없으면: ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md 안내 후 종료

2. novel-config.md 로드 및 **필수 필드 검증 게이트**
   - {PROJECT_DIR}/novel-config.md 읽기
   - **즉시 필수 필드 검증** (하나라도 누락 시 에러 출력 후 종료):
     ```
     필수 필드 체크리스트:
     - [ ] project.target_platform — 플랫폼명
     - [ ] project.episode_dir — 에피소드 저장 디렉토리
     - [ ] project.design_dir — 설계문서 디렉토리 (설정문서 변경 감지에 필요)
     - [ ] 설정문서 매핑.bootstrap — 부트스트랩 경로 (파일 존재 확인)
     - [ ] 설정문서 매핑.character_core — 캐릭터 핵심 경로 (파일 존재 확인)
     - [ ] 설정문서 매핑.character_detail — 캐릭터 상세 경로 (파일 존재 확인)
     - [ ] EP 범위별 플롯 가이드 — 최소 1개 행 존재 (파일 존재 확인)
     ```
     검증 실패 시:
     ```
     ❌ novel-config.md 필수 필드 누락: {누락 필드 목록}
     rewrite 스킬을 실행하려면 위 필드를 채워주세요.
     템플릿: ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md
     ```
   - **target_platform 허용 집합 검증**:
     - `project.target_platform`은 문피아, 네이버시리즈, 카카오페이지, 리디, 조아라, 노벨피아 중 하나여야 한다
     - rewrite 스킬은 novel-config.md의 canonical 값만 사용하며 자동 매핑하지 않는다
     - 비지원 값이면 에러 출력 후 종료:
       ```
       ❌ 허용되지 않는 target_platform: {현재값}
       novel-config.md의 project.target_platform을 허용 플랫폼명으로 수정해주세요.
       ```
   - 설정문서 경로 매핑 추출 → {CONFIG} 변수로 저장
   - 보존 가드레일 추출 → {GUARD_RAILS}
   - Rewrite 전용 설정 추출:
     - character_dialogue_dna 경로 → {DIALOGUE_DNA} (없으면 character_detail로 대체)
     - Rewrite 보존 가드레일 → {REWRITE_GUARD_RAILS} (polish 가드레일에 추가)
     - rewrite_work_dir → {REWRITE_WORK_DIR} (없으면 work_dir 사용)
   - 커스텀 축 존재 여부 확인 → {CUSTOM_AXES}

3. 필수 문서 로드:
   - {WRITING_RULES} — 집필 규칙 바이블 (novel-config.md의 writing_rules, 기본값: CLAUDE.md)
   - {CHAR_CORE} — 캐릭터 핵심 문서 (novel-config.md의 character_core)
   - {CHAR_DETAIL} — 캐릭터 상세 문서 (novel-config.md의 character_detail)
   - {PLOT_MACRO} — 전체 플롯 구조 (novel-config.md의 plot_macro)
   - {BOOTSTRAP} — 세계관·기술·경제 규칙 (novel-config.md의 bootstrap)

4. 작업 디렉토리 설정
   - {REWRITE_WORK_DIR}에서 rewrite-plan.md 읽기 또는 생성
   - rewrite-log.md 읽기 또는 생성

5. $ARGUMENTS 파싱
   - "EP{NNN}" → 해당 에피소드부터 시작
   - "EP{NNN}-EP{MMM}" → 범위 지정
   - "start" → EP001부터
   - 없음 → rewrite-plan.md의 다음 미완료부터
```

### Step 0.5: EP 범위별 설정문서 결정 (매 에피소드)

대상 에피소드 번호에 따라 참조할 설정문서를 결정한다.
novel-config.md의 **"EP 범위별 설정문서"** 테이블에서 해당 EP의 플롯 문서와 캐릭터 시트를 선택.

```
{PLOT_DOC} = novel-config.md의 EP 범위 매핑에서 해당 EP에 맞는 플롯 가이드 경로
             세부 플롯 가이드 열에 경로가 있고 파일이 존재하면 세부 가이드 우선
{CHAR_CORE} = novel-config.md의 공통 문서 중 character_core 경로
{CHAR_DETAIL} = EP 범위별 설정문서 테이블의 세부 캐릭터 시트 열에 경로가 있고 파일이 존재하면
                세부 캐릭터 시트 우선, 없으면 공통 문서의 character_detail 경로
{BOOTSTRAP} = novel-config.md의 공통 문서 중 bootstrap 경로
{VERIFY} = novel-config.md의 보조 참조 중 verification 경로 (있을 경우)
{DIALOGUE_DNA} = novel-config.md의 Rewrite 전용 설정 중 character_dialogue_dna 경로 (없으면 {CHAR_DETAIL})
{WRITING_RULES} = novel-config.md의 writing_rules 경로 (기본값: CLAUDE.md)
{EPISODE_DIR} = novel-config.md의 episode_dir 경로
{GUARD_RAILS} = novel-config.md의 보존 가드레일 목록
{REWRITE_GUARD_RAILS} = novel-config.md의 Rewrite 보존 가드레일 (있을 경우)
{CUSTOM_AXES} = novel-config.md의 커스텀 진단 축 섹션 (있을 경우)
```

이하 Phase 1~3의 에이전트 프롬프트에서 위 변수를 실제 값으로 치환하여 전달한다.

### Step 1: 대상 에피소드 결정

rewrite-plan.md에서 최상단 미완료(`[ ]`) 항목을 가져온다.

### Step 2: Phase 1 — 병렬 분석 (Fan-out)

**Agent 도구를 사용하여 2개 에이전트를 한 번의 메시지에 동시 호출한다.**

```
[동시 호출 — 하나의 응답에 2개 Agent 도구 호출]

Agent("revision-analyst"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md)에 대해 설정문서↔에피소드 괴리 분석 수행.

           필수 읽기:
           - {EPISODE_DIR}/ep{NNN}.md (대상 에피소드 전문)
           - {EPISODE_DIR}/ep{NNN-1}.md (직전 에피소드 — 연속성 확인)
           - {WRITING_RULES} (집필 규칙)
           - {PLOT_DOC} (해당 EP 비트·확정 수치 확인)
           - {BOOTSTRAP} (세계관·수치 규칙)
           - {CHAR_CORE} (캐릭터 핵심 정의·생년·고유 설정)

           ★ 커스텀 축 검증 (novel-config.md에 정의된 경우):
           {CUSTOM_AXES}
           커스텀 축이 정의되어 있으면 해당 레이어를 함께 검증한다.

           ★ 시간·수치 교차검증 (3단계 프로토콜 — 필수):
           [1단계] 대상 EP + 직전 EP에서 모든 시간 마커를 추출.
           명시적 날짜, 상대적 시간, 경과 시간을 모두 수집.
           상대적 시간은 해당 장면의 명시적 날짜 기준으로 절대 날짜로 변환.
           [2단계] 같은 사건/약속/예고에 대한 시간 참조가 인접 화에서 모순되는지 확인.
           [3단계] 같은 대상의 수치(프로젝트 핵심 수치)가 EP 간 일관되는지 확인.
           EP 내 산술(나눗셈·곱셈·합산)이 맞는지도 검증.
           불일치 발견 시 [TIMELINE] 또는 [NUMBER] 태그 + 등급 부여.

           Revision Analysis Report 형식으로 출력.
           괴리 항목별로 [CRITICAL/MAJOR/MINOR] 등급 부여."

Agent("character-sculptor"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md)에 대해 캐릭터 입체성 진단 수행.

           필수 읽기:
           - {EPISODE_DIR}/ep{NNN}.md (대상 에피소드 전문)
           - {EPISODE_DIR}/ep{NNN-1}.md (직전 에피소드 — 비언어 반복 대조용)
           - {CHAR_CORE} (캐릭터 핵심 정의)
           - {CHAR_DETAIL} (보이스표·비언어·경악·관계 변화표·호칭 규칙)
           - {WRITING_RULES} (캐릭터 보이스·비언어 태그)
           - {REWRITE_WORK_DIR}/alive-tracker.md (관계 아크 현황)
           - {DIALOGUE_DNA} (대화 DNA 가이드)

           출력 경로: {REWRITE_WORK_DIR}/_workspace/06_character-sculptor_report_EP{NNN}.md

           ★ 강화된 진단 기준 (7개 차원):
           1. 비언어가 서술 흐름에 녹아있는가? (태그 직결 vs 인과순서)
           2. 비언어 강도가 사건의 무게에 비례하는가? (5단계: 미약~압도)
           3. 화 내 동일 비언어 2회 초과 여부
           4. 직전 EP과 동일 표현 연속 반복 여부
           5. 같은 감탄사 화당 1회 초과 여부
           6. 3명+ 동시 반응 시 전원 동일 감정 여부
           7. 대화 톤이 캐릭터 아크 단계에 맞는가
           8. 조연 대사에 개인적 맥락(경험·기억·이해관계)이 실려있는가

           ★ 대화 DNA 진단 (⑦번 차원):
           9. 대사의 사고 패턴이 캐릭터 DNA와 정합하는가?
           10. 같은 캐릭터의 같은 사고경로가 화당 3회 이상 반복되지 않는가?
           11. 대사의 화자를 다른 캐릭터로 교체했을 때 어색한가? (교체 불가성)
           12. 정보 전달 대사에도 개인 맥락이 실려있는가? (기능 대사 검출)

           Character Sculpture Report 형식으로 출력.
           등장 캐릭터별로 입체성 점수 부여."
```

### Step 3: Phase 2 — 재작성 실행 (Fan-in → 순차)

2개 보고서가 돌아오면 통합하여 episode-rewriter 호출:

```
Agent("episode-rewriter"):
  prompt: "아래 2개 분석 보고서를 기반으로 {EPISODE_DIR}/ep{NNN}.md를 재작성.

           필수 읽기:
           - {EPISODE_DIR}/ep{NNN}.md (현재 에피소드 전문)
           - {EPISODE_DIR}/ep{NNN-1}.md (직전 에피소드 — 연속성 + 비언어/대화 반복 확인)
           - {EPISODE_DIR}/ep{NNN+1}.md (다음 에피소드 — 연속성, 있을 경우)
           - {WRITING_RULES} (집필 규칙 바이블)
           - {PLOT_DOC} (해당 EP 비트·확정 수치)
           - {CHAR_CORE} (캐릭터 핵심)
           - {CHAR_DETAIL} (보이스표·비언어·경악·관계 변화표)
           - {BOOTSTRAP} (세계관)
           - {REWRITE_WORK_DIR}/alive-tracker.md (관계 아크 현황)
           - {DIALOGUE_DNA} (대화 DNA 가이드)

           ★ Phase 1 분석 보고서 (파일로 읽기):
           - {REWRITE_WORK_DIR}/_workspace/06_character-sculptor_report_EP{NNN}.md

           [Revision Analysis Report]
           {revision-analyst 출력}

           [Character Sculpture Report — 위 파일의 내용도 참조]
           {character-sculptor 출력}

           ★ 캐릭터 입체성 서술 원칙 (필수 준수):
           1. 비언어를 태그 직결로 쓰지 않는다. 인과의 순서(자극→신체→행동→대사)를 따른다.
           2. 비언어 강도를 사건 무게에 비례시킨다 (미약~압도 5단계).
           3. 같은 캐릭터의 동일 비언어는 화당 2회 이하. 직전 EP에서 사용된 표현은 변주한다.
           4. 조연 대사에 개인적 맥락(경험·기억·이해관계) 1줄 이상.
           5. 3명+ 동시 반응 시 최소 1명은 다른 감정으로 반응.
           6. 대화 톤을 캐릭터 아크 단계에 맞춘다 (alive-tracker 참조).

           ★ 대화 DNA 원칙 (필수 준수 — {DIALOGUE_DNA} 참조):
           7. 대사 작성 3단계: ①누구의 DNA인가 → ②어떤 상황 유형인가 → ③교체 불가 검증.
           8. 같은 캐릭터의 같은 사고경로 화당 2회까지. 3회째는 다른 경로로 변주.
           9. 정보 전달 대사에도 캐릭터 DNA를 통과시킨다. 기능 대사 금지.
           10. 대사 전후에 행동·시선·공간 서술을 넣는다. 대본식 대화 나열 금지.
           11. 침묵 묘사는 질감이 있어야 한다. "침묵했다" 대신 공간·감각·행동으로 표현.

           ★ 보존 가드레일 (절대 훼손 금지):
           {GUARD_RAILS}
           {REWRITE_GUARD_RAILS}

           {WRITING_RULES}의 에피소드 집필 프로세스를 따라 재작성.
           재작성 결과를 {EPISODE_DIR}/ep{NNN}.md에 저장.
           Rewrite Execution Report 형식으로 출력."
```

### Step 4: Phase 3 — 검증 (순차)

```
Agent("quality-verifier"):
  prompt: "EP{NNN}의 재작성 결과를 검증하라. (REWRITE 모드)

           필수 읽기:
           - {EPISODE_DIR}/ep{NNN}.md (재작성된 에피소드 전문)
           - {EPISODE_DIR}/ep{NNN-1}.md (직전 에피소드)
           - {WRITING_RULES} (집필 규칙)
           - {PLOT_DOC} (해당 EP 비트·확정 수치)
           - {CHAR_CORE} (캐릭터 핵심)
           - {CHAR_DETAIL} (보이스표·비언어·경악·관계 변화표)

           [Rewrite Execution Report]
           {episode-rewriter 출력}

           7개 카테고리 QA 실행:
           1. ★수치일관성 (3단계 교차검증): 모든 시간 마커 추출 → 상대시간을 절대날짜로 변환
              → 인접 EP와 같은 사건의 시간 참조 대조 → 모든 수치 인접 EP 대조
              → EP 내 산술 검증. [TIMELINE]/[NUMBER] 태그 잔존 시 FAIL.
           2. 시간순서
           3. 캐릭터보이스
           4. 문체/금지표현
           5. 개연성
           6. 훅/페이싱
           7. ★커스텀 축 정합성 — novel-config.md에 정의된 커스텀 축 최종 확인

           ★ 보존 가드레일 확인:
           {GUARD_RAILS}
           {REWRITE_GUARD_RAILS}

           캐릭터 입체성 최종 확인.

           ★ 캐릭터 서술 품질 검증 (필수):
           1. 비언어 태그 직결 3건+ → REWRITE
           2. 동일 비언어 화당 3회+ 또는 3화 연속 반복 → REWRITE
           3. 전원 동일 감정 반응 장면 1건+ → REWRITE
           4. 직전 EP와 비언어 반복을 grep으로 대조한다.

           Verification Report 형식으로 PASS/REWRITE 판정."
```

- **PASS** → Step 5로
- **REWRITE** → 수정 지시를 포함하여 Phase 2 재실행 (최대 2회)

### Step 5: 기록 및 다음 에피소드로

1. rewrite-plan.md 업데이트
   ```
   - [x] EP{NNN} | REWRITTEN | 괴리:{C}c/{M}m | 캐릭터:{점수} | 훅:{유형}/{강도}
   ```
2. rewrite-log.md에 주요 변경 기록
3. **5화 완료마다**: 수치 연속성·캐릭터 아크·복선 추적 교차검증
4. **즉시 다음 에피소드로 → Step 1로 돌아간다**

---

## rewrite-plan.md 형식

```markdown
# Rewrite Plan — 에피소드 재작성 계획

## 대상: EP{NNN}~EP{MMM} (설정문서 반영)
## 현황: 0/{총수} 완료

### EP별 상태
- [ ] EP{NNN} | {EP 요약}
- [ ] EP{NNN+1} | {EP 요약}
...
- [x] EP{MMM} | REWRITTEN | 괴리:{C}c/{M}m | 캐릭터:{점수} | 훅:{유형}/★{N}
```

## rewrite-log.md 형식

```markdown
# Rewrite Log

## EP{NNN}
- [CRITICAL] {변경 내용}
- [MAJOR] {변경 내용}
- [MINOR] {변경 내용}
- 캐릭터 변경: {상세}
```

---

## 보존 가드레일

재작성 시에도 절대 훼손하지 않는 요소. **프로젝트별 novel-config.md에서 로드한다.**

기본 가드레일 (novel-config.md의 "보존 가드레일" 섹션):
- `{GUARD_RAILS}` — 프로젝트가 정의한 보존 항목

Rewrite 전용 가드레일 (novel-config.md의 "Rewrite 보존 가드레일" 섹션):
- `{REWRITE_GUARD_RAILS}` — 경악 방식 교차오염 금지, 주인공 동기 표현 규칙 등

두 가드레일을 합산하여 모든 Phase에서 적용한다.

---

## 캐릭터 입체성 서술 원칙 (방법론)

비언어 태그 표는 **팔레트의 기본색**이다. 이를 서술에 녹이는 원칙:

### 원칙 1: 인과의 순서
자극 → 신체 반응(무의식) → 의식적 행동 → 대사/침묵.

### 원칙 2: 감정 강도별 변주 (5단계)
| 강도 | 처리 원칙 |
|------|----------|
| 미약(1) | 동작이 시작되다 멈춤. 시선만 이동 |
| 주목(2) | 고유 동작의 축소판 |
| 놀람(3) | 고유 동작의 완전한 실행 |
| 충격(4) | 고유 동작 + 후속 행동 |
| 압도(5) | 고유 동작조차 나오지 않음. 대비로 충격 전달 |

### 원칙 3: 반복 방지
- **화 내**: 같은 캐릭터의 동일 비언어 2회 이하. 동일 감탄사 1회.
- **화 간**: 직전 2화에서 사용한 비언어를 grep으로 확인. 3화 연속 금지.

### 원칙 4: 대화에 생애를 싣기
조연의 대사에는 **자기 경험·기억·이해관계**가 한 줄 실려야 한다.

### 원칙 5: 캐릭터 아크 반영
직전 화까지의 관계 변화를 확인하고, 현재 EP의 대화 톤을 아크 단계에 맞춘다.
캐릭터 상세 문서({CHAR_DETAIL})의 관계 변화표와 alive-tracker.md를 참조한다.

### 원칙 6: 전원 동일 반응 금지
같은 사건에 3명+ 동시 반응 시, 최소 1명은 다른 감정으로 반응.

---

## 재작성 후 워크플로우 연결

### rewrite → polish 재실행 권장

rewrite 완료 후, 재작성된 에피소드는 **12축 윤문(polish)을 거치지 않은 상태**다.
rewrite의 quality-verifier는 6카테고리 QA만 수행하므로, polish의 12축+ALIVE 4축 진단과 범위가 다르다.

**REWRITE_COMPLETE 출력 후 다음 안내를 반드시 포함한다:**

```
## 다음 단계: 윤문(polish) 재실행 권장

재작성된 에피소드는 장면·캐릭터·플롯 레벨의 QA를 통과했지만,
문장·표현 레벨의 12축 윤문은 아직 수행되지 않았습니다.

재작성 과정에서 새로운 SILENCE 패턴, UNIFORM 반복, TRANS(번역체) 등이
발생할 수 있으므로, 아래 명령으로 윤문을 재실행하세요:

/polish {프로젝트명} EP{시작}

재작성 범위: EP{시작}~EP{끝}
```

### 재시도 횟수 정책

| 스킬 | 최대 재시도 | 근거 |
|------|-----------|------|
| **create** | 2회 | 창작은 0에서 시작하므로 수정 폭이 크고, 1회로 해결되기 어려움 |
| **polish** | 1회 | 문장 레벨 교정은 범위가 좁아 1회 추가로 대부분 해결 |
| **rewrite** | 2회 | 장면 재구성은 수정 폭이 커서 create와 동일 기준 적용 |

---

## 대화 DNA 원칙 (방법론)

비언어가 **몸**의 팔레트라면, 대화 DNA는 **말**의 팔레트다.
상세 가이드: `${CLAUDE_PLUGIN_ROOT}/skills/rewrite/references/character-dialogue-dna.md` (방법론)
캐릭터별 DNA 프로필: novel-config.md의 `character_dialogue_dna` 경로 (프로젝트 데이터)

### 대사 작성 3단계
1. **누가 말하는가** — 캐릭터 DNA 확인. 이 사람은 정보를 어떻게 처리하나?
2. **지금 어떤 상황인가** — 좋은 소식? 위기? 갈등? → 상황별 사고 경로 선택
3. **교체 불가 검증** — 이 대사의 화자를 다른 캐릭터로 바꿔도 자연스러운가? 그렇다면 DNA가 부족하다.

### 변주 규칙
- 같은 캐릭터의 같은 사고경로 화당 2회까지. 3회째는 다른 경로.
- 정보 전달 대사에도 캐릭터 DNA를 통과시킨다. 기능 대사 금지.
- 대사 전후에 행동·시선·공간 서술을 넣는다. 대본식 나열 금지.
- 침묵 묘사는 질감이 있어야 한다. "침묵했다" 대신 공간·감각·행동으로 표현.
