# ADR 0013: `_LightMinLimit` / `_BlendOpFA` をプリセットの管理対象から外す

- ステータス: 承認済み
- 日付: 2026-09-23
- 対象: `Presets/half-lambert.asset`

## 背景

`Half-Lambert影` は以下の値を設定していた。

| プロパティ | 値 |
| --- | --- |
| `_LightMinLimit` | `0.05` |
| `_LightMaxLimit` | `1` |
| `_BlendOpFA` | `0`（Add） |

VRChat のワールドはライトの設定がワールドごとに大きく異なり、明るすぎる・暗すぎるなど
破綻しているものが多い。これらのプロパティはそうしたワールドのライトに対してアバターの
明るさを合わせるための調整手段であり、プリセットを適用するたびに固定値へ戻されると
ユーザーが調整した値が失われる。

## 決定

`_LightMinLimit` と `_BlendOpFA` をプリセットから削除し、値をユーザー（マテリアル側）に委ねる。
プリセット適用後は、マテリアルに元から設定されていた値がそのまま使われる。

`_LightMaxLimit` は `1` のまま管理対象に残す。
`_ShadowEnvStrength` = `0.55`（[ADR 0010](0010-shadow-env-strength-under-light-max-limit.md)）は
一致条件 `e = M / (L + A)`（`M = min(L + A, _LightMaxLimit)`）を `_LightMaxLimit` = `1` で解いた値であり、
`_LightMaxLimit` が 1 以外になると影の濃さが理想値からずれるためである。

## 影響

- `_BlendOpFA` がプリセットで Add に揃えられなくなる。マテリアルの値によっては、
  リアルタイムポイントライト等の追加ライト（ForwardAdd パス）の合成結果がこれまでと変わる。
- `_LightMinLimit` は `L + A < _LightMinLimit` の暗いワールドでのみ `M` を変える。
  この範囲では ADR 0010 の誤差表は保証されない。ユーザーが `_LightMinLimit` を上げるほど、この範囲は広がる。
- 既存マテリアルへの影響はない。プリセットは適用時に値がマテリアルへ焼き込まれるため、
  パッケージ更新によって見た目が変わることはない。
