# ADR 0015: 影を影1+影2の2段構成にする

- ステータス: 承認済み
- 日付: 2026-10-18
- 対象: `Presets/half-lambert.asset`, `Presets/half-lambert-backlight.asset`
- 置き換え対象: [ADR 0011](0011-shadow-offset-back-to-shadow-color.md) の `_ShadowColor` の値、
  [ADR 0007](0007-fix-shadow-strength-gamma-conversion.md) の影1のランプ（`_ShadowBorder` / `_ShadowBlur`）

## 背景

従来は影1だけの直線ランプ1本で hL² を近似していた。最暗部は `0.068` に持ち上がり、
真裏まで黒に落ちない。

## 決定

影1と影2の2段構成にし、`NdotL` が 0〜1 の区画を影1、-1〜0 の区画を影2に担当させる。

| プロパティ | 値 |
| --- | --- |
| `_ShadowColor` | `(0.5371, 0.5371, 0.5371, 1)`（リニア値 `0.25` の sRGB 表記） |
| `_ShadowBorder` | `0.75` |
| `_ShadowBlur` | `0.5` |
| `_ShadowStrength` | `1`（変更なし） |
| `_Shadow2ndColor` | `(0, 0, 0, 1)` |
| `_Shadow2ndBorder` | `0.25` |
| `_Shadow2ndBlur` | `0.5` |
| `_Shadow2ndNormalStrength` | `1` |
| `_Shadow2ndReceive` | `0` |
| `_Shadow3rdColor` | `(0, 0, 0, 0)`（オフ、変更なし） |

`hL = NdotL * 0.5 + 0.5` より、影1は `hL ∈ [0.5, 1]`、影2は `hL ∈ [0, 0.5]` で遷移する。
lilToon 2.3.4 の `lilGetShading` に沿うと、環境光のない白色光下での出力は

```
NdotL ∈ [0, 1]  : 0.25 + 0.75 * NdotL
NdotL ∈ [-1, 0] : 0.25 * (1 + NdotL)
```

となり、hL² と `NdotL` = -1, 0, 1 の3点（`0`, `0.25`, `1`）で一致する折れ線になる。

`_Shadow2ndNormalStrength` は影1（`_ShadowNormalStrength` = `1`）に揃える。
lilToon は影1と影2で `hL` を別々に求めるため、ここが食い違うと区画の境目で階調が途切れる。

`_Shadow2ndReceive` は `0`（lilToon 既定値）とし、落ち影は影1だけが受ける。
オブジェクトが落ち影の中にあっても、ある程度の階調を残すためである。
影1と影2の両方が落ち影を受けると、落ち影の中は一律に黒（`0`）となり階調が消える。
影2が受けなければ、落ち影の中は `NdotL ∈ [0, 1]` で `0.25`、`NdotL ∈ [-1, 0]` で
`0.25 * (1 + NdotL)` となり、明暗境界から真裏に向かう階調が残る。
代わりに落ち影の中の光が当たる側は、真裏の自己影（`0`）より明るくなる。

## 影響

- **見た目が変わる。** 真裏が黒まで落ち、明暗境界付近は従来より明るくなる。
- `_ShadowEnvStrength` = `0.55`（[ADR 0010](0010-shadow-env-strength-under-light-max-limit.md)）の導出は
  影のランプ形状に依存しないため、そのまま維持する。
- `_ShadowBorderColor` の遷移帯（[ADR 0009](0009-shadow-border-color-out-of-scope.md)）は
  影1の範囲・ぼかしに従って `hL ∈ [0.42, 1.0]` に変わる。プリセットの対象外である点は変わらない。
- README の、`_ShadowColor` = `0.2895` を前提とした色味調整の数値表は無効になったため削除した。
- 逆光ライトの Blur / Border を [ADR 0001](0001-backlight-parameters.md) の式（k = 0.25）で再計算する。
  `Presets/half-lambert-backlight.asset` の `_BacklightBlur` を `0.16` → `0.125`、
  `_BacklightBorder` を `0.42` → `0.4375` とする。逆光の輪郭は従来より細くなる。
- 既存マテリアルはプリセット適用時に値が焼き込まれているため、反映には再適用が必要。
