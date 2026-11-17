# 분석 결과

> 이 문서의 모든 정규식·상수·목록은 스크래치 복사본에 실제로 적용해 검증했다.
> - 프로토타입 패치: `/private/tmp/claude-501/-Users-jh-workspace-insta/842875ae-9eef-4ee6-acaa-7481ae0210d2/scratchpad/analyzer/patch.py`
> - 프로토타입 결과 diff: `/private/tmp/claude-501/-Users-jh-workspace-insta/842875ae-9eef-4ee6-acaa-7481ae0210d2/scratchpad/analyzer/proto.diff`
> - 영어 회귀 스윕: `.../scratchpad/analyzer/regress.py` (레포 내 모든 .md 줄·문단 + hooks.json 필드 + slop 항목 합성문, 영어 텍스트 2140개 × 5개 함수, 구/신 JSON 비교) → **diff 0**
> - 한국어 프로브: `.../scratchpad/analyzer/ko_probe.py`
> - 하네스 `run.sh` 결과: `en-*` 5개 파일 모두 `cmp` 동일.
>
> Writer는 아래 문자열을 **그대로** 옮기면 된다. 다시 결정할 것은 없다.

## 현황 파악

| 파일 | 영어 전용 지점 | 한국어 입력 시 현재 증상 (base/) |
|---|---|---|
| `skills/ig-caption/caption.py` | `HASHTAG_RE`(38행) `[A-Za-z0-9_]`, `CONCRETE_RE`(43행) 끝 `\b`, `ASKS`(45–54행) 영어 CTA | 해시태그 0개, 구체성 0, ask 0개 |
| `skills/ig-reel/hookscore.py` | `WORD_RE`(39행), `STAKES`(67–79), `WEAK_OPENERS`(82–89), `DEALBREAKERS`(98–109), `check_address`(200–210) | 훅1: 4단어(LENGTH 80), STAKES 20 → 48.2. 인사 훅: 0단어 → LENGTH 0 → 8.4, dealbreaker 없음 |
| `skills/ig-reel/beats.py` | `WORD_RE`(29) 기반 낭독 시간, `CONCRETE_RE`(31) | 5줄 스크립트 1.8초, 2–4행 0.0초 |
| `skills/ig-human/detect.py` | `WORD_RE`(36) `[A-Za-z']`, `NUMBERS`(39), slop 패턴 `\b…\b`(92), `CONTRACTIONS/PRONOUNS` | 단어 0 → 모든 체크 "too short"/"empty" |
| `skills/ig-human/humanize.py` | slop 패턴 `\b…\b`(136) | em dash만 처리, 상투어 0개 |
| `skills/ig-human/slop.json` | 영어 항목만 (words 96 + phrases 58 = 154) | — |
| `skills/ig-viral/swipe.py` | `WORD_RE`(37), hooks.json `match` 26개 | 훅 단어 수 0 근처, 공식 전부 unclassified |

확인한 사실:
- Python 3 `str` 정규식에서 `\w`/`\b`는 유니코드 기준이라 한글은 단어 문자다. 그래서 `#\w+`(hookscore `HASHTAG_RE`, slop `hashtag-wall`)는 이미 한글을 잡는다. 실패 원인은 `[A-Za-z]` 명시 클래스, 그리고 `숫자\b`(27kg, 3가지에서는 숫자와 뒤 글자가 둘 다 단어 문자라 경계가 없음)다.
- `\b` 앞쪽 경계는 공백 뒤 한글에서 정상 작동한다. 뒤쪽 `\b`는 조사가 붙으면 실패한다(`게임체인저를`).
- `slop.json`은 `json.dumps(d, indent=2, ensure_ascii=False) + "\n"`과 바이트 단위로 같다(검증함). 그래서 Writer C는 같은 방식으로 다시 써도 diff가 추가분에만 생긴다.
- `slop.json` 최상위 키: `version, note, invisible, typographic, words, phrases, structures`. `words`/`phrases` 항목 스키마: `{"find","replace","family"}` (정확히 3필드). `structures` 항목: `{"id","regex","name","fix"}`. family 값은 `adjectives|verbs|openers|nouns|closers|connectives` 6개뿐이다.
- `swipe.py`는 `sys.path`에 `../ig-reel`을 넣고 `hookscore.run`을 import한다. 그래서 hookscore 수정은 swipe의 `hook_score`에 **자동 전파된다**(확인함).
- 기존 버그(영어에도 있음, 손대지 않음): humanize 삭제 뒤 `".!"` 잔여(`mindset.!`), beats `HOOK/CTA` 라벨 뒤 공백 없음, caption `$18,000` → `['$1','000']` 2개 계수, caption 박스에서 한글(전각)이 정렬 깨짐.

## 요구사항 정리

1. 한국어 캡션: `#애개육아` 계수, `27kg`/`7.5kg`/`3가지`/`5분` 구체성 계수, 한국어 CTA 탐지.
2. 한국어 훅: 어절 단위 길이, 한국어 숫자, 한국어 긴장어(STAKES), 인사·서두·"스크롤 멈춰"는 LENGTH가 아니라 **dealbreaker**로 걸린다.
3. 한국어 스크립트: 한글 음절 수 기반 낭독 시간. `--spm` 신설, 영어 `--wpm` 동작은 그대로.
4. 한국어 초안: 단어 계수, 조사가 붙어도 slop 탐지·치환, 치환 뒤에도 조사 문법 유지.
5. **하드 제약:** ASCII(영어) 입력에서 출력이 바이트 단위로 동일해야 한다. 이 사양의 모든 한국어 확장은 **매칭에 한글 문자(가-힣)가 반드시 필요한** 대안(alternation)이나 분기로만 추가한다. 그래서 ASCII 텍스트에서는 구조적으로 발동할 수 없다(코드 수준 보장, 리뷰어가 확인할 핵심 논리).

### Analyzer 결정 8개 (요약)

| # | 결정 |
|---|---|
| 1 | 한글 상수: 음절 `가-힣`(U+AC00–U+D7A3). 단어·해시태그 문자 클래스에는 자모 `ㄱ-ㆎ`(U+3131–U+318E, ㅋㅋ/ㅠㅠ)도 넣는다. 한국어 여부 판정과 음절 계수에는 `[가-힣]`만 쓴다. 각 스크립트에 복제하고 공용 모듈은 만들지 않는다. |
| 2 | caption: `HASHTAG_RE`에 한글을 허용한다. `KO_NUMBER` 대안(한글과 인접한 숫자 + 고유어 수사+단위명사)을 `CONCRETE_RE`의 `\b\d…\b` **앞**에 삽입한다. 기존 `ASKS` 8개 각각의 정규식 끝에 `|한국어` 대안을 붙인다(새 항목이 아니므로 ask 이름 중복이 없다). |
| 3 | hookscore: `WORD_RE`에 한글을 넣는다(어절=단어, 5–12 기준은 그대로). PROPER는 영어 전용으로 두고, 한국어 고유명사는 탐지하지 않는다. 한국어 dealbreaker 3종은 기존 3개 정규식에 대안으로 추가한다. 훅1 STAKES는 20으로 **오르지 않는 것이 정답**이다(아래 근거). |
| 4 | beats: `seconds() = 영어단어/wps + 한글음절/sps`. `--spm` 기본값 **300**. 한글이 하나라도 있으면 안내문 단위를 "syllables"로 바꾼다. JSON 키는 한국어일 때만 추가한다. |
| 5 | detect: WORD_RE·NUMBERS는 한국어로 확장한다. PROPER·CONTRACTIONS·PRONOUNS는 영어 전용으로 둔다. 한글이 다수인 텍스트에서는 contractions·pronouns 하위 점수를 중립 50으로 둔다. 한국어 slop 항목의 경계는 앞쪽 `(?<![가-힣])`, 뒤쪽은 `replace`가 비어 있으면(삭제) `(?![가-힣])`, 아니면(치환) 열어 둔다. |
| 6 | slop.json: 스키마는 그대로 두고, 기존 `words`/`phrases`/`structures` 배열 **끝에** 한국어 항목을 추가한다(words 7, phrases 9, structures 4). 치환어는 받침 유무를 원어와 일치시켜 조사를 보존한다. `note` 문자열 끝에 한 문장을 덧붙인다. |
| 7 | swipe: 점수는 자동 전파된다. `WORD_RE` 한 줄만 hookscore와 동일하게 바꾼다(범위 내). hooks.json 26개 `match`의 한국어화는 **범위 외**다. |
| 8 | 영어 회귀: 아래 체크리스트 참고. 프로토타입으로 하네스 동일 + 2140개 스윕 diff 0을 확인했다. |

## 제약사항 및 위험

- **영어 불변의 근거:** 추가한 모든 정규식 대안에는 `[가-힣]` 리터럴, 한글 리터럴 단어, lookahead/lookbehind 안의 `[가-힣]` 중 하나가 있다. ASCII 텍스트에서는 매칭이 불가능하다. 대안 순서상 한국어 대안을 영어 대안 **앞에** 두는 곳(CONCRETE_RE, detect NUMBERS)도 같은 이유로 ASCII에서는 영어 대안으로 그대로 떨어진다. 분기(`if HANGUL_RE.search(...)`, `ko` 플래그)는 ASCII에서 항상 False다. float 연산 `n/wps + 0/sps`는 `n/wps`와 IEEE 값이 같다(`x + 0.0 == x`).
- **JSON 출력:** beats `--json`에 추가하는 키(`syllables`, `spm`, `total_syllables`)는 한글이 있을 때만 붙인다. 그래서 영어 JSON은 바이트 동일하다(검증함).
- **오탐 위험(한국어):** "댓글", "저장", "공유", "꿀팁", "업그레이드", "알려드릴게요"처럼 일반 문장에 흔한 단어는 단독으로 넣지 않고, 명령형 어미나 특정 구 형태로만 매칭한다. 프로브로 음성 사례 11개(`저장 공간이 부족해요`, `사진을 저장해 둔 폴더`, `팔로우하고 있는 계정이 많아요`, `공유 주방`, `프로필 사진 바꿨어요` 등)가 모두 미검출인 것을 확인했다.
- **조사 문법:** 치환 항목은 원어 끝 음절과 치환어 끝 음절의 받침 유무(그리고 ㄹ받침 여부)를 맞췄다. 그래서 `를/을`, `가/이`, `로/으로`가 그대로 맞는다. `한 단계 업그레이드 → 개선`은 하다-동사 어근이라 뒤에 `해/된/하는`이 붙는 것이 일반적이다(명사 용법 `업그레이드를`은 드묾, 허용).
- **알려진 한계(문서화만 함):** 한국어 고유명사(우디, 아진이)는 대문자가 없어 탐지하지 못한다. beats는 `27kg` 같은 ASCII 토큰을 영어 단어 속도로 계산해서 실제 낭독("이십칠 킬로")보다 짧게 잡는다. 루프 탐지는 ASCII 토큰만 비교한다.
- **SPM 수치 출처:** 네트워크 없이 작업했다. 아래 범위는 한국어 발화 속도 연구와 방송·발표 가이드에서 흔히 인용되는 값이며, 원문으로 확인하지는 않았다. 코드 주석과 README에는 "범위를 단정하는 인용" 대신 "직접 재서 `--spm`으로 맞추라"는 기존 톤을 쓴다.

## Writer를 위한 구체적 지시사항

공통 규칙:
- 각 파일 안에 상수를 복제한다. import로 공유하지 않는다.
- 아래 주석 문구는 저자 톤(짧은 평서문, 영어)을 따르므로 그대로 쓴다.
- 요청 외 리팩터, 포맷 변경, 기존 주석 수정은 금지다.
- 검증: `V=/private/tmp/claude-501/-Users-jh-workspace-insta/842875ae-9eef-4ee6-acaa-7481ae0210d2/scratchpad/verify; zsh $V/run.sh $PWD/skills /private/tmp/claude-501/-Users-jh-workspace-insta/842875ae-9eef-4ee6-acaa-7481ae0210d2/scratchpad/<writer>/out` 실행 후 `en-*`를 `$V/base/en-*`와 `cmp`.

### Writer A (ig-caption) — `skills/ig-caption/caption.py`

**A1. 38행 `HASHTAG_RE`** 교체:
```python
HASHTAG_RE = re.compile(r"(?:^|\s)(#[A-Za-z0-9_가-힣ㄱ-ㆎ]+)")
```
(`MENTION_RE`는 그대로 둔다. 인스타그램 아이디는 ASCII뿐이다.)

**A2. 42행 `EMOJI_RE` 다음, 43행 `CONCRETE_RE` 앞**에 블록을 삽입하고, 43행을 교체한다:
```python
# Korean glues units and counters straight onto the number (27kg, 3가지, 5분),
# so the word boundary the English pattern ends on never comes. These only
# fire next to Hangul, which is what keeps English results where they were.
KO_NUMBER = (r"(?<![\d.,])\d[\d,.]*(?=[A-Za-z%]{0,3}\s?[가-힣])"
             r"|(?<=[가-힣])\d[\d,.]*|(?<=[가-힣]\s)\d[\d,.]*"
             r"|(?<![가-힣])(?:두|세|다섯|여섯|일곱|여덟|아홉|열)\s?(?:가지|번|명|개|살|마리|달)")
CONCRETE_RE = re.compile(r"\$\s?\d|" + KO_NUMBER + r"|\b\d[\d,.]*\b|(?<!^)\b[A-Z][a-z]{2,}\b", re.MULTILINE)
```
- 위치가 중요하다. `KO_NUMBER`는 `\b\d[\d,.]*\b` **앞**에 와야 한다. 뒤에 두면 `7.5kg`가 기존 대안에서 `7.`로 먼저 잡히고 `5`가 따로 잡혀 2개로 셈된다.
- 대안별 역할:
  - ① 숫자(+최대 3자의 ASCII 단위, `%`) 뒤 0–1칸 공백 후 한글: `27kg 리트리버`, `3가지`, `7.5kg 아기`, `27kg짜리`.
  - ② 한글 바로 뒤 숫자: `아기3명`.
  - ③ 한글+공백 뒤 숫자: `몸무게는 27kg.`.
  - ④ 고유어 수사+단위명사: `세 가지`, `두 번`, `열 달`.
- 수사 제외 근거:
  - `한`: "a/one"이라 영어 `one`을 뺀 것과 같은 이유다(`한 번 더`, `한 단계`).
  - `네`: "yes/your" 의미가 있다(`네 개가 짖어요` = 너의 개).
  - 서수 `첫째/둘째/셋째`: 영어 `first`가 SPOKEN_NUMBERS에서 빠진 것과 같다.
  - 단위명사 `분/초/주/배/시간/년`: `네 분명히`, `네 배가 고파`, `문을 열 시간`처럼 오탐이 있고, `년`은 고유어 수사와 쓰지 않는다.
  - 검증한 결과: `문을 열 시간`, `네 개가 짖어요`, `한 번 더`, `첫째, 둘째`는 0개다.

**A3. 45–54행 `ASKS`**: 각 튜플의 정규식 문자열 **끝에** 한국어 대안을 붙인다. 이름과 순서는 그대로 두고 항목 수도 늘리지 않는다. 최종 8줄은 아래와 같다(그대로 교체):
```python
ASKS = [
    (re.compile(r"(?i)\bcomment (?:the word |\")?[A-Z0-9]{2,}\b|댓글(?:에|로)?\s*(?:[\"\'“‘][^\"\'”’\n]{1,15}[\"\'”’]|[A-Za-z0-9가-힣]{1,15}?이?라고)"), "comment a keyword"),
    (re.compile(r"(?i)\b(?:dm|message) me\b|(?:\bdm|디엠|메시지|메세지)\s?(?:주세요|주시면|보내\s?주세요|보내주시면|남겨\s?주세요)"), "DM me"),
    (re.compile(r"(?i)\bsave (?:this|it)\b|저장\s?해\s?(?:두|놓)(?:세요|고|시면)|저장\s?(?:하세요|해\s?주세요|필수|각)"), "save this"),
    (re.compile(r"(?i)\bshare (?:this|it)\b|공유\s?(?:해\s?주세요|하세요|해\s?보세요|부탁)"), "share this"),
    (re.compile(r"(?i)\bfollow (?:me|for)\b|팔로우\s?(?:해\s?주세요|하세요|해\s?두세요|부탁|하고(?!\s?있))"), "follow"),
    (re.compile(r"(?i)\blink in (?:my )?bio\b|프로필\s?(?:상단\s?|의\s?)?링크|링크는\s?프로필"), "link in bio"),
    (re.compile(r"(?i)\b(?:swipe|tap) (?:through|left|right|for|to)\b|옆으로\s?(?:넘겨|밀어)|스와이프\s?해"), "swipe or tap"),
    (re.compile(r"(?i)\btell me\b|\bwhat would you\b|\bwhich one\b|알려\s?주세요|댓글\s?(?:로|에)?\s?(?:남겨|달아|적어)\s?주세요|(?:어떤\s?(?:게|걸)|뭐가)\s?더[^.!?\n]{0,15}\?"), "answer a question"),
]
```
- 키워드 댓글은 따옴표로 감싼 키워드(`댓글에 "체크리스트" 남겨주시면`)나 `~라고`(`댓글로 육아라고`) 형태가 있을 때만 잡는다. 키워드가 없는 일반 댓글 요청(`댓글 남겨주세요`, `댓글로 알려주세요`)은 "answer a question"으로 간다. 두 패턴이 같은 문장에서 이중 계수되지 않는 것을 확인했다.
- `팔로우하고(?!\s?있)`: `팔로우하고 있는 계정`(평서문)을 배제한다. `하고\s?(?!있)`로 쓰면 백트래킹으로 뚫리므로 반드시 이 형태로 쓴다.
- `저장\s?해\s?(?:두|놓)(?:세요|고|시면)`: `사진을 저장해 둔 폴더`(관형형)를 배제한다.
- 단독 `댓글`/`저장`/`공유`는 넣지 않는다. `댓글이 많이 달렸어요`, `저장 공간`, `공유 주방` 같은 오탐 때문이다(Director 예시 중 이 셋은 기각한다).

**A4. 그 외는 변경하지 않는다.**
- TAG PLACEMENT(134행)의 `re.escape(t) + r"\b"`는 한글 태그 뒤에서도 정상 작동한다.
- ko 샘플은 136자 캡션이라 125자 창 안에 `#애개육아`가 들어간다. 그래서 TAG PLACEMENT가 WARN으로 바뀌는데, 이것은 **올바른 결과**다.
- `TRUNCATE=125`, 박스 렌더링의 전각 정렬은 범위 외다.

**A 기대 결과 (ko-caption):** `3 hashtags · 1 ask(s)`, HOOK IS CONCRETE PASS 1개, HASHTAGS PASS `#애개육아 #대형견 #아기와강아지`, TAG PLACEMENT WARN, ONE ASK PASS `answer a question`, VERDICT REVIEW.

### Writer B (ig-reel + ig-viral) — `skills/ig-reel/hookscore.py`, `skills/ig-reel/beats.py`, `skills/ig-viral/swipe.py`

`hooks.json`은 **수정하지 않는다**(범위 외).

#### B1. hookscore.py

**39행:**
```python
WORD_RE = re.compile(r"[A-Za-z0-9$%'’가-힣ㄱ-ㆎ-]+")
```
(`-`는 클래스 맨 끝에 그대로 둔다.) 이렇게 하면 어절이 1단어가 된다. 길이 기준 5–12는 바꾸지 않는다. 한국어 어절은 평균 약 3음절이라 초당 2어절 안팎이고, 이는 영어 2.75단어/초와 같은 자릿수다. 그래서 "5–12어절 ≈ 2–5초"가 되어 기준의 취지가 유지된다. `~Xs spoken` 표시는 그대로 둔다(근사치).

**`SPOKEN_NUMBERS`(54–59행)** 마지막 줄 `"billion", "dozen", "half", "twice", "triple",` 다음, `}` 앞에 한 줄을 추가한다:
```python
    "두", "세", "다섯", "여섯", "일곱", "여덟", "아홉", "절반",
```
- 정확히 일치하는 토큰만 잡는다. 이 수사들은 단위명사 앞에서 띄어 쓰는 관형사라 단독 토큰이 된다.
- `한`, `네`, `열`은 제외한다(`열 받다`, `문을 열다`). 기존 주석 "one and first are deliberately absent"와 같은 원칙이다. 주석은 추가하지 않는다(세트 위 기존 주석이 이유를 이미 설명한다).

**`STAKES` 세트(79행 `}`) 다음, `# Openers that spend...`(81행) 앞**에 삽입한다:
```python
# Korean stakes words. Particles and endings attach to the stem (실수를,
# 몰랐어요), so these match the start of a word rather than the whole word.
# 않 and 없 match anywhere: every word they appear in is a negation.
KO_STAKES_RE = re.compile(
    r"(?<![가-힣])(?:절대|아무도|실수|실패|망했|망친|손해|후회|포기|그만두|멈춰|멈추"
    r"|위험|경고|금지|최악|비밀|몰랐|진실|거짓말|사기|낭비|문제|하지만|대신"
    r"|아직|이미|벌써|무료|공짜|처음)"
    r"|(?<![가-힣])(?:안|못)(?![가-힣])|않|없")

```
- 목록은 영어 STAKES를 의미 단위로 옮긴 것이다(never, nobody, mistake, failed, lost, regret, quit, stop, danger, warning, banned, worst, secret, "didn't know", truth, lie, scam, waste, problem, but, instead, still, already, free, first).
- `안`/`못`은 단독 어절일 때만 잡는다(`안 되는` ○, `안녕`·`안에서`·`못생긴` ×, 검증함).
- `그만`은 `그만큼` 오탐이 있어서 `그만두`만 넣었다.
- `진짜`/`딱`처럼 흔한 강조 부사는 제외한다.

**`WEAK_OPENERS`(88행) `"as a", "when it", "if you've", "you know",` 다음, `]` 앞**에 추가한다:
```python
    "안녕", "여러분", "오늘", "그래서", "솔직히", "그냥", "혹시", "사실",
    "이번 영상", "이 영상",
```
- 기존 로직 `opener.startswith(weak)`가 접두 매칭이라 `안녕하세요`, `오늘은`, `사실은`이 자연스럽게 잡힌다.
- `오늘`은 영어 `today`, `혹시`는 `have you`, `솔직히`는 `honestly`, `그냥`은 `just`에 대응한다.
- `자`(alright)는 `자면`·`자기` 접두 오탐 때문에 제외한다.

**`IMPERATIVES` 세트(96행 `}`) 다음, `DEALBREAKERS = [`(98행) 앞**에 삽입한다:
```python
# Korean person words. 너 needs a particle or a gap after it, or 너무
# ("very") would count as talking to someone.
KO_YOU_RE = re.compile(r"(?<![가-힣])(?:여러분|당신|너희|너(?=[는도를의가랑]|\s|$))")
KO_ME_RE = re.compile(r"(?<![가-힣])(?:우리|저희|제가|내가|나는|저는|(?:나|내|저|제)(?![가-힣]))")

```

**`DEALBREAKERS`(98–104행)**: 앞 3개 정규식만 아래처럼 교체한다. 메시지 문자열과 HASHTAG/EMOJI 항목은 그대로 둔다.
```python
    (re.compile(r"(?i)^\s*(?:stop scrolling|don'?t scroll|스크롤\s?(?:을\s?)?(?:멈|그만|내리지\s?마|넘기지\s?마)|잠깐만?요?[,!.]?\s?(?:스크롤|멈춰))"),
     "Opens with \"stop scrolling\". Asking for attention proves you have not earned it."),
    (re.compile(r"(?i)\b(?:in (?:this|today'?s) (?:video|reel)|i'?m going to show you|i'?ll show you how)\b"
                r"|(?<![가-힣])(?:이|이번|오늘)\s?(?:영상|릴스)\s?에서|(?:보여|알려|소개해)\s?드릴(?:게요|께요)"),
     "Video preamble. Delete it and open on the payoff."),
    (re.compile(r"(?i)^\s*(?:hey |hi |what'?s up |welcome |안녕|여러분[,!]?\s?안녕|반가워|반갑습니다)"),
     "Greeting. Nobody came to the feed to be greeted."),
```
- 인사(`^` 고정): `안녕하세요`, `여러분 안녕하세요`, `반갑습니다`. `하이`/`헬로`는 `하이킹` 오탐 때문에 제외한다.
- 서두: `이/이번/오늘 영상(릴스)에서`, `보여/알려/소개해 드릴게요`. 영어 "i'll show you how"에 대응한다. 훅(첫 줄)에만 적용되므로 `알려드릴게요`를 넣어도 된다. 캡션·초안용 slop structure에서는 일반 문장 오탐 때문에 제외했다(Writer C 참조).
- 스크롤(`^` 고정): `스크롤 멈춰/멈추세요/그만/내리지 마`, `잠깐만요, 스크롤`.

**`check_stakes` 154행·156행:**
```python
    hits = [x for x in w if x in STAKES] + KO_STAKES_RE.findall(text)
```
```python
    if re.search(r"\$\s?\d|₩\s?\d|\d\s?[만천억]?\s?원", text):
```
(가격 표시: `30만원`, `5,000원`, `₩5000`. 숫자가 앞에 있어야 하므로 `원래`·`병원` 오탐이 없다.)

**`check_frontload` 180–181행** 조건에 한 줄을 추가한다:
```python
        if (token in STAKES or token in SPOKEN_NUMBERS or token in MONEY_WORDS
                or NUMBER_RE.match(w[i]) or (i and PROPER_RE.match(w[i]))
                or KO_STAKES_RE.search(w[i])):
```

**`check_address` 204행·208행:**
```python
    if re.search(r"\b(you|your|you're|youre|yourself)\b", low) or KO_YOU_RE.search(text):
```
```python
    if re.search(r"\b(i|my|me|we|our)\b", low) or KO_ME_RE.search(text):
```
(한국어 명령형 어미 `-세요`를 IMPERATIVES 90점으로 처리하는 것은 범위 외다. `안녕하세요`도 `-세요`로 끝나기 때문이다.)

**변경하지 않는 것:**
- `NUMBER_RE`: `\b27`은 공백 뒤라 이미 잡히고, 단위 그룹이 optional이라 `27kg`에서 `27`이 매칭된다. 검증함.
- `PROPER_RE`: 한국어에는 대문자가 없다. 이름 사전이 없으면 탐지할 수 없으므로 영어 전용으로 둔다.
- `HASHTAG_RE`(`#\w+`는 이미 유니코드), `EMOJI_RE`, `MONEY_WORDS`(한국어 금액은 거의 항상 숫자에 붙음 → 숫자로 이미 잡힘), `IMPERATIVES`, 길이 점수 공식.

**훅 점수 기대치 (프로토타입 실측):**

| 훅 | 이전 | 이후 |
|---|---|---|
| `27kg 대형견이 7.5kg 아기 옆에서 하는 행동 3가지` | 48.2 WEAK (LENGTH 80, 4 words) | **50.6 OK**: LENGTH 100 (9 words), SPECIFICITY 100, STAKES **20**, FRONTLOAD 100, ADDRESS 35 |
| `안녕하세요 오늘은 우리 아진이 일상을 보여드릴게요` | 8.4 WEAK, weakest LENGTH(0), flag 없음 | **0.0 WEAK**: LENGTH 100 (6 words), FRONTLOAD 0 (weak opener "안녕"), ADDRESS 70 ("우리"), **dealbreaker 2개: Video preamble, Greeting** |

- **STAKES가 오르지 않는 것은 의도된 결과다.** 이 훅에는 긴장·손실·부정어가 하나도 없다. 같은 뜻의 영어 훅("What a 27kg dog does next to a 7.5kg baby: 3 things")도 STAKES 20이다. SPECIFICITY는 이미 base에서 100이었다(숫자 27, 7.5, 3). 상승 요인은 LENGTH(80→100)뿐이다. Director가 기대한 "STAKES 상승"은 이 사양에서 기각한다. `행동`·`아기`를 긴장어로 넣는 것은 점수 조작이다.
- 인사 훅은 이제 LENGTH가 아니라 **dealbreaker**(인사+서두, -30)와 FRONTLOAD 때문에 떨어진다. 이유가 정확해졌다.
- 추가 실측: `대형견이랑 아기 같이 키우면 절대 하면 안 되는 실수`는 STAKES 100(실수, 안, 절대), `아무도 안 알려주는 대형견 육아 비밀`은 STAKES 100, `월 30만원 아끼는 법`은 STAKES 70(a price), `너무 귀여운 우리 우디`는 ADDRESS 70(너무 오탐 없음).

#### B2. beats.py

**docstring 11–14행** 문단 끝(`... and the default here is 165.` 다음 줄)에 두 줄을 추가한다:
```
Korean is timed by the syllable instead, at --spm (default 300). A line that
mixes the two pays for both.
```

**31행 `CONCRETE_RE`**를 아래 블록으로 교체한다(caption.py와 동일한 `KO_NUMBER`를 복제하고, `SYLLABLE_RE`를 추가):
```python
SYLLABLE_RE = re.compile(r"[가-힣]")
# Korean glues units and counters straight onto the number (27kg, 3가지, 5분),
# so the word boundary the English pattern ends on never comes. These only
# fire next to Hangul, which is what keeps English results where they were.
KO_NUMBER = (r"(?<![\d.,])\d[\d,.]*(?=[A-Za-z%]{0,3}\s?[가-힣])"
             r"|(?<=[가-힣])\d[\d,.]*|(?<=[가-힣]\s)\d[\d,.]*"
             r"|(?<![가-힣])(?:두|세|다섯|여섯|일곱|여덟|아홉|열)\s?(?:가지|번|명|개|살|마리|달)")
CONCRETE_RE = re.compile(r"\$\s?\d|" + KO_NUMBER + r"|\b\d[\d,.]*\b|(?<!^)\b[A-Z][a-z]{2,}\b", re.MULTILINE)
```
`WORD_RE`(29행)와 `words()`는 **바꾸지 않는다**. 영어 단어는 여기서 세고, 한글은 `SYLLABLE_RE`로 따로 센다. 그래서 한 줄 안에서 이중 계수가 생기지 않는다.

**`words()` 다음, `pretty()` 앞**에 함수를 추가한다:
```python
def seconds(text, wps, sps):
    """English is timed by the word, Korean by the syllable. A mixed line pays both."""
    return len(words(text)) / wps + len(SYLLABLE_RE.findall(text)) / sps


```

**`split_beats`:** 시그니처를 `def split_beats(raw, wps, sps):`로 바꾸고, 66행 `len(words(line)) / wps`는 `seconds(line, wps, sps)`, 75행 `len(words(candidate)) / wps`는 `seconds(candidate, wps, sps)`로 바꾼다.

**`analyse` 85–87행:**
```python
def analyse(raw, wpm=165, target=None, spm=300):
    wps = wpm / 60.0
    sps = spm / 60.0
    beats = split_beats(raw, wps, sps)
    # A Korean script gets cut and padded in syllables, an English one in words.
    ko = bool(SYLLABLE_RE.search(raw))
    unit, rate = ("syllables", sps) if ko else ("words", wps)
```
- 94행 `dur = n / wps` → `dur = seconds(text, wps, sps)`. `n = len(words(text))`과 row의 `"words": n`은 그대로 둔다.
- 122행 `f"{int(HOOK_WINDOW * wps)} words or fewer, ...` → `f"{int(HOOK_WINDOW * rate)} {unit} or fewer, or the hook lands after "`
- 164행 → `notes.append(f"{delta:.1f}s over target. Cut about {int(delta * rate)} {unit}.")`
- 166행 → `notes.append(f"{-delta:.1f}s under target. Either add {int(-delta * rate)} {unit} "`

**169–175행 `return {...}`** 교체(키 순서 유지, 한국어일 때만 키를 끝에 추가):
```python
    out = {
        "wpm": wpm, "target": target,
        "total_seconds": round(total, 2),
        "total_words": sum(r["words"] for r in rows),
        "beats": rows,
        "notes": notes,
    }
    if ko:
        for r in rows:
            r["syllables"] = len(SYLLABLE_RE.findall(r["text"]))
        out["spm"] = spm
        out["total_syllables"] = sum(r["syllables"] for r in rows)
    return out
```

**`render` 179–180행 `head`** 교체(영어에서는 문자열이 동일):
```python
    syl = a.get("total_syllables")
    head = (f"BEAT SHEET  ·  {a['total_words']} words" + (f" + {syl} syllables" if syl else "")
            + f"  ·  ~{a['total_seconds']:.1f}s at {a['wpm']} wpm" + (f" / {a['spm']} spm" if syl else "")
            + (f"  ·  target {a['target']}s" if a["target"] else ""))
```

**`main`:** `--wpm` 다음, `--target` 앞에 인자를 추가하고 `analyse` 호출을 바꾼다.
```python
    ap.add_argument("--spm", type=float, default=300,
                    help="Korean speaking rate, syllables per minute (default 300)")
```
```python
    a = analyse(raw, wpm=args.wpm, target=args.target, spm=args.spm)
```

**CLI 동작 정의:**
- `--wpm`은 ASCII 단어에만, `--spm`은 한글 음절에만 적용된다. 둘은 독립이고, 섞인 줄은 두 시간을 합산한다.
- 영어 스크립트에서 `--spm`은 무시된다(음절 0 → 기여 0). 출력과 JSON 모두 변화가 없다.
- `--wpm`으로 spm을 유도하지 않는다. 언어별 속도 비율이 화자마다 달라 근거가 없다.

**기본 SPM 300의 근거:**
- 문헌과 가이드에서 흔히 인용되는 한국어 발화 속도는 대략 다음과 같다(오프라인에서 작성해 원문 미확인).
  - 낭독체: 약 4.5–5.5음절/초(≈270–330 SPM)
  - 방송 뉴스 아나운서: 약 5.5–6음절/초(≈330–360 SPM)
  - 발표 권장 속도: 분당 250–300음절
- 300 SPM(5음절/초)은 이 범위의 중간 아래다. 영어 기본값 165 wpm이 "150–200" 범위의 중간 아래인 것과 같은 위치다.
- 약간 느리게 잡아야 3초 훅 초과와 4초 비트 초과를 놓치지 않는다. 이것이 이 도구의 목적("편집 계획용 추정치")에 맞다.
- docstring의 "직접 재고 `--spm`으로 맞추라"는 기존 `--wpm` 안내와 같은 톤이다.

**ko-beats 기대 결과 (실측, 300 SPM):**
- 헤더: `BEAT SHEET  ·  5 words + 92 syllables  ·  ~20.2s at 165 wpm / 300 spm  ·  target 30.0s`
- 비트 길이: 4.5 / 5.2 / 4.6 / 3.4 / 2.6초
- 안내: 훅 4.5초 초과 플래그, "Cut it to 15 syllables or fewer", 1–3번 비트 4초 초과, Loops "27kg", "9.8s under target. Either add 48 syllables"
- 이전 결과 1.8초는 비현실적이었다. 이후 20.2초는 실제 낭독(약 100음절 이상)과 같은 자릿수다.

#### B3. swipe.py (범위 내, 1줄)

**37행:**
```python
WORD_RE = re.compile(r"[A-Za-z0-9$%'’가-힣ㄱ-ㆎ-]+")
```
- hookscore와 똑같은 문자열이다. `median hook length`가 한국어 어절 수로 정상 계산된다.
- `hook_score`는 import로 자동 반영된다.
- `classify`와 hooks.json은 그대로 둔다. 한국어 훅은 `unclassified`로 나오고, 스크립트가 이미 "Read those by hand"라고 안내한다.

### Writer C (ig-human) — `skills/ig-human/detect.py`, `skills/ig-human/humanize.py`, `skills/ig-human/slop.json`, `README.md`

#### C1. detect.py

**36행** 교체, 다음 줄에 상수를 추가한다:
```python
WORD_RE = re.compile(r"[A-Za-z'가-힣ㄱ-ㆎ]+")
HANGUL_RE = re.compile(r"[가-힣]")
```
**39행 `NUMBERS`** 교체(한국어 대안을 앞에 두고, 영어 대안은 그대로):
```python
NUMBERS = re.compile(r"(?<![\d.,])\d[\d,.]*(?=[A-Za-z%]{0,3}\s?[가-힣])|(?<=[가-힣])\d[\d,.]*|(?<=[가-힣]\s)\d[\d,.]*"
                     r"|\b\d[\d,.]*%?\b|\$\d")
```
(detect에는 영어에도 수사 단어 계수가 없으므로 한국어 고유어 수사도 넣지 않는다. caption/beats의 KO_NUMBER에서 ④ 대안을 뺀 것과 같다.)

**`PROPER`·`CONTRACTIONS`·`PRONOUNS`는 변경하지 않는다(영어 전용).**
- 한국어에는 대문자도 아포스트로피 축약도 없다.
- 한국어는 주어·인칭대명사를 일상적으로 생략한다(pro-drop). 그래서 영어 기준(사람 글 = 100단어당 8개)을 적용하면 사람이 쓴 한국어도 기계로 판정된다. 한국어 전용 임계값은 근거 없이 만들어야 하므로 만들지 않는다.

**`check_slop` 앞**에 함수를 추가한다(주석 포함):
```python
def slop_pattern(entry):
    """English entries match whole words. Korean glues particles onto the end
    of a word (게임체인저를), so a Korean entry only needs its front edge clear,
    unless it is deleted outright: then it has to end where the word ends, or
    the leftover ending (멈춰요 -> 요) would be stranded in the sentence."""
    find = entry["find"]
    if not HANGUL_RE.search(find):
        return re.compile(r"\b" + re.escape(find).replace(r"\ ", r"\s+") + r"\b", re.IGNORECASE)
    tail = r"(?![가-힣])" if not entry["replace"] else ""
    return re.compile(r"(?<![가-힣])" + re.escape(find).replace(r"\ ", r"\s*") + tail, re.IGNORECASE)


```
**`check_slop` 92–94행**의 `pattern = re.compile(...)` 두 줄과 `n = len(pattern.findall(text))`를 한 줄로 교체한다:
```python
        n = len(slop_pattern(entry).findall(text))
```

**경계 규칙:**
- 영어 항목: 기존 `\b…\b` + `\s+`를 바이트 그대로 유지한다.
- 한국어 항목(`find`에 한글 포함):
  - 앞쪽은 `(?<![가-힣])`. 앞이 공백, 문두, 구두점, ASCII면 허용한다.
  - 띄어쓰기는 `\s*`. 한국어 SNS 글은 띄어쓰기가 들쭉날쭉하다(`스크롤멈추세요`).
  - 뒤쪽은 두 경우로 나뉜다.
    - 치환 항목(`replace` 비어 있지 않음): 열어 둔다. 조사나 `하다` 활용이 그대로 남아 치환어에 붙는다(`게임체인저를` → `큰 변화를`).
    - 삭제 항목(`replace == ""`): `(?![가-힣])`. 항목이 어절 끝에서 끝나야 한다. `스크롤 멈추세요요`처럼 어미가 다른 경우 매칭하지 않아서, `요`만 남는 잔해가 생기지 않는다.
  - 이 구분은 기존 필드(`replace`)에서 유도하므로 **스키마 변경이 없다**.
- detect와 humanize는 같은 규칙을 써서 탐지 결과와 치환 결과가 일치한다.

**`check_voice`:**
- 128행 `person = ...` 다음에 추가한다:
```python
    # Korean has no apostrophe contractions and drops pronouns as a matter of
    # course, so neither count says anything about it. Score them as unknown.
    ko = sum(1 for x in w if HANGUL_RE.search(x)) * 2 > len(w)
```
- 141–143행 `score = (...)` 교체:
```python
    score = ((50.0 if ko else scale(contractions, human=3.0, machine=0.0)) * 0.35
             + (50.0 if ko else scale(person, human=8.0, machine=1.0)) * 0.35
             + clamp(100 - tells * 22) * 0.30)
```
- 147–148행 `detail = (...)` 바로 다음(`if names:` 앞)에 추가한다:
```python
    if ko:
        detail = f"contractions and pronouns not scored for Korean, {tells} structural tell(s)"
```
- 판정 기준은 "한글이 들어간 토큰이 과반"이다. 한글이 섞인 영어 글은 영어로 채점한다. 중립값 50은 이 파일의 "too short to judge" 관례와 같다.

**ko-detect 기대 결과 (실측):**
- SPECIFICITY 0.0 (0 markers / 27 words. 초안에 실제로 숫자가 없음)
- SLOP DENSITY 0.0 ("7 stock terms, 25.9 per 100 words (게임체인저, 마인드셋, 스크롤 멈추세요, 오늘 영상에서는, ...)")
- FINGERPRINT 0.0 (em dash)
- VOICE 38.6 ("contractions and pronouns not scored for Korean, 4 structural tell(s) [ko-not-just, ko-in-this-video, ko-follow-bait]")
- HUMAN SCORE 10.6 FLAGGED
- 참고: 사람이 쓴 ko-caption을 detect에 넣으면 50.0 REVIEW(SLOP 100, FINGERPRINT 100, VOICE 65)다.

#### C2. humanize.py

**40행 `SENT_RE` 다음** 줄에 추가한다:
```python
HANGUL_RE = re.compile(r"[가-힣]")
```
**`pass_lexical` 136–137행**의 `pattern = re.compile(...)` 두 줄을 교체한다(영어 분기는 들여쓰기만 한 단계 깊어지고 식은 그대로):
```python
        if not HANGUL_RE.search(find):
            pattern = re.compile(r"\b" + re.escape(find).replace(r"\ ", r"\s+") + r"\b",
                                 re.IGNORECASE)
        else:
            # Korean glues particles onto the end of a word (게임체인저를), so
            # only the front edge needs to be clear and the particle stays on the
            # replacement. A deletion has to end where the word ends, or the
            # leftover ending (멈춰요 -> 요) is stranded in the sentence.
            tail = r"(?![가-힣])" if not entry["replace"] else ""
            pattern = re.compile(r"(?<![가-힣])" + re.escape(find).replace(r"\ ", r"\s*") + tail,
                                 re.IGNORECASE)
```
- `_match_case`, `restore_capitals`, 후처리 정리 정규식은 변경하지 않는다. 한글에는 대소문자가 없어 `src[0].isupper()`가 False이므로 치환어가 그대로 들어간다.
- 영어 정렬(`len(find)` 내림차순, stable)은 그대로다. 한국어 항목은 ASCII에 매칭되지 않으므로 영어 치환 순서와 결과에 영향이 없다.

**ko-humanize 기대 결과 (실측):** 8 artefacts(em dash 1 + 어휘 7), 구조 1개(`"단순한 X가 아니라 Y"`). 출력:
```
여러분의 육아를 개선해 줄 큰 변화를 소개해 드릴게요, 이건 단순한 팁이 아니라 마음가짐입니다.!
```
(끝의 `.!`는 영어와 같은 기존 후처리 동작이다(`mindset.!`). 범위 외라 수정하지 않는다.)

#### C3. slop.json — 정확한 추가 항목

- 방법: `d = json.load(...)` → 아래 항목을 각 배열 **끝에** `append` → `json.dump(d, fh, indent=2, ensure_ascii=False)` + `"\n"`. 기존 포맷과 바이트 호환이 검증되어 있다. 손으로 편집해도 되지만 결과는 같아야 한다.
- `version`은 `"1.0"` 그대로 둔다.

**`note`** 끝에 한 문장을 덧붙인다(기존 문장 끝 `... it is meant to be yours.` 뒤에 공백 하나 후):
```
Korean entries sit at the end of each list. They match with the particle still attached (게임체인저를), so a replacement has to end the same way the original does, with or without a final consonant, or the particle after it stops fitting.
```

**`words` 끝에 추가 (7개):**
```json
{"find": "게임체인저", "replace": "큰 변화", "family": "nouns"},
{"find": "마인드셋", "replace": "마음가짐", "family": "nouns"},
{"find": "치트키", "replace": "노하우", "family": "nouns"},
{"find": "혁신적인", "replace": "새로운", "family": "adjectives"},
{"find": "획기적인", "replace": "새로운", "family": "adjectives"},
{"find": "결론적으로", "replace": "결국", "family": "connectives"},
{"find": "궁극적으로", "replace": "결국", "family": "connectives"}
```
**`phrases` 끝에 추가 (9개):**
```json
{"find": "한 단계 업그레이드", "replace": "개선", "family": "verbs"},
{"find": "오늘 영상에서는", "replace": "", "family": "openers"},
{"find": "이번 영상에서는", "replace": "", "family": "openers"},
{"find": "이 영상에서는", "replace": "", "family": "openers"},
{"find": "스크롤 멈추세요", "replace": "", "family": "openers"},
{"find": "스크롤을 멈추세요", "replace": "", "family": "openers"},
{"find": "팔로우하고 더 많은 꿀팁 받아가세요", "replace": "", "family": "closers"},
{"find": "필요한 친구를 태그해 주세요", "replace": "", "family": "closers"},
{"find": "필요한 친구에게 공유해 주세요", "replace": "", "family": "closers"}
```
**`structures` 끝에 추가 (4개).** JSON 안이므로 역슬래시는 이중으로 쓴다.
```json
{"id": "ko-not-just", "regex": "(?:단순한|단순히|그냥|단지)\\s*[^.!?\\n]{1,40}?\\s*아니라", "name": "\"단순한 X가 아니라 Y\" (Korean \"not just X, it's Y\")", "fix": "Say the one thing you actually mean. Delete the setup half."},
{"id": "ko-stop-scroll", "regex": "(?m)^\\s*(?:잠깐만?요?[,.!]?\\s*)?스크롤\\s*(?:을\\s*)?(?:멈|그만|내리지)", "name": "\"스크롤 멈추세요\" opener", "fix": "The scroll-stop has to be the claim itself. Asking for attention is the tell that you have not earned it."},
{"id": "ko-in-this-video", "regex": "(?<![가-힣])(?:이|이번|오늘)\\s*(?:영상|릴스)\\s*에서|에\\s*대해\\s*(?:알아볼게요|알아보겠습니다|이야기해\\s*볼게요)", "name": "Video preamble (\"이 영상에서는\", \"~에 대해 알아볼게요\")", "fix": "Delete it and start at the payoff. The first two seconds decide the whole reel."},
{"id": "ko-follow-bait", "regex": "팔로우하고\\s*더\\s*많은|필요한\\s*(?:친구|사람|분)(?:을|를|에게|한테)?\\s*(?:태그|공유)|공감(?:되면|하시면|하면)\\s*(?:좋아요|하트)", "name": "Reflex follow or tag bait (Korean)", "fix": "Ask for the one action this post actually earns. The generic ask trains the audience to ignore every ask."}
```
- `fix` 문구는 대응하는 영어 structure(`not-just`, `stop-scroll`, `in-this-video`, `follow-bait`)의 문장을 그대로 재사용했다. 출력 리포트가 영어이므로 톤을 맞춘 것이다.

**받침 검증 (치환이 조사를 보존하는 근거):**

| find → replace | 끝 음절 받침 | 실측 |
|---|---|---|
| 게임체인저 → 큰 변화 | 없음 / 없음 | `게임체인저를` → `큰 변화를`, `게임체인저가` → `큰 변화가` |
| 마인드셋 → 마음가짐 | ㅅ / ㅁ (둘 다 받침, ㄹ 아님) | `마인드셋이` → `마음가짐이`, `마인드셋으로` → `마음가짐으로` |
| 치트키 → 노하우 | 없음 / 없음 | `치트키는` → `노하우는` |
| 혁신적인·획기적인 → 새로운 | 관형형(조사 없음) | — |
| 결론적으로·궁극적으로 → 결국 | 부사(조사 없음) | `결론적으로,` → `결국,` |
| 한 단계 업그레이드 → 개선 | 하다-어근 | `한 단계 업그레이드해 줄` → `개선해 줄`, `…업그레이드된` → `개선된` |

**넣지 않은 후보와 이유(오탐 회피):**
- `꿀팁`: 사람이 쓴 한국어 캡션에서도 매우 흔하다. 긴 구 `팔로우하고 더 많은 꿀팁 받아가세요` 안에서만 잡는다.
- `업그레이드` 단독: `업그레이드된 사료` 같은 일상 용법이 있다. 검증: 미검출.
- `인사이트`: 받침이 맞는 자연스러운 치환어가 없다(배운 점/교훈/깨달음은 모두 받침 있음).
- `꿀팁 대방출`: ㄹ받침 치환어가 없다.
- `필수템`, `인생템`, `역대급`, `레전드`: 일상 구어다.
- `놓치지 마세요`, `결론부터 말씀드리면`: 삭제하면 문장 잔해가 남는다.
- `~에 대해 알아볼게요`: 앞에 가변 명사가 붙으므로 lexical이 아니라 structure(`ko-in-this-video`)로 보고만 한다.
- `소개해 드릴게요`, `알려드릴게요`: `친구 소개해 드릴게요`, `댓글 주시면 알려드릴게요` 같은 일상 용법 때문에 structure에서도 제외했다(hookscore 첫 줄에서만 서두로 본다).

**오탐 실측(모두 미검출):** `우디가 업그레이드된 사료를 먹어요. 결론은 우디가 최고예요. 스크롤을 내리다 보니 …`, `이 영상에 나온 장난감은 …`, `오늘 영상 찍느라 힘들었어요`, `스크롤 멈춰요 같은 건 안 써요`(문두 아님), `필요한 친구는 없어요`. (`그냥 산책이 아니라 훈련이에요`는 ko-not-just로 보고된다. 영어 not-just가 사람의 글에도 걸리는 것과 같은 동작이며, 보고만 하고 수정하지는 않는다.)

#### C4. README.md (짧게, 저자 톤, 영어)

- 위치: `### The swipe file` 섹션의 끝, `## What I actually measured, which is the part worth reading`(222행) **바로 앞**에 새 하위 섹션으로 넣는다.
- 그 외 README 문장, 특히 169행 "154 stock words and phrases"는 수정하지 않는다. 영어 목록 기준으로 여전히 사실이다.
```markdown
### Korean

The five tools read Korean too. `#애개육아` counts as a hashtag, `27kg` and
`3가지` count as numbers, `댓글로 알려주세요` counts as the ask, and
`안녕하세요` gets flagged as a greeting instead of quietly scoring low.
`beats.py` times Korean by the syllable, at 300 a minute by default; time
yourself and pass `--spm`. The humanizer has a short Korean block at the end
of `slop.json` that matches with the particle still attached, so 게임체인저를
comes out as 큰 변화를.

What it cannot do: spot a name, because Hangul has no capital letters, or put
a Korean hook into one of the 26 formulas. Those come back unclassified. The
English results have not moved.
```

### 영어 회귀 체크리스트

리뷰어와 TestRunner가 항목별로 확인한다. 괄호 안은 영어 불변 근거다.

- [ ] caption `HASHTAG_RE`: 클래스에 한글만 추가했다(ASCII 매칭 집합 동일).
- [ ] caption/beats `CONCRETE_RE`: `KO_NUMBER` 3개 대안은 모두 `[가-힣]` lookahead/lookbehind나 한글 리터럴이 필요하다. ASCII에서는 위치마다 실패하고 기존 대안으로 진행하므로 findall 결과가 동일하다. `$18,000` → `['$1','000']` 기존 동작도 유지된다.
- [ ] caption `ASKS`: 8개 모두 `|한글…` 대안만 추가했다. 이름·순서·개수가 같다. 한국어 대안 안의 `\bdm`은 `\s?(?:주세요|…)`(한글)가 필요하므로 영어 "dm"만으로는 매칭되지 않는다.
- [ ] hookscore `WORD_RE`, swipe `WORD_RE`: 클래스에 한글만 추가했다.
- [ ] hookscore `SPOKEN_NUMBERS`: 한글 문자열 원소만 추가했다(영어 토큰과 같아질 수 없음).
- [ ] hookscore `WEAK_OPENERS`: 한글 원소만 추가했다. `opener.startswith("안녕")`류는 ASCII opener에서 항상 False이고, 리스트 순서상 영어 항목이 먼저 검사되므로 `hit_opener`도 동일하다.
- [ ] hookscore `KO_STAKES_RE.findall(text)`: ASCII에서는 `[]`이므로 `hits`와 `markers` 정렬 결과가 동일하다.
- [ ] hookscore 가격 정규식: `₩`/`원` 대안은 비ASCII가 필요하다.
- [ ] hookscore frontload `or KO_STAKES_RE.search(w[i])`, address `or KO_YOU_RE/KO_ME_RE`: ASCII에서는 None이므로 분기가 동일하다.
- [ ] hookscore DEALBREAKERS: 3개 정규식의 영어 부분이 바이트 그대로이고, 대안은 한글이 필요하다. preamble 정규식은 문자열 연결(`"..."\n r"|..."`)로 영어 부분을 손대지 않았다.
- [ ] beats `seconds()`: ASCII에서는 `len(words)/wps + 0/sps`이고 IEEE상 `x + 0.0 == x`이므로 dur·clock·total이 비트 단위로 동일하다.
- [ ] beats `ko=False`이면 `unit, rate = "words", wps`이므로 안내문 3개 문자열이 동일하다.
- [ ] beats JSON 추가 키는 `if ko:`에서만 붙는다. head는 `syl`이 None이므로 기존 문자열과 동일하다(검증: `en_old.json` = `en_new.json`).
- [ ] beats `--spm` 인자 추가는 `--help` 출력만 바뀐다(의도된 변경이며 하네스는 확인하지 않음).
- [ ] detect `WORD_RE`: ASCII 매칭 집합이 동일하다.
- [ ] detect `NUMBERS`: 한국어 대안을 앞에 두었고 ASCII에서는 실패하므로 동일하다.
- [ ] detect/humanize slop 패턴: `HANGUL_RE.search(find)`가 False인 영어 항목은 기존 식과 문자 하나 다르지 않다. 한국어 항목 패턴은 한글 리터럴을 포함하므로 ASCII 텍스트에 매칭될 수 없다.
- [ ] detect voice `ko`: ASCII 토큰에는 한글이 없으므로 False이고, score와 detail이 동일하다.
- [ ] slop.json: 기존 항목의 순서와 내용이 불변이고, 추가분은 배열 끝에만 있다. `note` 텍스트 변경은 출력에 나타나지 않는다(어떤 스크립트도 note를 출력하지 않음). 신규 structures 4개의 정규식은 모두 한글 리터럴을 포함하므로 영어 텍스트의 tells와 flags가 동일하다. JSON 유효성은 `python3 -I -c "import json;json.load(open('skills/ig-human/slop.json'))"`로 확인한다.
- [ ] 실행 확인:
  - 하네스: `en-*` 5개 파일을 `cmp`해 동일해야 한다.
  - 광역 스윕: `python3 -I /private/tmp/claude-501/-Users-jh-workspace-insta/842875ae-9eef-4ee6-acaa-7481ae0210d2/scratchpad/analyzer/regress.py <원본 skills 복사본> $PWD/skills $PWD`에서 `diffs: 0`이어야 한다. 원본 skills는 HEAD에서 꺼내 쓴다. 예: `git archive HEAD skills | tar -x -C <tmp>` 후 `<tmp>/skills`를 첫 인자로 준다.

### 범위 외 항목

- **hooks.json 26개 `match`의 한국어화:** 공식마다 한국어 표현 대안을 설계해야 해서 소규모가 아니다. 한국어 훅은 `unclassified`로 표시된다. swipe가 이미 "손으로 읽으라"고 안내한다.
- 한국어 고유명사 탐지(이름 사전 필요), 한국어 명령형 어미(`-세요`)의 ADDRESS 90점 처리(`안녕하세요`와 충돌), 한국어 MONEY_WORDS(`월급`, `연봉`. 조사 결합 문제가 있고 숫자로 대부분 잡힘).
- beats의 ASCII 숫자·단위 토큰(`27kg`)을 한국어 낭독 음절로 환산하는 것, 한국어 어절 기반 루프 탐지(조사 차이 때문에 형태소 분석 필요).
- caption 박스 렌더링의 전각 문자 정렬(`textwrap`과 `:<width>`가 문자 수 기준), 한국어 `TRUNCATE` 값 재검증.
- detect의 한국어 전용 contractions·pronouns 대체 지표(구어체 어미 등). 근거 있는 임계값이 없어서 중립 처리했다.
- 기존 버그: humanize 삭제 뒤 `".!"` 잔여, beats `HOOK/CTA` 라벨 공백, caption `$18,000` 2개 계수. 영어에도 존재하며, 고치면 영어 출력이 바뀐다.
- 다른 스킬의 SKILL.md 문서 번역이나 한국어 안내 추가.
