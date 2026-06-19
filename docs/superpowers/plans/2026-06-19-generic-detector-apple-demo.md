# アプリ汎用化＋りんご測定器デモ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** judge アプリの被写体名を学習時に変更可能にし、公開サイトの同梱デモを著作権クリアな「りんご測定器」に差し替え、公開サイトの地図パネル非表示を修正する。

**Architecture:** 被写体名を `{name}` テンプレ化し localStorage/同梱メタから注入。りんごモデルは Wikimedia の PD/CC0 画像を取得→playwright で `train.html` をブラウザ駆動して学習→成果物を `model/` と `map-data.json` に書き出し。地図は同梱 `map-data.json` をフォールバック読込し公開で常時表示。

**Tech Stack:** バニラ HTML/JS、TensorFlow.js + MobileNet(CDN)、vis-network、playwright(ビルド/検証駆動)、Node 18+ fetch。

## Global Constraints

- 学習画像は **PD / CC0 のみ**。`extmetadata` のライセンスで絞る。それ以外は破棄。
- 被写体名は `{name}` プレースホルダ。表示時 `tpl()` で `subjectName` に置換。
- `subjectName` 既定値 = `'りんご'`(未設定フォールバック)。
- 同梱モデルパスは現状維持: `model/gyopi-classifier/`, `model/gyopi-feature-classifier/`, `model/gyopi-avg.json`(内部名なので改名しない)。
- `map-data.json` は **追跡・公開**。`training_data/`, `train.html` は **非公開のまま**。
- テストは playwright 駆動の `verify_*.mjs`(gitignore 済みパターン、ローカル検証専用)。ローカル静的サーバ越しに検証(file:// は fetch/CORS不可)。
- サーバ: 改善のたびローカルサーバ起動したまま残す(停止したら再起動)。

---

### Task 1: train.html に被写体名入力とクラス名汎用化

**Files:**
- Modify: `train.html`(名前入力欄・保存・クラス名汎用化)
- Create: `verify_serve.mjs`(共有ローカルサーバ helper、gitignore)
- Test: `verify_train_name.mjs`(gitignore)

**Interfaces:**
- Produces: localStorage `subject-name`(string)。train.html の名前入力 `#subject-name-input` の値を学習時に保存。
- Produces: `verify_serve.mjs` が `export function startServer(dir, port=8123)` → `{ url, close }`。

- [ ] **Step 1: 共有サーバ helper を作成**

`verify_serve.mjs`:

```js
import http from 'http';
import { readFile } from 'fs/promises';
import { extname, join, normalize } from 'path';
const MIME = { '.html':'text/html', '.js':'text/javascript', '.json':'application/json', '.bin':'application/octet-stream', '.png':'image/png', '.jpg':'image/jpeg', '.jpeg':'image/jpeg', '.svg':'image/svg+xml' };
export function startServer(dir, port = 8123) {
  const server = http.createServer(async (req, res) => {
    try {
      const urlPath = decodeURIComponent(req.url.split('?')[0]);
      const rel = normalize(urlPath).replace(/^([/\\])+/, '');
      const file = join(dir, rel === '' ? 'index.html' : rel);
      const body = await readFile(file);
      res.writeHead(200, { 'Content-Type': MIME[extname(file)] || 'application/octet-stream' });
      res.end(body);
    } catch { res.writeHead(404); res.end('not found'); }
  });
  return new Promise(resolve => server.listen(port, () => resolve({ url: `http://localhost:${port}`, close: () => server.close() })));
}
```

- [ ] **Step 2: 失敗するテストを書く**

`verify_train_name.mjs`:

```js
import { chromium } from 'playwright';
import { startServer } from './verify_serve.mjs';
const srv = await startServer(process.cwd());
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto(srv.url + '/train.html');
// 名前入力欄が存在する
const input = await page.$('#subject-name-input');
if (!input) throw new Error('FAIL: #subject-name-input が無い');
// 既定値が りんご
const def = await page.inputValue('#subject-name-input');
if (def !== 'りんご') throw new Error('FAIL: 既定名が りんご でない: ' + def);
// 入力→保存(blur/changeで保存される想定)
await page.fill('#subject-name-input', 'ねこ');
await page.dispatchEvent('#subject-name-input', 'change');
const saved = await page.evaluate(() => localStorage.getItem('subject-name'));
if (saved !== 'ねこ') throw new Error('FAIL: subject-name 未保存: ' + saved);
console.log('PASS verify_train_name');
await browser.close(); srv.close();
```

- [ ] **Step 3: テスト実行→失敗確認**

Run: `node verify_train_name.mjs`
Expected: FAIL（`#subject-name-input が無い`）

- [ ] **Step 4: train.html に名前入力欄を追加**

`train.html` の `<h1>🎓 ぎょぴちゃん学習モード</h1>`(91行)直後の `.card` 先頭(94行 `<div class="card">` 直後)に section を追加:

```html
    <div class="section">
      <h2>🏷️ 測定するもの の名前</h2>
      <input type="text" id="subject-name-input" value="りんご"
        style="width:100%;padding:0.7rem;border:2px solid #a5d6a7;border-radius:10px;font-size:1rem;font-family:inherit;">
      <div class="label" style="font-size:0.8rem;color:#757575;margin-top:0.3rem;">この名前で「○○測定器」が作られます</div>
    </div>
```

`<script>` 内(`let yesFiles...` 直後あたり)に保存処理を追加:

```js
    const nameInput = document.getElementById('subject-name-input');
    nameInput.value = localStorage.getItem('subject-name') || 'りんご';
    nameInput.addEventListener('change', () => {
      localStorage.setItem('subject-name', nameInput.value.trim() || 'りんご');
    });
```

- [ ] **Step 5: 学習時に名前を保存(startTraining 内)**

`startTraining()` の `localStorage.setItem('gyopi-avg-features', ...)`(342行)の直後に追加:

```js
      localStorage.setItem('subject-name', (document.getElementById('subject-name-input').value || '').trim() || 'りんご');
```

- [ ] **Step 6: クラス名を名前由来に(任意ラベル表示のみ)**

`train.html` の見出し `<h2>✅ ぎょぴちゃん の画像</h2>`(100行)を `<h2 id="zone-yes-title">✅ これ の画像</h2>`、`<h2>❌ ぎょぴちゃんじゃない の画像</h2>`(111行)を `<h2 id="zone-no-title">❌ これじゃない の画像</h2>` に変更し、`nameInput` 設定直後に同期処理を追加:

```js
    function syncZoneTitles() {
      const n = nameInput.value.trim() || 'りんご';
      document.getElementById('zone-yes-title').textContent = `✅ ${n} の画像`;
      document.getElementById('zone-no-title').textContent = `❌ ${n}じゃない の画像`;
    }
    nameInput.addEventListener('input', syncZoneTitles);
    syncZoneTitles();
```

注: 内部のクラス名文字列(`'ぎょぴちゃん'`/`'ぎょぴちゃんじゃない'`、autoLoad/index.json 用)と localStorage キー(`gyopi-*`)は **変更しない**(autoLoad 互換のため)。

- [ ] **Step 7: テスト実行→PASS 確認**

Run: `node verify_train_name.mjs`
Expected: `PASS verify_train_name`

- [ ] **Step 8: コミット**

```bash
git add train.html
git commit -m "feat(train): 被写体名の入力欄を追加し学習時に保存"
```

注: `verify_*.mjs` は .gitignore 済みのため add されない(想定通り)。

---

### Task 2: index.html の被写体名テンプレ化

**Files:**
- Modify: `index.html`(globals, tpl/L, LANG辞書, render, applyLang, loadSet/applyModel)
- Test: `verify_judge_name.mjs`(gitignore)

**Interfaces:**
- Consumes: localStorage `subject-name`(Task 1)、`model/meta.json`(Task 5/6 が `{ "name": "りんご" }` を同梱)。
- Produces: グローバル `subjectName`(string)。`tpl(s)` が `{name}`→`subjectName` 置換。`loadSet(where)` の返却に `name` を追加。

- [ ] **Step 1: 失敗するテストを書く**

`verify_judge_name.mjs`:

```js
import { chromium } from 'playwright';
import { startServer } from './verify_serve.mjs';
const srv = await startServer(process.cwd());
const browser = await chromium.launch();
const page = await browser.newPage();
// 公開相当: train.html / training_data を 404 に
await page.route('**/train.html', r => r.fulfill({ status: 404, body: '' }));
await page.route('**/training_data/**', r => r.fulfill({ status: 404, body: '' }));
// 自分の名前を ねこ に設定
await page.addInitScript(() => localStorage.setItem('subject-name', 'ねこ'));
await page.goto(srv.url + '/index.html');
await page.evaluate(() => localStorage.setItem('judge-welcomed','1'));
await page.waitForTimeout(2500); // モデル/MobileNet ロード
const title = await page.textContent('#t-tb-title');
if (!title.includes('ねこ')) throw new Error('FAIL: タイトルに名前未反映: ' + title);
const matchLabel = await page.textContent('#jc-rows');
if (!matchLabel.includes('ねこっぽさ')) throw new Error('FAIL: ラベル未反映: ' + matchLabel.slice(0,40));
console.log('PASS verify_judge_name');
await browser.close(); srv.close();
```

- [ ] **Step 2: テスト実行→失敗確認**

Run: `node verify_judge_name.mjs`
Expected: FAIL（タイトルに名前未反映、`ぎょぴちゃん` のまま）

- [ ] **Step 3: globals に subjectName と tpl を追加**

`index.html` の `let modelSet = { own: null, bundle: null };`(271行)直後に追加:

```js
    let subjectName = 'りんご';
    function tpl(s) { return typeof s === 'string' ? s.replaceAll('{name}', subjectName) : s; }
```

- [ ] **Step 4: L() を tpl 経由に**

`function L(key) { return LANG[curLang][key]; }`(365行)を置換:

```js
    function L(key) { return tpl(LANG[curLang][key]); }
```

- [ ] **Step 5: LANG 辞書の名前リテラルを {name} 化**

kid 辞書(287〜323行)を以下に変更:

```js
        title:'{name} はんていマシン', subtitle:'この えは {name} かな？ コンピュータに きいてみよう！',
```
```js
        matchLabel:'{name}っぽさ', effPrefix:'きいた ',
```
```js
        jcNote:'※「{name}っぽさ」= {name}に どれだけ にてるか。「きいた」= その とくちょうが はんていに どれだけ きいたか。',
```
```js
        verdict:{ yes:'{name}だ！', maybe:'どっちかな？', no:'ちがうみたい' },
        conf:['{name}じゃない','ちがう…かな？','うーん、まよう…','{name}…かな？','たぶん {name}','ぜったい {name}！'],
```
```js
        mapYes:'{name}', mapNo:'ちがう',
```
```js
        mapNote:'🔴{name} と 🔵ちがう が べつの かたまりに 分かれたら、AIが ちがいを おぼえた しるし。ノードに さわると どの えか わかるよ。',
```
```js
        ndImg:'🖼️ いれた え', ndHidden:'🧠 のうさいぼう', ndOut:'🐟 {name}度', ndAxis:'とくちょう：',
```

adult 辞書(326〜362行)を以下に変更:

```js
        title:'{name}判定器', subtitle:'この画像が{name}かどうか、AIに判定させます',
```
```js
        matchLabel:'{name}っぽさ', effPrefix:'効き ',
```
```js
        jcNote:'※「{name}っぽさ」={name}平均との近さ（一致度）。「きいた」=その特徴が判定に与えた影響（寄与度・近似）。',
```
```js
        verdict:{ yes:'{name}です！', maybe:'どちらとも言えません', no:'ちがいます' },
        conf:['{name}ではない','たぶん違う','判定に迷っています','{name}かも','たぶん{name}','ほぼ確実に{name}'],
```
```js
        mapYes:'{name}', mapNo:'ちがう',
```
```js
        mapNote:'🔴{name}と🔵ちがうがクラスタに分離していれば、AIが違いを学習できた証拠です。ノードにカーソルを合わせると画像名が出ます。',
```
```js
        ndImg:'🖼️ 入力画像', ndHidden:'🧠 ニューロン（中間層）', ndOut:'🐟 {name}確率', ndAxis:'特徴：',
```

- [ ] **Step 6: render() の verdict/conf を tpl 化**

`render()` 内(664〜672行)を変更。`verdict=x.verdict.yes` → `verdict=tpl(x.verdict.yes)`、`.maybe`/`.no` も同様。`meterEl.textContent = x.conf[ci];`(672行)→ `meterEl.textContent = tpl(x.conf[ci]);`:

```js
      if (prob >= 0.55) { badge='🐟'; cls='yes'; verdict=tpl(x.verdict.yes); barClass=''; }
      else if (prob >= 0.45) { badge='🤔'; cls='maybe'; verdict=tpl(x.verdict.maybe); barClass='mid'; }
      else { badge='🙅'; cls='no'; verdict=tpl(x.verdict.no); barClass='low'; }
```
```js
      meterEl.textContent = tpl(x.conf[ci]);
```

- [ ] **Step 7: applyLang() を L() 経由に**

`applyLang()`(369行〜)内で名前を含む代入を `x.<key>` から `L('<key>')` に変更する。対象:
- `x.title`(`w-title`/`t-tb-title`)→ `L('title')`
- `x.subtitle`(`w-subtitle`)→ `L('subtitle')`
- `x.mapTitle`/`x.mapDesc`/`x.mapYes`/`x.mapNo`/`x.mapNote` → `L('mapTitle')` 等

例(371〜379行該当部):

```js
      document.getElementById('w-title').textContent = L('title');
      document.getElementById('w-subtitle').textContent = L('subtitle');
      document.getElementById('t-tb-title').textContent = L('title');
      document.getElementById('nav-link').textContent = L('nav');
      document.getElementById('t-map-title').textContent = L('mapTitle');
      document.getElementById('t-map-desc').textContent = L('mapDesc');
      document.getElementById('t-map-yes').textContent = L('mapYes');
      document.getElementById('t-map-no').textContent = L('mapNo');
      document.getElementById('t-map-note').textContent = L('mapNote');
```

(他の `x.foo` 行も `L('foo')` に統一してよい。L は tpl 済みで名前なしキーは素通り。)

- [ ] **Step 8: loadSet が name を返し applyModel が反映**

`loadSet(where)`(405行)の return を変更。bundle は `model/meta.json`、own は localStorage から名前取得:

```js
        const name = idb
          ? (localStorage.getItem('subject-name') || null)
          : await fetch('model/meta.json').then(r => r.ok ? r.json() : null).then(m => m && m.name).catch(() => null);
        return { classifier: cls, featClassifier: feat, gyopiAvg: avg, name };
```

`applyModel()`(417行)内、`usingOwnModel = ...` の直後に追加:

```js
      subjectName = s.name || 'りんご';
      applyLang();
      buildJudgeCardRows();
```

- [ ] **Step 9: 初期表示も subject-name を反映**

`applyLang();`(808行、初期化呼び出し)の **前** に追加:

```js
    subjectName = localStorage.getItem('subject-name') || 'りんご';
```

- [ ] **Step 10: テスト実行→PASS 確認**

Run: `node verify_judge_name.mjs`
Expected: `PASS verify_judge_name`

- [ ] **Step 11: コミット**

```bash
git add index.html
git commit -m "feat: 被写体名を{name}テンプレ化し全ラベルに反映"
```

---

### Task 3: 地図パネルの同梱 map-data.json フォールバック

**Files:**
- Modify: `index.html`(makeMap フォールバック、setupNavLink 表示条件)
- Create: `map-data.json`(検証用の最小フィクスチャ。Task 5 で本物に上書き)
- Test: `verify_map_public.mjs`(gitignore)

**Interfaces:**
- Consumes: 同梱 `map-data.json` = 配列 `[{ thumb: dataURL, emb: number[], yes: boolean }, ...]`。
- Produces: `loadMapData()` async → 上記配列 or null。`makeMap()` がこれを使用。

- [ ] **Step 1: 最小フィクスチャを作成**

`map-data.json`(2点だけ。後で本物に置換):

```json
[
  { "thumb": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg==", "emb": [1, 0, 0], "yes": true },
  { "thumb": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNk+M9QDwAEhQGAhKmMIQAAAABJRU5ErkJggg==", "emb": [0, 0, 1], "yes": false }
]
```

- [ ] **Step 2: 失敗するテストを書く**

`verify_map_public.mjs`:

```js
import { chromium } from 'playwright';
import { startServer } from './verify_serve.mjs';
const srv = await startServer(process.cwd());
const browser = await chromium.launch();
const page = await browser.newPage();
await page.route('**/train.html', r => r.fulfill({ status: 404, body: '' }));
await page.route('**/training_data/**', r => r.fulfill({ status: 404, body: '' }));
await page.goto(srv.url + '/index.html');
await page.evaluate(() => localStorage.setItem('judge-welcomed','1'));
await page.waitForTimeout(2500);
const disp = await page.evaluate(() => getComputedStyle(document.getElementById('panel-map')).display);
if (disp === 'none') throw new Error('FAIL: 公開相当で地図パネル非表示');
await page.waitForTimeout(2000);
const nodeCount = await page.evaluate(() => document.querySelectorAll('#network canvas').length);
if (nodeCount < 1) throw new Error('FAIL: network canvas 未描画');
console.log('PASS verify_map_public');
await browser.close(); srv.close();
```

- [ ] **Step 3: テスト実行→失敗確認**

Run: `node verify_map_public.mjs`
Expected: FAIL（公開相当で地図パネル非表示）

- [ ] **Step 4: loadMapData helper を追加**

`makeMap()`(450行)の直前に追加:

```js
    async function loadMapData() {
      const cached = JSON.parse(localStorage.getItem('gyopi-map-data') || 'null');
      if (cached && cached.length) return cached;
      return await fetch('map-data.json').then(r => r.ok ? r.json() : null).catch(() => null);
    }
```

- [ ] **Step 5: makeMap をフォールバック対応に**

`makeMap()` 内、`const cached = JSON.parse(localStorage.getItem('gyopi-map-data') || 'null');`(454行)〜 `if (cached && cached.length) {`(455行)を以下に変更:

```js
        const cached = await loadMapData();
        if (cached && cached.length) {
```

(以降の `items = cached.map(...)` 等はそのまま。else の training_data フォールバックも残す。)

- [ ] **Step 6: setupNavLink の表示条件に同梱データを追加**

`setupNavLink()`(498行)冒頭の `const hasMapData = !!localStorage.getItem('gyopi-map-data');` を変更:

```js
      const lsMap = !!localStorage.getItem('gyopi-map-data');
      const bundledMap = await fetch('map-data.json', { method: 'HEAD' }).then(r => r.ok).catch(() => false);
      const hasMapData = lsMap || bundledMap;
```

- [ ] **Step 7: テスト実行→PASS 確認**

Run: `node verify_map_public.mjs`
Expected: `PASS verify_map_public`

- [ ] **Step 8: コミット**

```bash
git add index.html map-data.json
git commit -m "feat(map): 同梱map-data.jsonフォールバックで公開時も地図を表示"
```

---

### Task 4: ビルドスクリプト — PD/CC0 画像取得

**Files:**
- Create: `build_apple_model.mjs`(取得パートのみ。学習は Task 5)
- Test: 同スクリプトの取得関数を直接実行して確認

**Interfaces:**
- Produces: `fetchImages(query, n, outDir)` async → 保存ファイルパス配列。Wikimedia Commons から PD/CC0 のみ取得。
- Produces: 一時ディレクトリ `.build/apple/`(りんご), `.build/not-apple/`(非りんご)に jpg 保存。

- [ ] **Step 1: 取得スクリプトを作成**

`build_apple_model.mjs`:

```js
import { mkdir, writeFile } from 'fs/promises';
import { join } from 'path';

const PD = ['cc0', 'public domain', 'pd', 'cc-zero'];
function isFree(meta) {
  const lic = ((meta?.LicenseShortName?.value || '') + ' ' + (meta?.License?.value || '')).toLowerCase();
  return PD.some(t => lic.includes(t));
}

export async function fetchImages(query, n, outDir) {
  await mkdir(outDir, { recursive: true });
  const api = 'https://commons.wikimedia.org/w/api.php';
  const params = new URLSearchParams({
    action: 'query', generator: 'search', gsrsearch: `${query} filetype:bitmap`,
    gsrnamespace: '6', gsrlimit: '40', prop: 'imageinfo',
    iiprop: 'url|extmetadata', iiurlwidth: '320', format: 'json', origin: '*',
  });
  const data = await fetch(`${api}?${params}`).then(r => r.json());
  const pages = Object.values(data?.query?.pages || {});
  const saved = [];
  for (const p of pages) {
    if (saved.length >= n) break;
    const info = p.imageinfo?.[0]; if (!info) continue;
    if (!isFree(info.extmetadata)) continue;
    const url = info.thumburl || info.url; if (!url) continue;
    try {
      const buf = Buffer.from(await fetch(url).then(r => r.arrayBuffer()));
      const file = join(outDir, `${saved.length}.jpg`);
      await writeFile(file, buf);
      saved.push(file);
    } catch {}
  }
  return saved;
}

// 単独実行で取得確認
if (process.argv[2] === 'fetch') {
  const apples = await fetchImages('apple fruit red', 14, '.build/apple');
  const others = [];
  for (const q of ['banana fruit', 'orange fruit', 'car', 'dog', 'flower']) {
    others.push(...await fetchImages(q, 4, `.build/not-apple/${q.split(' ')[0]}`));
    if (others.length >= 14) break;
  }
  console.log(`apples=${apples.length} others=${others.length}`);
  if (apples.length < 8 || others.length < 8) { console.error('不足: 別クエリ/閾値調整が必要'); process.exit(1); }
}
```

- [ ] **Step 2: 取得を実行して確認**

Run: `node build_apple_model.mjs fetch`
Expected: `apples=… others=…`(各 8 以上)。`.build/apple/*.jpg` と `.build/not-apple/**/*.jpg` が生成される。不足時はクエリ語や `gsrlimit` を増やす。

- [ ] **Step 3: .gitignore に .build/ を追加**

`.gitignore` に追記:

```
# りんごモデルのビルド一時物
.build/
build_apple_model.mjs
```

- [ ] **Step 4: コミット**

```bash
git add .gitignore
git commit -m "chore: りんごモデルのビルド一時物を無視"
```

注: `build_apple_model.mjs` は再現用スクリプトだが成果物(model/・map-data.json)のみ公開する方針で gitignore する。再現手順は本プランに記録。

---

### Task 5: ビルドスクリプト — 学習と成果物書き出し

**Files:**
- Modify: `build_apple_model.mjs`(playwright 駆動の学習＋エクスポート追加)
- Test: ビルド実行で成果物検証

**Interfaces:**
- Consumes: `fetchImages`(Task 4)、`train.html`(Task 1: 名前入力＋ファイル投入可能)、`verify_serve.mjs`(Task 1)。
- Produces: `model/gyopi-classifier/{model.json,weights.bin}`, `model/gyopi-feature-classifier/{model.json,weights.bin}`, `model/gyopi-avg.json`, `model/meta.json`, `map-data.json`(本物)。

- [ ] **Step 1: 学習駆動＋エクスポートを追加**

`build_apple_model.mjs` の末尾(`fetch` 分岐の外)に追記:

```js
async function exportModelFromIDB(page, idbName, outDir) {
  const art = await page.evaluate(async (name) => {
    const m = await window.tf.loadLayersModel('indexeddb://' + name);
    let saved;
    await m.save(window.tf.io.withSaveHandler(async a => { saved = a; return { modelArtifactsInfo: { dateSaved: new Date(), modelTopologyType: 'JSON' } }; }));
    return { topology: saved.modelTopology, specs: saved.weightSpecs, weights: Array.from(new Uint8Array(saved.weightData)) };
  }, idbName);
  const modelJson = {
    format: 'layers-model', generatedBy: 'judge-build', convertedBy: null,
    modelTopology: art.topology,
    weightsManifest: [{ paths: ['weights.bin'], weights: art.specs }],
  };
  const { mkdir, writeFile } = await import('fs/promises');
  await mkdir(outDir, { recursive: true });
  await writeFile(join(outDir, 'model.json'), JSON.stringify(modelJson));
  await writeFile(join(outDir, 'weights.bin'), Buffer.from(art.weights));
}

if (process.argv[2] === 'build') {
  const { chromium } = await import('playwright');
  const { startServer } = await import('./verify_serve.mjs');
  const { writeFile } = await import('fs/promises');
  const { readdir } = await import('fs/promises');

  // 取得
  const appleDir = '.build/apple';
  await fetchImages('apple fruit red', 14, appleDir);
  const otherDirs = [];
  for (const q of ['banana fruit', 'orange fruit', 'car', 'dog', 'flower']) {
    const d = `.build/not-apple/${q.split(' ')[0]}`;
    await fetchImages(q, 4, d); otherDirs.push(d);
  }
  const ls = async d => (await readdir(d)).map(f => join(d, f));
  const yesFiles = await ls(appleDir);
  let noFiles = []; for (const d of otherDirs) noFiles = noFiles.concat(await ls(d));
  noFiles = noFiles.slice(0, yesFiles.length);

  const srv = await startServer(process.cwd());
  const browser = await chromium.launch();
  const page = await browser.newPage();
  page.on('console', m => console.log('[page]', m.text()));
  await page.goto(srv.url + '/train.html');
  await page.fill('#subject-name-input', 'りんご');
  await page.setInputFiles('#files-yes', yesFiles);
  await page.setInputFiles('#files-no', noFiles);
  await page.click('#train-btn');
  await page.waitForSelector('#result-wrap', { state: 'visible', timeout: 180000 });

  // エクスポート
  await exportModelFromIDB(page, 'gyopi-classifier', 'model/gyopi-classifier');
  await exportModelFromIDB(page, 'gyopi-feature-classifier', 'model/gyopi-feature-classifier');
  const avg = await page.evaluate(() => localStorage.getItem('gyopi-avg-features'));
  const mapData = await page.evaluate(() => localStorage.getItem('gyopi-map-data'));
  await writeFile('model/gyopi-avg.json', avg);
  await writeFile('model/meta.json', JSON.stringify({ name: 'りんご' }));
  await writeFile('map-data.json', mapData);
  console.log('build done');
  await browser.close(); srv.close();
}
```

- [ ] **Step 2: ビルド実行**

Run: `node build_apple_model.mjs build`
Expected: `[page]` ログ→`build done`。`model/gyopi-classifier/model.json`+`weights.bin`、`model/gyopi-feature-classifier/*`、`model/gyopi-avg.json`、`model/meta.json`、`map-data.json` が更新/生成される。

- [ ] **Step 3: 成果物の妥当性を確認**

Run: `node -e "const m=require('./map-data.json'); console.log('nodes', m.length, 'yes', m.filter(x=>x.yes).length, 'embLen', m[0].emb.length)"`
Expected: `nodes` ≥ 16、`yes` ≥ 8、`embLen` = 256(MobileNet v1 α0.25)。

- [ ] **Step 4: コミット**

```bash
git add model/gyopi-classifier model/gyopi-feature-classifier model/gyopi-avg.json model/meta.json map-data.json
git commit -m "feat: りんごモデルと地図データを同梱(PD/CC0画像で学習)"
```

---

### Task 6: 公開相当の総合検証

**Files:**
- Test: `verify_public_e2e.mjs`(gitignore)

**Interfaces:**
- Consumes: Task 2/3/5 の成果物すべて。

- [ ] **Step 1: 総合テストを書く**

`verify_public_e2e.mjs`:

```js
import { chromium } from 'playwright';
import { startServer } from './verify_serve.mjs';
const srv = await startServer(process.cwd());
const browser = await chromium.launch();
const page = await browser.newPage();
// 公開相当: train.html / training_data 404、localStorage 空
await page.route('**/train.html', r => r.fulfill({ status: 404, body: '' }));
await page.route('**/training_data/**', r => r.fulfill({ status: 404, body: '' }));
await page.goto(srv.url + '/index.html');
await page.evaluate(() => localStorage.setItem('judge-welcomed','1'));
await page.waitForTimeout(3000);
// 名前 = りんご(meta.json 由来)
const title = await page.textContent('#t-tb-title');
if (!title.includes('りんご')) throw new Error('FAIL: 既定名 りんご 未反映: ' + title);
// 地図パネル表示＋描画
const disp = await page.evaluate(() => getComputedStyle(document.getElementById('panel-map')).display);
if (disp === 'none') throw new Error('FAIL: 地図パネル非表示');
await page.waitForTimeout(2500);
const drawn = await page.evaluate(() => document.querySelectorAll('#network canvas').length);
if (drawn < 1) throw new Error('FAIL: network 未描画');
console.log('PASS verify_public_e2e');
await browser.close(); srv.close();
```

- [ ] **Step 2: テスト実行→PASS 確認**

Run: `node verify_public_e2e.mjs`
Expected: `PASS verify_public_e2e`

- [ ] **Step 3: 既存検証の回帰確認**

Run: `node verify_judge_name.mjs && node verify_map_public.mjs && node verify_train_name.mjs`
Expected: 3 件すべて `PASS`

- [ ] **Step 4: ローカルサーバを起動したまま残す**

Run(バックグラウンド): `python -m http.server 8123`(無ければ `node verify_serve.mjs` 相当を常駐)。ブラウザで `http://localhost:8123/index.html` を開き目視確認(りんご測定器・地図表示)。

- [ ] **Step 5: 最終コミット(必要なら)**

```bash
git add -A
git commit -m "test: 公開相当の総合検証を通過" || echo "差分なし"
```

---

## Self-Review

**Spec coverage:**
- 被写体名パラメータ化(train保存/index反映/デフォルトりんご) → Task 1,2 ✓
- 名前を全ラベル反映(っぽさ/度/判定文) → Task 2 ✓
- りんごモデル新規同梱(PD/CC0取得→学習→model/差替) → Task 4,5 ✓
- 地図サムネ=PD/CC0 → Task 5(map-data.json は学習画像サムネ) ✓
- 公開地図フォールバック＋常時表示 → Task 3 ✓
- 旧ぎょぴ公開差替 → Task 5(model/上書き)+meta.json名=りんご ✓
- 非対象(絵文字/内部キー改名/複数被写体) → 計画外、整合 ✓

**Placeholder scan:** TBD/TODO なし。各コード手順に実コードあり。

**Type consistency:**
- `tpl(s)`/`subjectName` Task2 で定義→render/applyModel で一貫使用 ✓
- `loadSet` 返却に `name` 追加 → applyModel が `s.name` 参照 ✓
- map-data 形式 `{thumb,emb,yes}` → フィクスチャ(Task3)・本物(Task5)・makeMap 消費で一致 ✓
- `model/meta.json` `{name}` → loadSet(bundle) と build 書き出しで一致 ✓
- `#subject-name-input` Task1 作成 → build(Task5) が fill で参照 ✓

ギャップなし。
