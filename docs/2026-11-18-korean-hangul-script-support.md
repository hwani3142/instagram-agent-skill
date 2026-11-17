# Director 실행계획

## 작업 분석
Instagram 에이전트 스킬 팩(Claude Code 플러그인)의 로컬 Python 도구 5종(caption.py, hookscore.py, beats.py, detect.py, humanize.py)과 slop 사전(slop.json)은 정규식·단어 목록이 영어 전용이라 한국어(한글) 캡션/훅/스크립트/초안을 넣으면 해시태그 0개, 구체성 0점, 낭독 시간 1.8초 등 잘못된 결과를 낸다. 사용자는 번역 우회가 아니라 스크립트에 직접 한국어 지원을 넣는 방식을 선택했다. 핵심 제약은 영어 입력 결과가 바이트 단위로 동일해야 한다는 것(회귀 금지)과, 각 스크립트가 독립 배포되므로 공용 헬퍼 없이 스크립트별로 작은 상수를 복제해야 한다는 것이다. 스크립트가 3개 폴더에 독립적으로 나뉘어 있어 폴더 단위 Writer 병렬화가 가능하나, 한국어 낭독 속도 기본값·조사 경계 매칭 방식 등 설계 판단이 필요하므로 Analyzer가 먼저 구체 사양을 확정한다.

## 선택한 파이프라인
템플릿: 기본 (코드/문서/버그) + 모듈별 Writer 병렬 + TestRunner 검증
순서: Director → Analyzer → Writer × 3 (병렬) → Reviewer → TestRunner

## 병렬화 계획
Analyzer: 단독, 포그라운드 (Writer 입력이 되므로 완료 후 Writer 디스패치). 5개 스크립트 + slop.json + swipe.py + hooks.json 전체를 읽고 파일별 변경 사양을 확정.
Writer: 복수 병렬 3개 — 폴더(=독립 배포 단위) 기준 분할, 공유 파일 없음
- Writer A (ig-caption): `skills/ig-caption/caption.py`
- Writer B (ig-reel): `skills/ig-reel/hookscore.py`, `skills/ig-reel/beats.py` (+ Analyzer가 필요하다고 판단한 경우에 한해 `skills/ig-reel/hooks.json`, `skills/ig-viral/swipe.py`)
- Writer C (ig-human): `skills/ig-human/detect.py`, `skills/ig-human/humanize.py`, `skills/ig-human/slop.json` (slop 사전을 공유하므로 한 Writer가 담당)
- README 한국어 지원 섹션(선택)은 Writer C가 마지막에 담당 (다른 Writer는 README를 건드리지 않음)
Reviewer: 단독 (모든 Writer 완료 후)
TestRunner: 메인 Claude가 Bash로 직접 실행 (Reviewer APPROVED 후)

## 각 역할 지시사항

### Analyzer
- 목표: 각 파일의 영어 전용 지점을 전수 조사하고, 영어 결과를 바꾸지 않는 최소 한국어 확장 사양을 파일별로 확정한다. 반드시 결정할 사항:
  1. 공통 한글 문자 범위 상수(예: `가-힣` 완성형 음절, 필요 시 자모 `ㄱ-ㆎ`)와 각 스크립트에 복제할 정확한 정규식 문자열.
  2. caption.py: HASHTAG_RE 한글 허용, CONCRETE_RE가 `27kg`/`7.5kg`/`3가지`/`5분`처럼 숫자 뒤 한글·단위가 붙는 경우를 잡도록 하는 방법(`\b` 문제), 한국어 숫자 단어(하나/둘/셋, 세 가지 등) 포함 여부와 범위, ASKS에 추가할 한국어 CTA 목록(예: "댓글로 알려주세요", "댓글", "저장해두세요", "저장", "공유", "DM 주세요", "프로필 링크" 등 — 오탐 가능성 검토 포함).
  3. hookscore.py: WORD_RE/NUMBER_RE/PROPER_RE의 한국어 처리(한국어에는 대문자 고유명사가 없음 — PROPER 점수를 한국어에서 어떻게 다룰지), 길이 점수가 한국어 어절 기준으로 합리적인지, 한국어 dealbreaker 패턴("안녕하세요", "여러분 안녕", "오늘은 ~ 보여드릴게요/알려드릴게요", "이 영상에서는", "스크롤 멈춰/멈추세요" 등). 예시 훅 "27kg 대형견이 7.5kg 아기 옆에서 하는 행동 3가지"의 기대 점수 방향(STAKES/구체성 상승)과 인사 훅이 LENGTH가 아닌 dealbreaker로 걸려야 함을 명시.
  4. beats.py: 한국어 낭독 시간을 한글 음절 수 기반으로 계산하는 방식, 근거 있는 기본 음절/분(SPM) 값(한국어 방송·낭독 속도 문헌 기준 범위 제시 후 하나 선택), 영어 단어와 한글 음절이 섞인 줄의 처리, `--wpm` 인자와의 상호작용(별도 `--spm` 추가 여부 포함; 추가 시 영어 기본 동작 불변 보장).
  5. detect.py: WORD_RE/NUMBERS/PROPER/CONTRACTIONS/PRONOUNS 중 한국어에서 의미 있는 것과 영어 전용으로 남겨야 하는 것 구분, slop 매칭의 `\b` 경계가 조사(은/는/이/가/을/를/에/로 등) 붙은 한국어에서 실패하는 문제의 해법(한국어 항목은 앞쪽만 경계 검사 또는 경계 없이 부분일치 등), 영어 항목 매칭은 기존 그대로 유지.
  6. humanize.py + slop.json: slop.json 기존 스키마(`version`, `note`, `invisible`, `typographic`, `words` 등)를 정확히 지키면서 한국어 항목을 추가하는 방법(기존 `words` 배열에 섞을지, 스키마가 허용하는 형태로 구분할지 — 스키마 변경 금지), 한국어 상투어 목록과 대체어(예: "게임체인저", "한 단계 업그레이드", "스크롤 멈추세요", "팔로우하고 더 많은 꿀팁", "필요한 친구를 태그", "~에 대해 알아볼게요", "결론적으로" 등), humanize의 치환이 조사를 보존하는지.
  7. swipe.py: hookscore를 import해 점수를 쓰므로 hookscore 수정의 효과가 자동 전파되는지 확인. WORD_RE와 hooks.json `match` 정규식 기반 공식 분류의 한국어 지원이 소규모(정규식 몇 개 수정)면 Writer B 범위에 포함, 아니면 "범위 외"로 명시.
  8. 영어 회귀 위험 지점 목록: 각 변경이 영어 샘플(`verify/en/*`)의 출력에 영향을 줄 수 있는지 판단하고, 영향 가능성이 있으면 회피 방법 제시.
- 산출물: `docs/2026-11-18-korean-hangul-script-support-analysis.md` — 파일별(Writer A/B/C별) 변경 사양(정확한 정규식·상수·목록·함수 수정 위치/라인), 결정 근거, 범위 외 항목, 영어 회귀 체크리스트.

### Writer A (ig-caption)
- 목표: Analyzer 사양대로 `skills/ig-caption/caption.py`에 한국어 지원 추가 (해시태그, 구체성 숫자/단위, 한국어 CTA). 기존 스타일·주석 톤 유지, 외과적 변경, 새 의존성 금지. 테스트 작성 불필요.
- 산출물: 수정된 `skills/ig-caption/caption.py`, 변경 요약(변경 라인과 영어 결과 불변 근거).

### Writer B (ig-reel)
- 목표: Analyzer 사양대로 `skills/ig-reel/hookscore.py`(단어/숫자/고유명사/dealbreaker 한국어 처리)와 `skills/ig-reel/beats.py`(한글 음절 기반 낭독 시간)를 수정. Analyzer가 swipe.py/hooks.json 한국어 지원을 범위 내로 판정한 경우에만 해당 파일도 수정. 다른 스크립트와 헬퍼 공유 금지(상수 복제). 테스트 작성 불필요.
- 산출물: 수정된 파일들, 변경 요약(`--wpm`/신규 인자 동작 설명 포함, 영어 결과 불변 근거).

### Writer C (ig-human)
- 목표: Analyzer 사양대로 `skills/ig-human/slop.json`에 한국어 상투어 추가(기존 스키마 그대로), `detect.py`와 `humanize.py`가 한국어 slop을 조사가 붙어도 탐지·치환하도록 수정, 한국어 단어 계수 처리. 영어 항목의 매칭 방식은 변경 금지. 마지막으로 README에 짧은 "Korean support" 섹션을 저자 톤으로 최소 추가(선택, Analyzer 사양에 따름). 테스트 작성 불필요.
- 산출물: 수정된 `slop.json`, `detect.py`, `humanize.py`, (선택) `README.md`, 변경 요약.

### Reviewer
- 검토 기준:
  1. 영어 입력 결과 불변: 변경된 정규식/분기가 ASCII 입력에서 기존과 동일하게 동작하는지 코드 수준으로 확인.
  2. 한국어 정확성: 해시태그 계수, `27kg`/`7.5kg` 구체성, 한국어 CTA, 한국어 인사/서두 dealbreaker, 음절 기반 낭독 시간, 조사 붙은 slop 탐지·치환이 Analyzer 사양대로 구현되었는지.
  3. 독립성·제약: 스크립트 간 공용 헬퍼 없음, stdlib만 사용, 네트워크 없음, slop.json 스키마 동일(JSON 유효성 포함).
  4. 외과적 변경: 요청과 무관한 리팩터·포맷 변경·주석 수정 없음, 기존 코드 스타일 일치.
  5. 오탐 위험: 한국어 CTA/slop/dealbreaker 패턴이 일반 문장에서 과도하게 걸리지 않는지.
- NEEDS_REVISION 조건: 위 기준 중 하나라도 위반(영어 결과 변경 가능성, 사양 미구현, 스키마 변경, 공용 헬퍼 도입, 무관한 변경 포함). 해당 파일 담당 Writer에게만 재디스패치.
- NEEDS_REANALYSIS 조건: Analyzer 사양 자체가 영어 회귀를 유발하거나, 한국어 낭독 속도 기본값/조사 경계 방식 등 설계 판단이 근거 없이 틀렸거나, 사양이 서로 모순되어 Writer가 따를 수 없는 경우.

### TestRunner
- 실행 명령어 (메인 Claude가 레포 루트에서 Bash로 실행):
  ```
  V=/private/tmp/claude-501/-Users-jh-workspace-insta/842875ae-9eef-4ee6-acaa-7481ae0210d2/scratchpad/verify; $V/run.sh $PWD/skills $V/after && diff -r <(cd $V/base; for f in en-*; do echo "== $f"; cat $f; done) <(cd $V/after; for f in en-*; do echo "== $f"; cat $f; done) && cat $V/after/ko-*
  ```
  추가로 `python3 -c "import json;json.load(open('skills/ig-human/slop.json'))"` 로 JSON 유효성 확인.
- PASS 조건: 영어 diff가 비어 있음(의도·정당화된 변경 제외) AND 한국어 출력에서 해시태그가 계수됨, 27kg이 구체성으로 계수됨, 한국어 CTA 탐지됨, 한국어 인사 훅이 dealbreaker로 표시됨, 한국어 스크립트가 현실적 길이로 계측됨, 한국어 slop 용어가 탐지/치환됨.
- FAIL 시 재 dispatch 대상 Writer:
  - `en-caption*` / `ko-caption*` 문제 → Writer A (`skills/ig-caption/caption.py`)
  - `en-hookscore*` / `ko-hookscore*`, `en-beats*` / `ko-beats*` 문제 → Writer B (`skills/ig-reel/hookscore.py`, `skills/ig-reel/beats.py`, 범위 내일 때 `skills/ig-viral/swipe.py`, `skills/ig-reel/hooks.json`)
  - `en-detect*` / `ko-detect*`, `en-humanize*` / `ko-humanize*`, slop.json 유효성 문제 → Writer C (`skills/ig-human/detect.py`, `skills/ig-human/humanize.py`, `skills/ig-human/slop.json`)

## 컨텍스트
- 레포 루트: git worktree `/Users/jh/workspace/insta/instagram-agent-skill/.claude/worktrees/korean-support` (브랜치 `worktree-korean-support`). 플러그인 = `.claude-plugin/`, 스킬 = `skills/ig-*/`.
- Python 3 표준 라이브러리만 사용. 새 의존성·네트워크 금지 (README의 "No dependencies, no network" 약속 유지).
- 각 스크립트는 자기 폴더에서 독립 실행·개별 복사되므로 스크립트 간 공용 헬퍼 모듈 금지 — 작은 정규식 상수는 복제한다. (기존 swipe.py → hookscore.py optional import는 기존 구조이므로 유지.)
- 사용자는 번역 전후 처리 방식이 아니라 직접 한국어 지원을 선택함.
- 하드 제약: 영어 기준 샘플 출력이 변경 후에도 바이트 단위 동일(명시적으로 의도·정당화된 경우 제외). 기존 코드 스타일 준수, 외과적 변경, 한국어 지원에 필요한 것 이상의 리팩터 금지. Python이므로 Writer는 단위 테스트를 작성하지 않음.
- 파일별 알려진 문제:
  - `skills/ig-caption/caption.py`: HASHTAG_RE `#[A-Za-z0-9_]+`(약 38행)가 `#애개육아`를 0개로 셈. CONCRETE_RE `\b\d[\d,.]*\b`가 `27kg`/`7.5kg`(숫자 뒤 한글·단위) 및 한국어 수사 누락. ASKS(약 45행)가 영어 CTA만 포함 — "댓글로 알려주세요", "저장해두세요", "DM 주세요", "프로필 링크" 등 누락.
  - `skills/ig-reel/hookscore.py`: WORD_RE `[A-Za-z0-9$%'’-]+`(39행), NUMBER_RE, PROPER_RE, dealbreaker 정규식(인사 "hey/hi", 서두 "in this video", "stop scrolling")이 영어 전용. 한국어 대응: "안녕하세요", "오늘은 ~ 보여드릴게요", "이 영상에서는", "스크롤 멈춰". 한국어 훅 "27kg 대형견이 7.5kg 아기 옆에서 하는 행동 3가지"가 48점 WEAK(STAKES 20), 인사 훅이 8.4점(LENGTH 0 때문 — 결과는 비슷하나 이유가 틀림).
  - `skills/ig-reel/beats.py`: 한국어를 거의 0단어로 세어 5줄 한국어 스크립트가 1.8초로 계측됨. 한국어는 영어 단어가 아닌 한글 음절 수 기반 낭독 속도가 필요.
  - `skills/ig-human/detect.py`: WORD_RE `[A-Za-z']+`, NUMBERS, PROPER, CONTRACTIONS, PRONOUNS 영어 전용. slop 매칭이 `\b` 경계를 써서 조사가 붙은 한국어 단어에서 실패.
  - `skills/ig-human/humanize.py`, `skills/ig-human/slop.json`: 영어 상투어 154개. slop.json 최상위 키 `version`, `note`, `invisible`, `typographic`, `words` 등 — 스키마 정확히 유지하며 한국어 항목 추가.
  - `skills/ig-viral/swipe.py`: `WORD_RE = [A-Za-z0-9$%'’-]+`(37행), `../ig-reel/hooks.json`의 `match` 정규식으로 공식 분류, `hookscore.run`을 optional import해 점수 산출. 한국어 공식 매칭은 소규모일 때만 범위 내.
- 검증 하네스(재생성 금지): `/private/tmp/claude-501/-Users-jh-workspace-insta/842875ae-9eef-4ee6-acaa-7481ae0210d2/scratchpad/verify/` — `en/`, `ko/` 샘플 입력(hooks.txt, script.txt, caption.txt, draft.txt), `base/` = 변경 전 출력, `run.sh <skills_dir> <out_dir>`가 양 언어에 대해 hookscore, beats, caption, detect, humanize 실행.
- 산출 문서 경로 규칙: Analyzer 산출물 `docs/2026-11-18-korean-hangul-script-support-analysis.md`.
