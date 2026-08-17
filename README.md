# ToonShader for MikuMikuDayo

[MikuMikuDayo](https://github.com/pennennennennennenem/MikuMikuDayo) 用のアニメ調トゥーンシェーダーです。

現在の推奨版は [`ToonAnime_32.fxdayo`](ToonAnime_32.fxdayo) です。`ToonAnime_32` は `ToonAnime_31` を基に、Edge・1号影・2号影の色をHSV形式で調整できるようにし、23個のPMXモーフを「目・まゆ・リップ・その他」へ整理した版です。

## 主な機能

- ライト・1号影・2号影による2段階アニメ影
- バックフェース拡張法によるシルエット輪郭線
- レイトレーシングによる SSAO
- PNG の連続アルファ、アルファマスク、透過テクセルへの明示対応
- 不透明、通常の半透明、半透明発光をピクセル単位で振り分ける描画
- AutoLuminous 方式を参考にした発光材質判定
- PPAL 用のモデル・面・重心座標・法線深度 GBuffer 出力

半透明面が深度を書いて発光ビームを消す問題、発光面の重なりが黒くなる問題、目材質が誤って発光・透過する問題、透明面が SSAO の不自然な影を作る問題への対策を含みます。

## ファイル

| ファイル | 内容 |
| --- | --- |
| [`ToonAnime_32.fxdayo`](ToonAnime_32.fxdayo) | 現行シェーダー。HSVカラーと23項目のPMXモーフ調整に対応 |
| [`ToonAnime.pmx`](ToonAnime.pmx) | `ToonAnime_32` のユーザー定義値コントローラー |
| [`initial.vmd`](initial.vmd) | 31版と同じ見た目になる初期値を23モーフのフレーム0へ設定 |
| [`CHANGELOG.md`](CHANGELOG.md) | `ToonAnime_2` から `ToonAnime_32` までの詳細な変更履歴 |
| [`PPAL.fxdayo`](https://github.com/opopoposiop/MMDayo_AL) | AutoLuminous ぽい Bloom を担当する外部ポストプロセス。本ディレクトリには未同梱 |
| `../hlsl/resources.hlsli` | MikuMikuDayo 側の共有リソース定義。シェーダーから相対参照 |
| `../hlsl/yrz.hlsli` | MikuMikuDayo 側の共有ヘルパー。シェーダーから相対参照 |

## 動作上の前提

- MikuMikuDayo の `.fxdayo` レンダラーとして読み込むことを前提としています。
- `ToonAnime_32.fxdayo` から見て `../hlsl/resources.hlsli` と `../hlsl/yrz.hlsli` を参照できる配置が必要です。
- 上記補足 `../MikuMikuDayo`傘下の`/renderer`に `ToonAnime_32.fxdayo`を配置
- `ToonAnime.pmx` を MikuMikuDayo に読み込み、モデルへ `initial.vmd` を適用してから使用してください。
- シェーダーはコントローラーをファイル名 `ToonAnime.pmx` で参照します。同じフォルダへの配置を推奨しますが、MikuMikuDayo がそのモデルを読み込める配置であれば必須ではありません。
- SSAO はレイトレーシングパス、TLAS、GBuffer、共有カメラ情報を利用します。これらを提供する MikuMikuDayo 環境が必要です。
- AutoLuminous 互換 Bloom を使用する場合は、対応する `PPAL.fxdayo` を別途用意してください。
- `PPAL.fxdayo` と併用する場合、AutoLuminous 判定定数と GBuffer レイアウトを両ファイルで一致させる必要があります。

詳細な前提は [`docs/compatibility.md`](docs/compatibility.md)、開発時の契約は [`docs/development.md`](docs/development.md)、公開前の確認項目は [`docs/validation.md`](docs/validation.md) を参照してください。

## ライセンスと謝辞

本プロジェクトの著作権者が権利を有するファイルには、ルートの [`LICENSE`](LICENSE) を適用します。外部プロジェクト、共有 HLSL、互換仕様に由来する部分の扱いは [`NOTICE.md`](NOTICE.md) を参照してください。

- [MikuMikuDayo](https://github.com/pennennennennennenem/MikuMikuDayo) の `.fxdayo` レンダラーと共有 HLSL 環境を利用しています。
- [MMDayo_AL](https://github.com/opopoposiop/MMDayo_AL) は、Bloom 用の外部 PPAL として連携できます。
- [AutoLuminous](https://www.nicovideo.jp/watch/sm16087751) の方式を参考にしています。そぼろ氏の作品との互換動作を目指したものであり、公式プロジェクトとの関係や動作保証を示すものではありません。

## 描画の流れ

```text
BG → GBuffer → Edge → MMD → MMDTrans → SSAO → Copy
```

| パス | 役割 |
| --- | --- |
| `BG` | カメラ方向からスカイボックスを描画 |
| `GBuffer` | 不透明ピクセルの法線、深度、モデル番号、面番号、重心座標を保存 |
| `Edge` | PMX のエッジフラグとテクスチャアルファを尊重して輪郭線を描画 |
| `MMD` | 不透明ピクセルと不透明発光を描画し、深度を書き込み |
| `MMDTrans` | 通常の半透明と半透明発光を、深度を書かずに描画 |
| `SSAO` | 不透明面を基準に短い AO レイを飛ばし、遮蔽量を計算 |
| `Copy` | SSAO を色へ適用し、PPAL 用 GBuffer をグローバル領域へ転写 |

## はじめて調整する場合

1. `ToonAnime_32.fxdayo` と `ToonAnime.pmx` を MikuMikuDayo から利用できる場所へ配置します。
2. `ToonAnime.pmx` を読み込み、そのモデルへ `initial.vmd` を適用します。
3. `ToonAnime.pmx` のモーフを一度に1項目だけ、小さな幅で変更します。
4. 同じモデル、カメラ、照明、背景で変更前後を比較します。
5. 不透明物、髪やレース、PNG の穴、発光ビーム、交差する半透明面を確認します。

モーフ値はすべて `0`～`1` です。負のしきい値、広い値域、整数値はシェーダー内で安全な範囲へ変換されます。処理コード、パス定義、リソース名、ブレンド設定を編集して見た目を調整することは避けてください。これらは複数のパスや PPAL と相互に依存しています。

### モーフグループ

| PMXグループ | 収録する設定 |
| --- | --- |
| `目` | `EdgeWidth`、`EdgeAlpha`、`EdgeH/S/V/A` |
| `まゆ` | `Shadow1Pos`、`Shadow1Mul`、`Shadow1H/S/V`、`ToonShadow1` |
| `リップ` | `Shadow2Pos`、`Shadow2Mul`、`Shadow2H/S/V`、`ToonShadow2` |
| `その他` | `ShadowSmooth`、`ToonLight`、`AORadius`、`AOSamples`、`AOStrength` |

HSVの `H` は色相、`S` は彩度、`V` は明度です。`S = 0` の間は無彩色になるため、`H` を変更しても見た目は変わりません。色を付ける場合は `H` を選んだ後に `S` を上げてください。

## ユーザーが変更できる設定

### 輪郭線

| PMXモーフ | initial.vmd | シェーダー値域 | 初期シェーダー値 | 効果 |
| --- | ---: | ---: | ---: | --- |
| `EdgeWidth` | `0.2` | `0`～`0.01` | `0.002` | 輪郭線の太さ |
| `EdgeAlpha` | `0.9` | `0`～`1` | `0.9` | 輪郭線を描ける不透明度の下限 |
| `EdgeH` | `0.0` | `0`～`1` | `0.0` | 灰色輪郭の色相 |
| `EdgeS` | `0.0` | `0`～`1` | `0.0` | 灰色輪郭の彩度 |
| `EdgeV` | `0.55` | `0`～`1` | `0.55` | 灰色輪郭の明度 |
| `EdgeA` | `1.0` | `0`～`1` | `1.0` | 灰色輪郭の不透明度 |

`Edge` パスの深度設定や頂点拡張式を太さ調整に使わないでください。表面との深度競合や、透明部分の黒塗りが再発する可能性があります。

### 2段階アニメ影

| PMXモーフ | initial.vmd | シェーダー値域 | 初期シェーダー値 | 効果 |
| --- | ---: | ---: | ---: | --- |
| `Shadow1Pos` | `0.55` | `-1`～`1` | `0.1` | ライトから1号影へ切り替わる向き |
| `Shadow2Pos` | `0.425` | `-1`～`1` | `-0.15` | 1号影から2号影へ切り替わる向き |
| `ShadowSmooth` | 約`0.09548` | `0.001`～`0.2` | `0.02` | 影境界の柔らかさ |
| `Shadow1Mul` | `0.65` | `0`～`1` | `0.65` | 1号影の明るさ |
| `Shadow2Mul` | `0.35` | `0`～`1` | `0.35` | 2号影の明るさ |
| `Shadow1H` | `0.0` | `0`～`1` | `0.0` | 1号影の色相 |
| `Shadow1S` | `0.0` | `0`～`1` | `0.0` | 1号影の彩度 |
| `Shadow1V` | `1.0` | `0`～`1` | `1.0` | 1号影の色乗算明度 |
| `Shadow2H` | `0.0` | `0`～`1` | `0.0` | 2号影の色相 |
| `Shadow2S` | `0.0` | `0`～`1` | `0.0` | 2号影の彩度 |
| `Shadow2V` | `1.0` | `0`～`1` | `1.0` | 2号影の色乗算明度 |
| `ToonLight` | `0.1` | `0`～`1` | `0.1` | Toon テクスチャのライト側サンプル位置 |
| `ToonShadow1` | `0.5` | `0`～`1` | `0.5` | Toon テクスチャの1号影サンプル位置 |
| `ToonShadow2` | `0.9` | `0`～`1` | `0.9` | Toon テクスチャの2号影サンプル位置 |

影を明るくする場合は `Shadow1Mul` と `Shadow2Mul` を少し上げます。色味を付ける場合は各影の `H` を選び、`S` を上げます。`V = 1` は色による追加減光なしです。影境界を柔らかくする場合は `ShadowSmooth` を少し上げます。`Shadow1Pos` が `Shadow2Pos` より大きい関係、1号影が2号影より明るい関係を保ってください。

### SSAO

| PMXモーフ | initial.vmd | シェーダー値域 | 初期シェーダー値 | 効果・注意 |
| --- | ---: | ---: | ---: | --- |
| `AORadius` | 約`0.24623` | `0.1`～`20` | `5.0` | 遮蔽物を探す最大距離。大きすぎると広い範囲が暗くなる |
| `AOSamples` | `0.2` | 整数`1`～`16` | `4` | 1ピクセルあたりのレイ本数。増やすと滑らかになるが重くなる |
| `AOStrength` | `0.5` | `0`～`1` | `0.5` | AO の暗さ。大きすぎると接触部が黒くなる |

`AO_BIAS = 1e-3` は自己遮蔽を避ける内部補正であり、PMXモーフからは変更できません。通常は変更せず、変更時はコードレビューと回帰確認を行ってください。

### initial.vmdと値の復元

PMXには既定のモーフ値を保存できないため、初期状態は `initial.vmd` のフレーム0キーで設定します。Edgeは `H=0、S=0、V=0.55` で31版の灰色を再現し、両方の影色は `H=0、S=0、V=1` の白にして追加の色変化を発生させません。その他の値域変換後の結果も31版と一致します。例えば、`Shadow2Pos = 0.425` は `-0.15`、`AOSamples = 0.2` は整数 `4` に変換されます。

## 変更してはいけない箇所

次の項目は単独で変更しないでください。

- `[YRZFX]` 内のテクスチャ名、形式、パス名、パス順
- ブレンド設定、深度書き込み設定、カリング設定
- `NURI` と bindless テクスチャ参照
- PMX フラグ値 `MAT_FLAG_DOUBLESIDE`、`MAT_FLAG_EDGE`
- `AL_SPEC_THRESHOLD`、`AL_SHIN_THRESHOLD`、`AL_MAX_OVERBRIGHT`
- `OPAQUE_CUT`
- `TEX_ALPHA_CUT`、`ALPHA_EPSILON`
- `MMDColor` の戻り値アルファと前乗算アルファの規約
- GBuffer のパッキングと PPAL 側のデコード契約
- GBuffer の半透明除外と、SSAO `AnyHit` の半透明 `IgnoreHit`

これらを変更する場合は、`MMD`、`MMDTrans`、`GBuffer`、`SSAO`、`Copy`、PPAL を一体としてコードレビューと回帰確認を行ってください。

## 透過と発光の扱い

`MMDColor` は前乗算アルファ形式で色を返します。

| 種類 | 出力 | 描画パス | 深度書き込み | 合成 |
| --- | --- | --- | --- | --- |
| 通常の不透明 | `rgb × a, a` | `MMD` | あり | 通常合成 |
| 通常の半透明 | `rgb × a, a` | `MMDTrans` | なし | 通常合成 |
| 不透明発光 | `emit, 1` | `MMD` | あり | 背景を遮蔽して上書き |
| 半透明発光 | `emit × coverage, 0` | `MMDTrans` | なし | 純加算 |

この分離により、半透明発光を重ねても背景が減衰して黒くならず、交差する発光面同士が深度で切り取られません。一方、不透明な目や発光パネルは深度を書き、背後を正しく隠します。

AutoLuminous 方式Aの発光判定は、概ね「スペキュラ色が黒に近い」かつ「`shininess >= 100`」です。`PPAL.fxdayo` と併用する場合は、同じ条件を使用してください。

## SSAO と半透明

SSAO は透明面の合成後に最終色へ適用されます。そのため半透明面を GBuffer や AO レイの遮蔽面として登録すると、透明面の奥にあるキャラクターまで不自然に暗くなります。

`ToonAnime_32` では、`ToonAnime_28` から継承した次の二重対策を行います。

1. `GBuffer` は `c.a >= OPAQUE_CUT` の不透明ピクセルだけを保存します。
2. SSAO の `AnyHit` は、材質アルファが `OPAQUE_CUT` 未満の三角形を `IgnoreHit()` で通過させます。

現行の `AnyHit` は材質アルファを参照し、PNG のピクセル単位アルファまでは参照しません。レイヒット地点の UV が現状の `AnyHit` にないため、不完全な推測でテクスチャを読むより誤判定を避ける設計です。

## 確認項目

変更後は少なくとも次を確認してください。

- 不透明な目、肌、衣装が発光せず、背後を正しく隠す
- 半透明の髪、レース、ビームが奥の不透明物を必要以上に隠さない
- 交差する半透明発光プレーンが黒くならず、交差線で欠けない
- PNG の完全透明部に色、輪郭線、深度、AO の影が残らない
- 半透明ポリゴンが SSAO の基準面やレイ遮蔽面にならない
- エッジ無効材質と AutoLuminous 材質に不要な輪郭線が出ない
- PPAL の発光判定と `IsALEmissiveMat` が一致する
- 透視投影と平行投影の両方で背景と SSAO の位置再構築が正しい

## 変更履歴の要点

| 版 | 主な変更 |
| --- | --- |
| `2` | 2トーン影、ハードスペキュラ、リムライト、画面空間輪郭、分離型Diffusion、FXAAを実装 |
| `3` | 彩度、拡散グロー、Sobel輪郭を`AnimeEffect`へ統合し、`TempBuffer`経由のFXAAへ再編 |
| `4` | 細輪郭とDiffusionを`Copy`へ統合し、モデル側1ピクセル輪郭とCopy後FXAAへ変更 |
| `6` | アニメ調トゥーン、シルエットエッジ、FXAA |
| `9` | 2段階アニメ影、レイトレーシング AO |
| `11` | SSAO への改称、HDR Bloom、高品質 FXAA |
| `12`～`20` | AutoLuminous 互換判定、発光バッファ、頂点発光、Bloom の検証と修正 |
| `21` | 発光と Bloom を PPAL へ分離 |
| `22` | PNG アルファの明示処理 |
| `23` | 透過部分を黒く塗る Edge パスの修正 |
| `24` | 前乗算アルファと発光の純加算 |
| `25`～`26` | 不透明・半透明・発光パスと深度書き込みの再編 |
| `27` | 不透明発光と半透明発光を分離し、目の発光・透過回帰を修正 |
| `28` | 半透明面を GBuffer と SSAO 遮蔽から除外 |
| `29` | `28` と同じコードへ編集ガイドと保守コメントを追加 |
| `30` | FXAAパス、FXAA定数、FXAAシェーダー本体を削除 |
| `31` | 17個のユーザー調整値を `ToonAnime.pmx` のモーフ制御へ変更し、`initial.vmd` を追加 |
| `32` | Edge色をHSV化し、1号影・2号影のHSV色を追加。23モーフを4グループへ整理 |

収録版に基づく変更履歴は [`CHANGELOG.md`](CHANGELOG.md) を参照してください。過去版シェーダー本体は公開対象外です。

### 履歴上の注意

- `ToonAnime_5`、`7`、`8`、`10` はアーカイブに存在しないため、個別の変更内容は未確認です。
- `ToonAnime_16` のリムライトは、`ToonAnime_15` をベースに再構成した `17` 以降へ継承されていません。
