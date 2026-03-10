# S2/S5 修正レポート（zmk-naginata 本体 / Method A）

## 1) イベント処理フロー（関数名ベース）

対象コードは `src/behaviors/behavior_naginata.c` です。

- `on_keymap_binding_pressed()`
  - `binding->param1`（押下キーコード）を受け取る
  - OS 切替キー（F15〜F19）のみ先に処理して `ZMK_BEHAVIOR_OPAQUE` で終了
  - それ以外は `timestamp` を更新して `naginata_press()` へ委譲
- `on_keymap_binding_released()`
  - `timestamp` を更新して `naginata_release()` へ委譲
- `naginata_press()`
  - Method A の判定順でキーイベントを分類
  - 薙刀対象キーの場合のみ `nginput`（バッファ）へ追加し、必要に応じて `ng_type()` を呼んで確定出力
- `naginata_release()`
  - Method A の判定順で release イベントを分類
  - 薙刀対象キーの release のみ、従来通り `pressed_keys` / `n_pressed_keys` を更新し、確定判定を実施
- `ng_type()`
  - `NGList` を薙刀かなテーブル（`ngdickana`）で変換し、`raise_zmk_keycode_state_changed_from_encoded()` で実キー出力

補助関数（今回追加）:
- `is_naginata_target_keycode()`
- `is_modifier_keycode()`
- `is_layer_control_keycode()`
- `cancel_naginata_buffer()`
- `commit_naginata_buffer()`
- `passthrough_keycode()`


## 2) 判定順 before / after

### before（変更前）
`naginata_press()` / `naginata_release()` は実質的に「薙刀対象キーかどうか」しか見ておらず、

- 修飾キー（Cmd/Shift/Ctrl/Alt）中の通常キー入力
- レイヤー制御キー（MO/LT/TO/TG など）

を先に除外・透過する仕組みがありませんでした。

結果として、`&ng` 経路に入ったイベントは薙刀側解釈が優先されやすく、
S2/S5 の不具合につながっていました。

### after（Method A）
`naginata_press()` / `naginata_release()` の先頭で次順に判定:

1. **LAYER_CONTROL 最優先**
   - `is_layer_control_keycode()` に該当したらバッファに入れず即 pass-through
   - press 時は未確定バッファを `commit_naginata_buffer()` してから透過
2. **MODIFIER**
   - `is_modifier_keycode()` に該当したら `active_modifiers` を更新して pass-through
   - press 時はショートカット優先のため `cancel_naginata_buffer()`
3. **NAGINATA_TARGET**
   - 上記に該当しない場合のみ従来の薙刀バッファ処理へ
   - さらに `active_modifiers > 0` の間は対象キーも pass-through（ショートカット優先）


## 3) バッファポリシー（commit/cancel）

- **レイヤー制御キー**: `commit`
  - 理由: レイヤー遷移前に未確定入力を確定しておくことで、入力取りこぼしを防ぎつつ `MO/LT` を確実に通す
- **修飾キー押下**: `cancel`
  - 理由: Cmd 系ショートカット直前の未確定バッファを破棄し、ショートカット文字（例: Cmd+C）を薙刀変換させない


## 4) 再現シナリオ検証（S2/S5）

本リポジトリ単体での確認として、イベント処理ロジックの遷移を追跡し、S2/S5 を回帰確認しました。

### S2: Cmd+C が薙刀変換される問題

想定入力: `Cmd down` -> `C down/up` -> `Cmd up`

- `Cmd down`
  - modifier 判定に入り `active_modifiers++`
  - `cancel_naginata_buffer()` 実行
  - Cmd を pass-through
- `C down/up`
  - `active_modifiers > 0` のため薙刀バッファ処理をスキップし、C を pass-through
- `Cmd up`
  - modifier release で `active_modifiers--`

**結果**: Cmd+C は薙刀側へ入らず、そのままショートカットとして送出される。

### S5: 薙刀入力中に MO/LT が通らない問題

想定入力: 薙刀キーを押している最中に `MO/LT` を押下

- `MO/LT down`
  - layer_control 判定が最優先で成立
  - 必要なら `commit_naginata_buffer()` 後、MO/LT を pass-through
- `MO/LT up`
  - 同様に pass-through

**結果**: レイヤー制御キーは薙刀バッファへ入らず、数字レイヤーなどへの遷移が可能。


## 5) 影響範囲

- 実装変更: `src/behaviors/behavior_naginata.c` のみ
- `include/` 側の公開 API 変更はなし

