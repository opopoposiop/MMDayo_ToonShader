# 互換性と動作前提

## 必須コンポーネント

- [MikuMikuDayo](https://github.com/pennennennennennenem/MikuMikuDayo) の `.fxdayo` レンダラー
- `ToonAnime_32.fxdayo` から相対参照される `../hlsl/resources.hlsli` と `../hlsl/yrz.hlsli`
- レイトレーシング、TLAS、GBuffer、共有カメラ情報を提供する MikuMikuDayo 環境

`PPAL.fxdayo` による Bloom は任意ですが、使用する場合は [MMDayo_AL](https://github.com/opopoposiop/MMDayo_AL) 側の対応版と、AutoLuminous 判定定数および GBuffer レイアウトを一致させてください。

## 未固定の項目

現時点のリポジトリには、MikuMikuDayo の対応コミット、GPU・ドライバーの対応表、PPAL の対応リリースが固定されていません。公開リリースでは、実際に確認した組み合わせだけを README またはリリースノートへ記載します。

## 配置例

シェーダーから共有 HLSL を相対参照できるよう、概ね次の配置にします。

```text
<MikuMikuDayo の配置先>/
├── hlsl/
│   ├── resources.hlsli
│   └── yrz.hlsli
└── shaders/
    ├── ToonAnime_32.fxdayo
    ├── ToonAnime.pmx
    └── initial.vmd
```
