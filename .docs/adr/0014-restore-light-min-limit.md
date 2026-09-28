# ADR 0014: `_LightMinLimit` をプリセットに戻し、ライト方向のオーバーライドをオブジェクト追従にする

- ステータス: 承認済み
- 日付: 2026-09-28
- 対象: `Presets/half-lambert.asset`
- 一部置き換え対象: [ADR 0013](0013-light-limit-out-of-scope.md)（`_LightMinLimit` に関する部分）

## 背景

[ADR 0013](0013-light-limit-out-of-scope.md) で `_LightMinLimit` をプリセットの管理対象から外したため、
ライトがほとんど無い暗いワールドでの最低限の明るさがマテリアル側の値に依存している。

暗いワールドでは明るさだけでなく、影の向きも定まらない。lilToon のライト方向は

```
normalize(SH の方向（y は絶対値） + メインライトの方向 × メインライトの輝度 + _LightDirectionOverride.xyz)
```

で求まるため、ワールドのライトがほぼ 0 のときは `_LightDirectionOverride.xyz` が方向を決める。
`_LightDirectionOverride.w` = `0`（lilToon の既定）ではこのベクトルがワールド空間で解釈されるため、
アバターの姿勢によって影の出る面が変わる。`w` ≠ `0` では、長さを保ったままオブジェクトの回転で変換される。

## 決定

`Half-Lambert影` に以下を設定する。

| プロパティ | 値 |
| --- | --- |
| `_LightMinLimit` | `0.05` |
| `_LightDirectionOverride` | `(0.001, 0.002, 0.001, 1)` |

`_LightMinLimit` は ADR 0013 以前の値に戻す。

`_LightDirectionOverride` は xyz を lilToon の既定値のまま、w（オブジェクト追従）だけを `1` にする。
xyz を既定の微小値に留めるのは、ワールドにライトがある場合はその方向を優先させ、
オーバーライドをライトがほぼ無い場合にだけ効かせるためである。

暗いワールドでの最低限の見え方は、`_LightMinLimit` が明るさの下限を、
`_LightDirectionOverride` がアバターに対する光の向きを固定することで担保する。

`_BlendOpFA` は ADR 0013 のとおり管理対象外のままとする。

## 影響

- プリセットを適用すると、マテリアル側で調整した `_LightMinLimit` と `_LightDirectionOverride` が上書きされる。
- `L + A < 0.05` の暗いワールドでは [ADR 0010](0010-shadow-env-strength-under-light-max-limit.md) の
  誤差表は保証されない（ADR 0013 以前と同じ状態）。
- ライトがほぼ無いワールドでは、影の向きがアバターのオブジェクトの回転に追従し、姿勢によらず同じ面が明るくなる。
- 既存マテリアルへの影響はない。プリセットは適用時に値がマテリアルへ焼き込まれるため、
  パッケージ更新によって見た目が変わることはない。
