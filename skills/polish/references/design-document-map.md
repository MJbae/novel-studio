# 설정문서 매핑 가이드 — Polish 에이전트용

윤문 에이전트가 참조하는 설정문서는 **프로젝트별 novel-config.md**에 정의한다.
이 파일은 novel-config.md 작성 시 참고하는 가이드다.

---

## 매핑 구조

각 소설 프로젝트의 `novel-config.md`에 아래 구조를 채운다:

### 1. 공통 문서 (모든 EP에 적용)

| 문서 키 | 용도 | 에이전트 |
|---------|------|---------|
| character_core | 캐릭터 핵심 정의, 고유 설정, 능력 매트릭스 | 전체 |
| character_detail | 보이스표, 호칭표, 비언어 태그, 관계 변화 | rule-checker, alive-enhancer, revision-* |
| bootstrap | 매크로 수치, 세계관 규칙, 시간선 | story-analyst, revision-* |

### 2. EP 범위별 플롯 가이드

소설의 막(Act) 구조에 따라 EP 범위를 분할하고, 각 범위에 해당하는 플롯 가이드 경로를 지정.

```
EP001~EP026 → plot-hook-guide_act1.md
EP027~EP076 → plot-hook-guide_act2.md
EP077~EP150 → plot-hook-guide_act3.md
```

윤문 스킬은 대상 EP 번호를 보고 자동으로 해당 플롯 가이드를 선택한다.

### 3. 보조 참조 (선택)

| 문서 키 | 용도 | 에이전트 |
|---------|------|---------|
| web_novel_guide | 모바일 최적화 원칙 | platform-optimizer |
| verification | 검증 완료 수치 기록 | story-analyst |
| plot_macro | 핵심 전환 포인트 매크로 | platform-optimizer |

---

## 수치 교차검증 정본 우선순위

수치 불일치 발견 시 어느 문서를 정본으로 삼을지 novel-config.md에 명시한다.
일반적 우선순위:

1. EP별 플롯 가이드 (가장 구체적)
2. 부트스트랩 (매크로 수치)
3. 검증 기록 (이미 확인된 수치)
4. 직전 에피소드 (서사 연속성)

---

## 새 프로젝트 설정 방법

1. 프로젝트 루트에 `novel-config.md` 생성
2. `${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md` 참조하여 섹션 채우기
3. `/polish {프로젝트명}` 으로 실행
