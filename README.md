# Timetable Cards UI (ICU) — README

## 目的

学科内の相談・調整用途として、**時間割グリッドに授業カードを貼り付ける**カード型インターフェースを提供する。

* 授業カードをドラッグ＆ドロップで配置
* 配置結果を **JSON** として出力
* **学期（春/秋/冬）別**にデータを管理（localStorageで十分）
* 複数科目が同一コマに入るケースも許容（別教員担当など）

本ツールは「ルールを完全に強制する時間割編成システム」ではなく、**可視化と調整支援**が主目的。

---

## 非目的（このプロジェクトで“やらないこと”）

実装コストと運用方針から、以下は **HTML側に実装しない**。

* 科目配置ルール（B1/B2/A3等）の自動チェック
  → **出力JSONをLLMでチェックする運用**を想定
* 学科内コンフリクト最小化（C1）や編集ロック（D1）
* Super時限の概念・制約（COIL等）

  * Super時限は **手元の時間割→JSON変換段階で吸収**する

---

## UIの仕様（確定）

### 1) グリッド（時間割マトリックス）

* day（列）：`M, TU, W, TH, F`
* period（行）：`1,2,3,L,4,5,6,7`

  * `L` は Lunch（昼休み）
* slot表記：`<period>/<day>`（例：`5/TH`, `L/TU`）

### 2) カード（授業）

* `id` は授業コードをそのまま使用（例：`ISC224a`, `ISC224b`）

  * 2コマ授業は `a/b` など **枝番**で表現
* カード表示：常に **`<id> <title>`**（例：`ISC224a OS`）
* `color` 指定があればカード背景色に反映（機能維持）

### 3) 同一コマへの複数配置

* 1つのセル（同一slot）に **複数カードを配置可能**
* 置換はしない（縦に積む）

### 4) 固定ブロック（教授会・チャペル等）

* **灰色のカード**を各学期のJSONに含めて表現する（A1の実装方針）
* 固定ブロックも通常カードと同じ挙動（配置/移動可能）

  * 「PTLは配置可能」等の制約は HTMLでは持たない

### 5) ドラッグ＆ドロップ挙動

* パレット → セル：配置
* セル → パレット：戻す
* セル → 別セル：移動
* **連続コマの一括移動は実装しない**（独立移動のみ）

---

## データ仕様（JSON）

### 入力JSON（授業カード一覧）

* JSONは **配列**で渡す
* 必須：`id`（string）
* 任意：`title`, `teacher`, `color`, `slot`
* `slot` があれば初期配置される。無ければパレットに残る。

例：

```json
[
  {"id":"ISC224a","title":"OS","teacher":"Kaburagi","color":"#fff6a8","slot":"5/TH"},
  {"id":"ISC224b","title":"OS","teacher":"Kaburagi","color":"#fff6a8","slot":"6/TH"},
  {"id":"BLOCK_FACULTY_MT","title":"教授会","color":"#d0d0d0","slot":"5/TU"},
  {"id":"BLOCK_CHAPEL","title":"Chapel Hour","color":"#d0d0d0","slot":"L/TU"}
]
```

### 出力JSON（slot付与）

* 現在の配置を反映して、各要素に `slot` を付与して出力
* 未配置のカードは `slot: null` を出力（LLMチェックや後処理で扱いやすい）

---

## 学期（春/秋/冬）と保存

* 学期は `spring / autumn / winter` の3種
* localStorageに **学期別キー**で保存する

  * 例：`timetable_v3_courses_spring`, `timetable_v3_placements_spring`
* 「保存データ読込」ボタンで学期別の保存状態を復元

---

## 操作方法

1. HTMLをブラウザで開く
2. 学期（春/秋/冬）を選択
3. JSONを **ファイル読込** または **貼り付け読込**
4. ドラッグでセルに配置（同一セルに複数OK）
5. 戻したい場合はカードをパレットへドロップ
6. 「slot付JSONを出力」で `slot` を含んだJSONを取得（テキスト欄に出力）

---

## ルールチェックの運用（LLM）

本ツールは **配置ルールを強制しない**。
運用としては：

1. ツールで配置
2. 出力JSONを取得（slot付き）
3. LLMに「ルールブック準拠チェック」を依頼し、違反・注意点を返す
4. 必要に応じて再配置

（例：縦組・横組の推奨、教員種別、学年帯の推奨、例外扱い等）

---

## 現状の実装（プロトタイプ）

* 1ファイルHTMLに、UI/ロジック（import/export, DnD, localStorage）を実装済み
* Super時限は **盤面上は表現しない**（事前変換で吸収）

---

## Codex引き継ぎ用：推奨リポジトリ構成

プロトタイプは1ファイルだが、拡張を考えると以下の分割が扱いやすい。

```
timetable/
  README.md
  public/
    index.html
    styles.css
    app.js
    storage.js         # localStorage read/write (term-aware)
    slot.js            # parse/format slot (day/period normalization)
    render.js          # grid/card rendering
    dnd.js             # drag & drop handlers (palette + cells)
  data/
    spring.json
    autumn.json
    winter.json
```

---

## 今後の拡張候補（任意）

※今回スコープ外だが、必要になった場合の候補

* JSONのダウンロードボタン（`export.json`として保存）
* セル内のカード順序入れ替え（同一slot内の並び）
* タグ表示（例：学年帯、言語、科目区分）※表示だけなら軽い
* LLMチェック結果の取り込み表示（警告の可視化）※強制しない方針のまま

---

## 対象

International Christian University の学科内時間割調整（相談用途）
