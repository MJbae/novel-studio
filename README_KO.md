<p align="center">
  <img src="https://img.shields.io/badge/Version-1.1.0-brightgreen.svg" alt="Version">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="License"></a>
  <img src="https://img.shields.io/badge/Claude_Code-Plugin-purple.svg" alt="Claude Code Plugin">
  <img src="https://img.shields.io/badge/Agents-18-orange.svg" alt="18 Agents">
  <img src="https://img.shields.io/badge/Skills-10-green.svg" alt="10 Skills">
  <a href="https://github.com/MJbae/awesome-novel-studio/stargazers"><img src="https://img.shields.io/github/stars/MJbae/awesome-novel-studio?style=social" alt="GitHub Stars"></a>
</p>

[English](README.md) | **한국어**

# Awesome Novel Studio

AI 기반 웹소설 창작 하네스. Claude Code 위에서 동작하며, 18개 전문 에이전트와 10개 스킬을 조합해 **기획 → 설계 → 집필 → 윤문 → 재작성** 전 과정을 자동화합니다.

> **프로덕션 검증 완료** — 본 워크플로우로 집필한 웹소설이 출판사와 정식 계약을 체결했습니다.
> 1일 조회수 2,500+ / 좋아요 1,000+ / 구독 300+

---

## 설치

### 마켓플레이스

#### 마켓플레이스 등록
```shell
/plugin marketplace add MJbae/awesome-novel-studio
```

#### 플러그인 설치
```shell
/plugin install novel-studio@awesome-ai-studio
```

#### 세션 재시작으로 활성화
```shell
/exit
```

---

## 빠른 시작

```bash
# 1. 새 소설 기획안 생성
/propose

# 2. 큰 설계 (부트스트랩 + 캐릭터시트 + 플롯훅가이드)
/design-big

# 3. 작은 설계 (25화 단위 세부 설계)
/design-small

# 4. 에피소드 집필
/create

# 5. 윤문
/polish
```

## 파이프라인

```
기획 ─────── 설계 ──────────────── 집필 ──── 윤문 ──── 출판
/propose       /design-big              /create   /polish
               /design-small
                    ↕ 설계 변경 시
                 /rewrite → /polish
```

---

## AI 웹소설 창작의 현실적인 벽

1화 5,000자는 어렵지 않습니다. 문제는 이걸 300화, 150만 자로 늘렸을 때 벌어집니다.

| 장편 연재의 벽 | 증상 | Awesome Novel Studio의 해법 |
|---------------|------|---------------------|
| **인물의 입체성 붕괴** | 수많은 등장인물의 고유한 성격과 화법, 내적 동기가 회차를 거듭할수록 흐릿해지고 획일화된다. | **캐릭터시트 + 보이스 테이블** — 캐릭터별 말투·어미·비언어 팔레트를 설계 단계에서 정의하고, 집필·윤문 시 `VOICE`·`TITLE`·`ALIVE` 축으로 매 화 자동 검증한다. |
| **스토리 개연성 파괴** | 방대한 세계관 속에서 촘촘하게 쌓아 올린 사건의 인과관계와 복선이 엇나간다. | **플롯훅가이드 + continuity-bridge** — 전체 서사 아크를 25화 단위로 비트 분해하고, 매 화 집필 전 `continuity-bridge`가 직전 2화의 타임라인·복선·인물 상태를 수집해 집필 에이전트에 전달한다. |
| **숫자·설정의 불일치** | 재화의 가치, 역사적 연도, 인물의 연령 등 작품의 뼈대를 지탱하는 핵심 수치들이 앞뒤가 맞지 않고 오락가락한다. | **guard_rails + LOGIC 축** — `novel-config.md`에 절대 규칙을 정의하고, 윤문 `LOGIC` 축이 직전 2화와의 수치·타임라인 교차 검증을 수행한다. |

---

## 주요 명령어

| 명령어 | 설명 | 비고 |
|--------|------|------|
| `/propose` | 장르+한국 플랫폼+컨셉으로 3개 기획안 생성 | 새 소설 시작점 |
| `/design-big` | 전체 소설 설계 (부트스트랩·캐릭터·플롯) | 자동 리서치 포함 |
| `/design-small` | 25화 단위 세부 설계 | 큰 설계 완료 후 사용 |
| `/design` | 설계 라우터 (큰+작은 설계 통합) | 범위 불분명 시 사용 |
| `/bootstrap` | 부트스트랩 문서 단독 생성 | 세계관·컨셉만 필요 시 |
| `/character` | 캐릭터 시트 단독 생성 | 인물 설계만 필요 시 |
| `/plot-hook` | 플롯훅 가이드 단독 생성 | 서사 구조만 필요 시 |
| `/create` | 에피소드 순차 집필 | 설계문서 기반 |
| `/polish` (= `/lint`) | 16축 윤문 (6에이전트 병렬) | 자동 순차 진행 |
| `/rewrite` (= `/revise`) | 설계 변경에 따른 에피소드 재작성 | 영향 범위 자동 산출 |

### 범위 지정

모든 집필/윤문/재작성 명령은 범위를 지정할 수 있습니다:

```bash
/create EP051          # EP051부터 시작
/create EP001-EP010    # EP001~EP010 범위
/polish start          # EP001부터 처음부터
/rewrite 프로젝트명 EP001-EP010
```

## 워크플로우

```
/propose                기획안 3개 생성, 1개 선택
       ↓
/design-big             부트스트랩 + 캐릭터시트 + 플롯훅가이드
       ↓                → novel-config.md 자동 생성
/design-small           25화 단위 세부 설계 (선택, 권장)
       ↓
/create                 에피소드 집필 (자동 연속)
       ↓
/polish                 16축 윤문 (자동 연속)
       ↓
[설계 변경 시] /rewrite → /polish  재작성 후 재윤문
```

## 디렉토리 구조

프로젝트를 시작하면 아래 구조가 자동 생성됩니다:

```
{프로젝트명}/
├── novel-config.md              # 프로젝트 설정 (핵심 파일)
├── design/                      # 설계 문서
│   ├── {이름}_부트스트랩.md       # 세계관·컨셉·플랫폼 전략
│   ├── {이름}_캐릭터시트.md       # 캐릭터 프로필·관계망·대화DNA
│   └── {이름}_플롯훅가이드.md     # 3막 구조·25화 비트·훅 전략
├── episode/                     # 에피소드 본문
│   ├── ep001.md
│   └── ...
├── revision/                    # 작업 파일
│   ├── fix_plan.md              # 윤문 진행 상황
│   └── learnings.md             # 윤문 중 발견된 패턴
└── _workspace/                  # 임시 작업 공간
    └── 00_research/             # 자동 리서치 결과
```

## novel-config.md

모든 파이프라인의 중심 설정 파일입니다. `/design-big` 실행 시 자동 생성되며, 사용자가 검토·수정합니다.

```yaml
project:
  name: "소설명"
  target_platform: "문피아"
  target_genre: "회귀+전문가물"
  episode_dir: "episode/"
  work_dir: "revision/"
  design_dir: "design/"

design_documents:
  bootstrap: "design/소설명_부트스트랩.md"
  character_core: "design/소설명_캐릭터시트.md"
  plot_guide: "design/소설명_플롯훅가이드.md"

ep_range_table:
  - range: "EP001-EP025"
    label: "1막: 기원"
    plot_guide: "design/소설명_플롯훅가이드.md"

guard_rails:
  - "주인공의 회귀 능력은 과거 정보 회상만 가능"

custom_axes:
  EXPERTISE: "전문 지식은 대사에서 냄새만 풍기되 설명하지 않음"
```

- **ep_range_table**: 플롯 가이드에서 자동 추출된 에피소드 범위
- **guard_rails**: 모든 집필/윤문 단계에서 검증되는 절대 규칙
- **custom_axes**: 프로젝트별 추가 윤문 기준

## 16축 윤문

| 축 | 이름 | 내용 |
|----|------|------|
| 1 | BANNED | 금지 표현 (시간 점프 클리셰, 메타 코멘터리) |
| 2 | VOICE | 캐릭터 대화 일관성 (보이스 테이블 대조) |
| 3 | TITLE | 호칭 규칙 (화자·청자·맥락) |
| 4 | SILENCE | 침묵 패턴 과다 사용 (에피소드당 최대 4회) |
| 5 | TRANS | 번역체·AI톤·의미적 직역 탐지 |
| 6 | SCENE | 장면 구조 (비트·갈등·해소) |
| 7 | LOGIC | 서사 논리·타임라인·수치 정합성 |
| 8 | SUMMARY | 장면별 존재 이유 |
| 9 | UNIFORM | 에피소드 간 일관성 |
| 10 | HOOK | 훅 강도 (오프닝·중간·클리프행어) |
| 11 | OPENING | 오프닝 200자 이내 훅 |
| 12 | MOBILE | 모바일 가독성 (문단 길이·대화 비율) |
| A1 | ALIVE | 메아리 대화 해소 |
| A2 | ALIVE | 침묵→비언어 교체 |
| A3 | ALIVE | 긴장점 캐릭터 생동감 |
| A4 | ALIVE | 거리감 관리 |

## 에이전트 구성

### 설계 에이전트 (5)
`concept-builder` · `character-architect` · `plot-hook-engineer` · `proposal-generator` · `domain-researcher`

### 집필 에이전트 (4)
`episode-architect` · `episode-creator` · `continuity-bridge` · `quality-verifier`

### 윤문 에이전트 (6)
`rule-checker` · `story-analyst` · `platform-optimizer` · `alive-enhancer` · `revision-executor` · `revision-reviewer`

### 재작성 에이전트 (4)
`revision-analyst` · `character-sculptor` · `episode-rewriter` · `quality-verifier`

> `quality-verifier`는 집필과 재작성 단계에서 공유됩니다 (고유 에이전트 총 18개).

## 감사의 말

[황민호(revfactory)](https://github.com/revfactory)님께 감사드립니다. [Harness](https://github.com/revfactory/harness) 플러그인 덕분에 초기 하네스 구축이 수월했습니다. 링크드인을 비롯한 여러 채널에서 나눠주시는 인사이트에 항상 큰 도움을 받고 있습니다.

## 라이선스

[Apache 2.0](LICENSE)
