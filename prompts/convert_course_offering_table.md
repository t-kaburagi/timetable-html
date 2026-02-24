````markdown
# Prompt: Convert ICU timetable table text → Timetable Cards JSON (for Cards UI)

You are a data-conversion assistant. Convert the following **raw timetable table text** (copy/paste from ICU listing) into a **valid JSON array** for a card-based timetable UI.

## Output requirements (STRICT)
- Output **ONLY** a single JSON array (no markdown, no commentary, no trailing text).
- Each element is a course card object with these keys:

```json
{
  "term": "spring|autumn|winter",
  "id": "COURSECODE[a|b|c...]",
  "title": "COURSECODE JP_TITLE (J|E)",
  "language": "J|E|B|Other",
  "teacher": "INSTRUCTOR(S) as one string",
  "units": 2,
  "color": "#RRGGBB",
  "slot": "1/M|2/TU|3/W|L/TH|4/F|5/M|6/TU|7/W|null",
  "notes": "optional free text"
}
````

* `day` must be one of: `M, TU, W, TH, F`
* `period` must be one of: `1,2,3,L,4,5,6,7`
* `slot` must be exactly `<period>/<day>` or `null`.

## Parsing rules (IMPORTANT)

### 1) Term normalization

* The raw text may include `SPRING` / `AUTUMN` / `WINTER` or `S` / `A` / `W`.
* Normalize to:

  * `SPRING` or `S` → `"spring"`
  * `AUTUMN` or `A` → `"autumn"`
  * `WINTER` or `W` → `"winter"`
* If term is missing for a row, infer from the section heading (e.g., the text begins with "Spring") and apply to all rows under that heading.

### 2) Course identifier & splitting into multiple cards

* Base course code is the **Course Number** (e.g., `ISC224`, `BIO214`).
* The timetable UI requires **one card = one slot**.
* If a course has **multiple slots**, split into multiple cards:

  * First slot: `ISC224a`
  * Second slot: `ISC224b`
  * Third slot: `ISC224c`
  * ...
* The base code remains in the title, but the `id` must be the branched one (with a/b/c...).
* If the same course has a single slot, do **NOT** add a suffix (keep `ISC224`).

### 3) Time Slot parsing

* Time slots appear like: `5/F,6/F` or `1/TU,2/TU`.
* They may include:

  * **Super4 slot** `*4/<day>` → expand into **2 cards**: `L/<day>` and `4/<day>`.
    * Example: `*4/M,*4/TH` → 4 cards with slots `L/M`, `4/M`, `L/TH`, `4/TH`
    * This is the only `*`-prefixed slot that requires expansion.
  * **Other `*`-prefixed slots** (e.g., `*5/W`) → strip the `*` and use the slot as-is (no expansion).
  * Parentheses for optional/alternative sessions: `(6/F,7/F)`

    * Convert each slot inside parentheses into cards too, and add `notes` like `"optional/parenthesized slot"`.
  * Mixed punctuation/spacing → normalize.
* Any slot outside the supported grid (e.g., `8/*`, `SA`, `Super`, etc.) must be handled as:

  * If it can be mapped to the supported scheme by the user's convention, apply mapping **ONLY if explicitly provided in the input**.
  * Otherwise, create the card with `slot: null` and put the original slot text into `notes` (do not invent mappings).

### 4) Title construction

* Use the **Japanese title only** (do not include the English title).
* Append the language indicator in parentheses at the end.
* Construct as:

  * `"title": "<COURSECODE> <JP_TITLE> (J)"` or `"title": "<COURSECODE> <JP_TITLE> (E)"`
* Remove extra blank lines.

### 5) Language

* Use the `Language` column value if present (`J` or `E`).
* If missing, set `"language": "Other"` and mention in `notes`.

### 6) Instructor(s)

* Put the instructor(s) as a single string.
* Keep `*` if present next to name (e.g., `"SHENG, Dachen *"`).
* If multiple instructors, join with ` / ` (e.g., `"OHTA, Keiji / TANAKA, Masayuki *"`).

### 7) Units

* Parse as an integer (e.g., `2`, `3`). If missing, set `null` and add note.

### 8) Color assignment

Assign `color` deterministically based on **course prefix and number**. Do not omit `color`.

| Condition | Color |
|-----------|-------|
| Graduate courses (prefix `QNMC`, `QNMS`, or any 500-level+) | `"#a8d4f8"` (blue) |
| GE courses (prefix `GEN`, `GEL`, etc.) | `"#b8f0c8"` (green) |
| `ISC` 100-level (ISC1xx) | `"#b8f0c8"` (green) |
| `ISC` 200-level (ISC2xx) | `"#fff6a8"` (yellow) |
| `ISC` 300-level (ISC3xx) | `"#ffb0cc"` (pink) |
| All other prefixes (BIO, ECO, LNG, etc.) | `"#d0d0d0"` (gray) |

* If the course looks like a fixed block (`BLOCK_...`) then `"#d0d0d0"` (only if such rows exist; do not invent).
* The `* KYOSHOKU` marker does **not** affect color — use the number-based rule above.

### 9) Duplicates and cleanliness

* Ensure:

  * No duplicate `id` values.
  * JSON is valid (double quotes, no trailing commas).
  * Preserve order from the input.

## Examples

**Example 1 — Parenthesized slots:**

Input: `5/F,(6/F,7/F)`
Output cards:

* `ISC222a` slot `5/F`
* `ISC222b` slot `6/F` with notes `"optional/parenthesized slot"`
* `ISC222c` slot `7/F` with notes `"optional/parenthesized slot"`

**Example 2 — Super4 (`*4/<day>`):**

Input: `*4/M,*4/TH`
Output cards (e.g., for ISC103):

* `ISC103a` slot `L/M`
* `ISC103b` slot `4/M`
* `ISC103c` slot `L/TH`
* `ISC103d` slot `4/TH`

**Example 3 — Title format:**

Input course: `ISC224` / Japanese title: `オペレーティングシステム` / Language: `J`
Output title field: `"ISC224 オペレーティングシステム (J)"`

## Now convert this input

(Place the raw timetable text below this line)

```
PASTE_INPUT_HERE
```

```
```
