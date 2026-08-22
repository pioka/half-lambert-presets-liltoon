# ADR 0008: _ShadowEnvStrength を 1.0 にする

- ステータス: [ADR 0010](0010-shadow-env-strength-under-light-max-limit.md) により置き換え
- 日付: 2026-08-22
- 置き換え対象: [ADR 0004](0004-shadow-env-strength-zero.md)

> **注記:** 本 ADR の導出は `_LightMaxLimit` によるライト色のクランプを考慮しておらず、
> 結論の `_ShadowEnvStrength` = `1.0` は誤りである。正しい値と導出は
> [ADR 0010](0010-shadow-env-strength-under-light-max-limit.md) を参照。
> 環境光をランプで減衰させるべきではないという本 ADR の問題提起自体は有効である。

## 背景

[ADR 0004](0004-shadow-env-strength-zero.md) では `_ShadowEnvStrength` を `0.2` から `0` に変更した。
理由は「影の明度に環境光依存の加算項が乗ると、hL² 近似の精度が環境依存で不定になる」というものである。

この判断は、本プリセットが再現対象としている Valve のモデルの形を取り違えていた。

### Valve のモデルと lilToon の差

Valve の Half-Lambert は、直接光の減衰カーブを置き換えるものであり、環境光（間接光）は
減衰させない独立した加算項として残る。

```
出力 = albedo * (L * hL² + A)        … L: 直接光, A: 環境光
```

一方 lilToon は、環境光をライト色に畳み込んでから影のランプを掛ける。

```
lightColor = LIL_MAINLIGHT_COLOR + shMax        // lil_common_functions.hlsl
directCol  = albedo * lightColor
出力       = lerp(indirectCol, directCol, ランプ)
```

つまり `_ShadowEnvStrength` = `0` のとき、出力は次の形になる。

```
出力 = albedo * (L + A) * G          … G: 本プリセットのランプ出力（ADR 0007 の近似値）
```

`A * G` の項が生じており、**環境光まで hL² のカーブで減衰している**。
これは Valve のモデルには存在しない項である。

すなわち `_ShadowEnvStrength` = `0` は「hL² を忠実にする設定」ではなく、
「Valve のモデルにない環境光の減衰を導入する設定」だった。

### 近似誤差は e = 0 の方が環境依存だった

ADR 0004 が避けようとした「環境依存で不定になる」現象は、実際には `_ShadowEnvStrength` = `0`
の側で発生する。Valve のモデルに対する最大誤差を環境光量ごとに比較すると以下の通り。

| A（環境光 / 直接光） | `_ShadowEnvStrength` = 0 | `_ShadowEnvStrength` = 1.0 |
| --- | --- | --- |
| 0.0 | 0.068 | 0.068 |
| 0.2 | 0.254 | 0.068 |
| 0.4 | 0.440 | 0.068 |
| 0.6 | 0.627 | 0.068 |
| 0.8 | 0.813 | 0.068 |
| 1.0 | 0.999 | 0.068 |

`e = 0` では誤差が環境光量に比例して増大し、`A ≒ 0.4` の一般的なワールドで既に
ADR 0007 の近似誤差 ±0.068 の6倍を超える。`e = 1.0` では環境光量に依らず ±0.068 で一定であり、
ADR 0004 が求めた「環境を問わず精度が保証される」状態は `e = 1.0` によって達成される。

## 決定

両プリセットの `_ShadowEnvStrength` を `0` から `1.0` に変更する。

- `Presets/half-lambert.asset`
- `Presets/half-lambert-with-backlight.asset`

あわせて ADR 0004 のステータスを「ADR 0008 により置き換え」に更新する。

## 理由

`_ShadowColor` = `(0, 0, 0, 1)` のとき、lilToon の影の合成は以下のようになる。

```
f          = saturate(shMin * _ShadowEnvStrength)
indirectCol = albedo * f                          // 黒影色のため lightColor 乗算後も 0、env で f まで持ち上がる
directCol   = albedo * (L + shMax)
出力        = albedo * (f * (1 - G) + (L + shMax) * G)
            = albedo * (L * G + shMax * G + f * (1 - G))
```

Valve のモデル `albedo * (L * hL² + A)` と一致する条件は、`G ≒ hL²` を前提として

```
shMax * G + f * (1 - G) = shMax
⇒ f * (1 - G) = shMax * (1 - G)
⇒ f = shMax
```

環境光が概ね等方であれば `shMin ≒ shMax` であり、`f = saturate(shMin * e)` から
**`_ShadowEnvStrength` = `1.0`** が得られる。

この条件は `G` に依存せずに成立する。すなわち特定の環境光量や特定の `N·L` の一点で
合わせ込んだ値ではなく、全階調・全環境光量にわたって成立する構造的な値である。

## 影響

- **明るいワールドで影が従来より明るくなる。** 環境光が影に落ちなくなるためであり、
  Valve のモデルに近づいた結果である。ADR 0004 の「明るいワールドでは従来より影がわずかに濃く見える。
  これは hL² 本来のカーブに近づいた結果であり、退行ではない」という記述は誤りであり、本 ADR で訂正する。
- 環境光のないシーン（`A = 0`）では挙動が変わらない。ADR 0007 の近似がそのまま出る。
- 既存マテリアルはプリセット適用時に値が焼き込まれているため、パッケージ更新だけでは反映されない。
  再適用が必要。
- ADR 0004 の方針に従いエントリ自体は残し、`value: 1` を明示する。
  過去に `0` や `0.2` が焼き込まれたマテリアルも再適用で確実に是正される。
- 挙動が変わる変更のため、`0.4.0` としてリリースする。

## 既知の制限

いずれも本 ADR の範囲では対処せず、制限として記録する。

- **`shMin` と `shMax` の乖離。** lilToon は `shMax` をライト方向、`shMin` をその逆方向の
  SH 評価値として取る（`lilGetToonSHDouble` / `lilGetSHToonMin`）。空のグラデーションが強い、
  あるいはワールドの直接光が SH に焼き込まれているシーンでは `shMin < shMax` となり、
  復元量が不足して影が Valve のモデルより濃く出る。過剰補正の方向には振れないため、安全側の誤差である。
- **`_LightMaxLimit` による影の消失。** 本プリセットは `_LightMaxLimit` = `1` を設定しており、
  `lightColor` は 1 で頭打ちになる。一方 `f = saturate(shMin * 1.0)` も 1 で頭打ちになるため、
  環境光が 1 を超えるワールドでは `indirectCol` と `directCol` が共に `albedo` に一致し、
  直後の `min(indirectCol, directCol)` を経て影が完全に消える。
  `_LightMaxLimit` はアバターの明るさ上限を担う別目的のプロパティであり、
  本プリセットの都合で引き上げるべきではないと判断する。
- **ForwardAdd パスには環境光の復元が入らない。** 環境光の項は
  `#if !defined(LIL_PASS_FORWARDADD)` の内側にあるため、点光源・スポットライトによる
  加算パスでは `_ShadowEnvStrength` が効かない。加算パスに環境光を足すのは
  二重計上であり、これは正しい挙動である。
- **`_ShadowColor` が純黒であることが前提。** 純黒以外を設定すると `indirectCol` が
  `lightColor` 乗算後に 0 にならず、上記の導出が崩れる。

## 関連

`_ShadowBorderColor` をプリセットの対象外とする判断については
[ADR 0009](0009-shadow-border-color-out-of-scope.md) を参照。
同プロパティが既定値のままのマテリアルでは、本 ADR および ADR 0007 の誤差は保証されない。
