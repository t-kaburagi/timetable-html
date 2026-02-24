以下は、そのまま **Codex にコピペして使える「作業指示書」**です（Markdown）。
プロジェクトの方針は「**UIは軽量・ルールチェックはLLM側**」「**学期別localStorage**」「**slotは `period/day`**」を厳守します。

---

# Codex 作業指示書 — Timetable Cards UI (ICU)

## 0. ゴール（最終成果物）

1. **複数ファイル構成**に分割されたフロントエンド（静的HTMLで動作）
2. 現行の1ファイルHTMLと **同じ機能**を維持（仕様は下記）
3. `data/` にサンプルJSON（spring/autumn/winter）を置き、読み込み確認できる
4. 出力JSON（slot付与）をテキスト欄に出し、LLMチェック用に使える

対象：International Christian University 学科内時間割相談用途（ルール強制はしない）

---

## 1. 実装仕様（必ず守る）

### 1.1 グリッド仕様

* day（列）：`M, TU, W, TH, F`
* period（行）：`1,2,3,L,4,5,6,7`（`L` = Lunch）
* slot表記：`<period>/<day>`（例：`5/TH`, `L/TU`）

### 1.2 カード仕様

* `id` は授業コードをそのまま（例：`ISC224a`, `ISC224b`）
* 表示は常に **`<id> <title>`**（例：`ISC224a OS`）
* JSONの `color` があれば背景色に反映（この機能は残す）
* 固定ブロック（教授会/チャペル/会議帯など）は **灰色カードをJSONに含める**
  → 通常カードと同じ挙動（移動可能・配置制約は作らない）

### 1.3 DnD挙動

* パレット → セル：配置
* セル → パレット：戻す
* セル → セル：移動
* **同一セルに複数カード配置OK**（置換しない）
* **連続コマの一括移動は実装しない**（カードは独立移動のみ）

### 1.4 JSON入出力

* 入力は **配列JSON**（必須：`id`）
* `slot` があれば初期配置、無ければパレット
* 出力は各要素に最新 `slot` を付与

  * 未配置は **`slot: null`**

### 1.5 ルールチェックはしない

* B1/B2/A3などの配置ルールチェックは **HTML側で実装しない**
* 出力JSONをLLMでチェックする運用前提

### 1.6 学期・保存

* 学期：`spring / autumn / winter`
* localStorageに **学期別**に保存

  * 例：`timetable_v3_courses_spring`, `timetable_v3_placements_spring`

---

## 2. リポジトリ構成（作ること）

以下の構成に分割（Vanilla JS、ビルドなしでOK）。

```
timetable/
  README.md
  public/
    index.html
    styles.css
    app.js
    modules/
      constants.js     # dayDefs, periodDefs, defaultColor, storage keys prefix
      slot.js          # parseSlot(), toSlotString(), normalizeDay/Period()
      storage.js       # loadState(term), saveState(term,state), clearTerm(term)
      model.js         # state shape helpers, validation, import/export transform
      render.js        # buildGrid(), renderAll(state), makeCard(course)
      dnd.js           # attachDnD({gridEl,paletteEl,onDropToCell,onDropToPalette})
      ui.js            # bind buttons, term select, file/text import, export
  data/
    spring.json
    autumn.json
    winter.json
```

---

## 3. 状態モデル（統一すること）

`state` は以下の形を固定：

```js
state = {
  courses: [ /* course objects from JSON, must include id */ ],
  placements: { 
    [id]: { day: "TH", period: "5" } 
  }
}
```

* `courses` は入力JSONのオブジェクトを基本的に保持（将来メタデータ追加しやすく）
* `placements` はUI内部の正規化データ（slotではなく day/period）

---

## 4. 実装タスク（順序つき）

### Phase 1: 静的サイトとして起動（骨格）

1. `public/index.html` を作成

   * 左：学期セレクタ、ファイル読込、テキスト貼り付け、各ボタン、パレット領域
   * 右：グリッド領域
2. `public/styles.css` に現行スタイルを移植（見た目は大体同等で良い）
3. `public/app.js` からモジュールを呼び出す（ES Modulesで `type="module"`）

**受け入れ条件**

* ブラウザで `public/index.html` を直接開いて動く（サーバなしでもOK）
* UIが表示される

---

### Phase 2: slotユーティリティ実装

`modules/slot.js` を作る。

* `normalizeDay(token)`：`M, TU, W, TH, F` のみ許可
* `normalizePeriod(token)`：`1,2,3,L,4,5,6,7` のみ許可（`lunch`等は `L`へ）
* `parseSlot("5/TH") -> {period:"5", day:"TH"}`
* `toSlotString({period:"L", day:"TU"}) -> "L/TU"`

**受け入れ条件**

* 不正slotは `null` または例外で判定できる（後段のバリデーションに使う）

---

### Phase 3: JSONインポート（ファイル/テキスト）とバリデーション

`modules/model.js` を作る。

* `parseCoursesJson(text)`：

  * JSON parse
  * 配列であること
  * 各要素が object
  * `id` 必須（string、trim、空文字不可）
  * `id` 重複禁止
  * `slot` があれば `parseSlot` で placements に変換（不正ならエラー）
  * 返り値：`{ courses, placements }`

**重要仕様**

* import時は「**importのslotを初期配置として採用**」
  （マージは不要。ユーザーがJSONを正としてやり直す運用。）

**受け入れ条件**

* 例外メッセージが分かりやすい（どのid/どのindexが悪いか）

---

### Phase 4: localStorage（学期別）

`modules/storage.js` を作る。

* `saveState(term, state)`
* `loadState(term) -> state`（無ければ `{courses:[], placements:{}}`）
* `clearTerm(term)`（該当キー削除）
* キーは `timetable_v3_courses_${term}`, `timetable_v3_placements_${term}` を厳守

**受け入れ条件**

* 学期を切り替えると、その学期の保存内容が復元される
* クリア/初期化が学期単位で動く

---

### Phase 5: レンダリング（カード/グリッド）

`modules/render.js` を作る。

* `buildGrid(gridEl)`：ヘッダ＋8行（1,2,3,L,4,5,6,7）×5列を生成

  * セルには `data-day`, `data-period`
* `makeCard(course)`：

  * 表示：`id title`（title無ければ idのみ）
  * `course.color` があれば背景色
  * `draggable = true` & `dataset.courseId = id`
* `renderAll({state, gridEl, paletteEl})`：

  1. パレットに全カード生成
  2. 全セルをクリア
  3. placementsに従って該当セルへ append（複数可）

**受け入れ条件**

* 同一セルに複数カードが積める
* `color` が反映される

---

### Phase 6: DnD（セル/パレット/セル間移動）

`modules/dnd.js` を作る（event delegation推奨）。

* ドロップ先がセルの場合：

  * 対象カードをそのセルへ `appendChild`
  * `state.placements[id] = {day, period}`
  * `saveState`
* ドロップ先がパレットの場合：

  * `delete state.placements[id]`
  * パレットへ `appendChild`
  * `saveState`

**注意**

* 「セル内に別カードがあっても置換しない」
* 「まとめ移動は実装しない」

**受け入れ条件**

* パレットへ戻せる
* セル→セルが自然に動く

---

### Phase 7: UI結線（ボタン群）

`modules/ui.js` を作る。

必要ボタン：

* 保存データ読込（= `loadState` → `renderAll`）
* ファイル読込（FileReader or `file.text()` → `parseCoursesJson` → state更新 → save → render）
* 貼り付け読込（同上）
* slot付JSON出力：

  * `courses.map(c => ({...c, slot: placements[c.id] ? toSlotString(...) : null }))`
  * 結果はテキスト欄に書き戻す（alertは使わない）
* 配置クリア：

  * `state.placements = {}`
  * save → render
* この学期を初期化：

  * `clearTerm(term)` + state初期化 + render

**受け入れ条件**

* 現行1ファイルHTMLと同等の操作が一通りできる

---

## 5. サンプルデータ（data/*.json）

`data/spring.json` 等を用意。最低限：

* 通常カード（色あり）
* 灰色固定ブロック（`color:"#d0d0d0"`）
* `slot` あり・なし混在
* 同一slotに2科目入る例

例（spring.json）：

```json
[
  {"id":"ISC224a","title":"OS","teacher":"Kaburagi","color":"#fff6a8","slot":"5/TH"},
  {"id":"ISC224b","title":"OS","teacher":"Kaburagi","color":"#fff6a8","slot":"6/TH"},
  {"id":"MAT101","title":"Math","teacher":"Suzuki","color":"#c8f7ff","slot":"5/TH"},
  {"id":"BLOCK_FACULTY","title":"教授会","color":"#d0d0d0","slot":"5/TU"},
  {"id":"BLOCK_CHAPEL","title":"Chapel Hour","color":"#d0d0d0","slot":"L/TU"},
  {"id":"FREE_CARD","title":"未配置サンプル","color":"#ffd2d2"}
]
```

---

## 6. テスト観点（最低限これだけ手動確認）

1. **slotパース**

   * `5/TH` OK、`L/TU` OK、`8/M` NG、`5/Mon` NG（あくまで指定表記のみ）
2. **同一セルに複数**

   * `5/TH` に2カード置ける
3. **戻す**

   * セルからパレットへ戻せる、slotがnullになる
4. **色**

   * `color` が反映される（灰色カードも同様）
5. **学期別**

   * springで置いた状態がautumnに影響しない
6. **出力**

   * 出力JSONに `slot` が正しい形式で入る、未配置は `slot:null`

---

## 7. 追加でやってよい改善（必須ではない）

* 出力JSONをダウンロードするボタン（`download` link生成）
* セル内カードの順序入れ替え（ただし仕様外なので、やるなら後回し）

---

## 8. 完了条件（Definition of Done）

* 分割後の `public/` を開くと、プロトタイプ同等に動作
* 仕様（day/period/slot、複数同居、色、学期別保存、slot:null出力）を満たす
* `data/*.json` の読み込みでデモできる
* ルールチェックは実装していない（LLM運用前提のまま）

---

必要なら、次の段階として「出力JSONをそのままLLMに投げるためのプロンプト雛形」も同じリポジトリに `prompts/` として追加できます。
