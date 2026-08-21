# ADR-0001: 逆光ライトプリセットの Blur / Border 決定則

- Status: Accepted
- Date: 2026-08-22 (記録日 / 決定自体は `b2ed873` 逆光ライト用プリセットの追加時)
- Scope: `Presets/half-lambert-squared-backlight.asset`

## Context

逆光ライト (`_UseBacklight`) のパラメータのうち `_BacklightBlur` と `_BacklightBorder` を
どのような根拠で決めるかが不明瞭だった。逆光は影 (`_UseShadow`) と同じ `lilTooningScale` を
通るため階調カーブは共通だが、**入力の定義域が異なる**ため、影のパラメータをそのまま流用できない。

### 前提となる制約

完全逆光時、`E = normalize(-headV * VS + L)` に `L = -headV` を代入すると、
`VS` (`_BacklightViewStrength`) の値に関わらず `E` は `-headV` に一致する。
したがって可視面 (`dot(headV, N) >= 0`) では常に

```
LN = dot(-headV, N) * 0.5 + 0.5 <= 0.5
```

つまり**逆光の入力は最大 0.5 までしか来ない**。`_BacklightDirectivity` や
`_BacklightViewStrength` をどう振ってもこの上限は変わらない。
影側の `ln` が 0〜1 を使うのに対し、逆光は上半分を永久に使わないという非対称がある。

## Decision

逆光の Blur / Border を、影の Blur と定数 `k` から一意に決める。

```
_BacklightBlur   = k * _ShadowBlur
_BacklightBorder = 0.5 - _BacklightBlur / 2
```

本プリセットでは **k = 0.25** を採用する。

### Border は自由変数ではなく従属変数

上記の制約より、`borderMax = Border + Blur / 2` が 0.5 を超えると
シルエットでも最大輝度に到達しない。逆に 0.5 未満だと縁で飽和して階調が潰れる。
よって `_BacklightBorder = 0.5 - _BacklightBlur / 2` に固定される。

影側の `borderMax` が 1.0 であることから、これは
「**borderMax だけが影の 0.5 倍に固定される**」と言い換えられる。
Border 単体に自然な倍率は存在せず、Blur を決めれば一意に決まる。

### Blur が唯一の調整箇所であり、影に対する倍率で語れる

| k (= Blur / 影のBlur) | 見え方 | 階調が乗る画面半径 |
| --- | --- | --- |
| 0.5 | シェーディングと同一の階調法則で回り込む「透過・ラップ」 | hL: r=0〜1 (全面) / hL²: r=0.78〜1 |
| 0.25 | 明確なリムライト | r≒0.95〜1 |
| 0.5 超 | 逆光の方が地の陰影より緩くなり、ブルーム/靄に見える (非推奨の上限) | — |

k = 0.5 が理屈上の基準点であり、影が hL なら逆光も hL、hL² なら逆光も hL² という
同一カーブになるため無調整で破綻しない。ただしリムではなく面全体の透過光になる。

本プリセットは**逆光リムとして使いたい**ため k = 0.25 を採用した。
k > 0.5 は実質的な上限であり、超えてはならない。

## Consequences

- `half-lambert-squared` の `_ShadowBlur = 0.6425` に対し、
  `_BacklightBlur = 0.25 * 0.6425 ≒ 0.16`、`_BacklightBorder = 0.5 - 0.16 / 2 = 0.42`。
  現行アセットの値はこの式から導出されたものである。
- 影側の Blur を変更した場合、逆光の Blur / Border も上式で再計算する必要がある。
  両者は独立に調整してはならない。
- k は 0.25〜0.5 の範囲でのみ意匠上の選択肢となる。

## Notes

- 当時は線形 Half-Lambert (hL) 用の逆光プリセットも存在したが、
  `a2078c2` で hL プリセットごと削除された。上表の hL 列はその当時の記録である。
- 本 ADR は既に実装済みの決定を後から文書化したものであり、新たな変更を伴わない。
