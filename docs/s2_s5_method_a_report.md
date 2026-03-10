# S2/S5 修正レポート（zmk-naginata 本体 / v16ベース）

## 変更要点

- 判定順は Method A を維持:
  1. LAYER_CONTROL を優先 pass-through
  2. MODIFIER を pass-through
  3. NAGINATA_TARGET を処理
- S2 対策として、`&ng` 経路だけでなく **他 behavior 由来の修飾キー状態** も
  `zmk_keycode_state_changed` で監視するようにした。

## なぜ前回修正で S2 が残ったか

`Cmd` が `&kp LCMD` などで定義され、`C` が `&ng C` の場合、
修飾キー押下イベントは `behavior_naginata` に直接入らない。
そのため `active_modifiers` だけでは Cmd 押下中を検知できず、
`C` が薙刀変換されるケースが残る。

## 今回の対応

- `on_keycode_state_changed()` を追加し、全体の keycode イベントから
  修飾キー押下状態（`external_modifiers`）を追跡。
- `total_active_modifiers()` で `active_modifiers + external_modifiers` を評価し、
  Cmd/Shift/Ctrl/Alt 押下中は薙刀変換に入れず pass-through。

## S2/S5 シナリオ

- S2: Cmd+C
  - Cmd が `&kp` 側で押されても `external_modifiers` が立つ
  - `&ng C` は shortcut pass-through され、Cmd+H 化を抑止
- S5: 薙刀入力中のレイヤー制御
  - NAGINATA_TARGET/修飾キー以外は先に pass-through
  - `MO/LT/TO/TG` 系は薙刀バッファに入れず透過

## 影響範囲

- `src/behaviors/behavior_naginata.c`
- `docs/s2_s5_method_a_report.md`
