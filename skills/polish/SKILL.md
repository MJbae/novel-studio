---
name: polish
description: "웹소설 윤문 스킬. 6명의 전문 에이전트를 병렬/순차 조합으로 운용하여 에피소드를 순차 윤문. '/polish', '/lint', '/polish start', '/polish EP051', '/polish {프로젝트명}', '/polish {프로젝트명} EP051' 으로 실행. 프로젝트별 novel-config.md를 읽어 설정문서·가드레일·커스텀 축을 자동 적용. '윤문', '린트', 'polish', 'lint' 모두 이 스킬을 트리거한다."
---

# Polish — 웹소설 윤문 팀 오케스트레이터

6명의 전문 에이전트를 **Fan-out/Fan-in 병렬 구조**로 운용하여
에피소드를 순차적으로 12축(+ALIVE 4축, +프로젝트 커스텀 축) 정밀 윤문하는 자기 반복(self-loop) 스킬.

**프로젝트 독립적**: `novel-config.md`를 읽어 설정문서·가드레일·커스텀 축을 자동 적용한다.
다른 소설 프로젝트에서도 novel-config.md만 작성하면 동일하게 사용 가능.

---

## 팀 구성

```
Phase 1 (병렬 진단 — Agent 도구로 4개 동시 호출)
         ┌→ [rule-checker]       ── BANNED, VOICE, TITLE, SILENCE, TRANS ─┐
[에피소드] ├→ [story-analyst]       ── SCENE, LOGIC, UNIFORM (+커스텀)     ─┤
         ├→ [platform-optimizer]  ── HOOK, OPENING, MOBILE, SUMMARY       ─┼→ [진단 통합]
         └→ [alive-enhancer]      ── ALIVE-1~4 (메아리,침묵,긴장점,거리감) ─┘

Phase 2 (순차 교정)
[진단 통합] → [revision-executor] → [교정된 에피소드]

Phase 3 (순차 검증)
[교정된 에피소드] → [revision-reviewer] → PASS / REVISE → (REVISE시 Phase 2 재실행)
```

| 에이전트 | 역할 | 담당 축 | Phase |
|---------|------|---------|-------|
| rule-checker | 규칙 위반 진단 | BANNED, VOICE, TITLE, SILENCE, TRANS | 1 (병렬) |
| story-analyst | 서사·논리 분석 | SCENE, LOGIC, UNIFORM (+커스텀 축) | 1 (병렬) |
| platform-optimizer | 플랫폼 최적화 | HOOK, OPENING, MOBILE, SUMMARY | 1 (병렬) |
| alive-enhancer | 캐릭터 생동감 | ALIVE-1~4 | 1 (병렬) |
| revision-executor | 교정 실행 | 전체 통합 교정 | 2 (순차) |
| revision-reviewer | 교정 검증 | 과교정·신규오류·가드레일 | 3 (순차) |

---

## 실행 방법

```
/polish                      ← 기본 프로젝트의 다음 미완료 에피소드부터
/polish EP051                ← EP051부터 시작
/polish start                ← EP001부터 전체 시작
/polish {프로젝트명}          ← 특정 프로젝트 지정 (예: /polish 36억평)
/polish {프로젝트명} EP051    ← 특정 프로젝트 + 특정 에피소드
```

---

## 자기 반복 루프 (Self-Loop Protocol)

**이 스킬의 핵심: 에피소드 하나를 완료한 뒤 스스로 다음 에피소드로 넘어간다.**
외부 루프가 필요 없다. Claude 자체의 에이전틱 루프로 반복한다.

### 루프 구조

```
LOOP:
  1. fix_plan.md에서 다음 미완료 에피소드 결정
  2. 해당 에피소드에 대해 Phase 1→2→3 실행
  3. fix_plan.md 업데이트
  4. learnings.md 업데이트 (새 패턴 발견 시만)
  5. alive-tracker.md 업데이트 (alive-enhancer 보고서 기반)
  6. 전체 완료 확인
     - 미완료 있음 → LOOP 반복 (다음 에피소드)
     - 전체 완료 → "REVISION_COMPLETE" 출력하고 종료
       완료 시 안내: "설정문서를 변경할 예정이라면 `/rewrite` 실행 후 `/polish`를 다시 실행하세요."
```

### 반복 지속 규칙

- **멈추지 않는다**: 에피소드 하나를 끝내면 즉시 다음으로 진행한다.
- **사용자 입력을 기다리지 않는다**: 자동으로 계속한다.
- **종료 조건**: fix_plan.md의 모든 항목이 `[x]`일 때만 멈춘다.
- **중단 시 재개**: `/polish`를 다시 치면 마지막 미완료 에피소드부터 이어간다.

---

## 워크플로우 상세

### Step 0: 초기화 (최초 1회)

```
1. 프로젝트 결정
   - $ARGUMENTS에서 프로젝트명 추출 (예: "36억평")
   - 프로젝트명 지정 없으면: 프로젝트 디렉토리 탐색하여 novel-config.md가 있는 디렉토리 자동 감지
   - novel-config.md가 여러 개면: 사용자에게 선택 요청
   - novel-config.md가 없으면: references/project-config-template.md 안내 후 종료

2. novel-config.md 로드 및 **필수 필드 검증 게이트**
   - {PROJECT_DIR}/novel-config.md 읽기
   - **즉시 필수 필드 검증** (하나라도 누락 시 에러 출력 후 종료):
     ```
     필수 필드 체크리스트:
     - [ ] project.target_platform — 플랫폼명
     - [ ] project.episode_dir — 에피소드 저장 디렉토리
     - [ ] project.work_dir — 작업 디렉토리
     - [ ] 설정문서 매핑.bootstrap — 부트스트랩 경로 (파일 존재 확인)
     - [ ] 설정문서 매핑.character_core — 캐릭터 핵심 경로 (파일 존재 확인)
     - [ ] 설정문서 매핑.character_detail — 캐릭터 상세 경로 (파일 존재 확인)
     - [ ] EP 범위별 플롯 가이드 — 최소 1개 행 존재 (파일 존재 확인)
     ```
     검증 실패 시:
     ```
     ❌ novel-config.md 필수 필드 누락: {누락 필드 목록}
     polish 스킬을 실행하려면 위 필드를 채워주세요.
     템플릿: ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md
     ```
   - **target_platform 허용 집합 검증**:
     - `project.target_platform`은 문피아, 네이버시리즈, 카카오페이지, 리디, 조아라, 노벨피아 중 하나여야 한다
     - polish 스킬은 novel-config.md의 canonical 값만 사용하며 자동 매핑하지 않는다
     - 비지원 값이면 에러 출력 후 종료:
       ```
       ❌ 허용되지 않는 target_platform: {현재값}
       novel-config.md의 project.target_platform을 허용 플랫폼명으로 수정해주세요.
       ```
   - 설정문서 경로 매핑 추출 → {CONFIG} 변수로 저장
   - 보존 가드레일 추출
   - 커스텀 축 존재 여부 확인
   - 침묵 패턴 예외 캐릭터 확인

3. 작업 디렉토리 설정
   - novel-config.md의 work_dir에서 fix_plan.md 읽기
   - 없으면 생성 (EP001~마지막 화)
   - learnings.md 읽기 (없으면 생성)
   - alive-tracker.md 읽기 (없으면 생성)

4. $ARGUMENTS 확인
   - "EP{NNN}" → 해당 에피소드부터 시작
   - "start" → EP001부터
   - 없음 → fix_plan.md의 다음 미완료부터
```

### Step 0.5: EP 범위별 설정문서 결정 (매 에피소드)

대상 에피소드 번호에 따라 참조할 설정문서를 결정한다.
novel-config.md의 **"EP 범위별 설정문서"** 테이블에서 해당 EP의 플롯 문서와 캐릭터 시트를 선택.

```
{PLOT_DOC} = novel-config.md의 EP 범위 매핑에서 해당 EP에 맞는 플롯 가이드 경로
             세부 플롯 가이드 열에 경로가 있고 파일이 존재하면 세부 가이드 우선
             (범위 중첩 발견 시 경고 출력 후 첫 번째 매칭 행 사용)
{CHAR_CORE} = novel-config.md의 공통 문서 중 character_core 경로
{CHAR_DETAIL} = EP 범위별 설정문서 테이블의 세부 캐릭터 시트 열에 경로가 있고 파일이 존재하면
                세부 캐릭터 시트 우선, 없으면 공통 문서의 character_detail 경로
{BOOTSTRAP} = novel-config.md의 공통 문서 중 bootstrap 경로
{GUIDE} = novel-config.md의 보조 참조 중 web_novel_guide 경로
{VERIFY} = novel-config.md의 보조 참조 중 verification 경로
{PLOT_MACRO} = novel-config.md의 보조 참조 중 plot_macro 경로
{EPISODE_DIR} = novel-config.md의 episode_dir 경로
{GUARD_RAILS} = novel-config.md의 보존 가드레일 목록
{CUSTOM_AXES} = novel-config.md의 커스텀 진단 축 섹션 (있을 경우)
{SILENCE_EXCEPT} = novel-config.md의 침묵 패턴 예외 캐릭터 (있을 경우)
```

이하 Phase 1~3의 에이전트 프롬프트에서 위 변수를 실제 값으로 치환하여 전달한다.

### Step 1: 대상 에피소드 결정

fix_plan.md에서 최상단 미완료(`[ ]`) 항목을 가져온다.
**SKIP 절대 금지.** 모든 에피소드 전문 정독.

### Step 1.5: 직전 에피소드 로드 (매 화 필수)

대상 에피소드(EP{NNN})의 **직전 2화**를 반드시 읽는다:
- `{EPISODE_DIR}/ep{NNN-2}.md` — 직전전 화 전문 (존재할 경우)
- `{EPISODE_DIR}/ep{NNN-1}.md` — 직전 화 전문

**검증 목적**:
1. **내용 중복**: 동일 사건·대화·묘사가 직전 2화에 겹치는지 확인
2. **개연성 연속성**: 직전 화 결말 → 현재 화 도입이 자연스러운지 확인
3. **시간·수치 교차검증** (아래 프로토콜 참조)
4. **캐릭터 위치**: 직전 화에서 퇴장한 인물이 현재 화에 설명 없이 등장하거나, 반대의 경우
5. **감정 흐름**: 직전 화에서 분노한 인물이 현재 화에서 갑자기 태평한 등 급변 없는지

### 시간·수치 교차검증 프로토콜 (story-analyst 필수)

에피소드 간 **시간 참조**와 **수치**의 정합성을 구조적으로 검증한다.
단순히 "수치 확인"이 아니라, 아래 3단계를 반드시 수행한다.

**1단계: 타임라인 추출**
대상 에피소드와 직전 2화에서 **모든 시간 마커**를 추출한다:
- 명시적 날짜: "10월 15일", "1993년 3월"
- 상대적 시간: "다음 주", "내일", "이번 주 토요일", "연말까지", "1월에"
- 계절/시기: "봄 이앙", "수확 후", "겨울"
- 경과 시간: "2주가 걸렸다", "6개월째"
상대적 시간은 해당 장면의 명시적 날짜를 기준으로 **절대 날짜로 변환**한다.

**2단계: 타임라인 교차대조**
같은 사건/약속/예고에 대한 시간 참조가 직전 2화에서 모순되는지 확인한다.

**3단계: 수치 교차대조**
같은 대상에 대한 수치가 에피소드 간 일관되는지 확인한다:
```yaml
검증_대상:
  면적: ha, 에이커, 평 — 같은 대상의 면적이 화마다 다르면 [NUMBER] CRITICAL
  자금: 달러, 원 — 같은 거래/빚/매출이 화마다 다르면 [NUMBER] CRITICAL
  수확량: t/ha, 톤, 부셸 — 같은 수확의 수치가 화마다 다르면 [NUMBER] CRITICAL
  인원: 가구 수, 명 — 같은 모임/조합의 인원이 화마다 다르면 [NUMBER] MAJOR
  산술검증: 에피소드 내 계산이 맞는지 [NUMBER] MAJOR

교차대조_방법:
  1. 대상 EP에서 모든 숫자+단위를 추출
  2. 직전 2화에서 같은 대상의 숫자를 찾아 대조
  3. 불일치 발견 시 [NUMBER] 또는 [TIMELINE] 태그 + 등급 부여
  4. 어느 쪽이 정본인지 판단이 어려우면 양쪽 모두 보고
```

EP001은 직전 화 없이, EP002는 직전 1화만으로 진행한다.
직전 에피소드 정보는 Phase 1의 **모든 에이전트에게** 전달한다.

### Step 2: Phase 1 — 병렬 진단 (Fan-out)

**Agent 도구를 사용하여 4개 에이전트를 한 번의 메시지에 동시 호출한다.**

**보고서 저장 규칙**: 각 에이전트의 진단 보고서를 `_workspace/`에 파일로 저장한다.
Phase 2의 revision-executor가 보고서 파일을 직접 Read하여 컨텍스트 전달 부담을 줄인다.

| 에이전트 | 보고서 저장 경로 |
|---------|----------------|
| rule-checker | `{WORK_DIR}/_workspace/07_rule-checker_report_ep{NNN}.md` |
| story-analyst | `{WORK_DIR}/_workspace/07_story-analyst_report_ep{NNN}.md` |
| platform-optimizer | `{WORK_DIR}/_workspace/07_platform-optimizer_report_ep{NNN}.md` |
| alive-enhancer | `{WORK_DIR}/_workspace/07_alive-enhancer_report_ep{NNN}.md` |

```
[동시 호출 — 하나의 응답에 4개 Agent 도구 호출]

Agent("rule-checker"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md)에 대해 규칙 검증 5축 진단 수행.
           직전 2화({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md)도 전문 정독하여
           호칭·금지표현·침묵 패턴이 직전 화와 겹치거나 충돌하는지 확인.

           ★ 설정문서 로드 (novel-config.md 기준):
           - {CHAR_DETAIL} — 보이스표(종결어미·길이·패턴), 호칭 규칙표(화자×청자)
           - {CHAR_CORE} — 주인공 감정 표현 규칙 (크랙 문법 등)

           침묵 패턴 예외 캐릭터: {SILENCE_EXCEPT}

           에이전트 정의(rule-checker.md)의 축별 체크리스트와 등급 기준을 따라
           ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/12-axes.md 축1~5 참조하여 진단.
           각 축의 grep 패턴으로 기계적 검출 후 정독으로 오탐 제거.
           VOICE는 설정문서 보이스표, TITLE은 호칭 규칙표와 1:1 대조.
           Rule Check Report 형식으로 출력."

Agent("story-analyst"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md)에 대해 SCENE + LOGIC + UNIFORM 3축 진단 수행.
           ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/12-axes.md 축6·7·9 기준 참조.
           직전 2화({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md)도 전문 정독.

           ★ 설정문서 로드 (novel-config.md 기준 — 수치·시간·설정 정본):
           - {PLOT_DOC} — 해당 EP의 확정 수치 (면적·자금·수확량·인원·시간대)
           - {BOOTSTRAP} — 매크로 수치 정본
           - {CHAR_CORE} — 캐릭터 나이(생년 기준), 고유 설정 정본
           - {VERIFY} — 검증 완료 수치 (있을 경우)

           ★★★ LOGIC 축은 반드시 서브카테고리를 독립 수행 ★★★

           [TIMELINE] 시간 정합성 — 3단계 프로토콜 (생략 불가):
             1단계: 대상 EP + 직전 2화에서 모든 시간 마커 추출. 상대 시간은 절대 날짜로 변환.
             2단계: 같은 사건/약속에 대한 시간 참조가 EP 간 모순되는지 확인.
             3단계: 추출 결과를 표로 출력. '문제 없음'이라도 표는 반드시 출력.

           [NUMBER] 수치 정합성:
             대상 EP에서 모든 숫자+단위 추출 → 직전 2화에서 같은 대상 grep 대조.
             EP 내 산술(나눗셈·곱셈·합산) 직접 계산 검증.
             추출 결과를 표로 출력.

           [PLAUSIBILITY] 개연성:
             캐릭터 반응 모순, 비용 누락, 시대 고증 위반 검출.

           {CUSTOM_AXES_PROMPT}

           에이전트 정의(story-analyst.md)의 출력 형식을 정확히 따라
           TIMELINE 추출 표, NUMBER 추출 표, 교차대조 결과를 반드시 포함한
           Story Analysis Report를 출력하라."

Agent("platform-optimizer"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md)에 대해 플랫폼 최적화 4축+특화 진단 수행.
           직전 2화({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md)도 읽기.
           직전 EP 마지막 500자 → 현재 EP 첫 1,000자 연결 자연스러움 확인.

           ★ 설정문서 로드 (novel-config.md 기준):
           - {PLOT_DOC} — 해당 EP의 훅 유형·감정강도·비트 구조
           - {PLOT_MACRO} — 핵심 전환 포인트 (있을 경우)
           - {GUIDE} — 모바일 최적화 원칙 (있을 경우)

           ★ 플랫폼별 기준:
           - novel-config.md의 target_platform은 문피아, 네이버시리즈, 카카오페이지, 리디, 조아라, 노벨피아 중 하나여야 함
           - novel-config.md의 target_platform에 해당하는 플랫폼 가이드 참조
           - _workspace/platform-guide-{platform}.md가 있으면 해당 파일의 기준 적용

           에이전트 정의(platform-optimizer.md)의 축별 체크리스트를 따라
           ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/12-axes.md 축8·10·11·12 참조하여 진단.
           HOOK·OPENING·MOBILE·SUMMARY 각각 정량 측정(훅 강도, 대화 비율 등).
           HOOK 유형은 설정문서의 해당 EP 훅 유형과 대조하여 평가.
           핵심 전환 포인트 해당 시 훅 강도 4+ 필수.
           Platform Optimization Report 형식으로 출력."

Agent("alive-enhancer"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md)에 대해 캐릭터 생동감 4축 진단 수행.
           직전 2화({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md)도 읽어
           동일 비언어 표현이 직전 화에서 반복 사용되는지 확인.

           ★ 설정문서 로드 (novel-config.md 기준):
           - {CHAR_CORE} — 조연별 고유 긴장점, 주요 관계 곡선
           - {CHAR_DETAIL} — 비언어 태그 팔레트, 호칭 규칙, 관계 변화표

           alive-tracker.md에서 조연별 마지막 관계 이벤트 확인.
           Alive Enhancement Report 형식으로 출력."
```

**커스텀 축 프롬프트 삽입 ({CUSTOM_AXES_PROMPT})**:
novel-config.md에 "커스텀 진단 축" 섹션이 있으면, story-analyst 프롬프트에 해당 축의
정본·탐지 키워드·판별 기준을 추가한다. 예:
```
[PASTLIFE] 전생 설정 정합성 (novel-config.md 커스텀 축):
  레이어1: 'grep "전생"' → 키워드 문장에서 정본과 모순 검출.
  레이어2: 탐지_키워드 grep → 서술자의 능력 부정합 검출.
  판별: 서술자 사실 진술=VIOLATION / 전략적 은폐=ALLOWED.
  novel-config.md의 PASTLIFE 섹션 전문을 참조하라.
```
커스텀 축이 없으면 이 부분은 생략한다.

### Step 3: Phase 2 — 교정 실행 (Fan-in → 순차)

4개 보고서가 돌아오면 통합하여 revision-executor 호출:

```
Agent("revision-executor"):
  prompt: "아래 4개 진단 보고서를 통합하여 {EPISODE_DIR}/ep{NNN}.md를 교정.
           교정 전 직전 2화를 반드시 읽어 중복 내용 제거 및 연속성 보장을 확인하라.

           ★ 설정문서 로드 (novel-config.md 기준):
           - {PLOT_DOC} — 해당 EP의 확정 수치·비트
           - {BOOTSTRAP} — 매크로 수치 정본
           - {CHAR_CORE} — 주인공 고유 설정, 나이(생년 기준)
           - {CHAR_DETAIL} — 보이스표·호칭표·비언어 태그

           ★ Phase 1 진단 보고서 (파일로 읽기):
           - {WORK_DIR}/_workspace/07_rule-checker_report_ep{NNN}.md
           - {WORK_DIR}/_workspace/07_story-analyst_report_ep{NNN}.md
           - {WORK_DIR}/_workspace/07_platform-optimizer_report_ep{NNN}.md
           - {WORK_DIR}/_workspace/07_alive-enhancer_report_ep{NNN}.md

           위 4개 파일을 Read한 후, 아래 보고서도 참조하라:

           [Rule Check Report — 위 파일 참조]
           {rule-checker 출력 요약}

           [Story Analysis Report — 위 파일 참조]
           {story-analyst 출력 요약}

           [Platform Optimization Report — 위 파일 참조]
           {platform-optimizer 출력 요약}

           [Alive Enhancement Report — 위 파일 참조]
           {alive-enhancer 출력 요약}

           ★★★ 교정 우선순위 (에이전트 정의 참조) ★★★
           1순위: [TIMELINE] CRITICAL, [NUMBER] CRITICAL — 직전 2화 grep 후 정본 확인하여 수정
           2순위: BANNED, LOGIC(개연성), 커스텀 축 CRITICAL
           3순위: TITLE, VOICE, SILENCE, TRANS
           4순위: HOOK, OPENING, SCENE, ALIVE, SUMMARY, MOBILE, UNIFORM

           시간·수치 교정 시 특별 규칙:
           - 정본 확인: novel-config.md의 수치 교차검증 정본 우선순위를 따른다
           - 파급 확인: 수치 변경 시 같은 EP 내 참조 문장도 함께 수정
           - 산술 재검증: 수정 후 관련 계산이 맞는지 직접 확인
           - 새 시간 마커 도입 시 직전 2화 타임라인과 모순 없는지 확인

           보존 가드레일: {GUARD_RAILS}
           [ADJACENT] 태그 항목은 우선 교정 대상이다.
           교정 후 fix_plan.md를 업데이트하라."
```

### Step 4: Phase 3 — 교정 검증 (순차)

```
Agent("revision-reviewer"):
  prompt: "EP{NNN}의 교정 결과를 검증하라.
           교정된 {EPISODE_DIR}/ep{NNN}.md를 전문 정독하라.
           직전 2화도 반드시 참조하라.

           ★ 설정문서 로드 (novel-config.md 기준):
           - {PLOT_DOC} — 해당 EP의 확정 수치·비트
           - {BOOTSTRAP} — 매크로 수치 정본
           - {CHAR_CORE} — 주인공 고유 설정 정본
           - {CHAR_DETAIL} — 보이스표·비언어 확인

           [Revision Execution Report]
           {revision-executor 출력}

           ★★★ 7대 검증 수행 ★★★

           1. 과교정: 자연스러운 표현을 기계적으로 바꿔 어색해진 곳
           2. 신규 오류: 교정으로 인해 새로 발생한 수치 불일치, 호칭 오류, 문맥 단절
           3. ★TIMELINE/NUMBER 최종 검증:
              교정된 EP에서 시간 마커와 숫자+단위를 재추출하여 직전 2화와 교차대조.
              [TIMELINE] CRITICAL 잔존 → REVISE
              [NUMBER] CRITICAL 잔존 → REVISE
              교정으로 인한 신규 시간·수치 불일치 → REVISE
              산술 오류 잔존 → REVISE
              검증 결과를 표 형식으로 출력
           4. 커스텀 축 최종 확인 (novel-config.md에 커스텀 축이 있으면)
           5. 보존 가드레일: {GUARD_RAILS}
           6. 훅 최종 확인
           7. 분량 ±15% 이내

           PASS/REVISE 판정."
```

- **PASS** → Step 5로
- **REVISE** → 수정 지시를 포함하여 Step 3 재실행 (최대 1회)

### Step 5: 기록 및 다음 에피소드로

1. fix_plan.md 현황 카운트 업데이트
2. learnings.md 업데이트 (새 패턴 발견 시만)
3. alive-tracker.md 업데이트 (alive-enhancer 보고서의 관계 이벤트)
4. **즉시 다음 에피소드로 → Step 1로 돌아간다**

---

## 강제 교정 정책 (Zero-Skip)

- **SKIP 절대 금지**: 모든 에피소드 전문 정독
- **CLEAN 조건**: 4개 보고서 합산 위반 0건 + 개선 후보 3건 미만 + 훅 강도 3+
- **CLEAN 비율 모니터링**: 10화 연속 CLEAN 50%+ → learnings에 경고 기록
- **교정률 목표**: 90%+

## Back-pressure 규칙

- 5화 연속 교정 15건+ → learnings에 공통 원인 분석 기록 (중단하지 않음)
- REVISE 3회 연속 → learnings에 교정 기준 재검토 기록
- 50화마다 교정률 집계

## 보존 가드레일

novel-config.md의 "보존 가드레일" 섹션에서 로드한다.
교정 시 이 목록의 요소를 절대 훼손하지 않는다.

## 핵심 전환 포인트

novel-config.md에서 프로젝트별 핵심 전환 포인트를 정의한다.
platform-optimizer에게 플래그 전달. 훅 강도 4+ 필수.
