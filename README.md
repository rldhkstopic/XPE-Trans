# 초보자 마을에서 온 편지

폰에서 열어두는 트릭스터풍 방치형 모험 게임. 초보자 마을에서 출발한 신출내기
모험가가 당신이 자리를 비운 동안에도 계속 걷고 드릴로 발굴하며 기록을 쌓는다.
돌아오면 모험 보고와 모험가가 쓴 편지를 읽고, 귓속말로 대화한다.

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
- 플레이어 행위(방침 변경, 강화, 소포)는 `{t, k, v}`로 기록되어 해당 틱에 적용된다
- 같은 씨앗 + 같은 행위 기록 → 항상 같은 결과. 기기가 달라도 같은 모험가를 만난다
- 400일치(115,200틱) 재현에 약 80ms

필드 6곳(초보자 마을 근교 → 클로버 들판 → 반딧불 숲 → 수정 폐광 → 노을 사막 →
용의 옛 둥지)이 **회차**로 돈다. 마지막 필드를 지나면 다음 회차의 첫 필드에서 다시
시작하고, 회차마다 위험과 소득이 함께 오른다. 끝이 없다.

## Artifact 런타임 기능

- `db` — `save/main`(진행), `save/chat`(교신 기록). localStorage에도 미러링하고,
  db가 없는 화면에서는 localStorage만으로 동작한다
- `sample` — 모험 보고 말미의 「모험가의 편지」와 귓속말 탭의 대화를 생성한다.
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
  console.log(h+"h", "LV"+s.level, "여정"+s.depth, s.locName, s.hp+"/"+s.maxHp, "패배"+s.retreats);
}'
```
