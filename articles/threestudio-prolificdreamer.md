---
title: "threestudioでProlificDreamerを動かす"
emoji: "🥐"
type: "tech"
topics:
  - "threestudio"
  - "ProlificDreamer"
  - "3d"
published: true
---

## はじめに

画像生成だけでなく、3Dモデル生成の品質もかなり上がってきました。特にtext-to-3Dモデルは、驚異的な進歩を見せています。

https://twitter.com/shinya7y/status/1662078190989484033

一番左の[DreamFusion](https://arxiv.org/abs/2209.14988)（2022年9月公開）から、一番右の[ProlificDreamer](https://arxiv.org/abs/2305.16213)（2023年5月公開）まで、僅か8ヶ月しか経っていません。
研究者であれ開発者であれクリエイターであれ経営者であれ、このぐらいの技術進歩速度は予見して行動すべきでしょうし、少なくとも追従できる必要があると思います。

幸いなことにProlificDreamerの論文公開3日後には、[技術解説記事](https://zenn.dev/kamata1729/articles/55fda1a884bdfe)が公開されています。
また、幸いなことに論文公開当日には、3Dモデル生成のライブラリである[threestudio](https://github.com/threestudio-project/threestudio)で非公式実装が公開されています。
公開当初から品質が改善され、昨日2023/6/3時点では以下のような生成ができるようです。

https://twitter.com/superbennyguo/status/1665050174907904000

本記事では、threestudioの環境を構築し、ProlificDreamer非公式実装を動かします。
まだ実装が不完全らしく論文ほどの生成品質ではありませんが、少なくとも[公式実装](https://github.com/thu-ml/prolificdreamer)が公開されるまでの検証には役立つと思います。

## 環境

- Ubuntu 22.04（WSL2上）
- GPU：RTX 3090
- VRAM：最低6GB、使用手法・実験条件次第では24GB以上
  - ProlificDreamer 64x64：約14GB
  - ProlificDreamer 256x256：約23GB
  - ProlificDreamer 512x512：OOMのため不明

NVIDIAドライバー・WSL2・Minicondaのインストール、conda-forgeチャンネル指定については、[別の記事](https://zenn.dev/shinya7y/articles/wsl2-conda-mmdet3)を参照。

## 環境構築

CUDA 11.8を使用する場合のインストール方法を示します。インストール済みのCUDA Toolkitを使用する場合は適宜省略・変更してください。

### CUDA Toolkitのインストール

- Ubuntu 22.04の場合
  - [CUDA 11.8 Ubuntu 22.04用コマンド](https://developer.nvidia.com/cuda-11-8-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_local)を実行
- WSL2上のUbuntuの場合
  - `sudo apt-key del 7fa2af80`
  - [CUDA 11.8 WSL-Ubuntu用コマンド](https://developer.nvidia.com/cuda-11-8-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=WSL-Ubuntu&target_version=2.0&target_type=deb_local)を実行

### threestudioのインストール

基本的には[README.md](https://github.com/threestudio-project/threestudio)記載の流れに従ってインストールします。
ただし、ここではvirtualenvを使用せず、仮想環境作成とPyTorchインストールにcondaを使用することにします。
また、[tiny-cuda-nn](https://github.com/NVlabs/tiny-cuda-nn)インストール時に、[`ld: cannot find -lcuda: No such file or directory`](https://github.com/NVlabs/tiny-cuda-nn/issues/183)というエラーが出たため、exportを追加しています。

```bash
git clone https://github.com/threestudio-project/threestudio.git
cd threestudio/

conda create -n threestudio python=3.11 -y
conda activate threestudio
conda install pytorch=2.0.1 torchvision=0.15.2 pytorch-cuda=11.8 -c pytorch -c nvidia

export PATH="/usr/local/cuda/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda/lib64:$LD_LIBRARY_PATH"
export LIBRARY_PATH="/usr/local/cuda/lib64/stubs:$LIBRARY_PATH"

pip install ninja
pip install -r requirements.txt
```

## ProlificDreamerの実行

### 訓練

[README.md](https://github.com/threestudio-project/threestudio#prolificdreamer-)記載のサンプルを実行します。

```bash
# object generation with 64x64 NeRF rendering, ~14GB VRAM
python launch.py --config configs/prolificdreamer.yaml --train --gpu 0 system.prompt_processor.prompt="a DSLR photo of a delicious croissant" data.width=64 data.height=64
# object generation with 512x512 NeRF rendering (original paper), >24GB VRAM
python launch.py --config configs/prolificdreamer.yaml --train --gpu 0 system.prompt_processor.prompt="a DSLR photo of a delicious croissant" data.width=512 data.height=512
# scene generation
python launch.py --config configs/prolificdreamer-scene.yaml --train --gpu 0 system.prompt_processor.prompt="Inside of a smart home, realistic detailed photo, 4k" data.width=64 data.height=64
```

訓練ログ等は`trial_dir`（上記configの場合、`outputs/prolificdreamer/[prompt]@[timestamp]/`）に保存されます。途中経過の画像や訓練完了時の動画は、`[trial_dir]/save/`に保存されます。

### メッシュ出力

メッシュを出力する場合は、`--export`オプションを使用します。

```bash
# 上記訓練で使用されたtrial_dirに置き換えてください
TRIAL_DIR=outputs/prolificdreamer/a_DSLR_photo_of_a_delicious_croissant@20230604-192515
CONFIG_FILE=${TRIAL_DIR}/configs/parsed.yaml
CHECKPOINT_FILE=${TRIAL_DIR}/ckpts/last.ckpt
python launch.py --config ${CONFIG_FILE} --export --gpu 0 resume=${CHECKPOINT_FILE} system.exporter_type=mesh-exporter system.exporter.context_type=cuda
```

`RuntimeError: Error building extension 'nvdiffrast_plugin_gl'`が出たため、`system.exporter.context_type=cuda`の指定を追加しています。

### テクスチャ出力

上記訓練で使用したconfigでは、`[WARNING] save_texture is True but no albedo texture found, using default white texture`という警告が出て、テクスチャが出力されません。
`material_type: "diffuse-with-point-light-material"`の場合のみテクスチャを出力できる実装のようです。
根本的な解決ではありませんが、以下のようにconfigの2箇所を変更し、訓練し直せば、テクスチャを出力できるようになります。

```yaml:prolificdreamer-dpl.yaml
  geometry:
    radius: 2.
    normal_type: analytic

  material_type: "diffuse-with-point-light-material"
  material:
    ambient_only_steps: 2001
    soft_shading: true
```

変更方法の妥当性や生成品質は未検証ですのでご注意ください。
