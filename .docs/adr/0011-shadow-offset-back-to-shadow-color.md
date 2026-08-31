# ADR 0011: 切片オフセットを `_ShadowStrength` から `_ShadowColor` へ戻す

- ステータス: 承認済み
- 日付: 2026-08-31
- 置き換え対象: [ADR 0002](0002-strength-based-offset.md), [ADR 0007](0007-fix-shadow-strength-gamma-conversion.md)

## 背景

[ADR 0002](0002-strength-based-offset.md) は hL² 近似の切片オフセット（影の最暗部を 0 まで
落とさず持ち上げる量）の表現を `_ShadowColor` から `_ShadowStrength` へ移し、
[ADR 0007](0007-fix-shadow-strength-gamma-conversion.md) がその値を `0.9319` に是正した。
結果として `_ShadowColor` は純黒 `(0, 0, 0, 1)` に固定されている。

ADR 0002 はこの配置の理由として「`_ShadowColor` が純黒になることで、ユーザーが影に色味を
付けたい場合にオフセット量を壊さずに色だけを変更できる」を挙げていた。**この理由は成立しない。**

lilToon の影色は乗算で効く。

```
indirectCol = lerp(albedo, shadowColorTex.rgb, shadowColorTex.a) * _ShadowColor.rgb
```

`_ShadowColor` を非黒にすると、色相と同時に切片オフセットが増える。環境光のないシーンでの
最暗部は、`_ShadowColor` を `S`（Linear）として

```
最暗部 = S * _ShadowStrength + (1 - _ShadowStrength)
       = S * 0.9319 + 0.0681
```

となる。たとえばカラーピッカーで赤 `(0.5, 0, 0)` を選ぶと `S` の R は
`sRGBToLinear(0.5) = 0.214` であり、最暗部の R は `0.268` になる。純黒時の `0.068` の
約 3.9 倍であり、「同じ明るさで赤っぽい影」にはならず単に影が明るくなる。
明るさを保つには `_ShadowStrength` を同時に引き下げる必要があり、その値は
ユーザーが手計算しない限り求まらない。ADR 0007 の影響欄が既に
「ユーザーが `_ShadowColor` に純黒以外を設定した場合、切片オフセットは `_ShadowStrength`
単独では決まらなくなる」と記録していた問題が、そのまま残っている。

肌のマテリアルで、明るさを保ったまま暖色寄りの影を作れるようにしたい。これは本プリセットの
利用者が求める代表的な調整であり、その受け皿として `_ShadowColor` は適している。

## 決定

切片オフセットの表現を `_ShadowStrength` から `_ShadowColor` に戻す。

- `_ShadowStrength`: `0.9319` → `1`
- `_ShadowColor`: `(0, 0, 0, 1)` → `(0.2895, 0.2895, 0.2895, 1)`

対象は両プリセット。

- `Presets/half-lambert.asset`
- `Presets/half-lambert-with-backlight.asset`

`0.2895` は sRGB 値であり、Unity の Color プロパティのガンマ変換を経てシェーダーには
`sRGBToLinear(0.2895) = 0.06815` として渡る（ADR 0007 の分析どおり）。これは
`1 - 0.9319 = 0.0681` と一致する。すなわち ADR 0002 以前の初期実装が採っていた値へ戻ることになる。

`_ShadowStrength` のエントリは削除せず `1` を明示する。過去に `0.9319` / `0.7105` が
焼き込まれたマテリアルを、再適用で確実に是正するためである（[ADR 0008](0008-shadow-env-strength-one.md)
が `_ShadowEnvStrength` について採ったのと同じ方針）。

## 理由

- **`_ShadowColor` が「影の最暗部の色」そのものになる。** `_ShadowStrength` = `1` のとき、
  環境光のない白色光下での最暗部の出力は `albedo * _ShadowColor` に一致する。
  ユーザーはカラーピッカーで見たままの色を指定でき、切片オフセットとの連動を意識する必要がない。
- **切片オフセットと色相を1つのプロパティで両立できる。** オフセットは「最暗部をどこまで
  持ち上げるか」であり、`_ShadowColor` の明度がそれを担い、色相・彩度が味付けを担う。
  ADR 0002 は「意味的には色ではなく強度である」としたが、`_ShadowColor` が乗算で効く以上、
  色の明度成分がオフセットを兼ねる構造は避けられない。分離できるという前提が誤りだった。
- **見た目が変わらない。** 後述のとおり、`lightColor` がクランプされている限り両者の出力は
  代数的に一致する。

## 等価性の検証

lilToon の実装（`lil_common_frag.hlsl` の Shadow Color 1 〜 Mix、`lil_common_macro.hlsl` の
`LIL_CORRECT_LIGHTCOLOR_*`）より、本プリセットの設定（`_ShadowMainStrength` = `0`、
`_Shadow2ndColor.a` = `_Shadow3rdColor.a` = `0`、影色テクスチャなし）における1チャンネル分の
出力は以下になる。

```
S    = _ShadowColor (Linear)
M    = clamp(LIL_MAINLIGHT_COLOR + shMax, _LightMinLimit, _LightMaxLimit)
f    = saturate(shMin * _ShadowEnvStrength)
m    = saturate((hL - borderMin) / (borderMax - borderMin))
c    = lns.w * _ShadowBorderColor
lns.x = lerp(1, m, _ShadowStrength)

indirect  = lerp(S * M, 1, f) → min(, M) → lerp(, M, c)
出力/albedo = lerp(indirect, M, lns.x)
```

案 A（変更前: `S` = `0`, `_ShadowStrength` = `0.9319`）と
案 B（変更後: `S` = `0.0681`, `_ShadowStrength` = `1`）を比較する。`g = 0.0681` として、

```
lerp(1, m, 1 - g) = g + (1 - g)m = lerp(g, 1, m)
```

という恒等式が成り立つ。これにより、**`M = 1` のとき両案の出力は任意の `f`・`m`・`c` に
ついて厳密に一致する**。`_ShadowBorderColor` の gradation 項 `c` を含めても差は 0 になる
（`c` の係数が両案で `g` の分だけずれるが、`lns.x` 側のずれと相殺する）。

`M < 1` のときのみ差が生じ、その量は

```
差 = g * f * (1 - M) * (1 - m)
```

`_LightMaxLimit` = `1` より `M = min(L + A, 1)` であり、`M < 1` は `L + A < 1` の暗いワールドに
限られる。`f = saturate(A * 0.55)` とあわせた上限は **0.0094** で、
[ADR 0007](0007-fix-shadow-strength-gamma-conversion.md) の近似誤差 ±0.068 の約 1/7 である。

`hL ∈ [0, 1]`、`L ∈ [0, 1]`、`A ∈ [0, 1.5]`、`_ShadowBorderColor ∈ {黒, 既定の橙}` を掃引した
数値検証の結果は以下の通り。

| 条件 | 最大差 |
| --- | --- |
| `M = 1`、`S` = `1 - 0.9319` （端数なし） | 2.2e-16（浮動小数点誤差） |
| `M = 1`、`S` = `sRGBToLinear(0.2895)` | 4.7e-05 |
| 全域（`M < 1` を含む） | 0.0094 |

`M = 1` での 4.7e-05 は、既存の `_ShadowStrength` = `0.9319` が4桁に丸められていることによる
残差であり（`sRGBToLinear(0.2895) - (1 - 0.9319) = 4.7e-05`）、構造的なずれではない。

## 明るさを保ったまま色相を変える基準

ユーザーが `_ShadowColor` の色相を変える際、「明るさを保つ」基準には
**CIELAB / LCh の `L*`（＝相対輝度）を用いる。HSV の `V` ではない。**

`L*` は `L* = 116 f(Y / Yn) - 16` と相対輝度 `Y` のみの単調関数であり、色相・彩度に依存しない。
一方 HSV の `V` は `max(R, G, B)` にすぎず明るさの指標ではないため、`V` を保ったまま彩度を
上げると影は暗くなる。既定値 `(0.2895, 0.2895, 0.2895)` の `L*` は `31.4` である。

| HSV `S`（hue 20°） | `V` 固定 `0.2895` の `L*` | `L*` を `31.4` に保つ `V` |
| --- | --- | --- |
| 0.0 | 31.4 | 0.2895 |
| 0.2 | 27.9 | 0.324 |
| 0.4 | 24.6 | 0.366 |

具体的な値と、hL² のチャンネル別誤差への影響は README に記載する。

## 却下した案

### 案1: `_ShadowColor` の既定値を暖色寄りにする

- 却下理由: 本プリセットは「マテリアルを選ばず共通プリセットとして適用しても違和感が出ない」
  ことを特徴としており、布や金属にも一律に暖色が乗るのは方針に合わない。
  またチャンネル別の誤差が既定で最大 +0.03 増える。暖色は用途を判断できるユーザーの
  オプトインとし、既定は見た目の変わらない中立グレーとする。

### 案2: `_ShadowColor` は純黒のまま、色相調整を `_ShadowBorderColor` に一本化する

- 却下理由: `_ShadowBorderColor` が着色するのは影境界の遷移帯のみであり
  （[ADR 0009](0009-shadow-border-color-out-of-scope.md)）、影全体の色味は変えられない。
  「肌の影を全体的に赤寄りにする」という要求は満たせない。両者は競合ではなく補完関係にある。

## 影響

- **見た目は変わらない。** `M = 1` で厳密に一致し、`M < 1` でも差は最大 0.0094。
  既存ユーザーが再適用しなくても実害はない。
- 色相を調整したいユーザーは、プリセット再適用後に `_ShadowColor` を変更するだけでよくなる。
  `_ShadowStrength` を触る必要はない。
- 既存マテリアルはプリセット適用時に値が焼き込まれているため、パッケージ更新だけでは
  新しい配置にならない。再適用が必要。
- ADR 0002 の決定と、ADR 0007 の決定（`_ShadowStrength` = `0.9319`）は本 ADR で置き換わる。
  ただし **ADR 0007 のガンマ変換の分析と誤差表は引き続き有効**であり、
  `0.2895` という値そのものがその分析から導かれている。無効になるのは
  「オフセットを `_ShadowStrength` に載せる」という配置の決定のみである。
- プロパティの意味づけが変わり再適用の案内が必要なため、`0.6.0` としてリリースする。

## 既知の制限

- **チャンネル別の誤差は色相を付けた分だけ増える。** hL² の近似誤差 ±0.068 は
  `_ShadowColor` が中立グレーであることを前提とする。色相を付けると各チャンネルの切片が
  `0.0681` からずれ、その分だけ誤差が増える。`L*` を保つ範囲で彩度 0.3 程度までなら
  ずれは最大 ±0.041 で、元の誤差の範囲内に収まる。
- **アルベドに元から無い色相は出ない。** `indirectCol = albedo * _ShadowColor` の乗算構造の
  ため、影の色はアルベドとの積で決まる。肌のように元から赤味のあるアルベドでは暖色の影が
  よく効く一方、寒色のアルベドに暖色を入れると単に暗くなる。
- **影色テクスチャが有効になる。** `_ShadowColorTex` や `_ShadowColorType` = LUT が既に
  設定されているマテリアルでは、従来 `_ShadowColor` = 純黒によって結果が 0 に潰れていた
  影色テクスチャが有効になる。本プリセットはテクスチャを設定しないため、該当するのは
  既にこれらを設定済みのマテリアルに限られる。
- `M < 1`（`L + A < 1` の暗いワールド）では、変更前より最大 0.0094 だけ影が明るくなる。

## 再検討の条件

本 ADR により影の色相を調整する窓口が `_ShadowColor` と `_ShadowBorderColor` の2つになった。
[ADR 0009](0009-shadow-border-color-out-of-scope.md) が `_ShadowBorderColor` を対象外とした
理由の1つは「SSS 調の表現余地をユーザーに残すため」であり、その役割の一部を `_ShadowColor` が
担うようになったことで、ADR 0009 の前提は弱まっている。

以下に該当する場合、ADR 0009 の案1（プリセットで `_ShadowBorderColor` = `(0, 0, 0, 1)` を
設定する）への切り替えを別 ADR で検討する。

- 影全体の色味は `_ShadowColor` で足り、影境界の橙は不要という判断が実運用で固まった場合
- 2つのプロパティの使い分けがユーザーに伝わらないことが確認された場合

## 関連

- [ADR 0009](0009-shadow-border-color-out-of-scope.md): `_ShadowBorderColor` は引き続き
  プリセットの対象外である。既定値のままのマテリアルでは、本 ADR の等価性は成立するが
  hL² の階調そのものは保証されない。
- [ADR 0010](0010-shadow-env-strength-under-light-max-limit.md): `_ShadowEnvStrength` = `0.55`
  の導出は `M = 1` 前提の相対コントラスト一致条件であり、上記の等価性がそのまま効くため、
  中立グレーの `_ShadowColor` のもとでも成立する。
