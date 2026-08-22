# ADR 0009: `_ShadowBorderColor` をプリセットの対象外とする

- ステータス: 承認済み
- 日付: 2026-08-22
- 関連: [ADR 0007](0007-fix-shadow-strength-gamma-conversion.md)

## 背景

README に記載の通り、本プリセットが変更するのは「ライティングと影に関するプロパティのみ」である。
`_ShadowBorderColor` / `_ShadowBorderRange` は影に関するプロパティでありながら、
初回コミット以来プリセットに含まれていない。この状態を意図的なものとして確定させる。

### lilToon における gradation 処理

`_ShadowBorderColor` は、影の合成の最終段で影境界に色を加算する。

```
lns.w      = lilTooningScale(_AAStrength, hL, _ShadowBorder, _ShadowBlur, _ShadowBorderRange)
indirectCol = lerp(indirectCol, directCol, lns.w * _ShadowBorderColor.rgb)
出力        = lerp(indirectCol, directCol, lns.x)
```

この処理は `if(_UseShadow)` の内側で**無条件に実行される**。有効・無効を切り替えるフラグは存在しない。
`_ShadowBorderRange` を `0` にしても `lns.w` が `lns.x` と一致するだけで、処理自体は消えない。

### 既定値が無色ではない

lilToon の既定値は以下の通りで、プリセットが触れない場合はこの値が使われる。

```
_ShadowBorderColor ("sShadowBorderColor", Color) = (1,0.1,0,1)
_ShadowBorderRange ("sShadowBorderRange", Range(0, 1)) = 0.08
```

さらに本プリセットは `_ShadowBlur` = `0.6425` と幅が広いため、`lns.w` の遷移帯は
`hL ∈ [0.28945, 1.0]` に及ぶ。lilToon 既定値（`_ShadowBlur` = `0.1`）における
`hL ∈ [0.37, 0.55]` と比べて格段に広い。

環境光のないシーンにおける、既定値のままのマテリアルでの明度の持ち上がり量は以下の通り。

| チャンネル | `_ShadowBorderColor` 既定値 | 最大増分 | 発生位置 |
| --- | --- | --- | --- |
| R | 1.0 | +0.245 | hL = 0.669 (N·L = 0.338) |
| G | 0.1 | +0.003 | hL = 0.887 |
| B | 0.0 | 0.000 | — |

R チャンネルの増分 +0.245 は、[ADR 0007](0007-fix-shadow-strength-gamma-conversion.md) の
近似誤差 ±0.068 の約 3.6 倍である。実質的には影境界が橙方向に色相シフトする。

## 決定

`_ShadowBorderColor` / `_ShadowBorderRange` をプリセットに追加せず、非設定を維持する。

あわせて README に以下を明示する。

- 本プリセットは影境界の色（`_ShadowBorderColor`）を変更しない
- hL² 本来の階調を得たい場合は `_ShadowBorderColor` を `(0, 0, 0, 1)` に設定する
- 肌マテリアル等で SSS 調の表現を載せたい場合はこのプロパティを用いる

## 理由

- **SSS 調の表現余地をユーザーに残すため。** 肌マテリアルにおいて影境界を暖色寄りにする表現は
  本プリセットの利用者が求める代表的な調整であり、その受け皿としてこのプロパティは適している。
- **ランプ形状を壊さないため。** gradation 処理は影境界にのみ作用し、`lns.x` によるランプ本体には
  影響しない。`_ShadowStrength` や `_ShadowBorder` を触って同種の表現を狙う場合と異なり、
  hL² の骨格を保ったまま質感の味付けだけを重ねられる。
- **汎用プリセットとしての性質に合わないため。** README に記載の通り、本プリセットは
  「マテリアルを選ばず共通プリセットとして適用しても違和感が出ない」ことを特徴としている。
  一方で最適な影境界色は肌・布・金属で異なり、プリセットが一律に決めるべき値ではない。

## 却下した案

### 案1: `_ShadowBorderColor` = `(0, 0, 0, 1)` をプリセットに追加する

- 利点: 既定で hL² の再現が保証される。プリセットは適用時に一度書き込むだけで以後の編集を妨げないため、
  ユーザーが後から色を入れる余地も失われない。
- 却下理由: 「既定でオンか、オプトインか」の選択であり、SSS 調の表現を積極的に活かす方針を採る。
  ただし本案は技術的な欠点を持たないため、後述の条件で再検討する。

### 案2: `_ShadowBorderRange` = `0` を設定する

- 却下理由: 効果がない。`lns.w` が `lns.x` と一致するだけで gradation 処理自体は無効化されず、
  `_ShadowBorderColor` が非ゼロである限り色は乗る。

## 影響

- `_ShadowBorderColor` に触れたことのないマテリアルへ本プリセットを適用すると、
  影境界に橙のグラデーションが乗る。ADR 0007 の近似誤差 ±0.068 はこのマテリアルでは保証されない。
- **ADR 0007 の誤差表は `_ShadowBorderColor` = `(0, 0, 0, 1)` を前提とした値である。**
- 両プリセット（`half-lambert.asset` / `half-lambert-with-backlight.asset`）に共通する。
- プリセットの asset ファイルは変更しないため、リリース対象は README のみ。
  挙動の変更を伴わないが、ADR 0007 / ADR 0008 と同じ `0.4.0` に含める。

## 再検討の条件

以下のいずれかに該当する場合、案1（プリセットで黒を設定する）へ切り替える。

- 「hL² の階調が既定で出ない」「影境界が意図せず橙になる」といった問い合わせや Issue が継続的に発生する場合
- README への明示だけでは利用者が `_ShadowBorderColor` の存在に気付かないことが実運用で確認された場合
