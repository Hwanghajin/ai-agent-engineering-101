# Observations — Week 01

## 1. Model selection: README example no longer free

README에 예시로 나온 `meta-llama/llama-3.3-70b-instruct:free`를 처음 시도했으나,
OpenRouter에서 더 이상 무료로 제공되지 않아 404 에러가 발생했다:

> Error code: 404 - 'This model is unavailable for free. The paid version is
> available now - use this slug instead: meta-llama/llama-3.3-70b-instruct'

대신 OpenRouter의 Free Models Router(`openrouter/free`)로 교체하여 해결함 —
요청 특성(tool calling 등)에 맞는 무료 모델을 자동으로 골라주는 라우터.
이후 `AGENT_MODEL=openrouter/free`로 실행.

## 2. Observation: non-deterministic tool calls under ambiguous instructions

동일한 프롬프트("read notes.txt and calculate the total amount to report")를 세 번 실행한 결과, calculator에 전달된 수식이 매번 달랐다:

| 실행 | calculator 수식 | 결과 | 해석 |
|---|---|---|---|
| run1 | 48000 + 9500 - 12000 | 45500 | 환급분을 차감 (맥락 추론) |
| run2 | 48000 + 9500 + 12000 | 69500 | 참석자 수 제외, 나머지 합산 |
| run3 | 4 + 48000 + 9500 + 12000 | 69504 | 지시문 문자 그대로 전부 합산 |

notes.txt의 "위의 숫자들을 모두 더하여"라는 지시가 참석자 수까지 포함하는지,
그리고 "환급받은 금액"을 총액에 더할지 뺄지가 명시되지 않아 생긴 차이로 보인다.
같은 모델·같은 프롬프트에서도 매 실행마다 다른 그럴듯한 해석을 택했으며,
이는 LLM의 확률적 생성 특성과 프롬프트의 모호성이 결합된 결과로 보인다.

세 실행의 원본 콘솔 출력은 `logs/run2.log`, `logs/run3.log`