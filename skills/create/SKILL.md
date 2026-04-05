---
name: create
description: "웹소설 에피소드 창작 오케스트레이터. 설정문서(캐릭터시트·플롯가이드·부트스트랩)에 따라 에피소드를 순차 창작한다. '/create', '/create {프로젝트명}', '/create EP051', '/create {프로젝트명} EP001-EP010', '에피소드 창작', '에피소드 집필', '에피소드 작성', '새 에피소드', '본문 작성', '에피소드 쓰기' 으로 실행. novel-config.md의 설정문서 매핑과 가드레일을 적용하여 캐릭터 입체성, 수치 일관성, 훅 강도, 개연성을 보장한다."
---

# 에피소드 창작 오케스트레이터

설정문서(캐릭터시트·플롯가이드·부트스트랩)에 따라 에피소드를 순차 창작하는 4-에이전트 파이프라인.

## 아키텍처

```
Phase 1 (병렬)              Phase 2 (순차)          Phase 3 (순차)
┌──────────────────┐       ┌──────────────┐       ┌──────────────┐
│ episode-architect│──┐    │  episode-    │       │  quality-    │
│ (설계도 추출)      │  ├──▶│  creator     │──────▶│  verifier    │
└──────────────────┘  │    │  (본문 집필)   │       │  (7축 검증)   │
┌──────────────────┐  │    └──────────────┘       └──────┬───────┘
│ continuity-bridge│──┘          ◀── REWRITE (max 2) ───┘
│ (연속성 수집)      │                     │
└──────────────────┘             PASS ───▶ 다음 EP
```

**실행 모드**: 서브 에이전트 (파이프라인 패턴, lint/revise와 동일)
- 이유: 각 Phase가 이전 Phase 출력에 순차 의존, 에이전트 간 실시간 통신 불필요

## 전제 조건

1. **novel-config.md** 존재 (프로젝트 디렉토리 내)
2. **설정문서** 존재: 부트스트랩, 캐릭터시트(core+detail+dialogue DNA), 플롯가이드
3. **에피소드 디렉토리** 존재 (`episode_dir` in novel-config.md)

novel-config.md가 없으면 에러를 출력하고 종료한다.

## 인자 파싱

| 입력 형태 | 해석 |
|----------|------|
| `/create` | 현재 디렉토리에서 novel-config.md 탐색, 마지막 EP 다음부터 |
| `/create 36억평` | 36억평 프로젝트, 마지막 EP 다음부터 |
| `/create EP051` | 현재 프로젝트, EP051만 |
| `/create 36억평 EP051` | 36억평 프로젝트, EP051만 |
| `/create 36억평 EP051-EP060` | 36억평 프로젝트, EP051~EP060 범위 |

## 실행 흐름

### Step 0: 초기화

0. **novel-config.md 필수 필드 검증 게이트**:
   novel-config.md를 로드한 직후, 아래 필수 필드가 모두 존재하고 비어있지 않은지 검증한다.
   하나라도 누락되면 **에러를 출력하고 즉시 종료**한다.

   ```
   필수 필드 체크리스트:
   - [ ] project.target_platform — 플랫폼명
   - [ ] project.episode_dir — 에피소드 저장 디렉토리
   - [ ] project.work_dir — 작업 디렉토리
   - [ ] project.design_dir — 설계문서 디렉토리
   - [ ] 설정문서 매핑.bootstrap — 부트스트랩 경로 (파일 존재 확인)
   - [ ] 설정문서 매핑.character_core — 캐릭터 핵심 경로 (파일 존재 확인)
   - [ ] 설정문서 매핑.character_detail — 캐릭터 상세 경로 (파일 존재 확인)
   - [ ] EP 범위별 플롯 가이드 — 최소 1개 행 존재 (파일 존재 확인)
   ```

   검증 실패 시 출력:
   ```
   ❌ novel-config.md 필수 필드 누락:
   - {누락 필드 목록}

   create 스킬을 실행하려면 위 필드를 채워주세요.
   템플릿: ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md
   ```

0.5 **target_platform 허용 집합 검증 게이트**:
   - novel-config.md의 `project.target_platform`은 아래 canonical name 중 하나여야 한다:
     - 문피아, 네이버시리즈, 카카오페이지, 리디, 조아라, 노벨피아
   - create 스킬은 novel-config.md에 저장된 canonical 값만 사용한다. 별칭 정규화나 자동 치환은 수행하지 않는다.
   - 비지원 값이면 에러를 출력하고 즉시 종료한다.

   ```
   ❌ 허용되지 않는 target_platform: {현재값}

   허용 플랫폼:
   - 문피아
   - 네이버시리즈
   - 카카오페이지
   - 리디
   - 조아라
   - 노벨피아

   novel-config.md의 project.target_platform을 허용 플랫폼명으로 수정한 뒤 다시 실행해주세요.
   ```

1. novel-config.md를 파싱한다:
   ```
   {CONFIG}       ← novel-config.md 전체
   {TARGET_PLATFORM} ← config.target_platform
   {DESIGN_DIR}   ← config.design_dir
   {EPISODE_DIR}  ← config.episode_dir
   {WORK_DIR}     ← config.work_dir
   {GUARD_RAILS}  ← config.guard_rails + config.create_guard_rails (있으면)
   {CUSTOM_AXES}  ← config.custom_axes (있으면)
   {CREATE_CFG}   ← config.create 설정 (있으면, 없으면 기본값)
   ```

2. 기본값 (`[create]` 섹션 없을 때):
   ```
   draft_chars: 6000-10000           # Phase 2 초안 목표 (넉넉하게 작성)
   final_chars: 5000-8000            # Step 2.5 트리밍 후 최종 목표
   dialogue_ratio: 40-60%
   max_scenes: 4
   hook_targets:
     opening_intensity: 4
     ending_intensity: 5
   continuity_lookback: 2
   ```

   > **분량 전략: "넘치게 쓰고 줄이기"**
   > episode-creator는 `draft_chars`(6000-10000자) 목표로 초안을 작성한다.
   > Step 2.5에서 오케스트레이터가 `final_chars`(5000-8000자)로 트리밍한다.
   > 이유: 부족해서 살을 붙이면 품질이 낮아진다. 충분히 쓰고 불필요한 부분을 삭제해야 밀도가 높아진다.

3. EP 범위 결정:
   - 지정 EP가 있으면 해당 범위 사용
   - 없으면 `{EPISODE_DIR}`에서 마지막 EP 번호를 찾고, 다음 번호부터 시작
   - 종료 EP를 지정하지 않으면 **1화만** 창작 (무한 루프 방지)

4. EP 범위에 따른 설정문서 매핑:
   - novel-config.md의 `ep_range_table`에서 해당 EP가 속하는 act 확인
   - **EP 범위 중첩 검출**: ep_range_table의 모든 행을 순회하여 범위가 겹치는 행이 있는지 확인. 중첩 발견 시 경고를 출력하고 사용자에게 확인
   - 해당 act의 플롯가이드 결정:
     - 세부 플롯 가이드 열에 경로가 있고 파일이 존재하면 → `{PLOT_DOC}` = 세부 플롯 가이드
     - 없으면 → `{PLOT_DOC}` = 큰 설계 플롯 가이드
   - 해당 act의 캐릭터시트(detail) 결정:
     - 세부 캐릭터 시트 열에 경로가 있고 파일이 존재하면 → `{CHAR_DETAIL_EP}` = 세부 캐릭터 시트
     - 없으면 → `{CHAR_DETAIL_EP}` = 공통 character_detail
   - **세부 설계 문서 유무 확인**: 해당 EP 범위에 세부 플롯 가이드(작은 설계 산출물)가 없으면 경고 출력:
     ```
     ⚠️ EP{NNN}이 속하는 범위에 세부 플롯 가이드가 없습니다.
     큰 설계 플롯 가이드만으로 EP별 비트를 추출하므로 설계도 품질이 저하될 수 있습니다.
     작은 설계(design-small)를 먼저 실행하는 것을 권장합니다.
     그래도 진행하시겠습니까?
     ```

5. `{WORK_DIR}/_workspace/` 디렉토리 확인 (없으면 생성)

6. `{WORK_DIR}/create-plan.md` 생성/갱신:
   ```markdown
   # 에피소드 창작 계획
   ## 대상: EP{start}-EP{end}
   - [ ] EP{NNN} — 미착수
   ...
   ```

### Step 1: Phase 1 — 에피소드 설계 (병렬)

두 에이전트를 **병렬로** (한 메시지에 Agent 2회 호출) 실행한다. `run_in_background: true`로 배경 실행한다.

> ⚠️ **Phase 1 대기 방법 — 절대 금지/필수 준수**:
> - **금지**: TaskOutput 사용 (팀 네임스페이스 충돌으로 "No task found" 오류 발생)
> - **금지**: Bash sleep 폴링 (`sleep N && ls` 반복 금지 — 불필요한 대기 + 승인 요청 유발)
> - **필수**: 배경 에이전트는 완료 시 시스템이 **자동으로 알려준다**. 두 에이전트 모두 완료 알림을 받은 후 Phase 2로 진행한다.

> ⚠️ **에이전트 공통 경로 제한 지시** (episode-architect, continuity-bridge 프롬프트에 반드시 포함):
> "아래 명시된 파일 경로만 읽어라. design/ 디렉토리를 Glob으로 탐색하거나,
> 명시되지 않은 파일을 스스로 찾아 읽으려 시도하지 마라.
> 필요한 모든 파일 경로는 이 프롬프트에 포함되어 있다."

**Agent 1: episode-architect**
- subagent_type: `general-purpose`
- `run_in_background: true`
- 프롬프트에 포함할 정보:
  - 에이전트 정의 파일 경로 → 읽으라고 지시
  - EP 번호
  - 읽을 설정문서 경로 (novel-config.md에 정의된 경로를 사용):
    - 부트스트랩: `{CONFIG.bootstrap}` (novel-config.md의 bootstrap 경로)
    - 플롯가이드: `{PLOT_DOC}` (Step 0.4에서 ep_range_table로 결정된 경로)
    - 캐릭터시트 core: `{CONFIG.character_core}` (novel-config.md의 character_core 경로)
    - 캐릭터시트 detail: `{CHAR_DETAIL_EP}` (ep_range_table 세부 캐릭터 시트 우선, 없으면 공통 character_detail)
  - novel-config.md의 가드레일, 커스텀 축
  - **목표 분량 명시** (반드시 포함):
    ```
    blueprint 헤더의 '목표 분량' 항목은 반드시 아래 값을 사용한다:
    - 초안 목표(draft_chars): {CREATE_CFG.draft_chars}자
    플랫폼 상식(문피아 3,000~4,000자 등) 기반의 자체 추론을 절대 사용하지 말 것.
    novel-config.md의 [create] 섹션 값이 없을 경우 기본값 8,000~10,000자 사용.
    ```
  - 경로 제한 지시 (위 공통 지시 포함)
  - **허용 경로 화이트리스트** (프롬프트에 반드시 포함):
    ```
    이 작업에서 Read할 수 있는 파일은 아래 목록이 전부다.
    목록에 없는 파일은 어떤 이유로든 Read하지 마라:
    0. ${CLAUDE_PLUGIN_ROOT}/agents/episode-architect.md (자신의 에이전트 정의)
    1. {CONFIG.bootstrap}
    2. {PLOT_DOC}
    3. {CONFIG.character_core}
    4. {CHAR_DETAIL_EP}
    5. novel-config.md
    위 6개 파일 외의 Read 시도는 경로 제한 위반이다.
    ```
  - 출력 경로: `{WORK_DIR}/_workspace/01_episode-architect_blueprint_EP{NNN}.md`

**Agent 2: continuity-bridge**
- subagent_type: `general-purpose`
- `run_in_background: true`
- 프롬프트에 포함할 정보:
  - 에이전트 정의 파일 경로 → 읽으라고 지시
  - EP 번호
  - 읽을 문서 경로:
    - 이전 N화: `{EPISODE_DIR}/ep{N-2}.md`, `{EPISODE_DIR}/ep{N-1}.md`
    - alive-tracker: `{WORK_DIR}/alive-tracker.md` (존재 시)
    - delta-tracker: `{DESIGN_DIR}/delta_tracker_*.md` (존재 시)
    - verification: `{DESIGN_DIR}/verification_*.md` (존재 시)
  - EP001 창작 시: 이전 에피소드 대신 부트스트랩 초기 상태를 요약하라고 지시
  - 경로 제한 지시 (위 공통 지시 포함)
  - 출력 경로: `{WORK_DIR}/_workspace/02_continuity-bridge_report_EP{NNN}.md`

### Step 2: Phase 2 — 에피소드 집필 (순차)

Phase 1 완료 후 episode-creator를 실행한다.

**Agent 3: episode-creator**
- subagent_type: `general-purpose`
- 프롬프트에 포함할 정보:
  - 에이전트 정의 파일 경로 → 읽으라고 지시
  - 읽을 문서:
    - `{WORK_DIR}/_workspace/01_episode-architect_blueprint_EP{NNN}.md`
    - `{WORK_DIR}/_workspace/02_continuity-bridge_report_EP{NNN}.md`
    - 캐릭터시트 core + detail + dialogue DNA
    - novel-config.md의 가드레일
  - `{CREATE_CFG}` 설정 (목표 글자수, 대화 비율 등)
  - 출력 경로: `{EPISODE_DIR}/ep{NNN}.md`
  - 상세 집필 원칙이 필요하면 `references/creation-principles.md`를 읽으라고 지시
  - **Bash 금지 지시**: "집필 완료 후 Bash/python3 스크립트를 실행하지 마라. 글자수·대화비율·장면수 집계는 quality-verifier가 수행한다. 파일 저장 후 셀프체크(있었다/고 있었다/것이었다 Grep 카운트)를 수행한 뒤 종료하라."
  - 경로 제한 지시 (Step 1의 공통 지시 포함)

**재작성 모드** (REWRITE 후 재실행 시):
- 추가 프롬프트:
  - `{WORK_DIR}/_workspace/04_quality-verifier_verdict_EP{NNN}.md`를 읽으라고 지시
  - 기존 초안 `{EPISODE_DIR}/ep{NNN}.md`를 읽으라고 지시
  - "수정 지시에 따라 해당 부분만 수정하라. 전체 재작성 금지."

### Step 2.5: 트리밍 게이트 (오케스트레이터 직접 수행)

Phase 2 완료 후, Phase 3 진입 전에 오케스트레이터가 초안을 트리밍한다.

**전략: "넘치게 쓰고 줄이기"**
episode-creator는 6000-10000자 초안을 작성한다. 오케스트레이터가 5000-8000자로 트리밍한다.
부족해서 살을 붙이는 것보다 충분히 쓰고 줄이는 것이 품질이 높다.

**측정 기준**: `wc -m`으로 파일 전체 글자수를 측정한다 (마크다운 헤더·메타데이터 포함).

1. **측정**: Bash `wc -m {EPISODE_DIR}/ep{NNN}.md` → 글자수 확인
2. **판정**: 최종 목표 범위(기본 final_chars: 5000-8000) 대비 확인

   - **상한 초과** (8000자 초과 — 일반적 케이스):
     - episode-creator를 트리밍 모드로 재호출한다
     - 프롬프트에 다음을 포함:
       ```
       트리밍 모드:
       - 현재 글자수: {N}자 (wc -m 기준, 헤더 포함)
       - 목표 범위: {FINAL_MIN}~{FINAL_MAX}자
       - 초과분: {N - FINAL_MAX}자
       - 지시: 아래 삭제 우선순위에 따라 불필요한 부분을 제거하여 {FINAL_MAX}자 이내로 만들어라.
         삭제 우선순위:
         1순위: 서술자의 상황 정리/요약 문장 (메타서술)
         2순위: 같은 감정을 다른 비유로 반복 표현한 문장
         3순위: 플롯 비트에 기여하지 않는 배경 묘사
         4순위: 이미 행동으로 보여준 감정을 내면 독백으로 재확인하는 문장
         5순위: 과도한 감각 묘사 (핵심 장면이 아닌 전환 구간)
         주의: 플롯 비트, 대사, 클리프행어, 핵심 감각 묘사는 보존. 엔딩은 대사/행동으로 유지.
       ```
     - 트리밍 후 다시 글자수를 측정한다. 1회까지 재시도, 이후 Phase 3으로 진행

   - **범위 내** (5000-8000자): Phase 3으로 진행

   - **하한 미달** (5000자 미만 — 초안이 부족한 비정상 케이스):
     - episode-creator를 즉시 재호출한다 (Phase 3 진입 전 보정)
     - 프롬프트에 다음을 포함:
       ```
       글자수 부족 보정 모드:
       - 현재 글자수: {N}자
       - 목표 하한: {FINAL_MIN}자
       - 지시: 기존 장면을 확장하거나 부족한 묘사/대화를 보강하여 {FINAL_MIN}자 이상으로 만들어라.
               새로운 플롯 비트를 추가하지 말고, 기존 장면의 밀도를 높여라.
       ```
     - 보정 후 다시 글자수를 측정한다. 2회까지 시도, 이후 Phase 3으로 진행

### Step 3: Phase 3 — 품질 검증 (순차)

**Agent 4: quality-verifier** (CREATE 모드)
- subagent_type: `general-purpose`
- 프롬프트에 포함할 정보:
  - 에이전트 정의 파일 경로 → 읽으라고 지시
  - CREATE 모드로 동작하라고 지시 (창작 품질 검증 — 아래 8축 적용):
    ```
    CREATE 모드 검증 8축 (구조적 정합성 중심 — polish의 문체·표현 레벨 교정과 구분):
    1. PLOT_BEAT — 설계도의 플롯 비트가 에피소드에 모두 반영되었는가
    2. TIMELINE — 시간 마커·날짜·나이가 설정문서 및 이전 에피소드와 일치하는가
    3. NUMBER — 핵심 수치(면적·자금·수확량 등)가 설정문서와 일치하는가
    4. GUARDRAIL — novel-config.md의 보존 가드레일을 위반하지 않았는가
    5. CONTINUITY — continuity-bridge 보고서의 미해결 복선/관계 상태가 반영되었는가
    6. HOOK — 오프닝/엔딩 훅 강도가 설정된 목표치 이상인가
    7. CHAR_VOICE — 등장인물의 대사가 캐릭터 보이스표/대화 DNA와 일관되는가
    8. CUSTOM — novel-config.md의 커스텀 축 위반이 없는가
    ```
    > **polish와의 관계**: create의 quality-verifier는 구조적 정합성(플롯·수치·가드레일)을 검증한다.
    > polish는 문장 품질(금지표현·침묵패턴·번역투·모바일 가독성·캐릭터 생동감)을 교정한다.
    > 따라서 create PASS 에피소드도 polish를 거쳐야 최종 품질에 도달한다.
  - **재검증 지시** (REWRITE 후 재실행 시 반드시 포함):
    "기존 verdict 파일이 있더라도 절대 읽지 마라. 에피소드 원문 파일을 가장 먼저 읽고,
     모든 검증축을 처음부터 수행하라. 결과를 verdict 파일에 새로 덮어써라."
  - 경로 제한 지시 (Step 1의 공통 지시 포함)
  - 읽을 문서:
    - `{EPISODE_DIR}/ep{NNN}.md` (창작된 에피소드)
    - `{WORK_DIR}/_workspace/01_episode-architect_blueprint_EP{NNN}.md`
    - `{WORK_DIR}/_workspace/02_continuity-bridge_report_EP{NNN}.md`
    - 설정문서 (부트스트랩, 캐릭터시트, 플롯가이드)
    - novel-config.md (가드레일, 수치 검증 우선순위, 커스텀 축)
  - 출력 경로: `{WORK_DIR}/_workspace/04_quality-verifier_verdict_EP{NNN}.md`

### Step 4: 판정 분기

verdict 파일을 읽어 판정을 확인한다.

**PASS 판정 시 — 오케스트레이터 교차검증 (필수):**
> 이중 안전장치: quality-verifier가 직접 Grep 카운트를 수행하더라도,
> 오케스트레이터는 독립적으로 재확인한다. 두 레이어가 동일 결과를 내야 PASS가 확정된다.

verdict가 PASS인 경우, 오케스트레이터가 핵심 가드레일을 직접 교차검증한다:
1. Grep `있었다` in `{EPISODE_DIR}/ep{NNN}.md` → 5회 이하 확인
2. Grep `고 있었다` in `{EPISODE_DIR}/ep{NNN}.md` → 3회 이하 확인
3. Bash: `wc -m {EPISODE_DIR}/ep{NNN}.md` → 목표 글자수 범위(기본 final_chars: 5000-8000) 확인

3개 중 하나라도 위반 시, 위반 유형에 따라 분기한다:

**경로 A — 오케스트레이터 직접 Edit (MINOR 위반)**:
다음 조건을 **모두** 충족하면 오케스트레이터가 직접 수정한다:
- 글자수 위반이 아님 (글자수 미달/초과는 경로 B)
- 위반 항목의 **초과분이 3건 이하** (예: '있었다' 7회 → 2건 초과)
- 위반이 **단일 표현의 텍스트 치환**으로 해소 가능

직접 Edit 절차:
1. Grep으로 위반 표현의 전체 출현 위치를 파악한다 (output_mode: "content", -n: true)
2. 초과분에 해당하는 출현을 선택한다 (에피소드 후반부 → 전반부 순으로 선택)
3. 각 출현에 대해 **전후 2문장을 Read로 확인**하여 문맥을 파악한 후 대체한다
4. 대체 전략 (보수적 — 시제 보존 우선):
   - '있었다' → '~였다' (과거 유지, 가장 안전) 또는 '~이다' (현재 문맥인 경우만)
   - '고 있었다' → '~했다' (단순 과거 전환, 시제 오류 최소)
   - '것이었다' → '~였다', '~인 셈이었다'
   - **시제를 과거→현재로 변경해야 하는 경우는 경로 B로 분류**한다 (문맥 의존적 수정)
5. 수정 후 다시 Grep으로 카운트가 상한 이내인지 확인한다
6. 확인 완료 시 PASS 경로를 계속한다 (verdict 파일에 "[오케스트레이터 직접 수정: {항목} {원래값}→{수정값}]" 추기)

**경로 B — REWRITE 절차 (MAJOR 위반)**:
다음 중 하나라도 해당하면 기존 REWRITE 절차를 따른다:
- 글자수 위반 (구조적 수정 필요)
- 초과분이 4건 이상 (광범위 수정 필요)
- 시제 변경이 필요한 대체 (문맥 의존적 수정)
- 단순 치환으로 해소 불가 (문맥 의존적 수정 필요)

REWRITE 절차:
- "⚠️ quality-verifier PASS 판정이 원문과 불일치. 오케스트레이터가 REWRITE로 재판정합니다."
- 기존 REWRITE 절차(Step 4 후반부)로 진행한다

교차검증 통과 시:
1. `create-plan.md`에서 해당 EP를 `[x]`로 갱신하고 요약 추가
2. 다음 EP로 이동 (Step 1부터)

**REWRITE:**
1. 재시도 횟수를 확인한다 (해당 EP의 누적 REWRITE 횟수)
2. **기존 verdict 파일을 삭제한다**: Bash `rm -f {WORK_DIR}/_workspace/04_quality-verifier_verdict_EP{NNN}.md`
   > verdict 파일이 남아있으면 quality-verifier가 재검증 없이 이전 결과를 재사용한다. 반드시 삭제한다.
3. **2회 미만**: Step 2로 돌아간다 (재작성 모드, 수정 지시 포함)
4. **2회 이상**: 사용자에게 알리고 해당 EP를 `[△]`(부분통과)로 표시, 다음 EP로 이동

### Step 5: 자기 루프

대상 범위의 모든 EP가 처리될 때까지 Step 1~4를 반복한다.

완료 시:
1. `create-plan.md` 최종 갱신
2. **`.design-hashes` 베이스라인 생성**: `{WORK_DIR}/.design-hashes` 파일을 생성/갱신한다.
   설정문서(부트스트랩, 캐릭터시트, 플롯가이드 등 `{DESIGN_DIR}/` 내 모든 설정문서)의 SHA256 해시를 기록한다.
   이 파일은 rewrite 스킬이 설정문서 변경을 감지하는 베이스라인으로 사용된다.
   ```
   .design-hashes 형식:
   {파일경로}\t{SHA256해시}\t{생성시각}
   ```
   > **목적**: rewrite 최초 실행 시 "모든 설정문서가 변경됨"으로 오탐하는 것을 방지.
   > create가 완료된 시점의 설정문서 상태가 베이스라인이 된다.
3. 결과 요약 출력:
   ```
   ## 창작 완료 요약
   - 총 에피소드: N화
   - PASS: X화
   - 부분통과(△): Y화
   - REWRITE 발생: Z회

   ### 다음 단계
   - 문장 품질 향상: `/polish` — 12축 진단으로 문체·훅·수치·캐릭터 생동감을 교정
   - 설정 변경 반영: `/rewrite` — 설정문서 변경 시 영향받는 에피소드를 재작성
   - ⚠️ 설정을 변경할 예정이라면 `/rewrite` → `/polish` 순서로 실행하세요
   ```

## 에러 핸들링

| 에러 유형 | 전략 |
|----------|------|
| 에이전트 실패 | 1회 재시도 → 실패 시 해당 EP를 `[!]`로 표시하고 다음으로 |
| 설정문서 누락 | 사용자에게 경고 후 가용한 문서로 진행. 플롯가이드 누락 시 종료 |
| 이전 EP 없음 (EP001) | continuity-bridge가 부트스트랩 초기 상태 요약 |
| novel-config.md 누락 | 에러 출력 후 종료 |
| 가드레일 충돌 | episode-creator가 `[가드레일우회]` 태그 → auditor가 해당 부분 별도 평가 |
| Phase 1 한쪽만 실패 | 성공한 보고서로 Phase 2 진행, 누락 명시 |

## novel-config.md 확장 — [create] 섹션

novel-config.md에 다음 섹션을 추가하면 창작 설정을 커스터마이즈할 수 있다. 없으면 기본값을 사용한다.

```markdown
## create 설정
draft_chars: 6000-10000          # Phase 2 초안 목표 (넉넉하게 작성)
final_chars: 5000-8000           # Step 2.5 트리밍 후 최종 목표
dialogue_ratio: 40-60%           # 대화 비율 (기본: 40-60%)
max_scenes: 4                    # 최대 장면 수 (기본: 4)
hook_targets:
  opening_intensity: 4           # 1-5, 오프닝 훅 강도 (기본: 4)
  ending_intensity: 5            # 1-5, 엔딩 클리프행어 강도 (기본: 5)
continuity_lookback: 2           # 이전 참조 화수 (기본: 2)
point_scenes_per_ep: 2-3         # 포인트 장면 수 (기본: 2-3)
dead_zone_threshold: 3500        # 포인트 장면 없는 최대 허용 글자수 (기본: 3500)

## create 사전 예방 (shift-left)
# create는 lint와 달리 문제를 사후 검출이 아닌 사전 예방한다.
# 아래 항목을 episode-architect의 설계도에 사전 포함하여 episode-creator가 준수하게 한다:
timeline_lock: true              # 에피소드별 절대 날짜/나이 사전 확정 (기본: true)
voice_quickref: true             # 등장인물별 보이스 퀵레프 카드 사전 추출 (기본: true)
nonverbal_memory: true           # 이전 2화 비언어 반복 메모리 수집 (기본: true)
echo_dialogue_filter: true       # 메아리 대화 방지 규칙 적용 (기본: true)
cider_gogumo_balance: true       # 사이다/고구마 밸런스 추적 (기본: true)

## create 가드레일
# lint 가드레일에 추가로 적용할 창작 전용 가드레일
# 예시:
# - 주인공이 1화에 전문 분야 외 지식을 시연하면 안 됨
# - 회귀 사실을 3막 이전에 타인에게 노출하면 안 됨
```

## 테스트 시나리오

### 정상 흐름
1. `/create 36억평 EP051` 실행
2. novel-config.md 파싱 → EP051은 Act2 범위
3. Phase 1: episode-architect(Act2 플롯가이드 + 부트스트랩 + 캐릭터시트) ‖ continuity-bridge(EP049, EP050) 병렬
4. Phase 2: episode-creator가 설계도+연속성+캐릭터 DNA로 집필 → `36억평/episodes/ep051.md`
5. Phase 3: quality-verifier가 7축 검증 → PASS
6. create-plan.md 갱신 `[x] EP051`, EP052로 이동

### 에러 흐름 — REWRITE
1. Phase 3에서 REWRITE 판정 (TIMELINE CRITICAL: 날짜 불일치)
2. verdict의 수정 지시를 episode-creator에 전달
3. episode-creator가 해당 부분만 수정
4. Phase 3 재실행 → PASS
5. create-plan.md 갱신, 다음 EP로 이동

### 에러 흐름 — 첫 에피소드
1. `/create 36억평 EP001` 실행
2. continuity-bridge: 이전 에피소드 없음 → 부트스트랩 초기 상태 요약
3. 이후 정상 흐름과 동일

### 에러 흐름 — 범위 창작
1. `/create 36억평 EP051-EP055` 실행
2. EP051 PASS → EP052 REWRITE(1차) → REWRITE(2차) → 부분통과(△) → EP053 PASS → ...
3. 최종 요약: PASS 4화, 부분통과 1화
