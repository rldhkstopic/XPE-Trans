# 클럭 없는 대륙

폰에서 열어두는 방치형 탐사 게임. 클럭이 멈춘 실리콘 대륙을 탐사체 하나가 홀로
걸어다니며, 당신이 자리를 비운 동안에도 계속 기록을 쌓는다. 돌아오면 부재 보고서를
읽는다.

claude.ai Artifact로 배포된다. `app/index.html`이 게시되는 페이지 본문이다
(게시 시점에 `<!doctype html>…<body>` 껍데기가 씌워지므로 파일에는 본문만 둔다).

## 구조

| | |
|---|---|
| `app/index.html` | 페이지 전체 — 스타일, 마크업, 시뮬레이션, 렌더링 |

의존 라이브러리 없음. 바닐라 JS 한 파일.

## 진행 방식

서버에서 도는 루프가 없다. 저장하는 것은 **씨앗과 행위 기록**뿐이고, 화면을 열 때마다
0틱부터 현재까지를 다시 재현한다.

- 1틱 = 실제 5분. `틱 = floor((now - startedAt) / 300000)`
- 틱마다 `mulberry32(hash32(seed, tick))`로 난수를 만들어 사건을 하나 뽑는다
  (이동 / 회수 / 조우 / 발굴 / 정비 / 관측)
- 플레이어 행위(지시 변경, 개장, 보급)는 `{t, k, v}`로 기록되어 해당 틱에 적용된다
- 같은 씨앗 + 같은 행위 기록 → 항상 같은 결과. 기기가 달라도 같은 탐사체를 만난다
- 400일치(115,200틱) 재현에 약 80ms

지형은 6개 지층이 **순환**한다. 마지막 지층을 지나면 다음 순환의 첫 지층에서 다시
시작하고, 순환마다 위험과 소득이 함께 오른다. 끝이 없다.

## Artifact 런타임 기능

- `db` — `save/main`(진행), `save/chat`(교신 기록). localStorage에도 미러링하고,
  db가 없는 화면에서는 localStorage만으로 동작한다
- `sample` — 부재 보고서 말미의 「탐사체의 기록」과 교신 탭의 대화를 생성한다.
  사용할 수 없으면 템플릿 문장으로 대체되고 게임 자체는 그대로 돌아간다

## 균형 확인

`app/index.html`에서 시뮬레이션 구간만 떼어내 node로 돌린다.

```sh
python3 - <<'PY'
s = open('app/index.html', encoding='utf-8').read()
i, j = s.index('var TICK_MS'), s.index('/* ── 저장 ')
open('/tmp/sim.js', 'w', encoding='utf-8').write(s[i:j] + "\nmodule.exports={simulate};\n")
PY
node -e '
const {simulate} = require("/tmp/sim.js");
const save = {seed:12345, startedAt:0, acts:[{t:0,k:"dir",v:"explore"}]};
for (const h of [9, 24, 240]) {
  const s = simulate(save, h*12);
  console.log(h+"h", "LV"+s.level, "심도"+s.depth, s.locName, s.hp+"/"+s.maxHp, "후퇴"+s.retreats);
}'
```
