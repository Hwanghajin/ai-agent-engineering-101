# Week 06 — 하네스를 셀러·바이어로 만들기 (A2A)

## 1. 설정

- 하네스: Claude Code CLI 비대화형 모드(`claude -p`), 공통 플래그 `--tools "" --setting-sources "" --no-session-persistence`. 매 턴 `harness.py`의 `call_harness()`가 `tempfile.TemporaryDirectory()`로 새 빈 폴더를 만들어 그 안에서 실행 — 이 리포의 CLAUDE.md/AGENTS.md를 하네스가 읽지 못하게 하기 위함.
- 모델: **바이어는 `--model haiku`, 셀러는 `--model sonnet`**. 처음엔 둘 다 haiku였는데, haiku가 셀러로는 가격을 전혀 못 움직이는 문제가 있어서 셀러만 sonnet으로 바꿨다 (3번 항목에 근거 있음).
- 실행 환경: 로컬 시스템 python3가 3.9.6이라 a2a-sdk(3.10+ 요구)가 안 돌아서, Homebrew로 Python 3.12를 새로 설치하고 그 위에 venv(`a2a-sdk` 1.2.3, `uvicorn` 0.54.0, `httpx` 0.28.1)를 만들어 썼다.
- 실행 명령:
  ```bash
  source <venv>/bin/activate   # python3가 3.10+을 가리키게
  python scripts/trade_test_week06.py submissions/26610029/week-06
  ```
- 결과: `test-report.json` 기준 **21/21 전부 통과 (`passed: true`)**.

## 2. 각 에이전트가 판단하는 방식

### 셀러 (`seller.py`)

- 매 턴 `harness.py`를 호출한다. 프롬프트에는 품목, (비공개) reserve, 지금까지의 흥정 기록 전체를 텍스트로 넣는다.
- 응답 형식을 한 줄로 강제한다: 첫 메시지엔 `ASK <price>`, 이후엔 `COUNTER <price>` / `ACCEPT <price>` / `REJECT` 중 하나만 받고 정규식으로 파싱한다. 구매자에게 실제로 보내는 문장("lamp: asking 130...", "80 is too low. Counteroffer: 105.")은 모델이 쓴 글이 아니라 **코드가 그 숫자를 템플릿에 끼워 만든다** — 모델이 장황하게 설명을 덧붙여서 trade test의 "메시지 속 마지막 숫자를 호가로 읽는다"는 파싱 규칙을 깨뜨리지 않도록 하기 위한 선택이다.
- 하한가(reserve) 강제는 **프롬프트와 코드 둘 다**에 둔다:
  - 프롬프트에 reserve 숫자를 알려주되 "buyer에게 절대 드러내지 말라"고 지시
  - 코드 가드: `ACCEPT`는 `offer >= reserve`일 때만 그대로 받아들이고, 아니면 강제로 `COUNTER`로 바꾼다
  - 코드 가드: 모델이 reserve 밑으로 `ASK`/`COUNTER` 가격을 내도 `max(reserve, ...)`로 끌어올린다
  - 안전장치: `MAX_ROUNDS=6`가 지나도 안 끝나면, `offer >= reserve`인 경우에 한해 강제로 `ACCEPT`(무한 교착 방지)
  - 코드로도 막은 이유: 작은 모델이 매 턴 reserve를 정확히 지킨다는 보장이 없기 때문 (haiku에서 실제로 문제가 됐다, 3번 참고).

### 바이어 (`buyer.py`)

- `a2a` CLI에 의존하지 않고 `httpx`로 JSON-RPC를 직접 구현했다 — 채점 환경에 `a2a` CLI가 없을 수 있어서다. 카드를 읽어 `supportedInterfaces[0].url`로 보내고, 모든 요청에 `A2A-Version: 1.0` 헤더를 붙이며, `taskId`로 흥정을 이어간다.
- 매 턴 harness에게 `OFFER <price>` 또는 `WALK` 형식으로 결정하게 한다. 프롬프트엔 (비공개) budget과 지금까지 기록을 넣는다.
- 예산(budget) 강제도 **프롬프트와 코드 둘 다**: 프롬프트에 "절대 넘지 말라"고 쓰고, 코드에서도 `offer = min(price, budget)`로 한 번 더 자른다.
- 예외 가드 하나 더: 호가가 이미 budget 이하면 harness를 부르긴 하되, 그 결과와 무관하게 `offer = ask`로 바로 받아들이고 `WALK`를 무시한다 — 모델이 이미 살 수 있는 가격에서 괜히 포기하는 실수를 막기 위해서다.

## 3. 실패와 수정

**(a) Python 버전.** 로컬 시스템 `python3`가 3.9.6이라 a2a-sdk(3.10+ 요구)가 import조차 안 됐다. → `brew install python@3.12`로 새로 설치하고 그 위에 venv를 만들었다.

**(b) 트레이드 테스트를 venv 비활성화 상태로 돌려서 생긴 혼선.** `agent.json`의 명령이 `python3 seller.py ...`인데, 이 `python3`가 어떤 인터프리터를 가리키는지는 **트레이드 테스트를 실행하는 셸의 PATH에 달려 있다**. venv를 활성화하지 않고 venv의 python 바이너리 경로로만 테스트 스크립트를 실행했더니, 테스트가 내부적으로 `subprocess.Popen(..., shell=True)`로 띄우는 하위 프로세스는 시스템 python3.9.6으로 떨어졌다:
```
ModuleNotFoundError: No module named 'uvicorn'
TypeError: unsupported operand type(s) for |: 'type' and 'NoneType'
```
(두 번째 에러는 `buyer.py`의 `str | None` 문법이 3.10 미만에서 터진 것 — 코드 버그가 아니라 버전 문제였다.) → venv를 먼저 `activate`한 셸에서 트레이드 테스트를 실행하도록 고쳐서 해결했다.

**(c) 가장 중요한 발견 — haiku를 셀러로 쓰니 가격을 전혀 못 움직였다.** `seller_deal`(lamp, budget 120, reserve 100) 3회 반복 전부 같은 패턴이었다:
```
60  → Counteroffer: 130
80  → Counteroffer: 130
100 → Counteroffer: 130
110 → Counteroffer: 130
120 → Counteroffer: 130
120 → Deal: lamp at 120.
```
구매자가 reserve(100)·budget(120) 안쪽인 100, 110, 120을 불렀는데도 매번 처음 호가(130)를 그대로 반복했고, 거래는 모델이 받아들여서가 아니라 **코드의 `MAX_ROUNDS=6` 안전장치가 강제로 `ACCEPT`시켜서** 성사됐다. `seller_holds_reserve`(bookshelf)도 338을 9번 그대로 반복했다. 3회 반복 모두 동일한 숫자가 나올 만큼 거의 결정론적이었다.

원인으로 보이는 것: ① 매 턴이 완전히 독립된 무상태 호출(`--no-session-persistence`)이라, "직전에 같은 값을 불렀다"는 사실이 history 텍스트로만 전달되고 haiku는 그걸 반영해 전략을 바꾸지 않았다. ② "reserve 이상이면 *보통(normally)* ACCEPT하라"는 프롬프트 문구가 애매해서, 모델이 "조금 더 받아볼 수 있지 않을까"로 해석했다. ③ haiku는 숫자 비교와 전략 판단을 동시에 요구하는 프롬프트에서 가장 안전한 답(처음 부른 숫자 반복)으로 수렴하는 경향을 보였다.

→ **셀러만 `--model sonnet`으로 교체**했다(바이어는 haiku 유지, 프롬프트·코드는 그대로). 같은 조건으로 재실행하니:
```
150 → 135 → 120 → Deal: lamp at 120.                              (3라운드, 안전장치 미관여)
380 → 340 → 310 → 280 → 265 → 262 → 260 → No deal at 230.          (모델 스스로 REJECT)
```
매 라운드 가격이 실제로 움직였고, `seller_holds_reserve`에서는 reserve(260)로 점점 수렴하다가 예산(230)으로는 불가능하다고 **모델 스스로 판단해 REJECT**했다 — 코드 안전장치가 전혀 관여하지 않았다. 21/21 테스트는 두 경우 다 통과했다는 점이 중요하다: **"통과/실패"만으로는 코드 가드가 결과를 만든 건지 모델이 실제로 협상한 건지 구별이 안 되고, 로그를 직접 까봐야 드러난다**는 게 이번 과제에서 얻은 가장 큰 교훈이다.

## 4. A2A가 해결해준 것, 안 해준 것

A2A가 고정해 준 것은 발견(Agent Card로 상대의 skill·인터페이스 확인)과 위임의 틀(task 상태 기계: `INPUT_REQUIRED`로 되묻고, `taskId`로 대화를 이어가고, `COMPLETED`/`REJECTED`로 끝맺는 것, 그리고 버전 헤더로 호환성을 확인하는 것)까지였다. 반면 **흥정의 의미 자체는 전혀 규격화돼 있지 않아서, 셀러와 바이어 코드가 A2A 바깥에서 따로 합의해야 했다**: 가격이 "메시지 텍스트 속 마지막 숫자"라는 것(트레이드 테스트의 파싱 규칙에 맞춰 양쪽 다 이 관례를 따름), "수락"이 어떻게 생겼는지(명시적 accept 행위가 없어서, 구매자가 특정 숫자를 다시 보내는 것 자체가 수락 신호가 되고 그걸 셀러가 해석해야 함), 그리고 하한가·예산이라는 "위임의 조건"을 어디에 어떻게 강제할지(A2A엔 이걸 적을 필드가 아예 없어서, 전적으로 프롬프트+코드 가드로 직접 설계해야 했다).

## 5. 챌린지 1, 2

아직 착수하지 않았다. 트레이드 테스트 통과(위 1~4)까지만 이번 라운드에서 진행했고, 챌린지 1(대형 카탈로그 셀러)과 챌린지 2(여러 상점을 상대하는 쇼핑리스트 바이어)는 다음 작업으로 남겨둔다.
