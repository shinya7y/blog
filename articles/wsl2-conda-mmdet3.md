---
title: "WSL2, conda, MMDetection v3.0.0 環境構築"
emoji: "Ⓜ️"
type: "tech"
topics:
  - "wsl2"
  - "openmmlab"
  - "mmdetection"
published: true
---

## 環境

- Windows 11 Home (22H2)
- Ubuntu 22.04
- GPU: RTX 3090

## NVIDIAドライバーのインストール（Windows上）

未インストールの場合は、 https://www.nvidia.com/Download/index.aspx からダウンロードしてインストールする。

インストール済みの場合でも、使用予定のCUDAバージョンによっては更新が必要。
[CUDA Toolkit Release Notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)でMinimum Required Driver Versionを確認する。
CUDA 11で十分な環境であれば452.39以上に上げ、CUDA 12も使用しそうな環境（RTX 4090等）であれば527.41以上に上げることになるだろう。筆者の場合、インストール済みの512.77をそのまま使用した。

## WSL2のインストール（Windows上）

スタートボタンを右クリックし、ターミナル (管理者) を開く。
「管理者: Windows PowerShell」タブ上で、以下を実行する。

```powershell
wsl --install
```

参考：

- https://thinkit.co.jp/article/19737
- https://learn.microsoft.com/ja-jp/windows/wsl/setup/environment

筆者の場合エラーが出たため、[金子邦彦先生の記事](https://www.kkaneko.jp/tools/wsl/wsl2.html)を参考にインストールした。差分は以下。

- Hyper-Vのチェックボックスは無かった
- dism.exeの実行は飛ばした
- `wsl --update`を実行してから`wsl --install`を実行する必要があった

## apt（WSL2のUbuntu上）

update, upgradeの実行に加え、とりあえず必要そうなものをインストールする。

```bash
sudo apt update && sudo apt upgrade
sudo apt -y install build-essential zip unzip
```

## WSL2のネットワークが遅い場合

[WSL2のネットワークが遅いという問題](https://github.com/microsoft/WSL/issues/4901)が頻発している。
筆者の場合、以下の(1), (2)を実行し、(2)の実行後にまともなダウンロード速度になった。

### (1) nameserver修正（WSL2のUbuntu上）

参考： https://taka-say.hateblo.jp/entry/2022/03/06/171935

### (2) Large Send Offload無効化（Windows上）

スタートボタンを右クリックし、ターミナル (管理者) を開く。
「管理者: Windows PowerShell」タブ上で、以下を実行する。

```powershell
# コントロールパネル経由で"vEthernet (WSL)"が見えない場合もipconfigでは見える
ipconfig.exe /all
# ネットワークアダプター設定を確認
Get-NetAdapterAdvancedProperty -IncludeHidden -Name "vEthernet (WSL)"
# Large Send Offloadを無効化
Disable-NetAdapterLso -IncludeHidden -Name "vEthernet (WSL)"
```

## VSCodeのインストール（Windows上）

https://code.visualstudio.com/ からダウンロードしてインストールする。

## VSCodeのWSL拡張機能のインストール（Windows上）

拡張機能ボタン → "wsl"で検索 → [WSL by Microsoft](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)をインストールする。

## Minicondaのインストール（WSL2のUbuntu上）

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
rm Miniconda3-latest-Linux-x86_64.sh
```

詳細は https://www.macnica.co.jp/business/semiconductor/articles/qualcomm/142362/ を参照。

## Minicondaのチャンネル指定変更（WSL2のUbuntu上）

```bash
conda config --remove channels defaults
conda config --add channels conda-forge
```

詳細は https://inorio.hatenablog.com/entry/2022/10/14/190000 を参照。

## MMDetectionのインストール（WSL2のUbuntu上）

[MMDetectionのドキュメント](https://mmdetection.readthedocs.io/en/latest/get_started.html)に従ってインストールする。以下は、PyTorch 2.0.0, CUDA 11.7を使用する場合の例である（RTX 4090等ではCUDA 11.8以上が必要）。

```bash
conda create --name mmdet300 python=3.9 -y
conda activate mmdet300
conda install pytorch=2.0.0 torchvision=0.15.0 pytorch-cuda=11.7 -c pytorch -c nvidia

pip install -U openmim
mim install mmengine
mim install "mmcv>=2.0.0"

git clone https://github.com/open-mmlab/mmdetection.git
cd mmdetection
pip install -v -e .
```

## MMDetectionの動作確認：推論（WSL2のUbuntu上）

```bash
mim download mmdet --config rtmdet_tiny_8xb32-300e_coco --dest work_dirs/rtmdet/
python demo/image_demo.py \
    demo/demo.jpg \
    work_dirs/rtmdet/rtmdet_tiny_8xb32-300e_coco.py \
    --weights work_dirs/rtmdet/rtmdet_tiny_8xb32-300e_coco_20220902_112414-78e30dcc.pth \
    --out-dir work_dirs/rtmdet/
```

インストールに成功していれば、`work_dirs/rtmdet/vis/demo.jpg`に結果が出力される。

### エラー1つ目の修正

> ImportError: libGL.so.1: cannot open shared object file: No such file or directory

```bash
sudo apt -y install libgl1-mesa-glx
```

[MMSegmentationのdandelionさんのプルリク](https://github.com/open-mmlab/mmsegmentation/pull/1568)参照。[MMDetectionでの修正方法](https://github.com/open-mmlab/mmdetection/pull/3891)より軽いはず。

### エラー2つ目の修正

> Could not load library libcudnn_cnn_infer.so.8. Error: libcuda.so: cannot open shared object file: No such file or directory

`libcuda.so`は`/usr/lib/wsl/lib/`にいるので`$LD_LIBRARY_PATH`に追加する。なお、`$PATH`には元々入っていたため、`nvidia-smi`は普通に実行できた。

```bash:.bashrc
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/wsl/lib
```

参考： https://stackoverflow.com/questions/54249577/importerror-libcuda-so-1-cannot-open-shared-object-file

## MMDetectionの動作確認：テスト・訓練（WSL2のUbuntu上）

### COCOデータセットのダウンロード

[download_dataset.py](https://github.com/open-mmlab/mmdetection/blob/ecac3a77becc63f23d9f6980b2a36f86acd00a8a/tools/misc/download_dataset.py#L138-L148)を編集し、使用予定の無いファイルはコメントアウトしておく。

```python
coco2017=[
    'http://images.cocodataset.org/zips/train2017.zip',
    'http://images.cocodataset.org/zips/val2017.zip',
    # 'http://images.cocodataset.org/zips/test2017.zip',
    # 'http://images.cocodataset.org/zips/unlabeled2017.zip',
    'http://images.cocodataset.org/annotations/annotations_trainval2017.zip',  # noqa
    # 'http://images.cocodataset.org/annotations/stuff_annotations_trainval2017.zip',  # noqa
    # 'http://images.cocodataset.org/annotations/panoptic_annotations_trainval2017.zip',  # noqa
    # 'http://images.cocodataset.org/annotations/image_info_test2017.zip',  # noqa
    # 'http://images.cocodataset.org/annotations/image_info_unlabeled2017.zip',  # noqa
],
```

データセットはMMDetection以外のライブラリからもアクセスしたいため、MMDetection外のディレクトリ（例：`${HOME}/data/coco`）にダウンロードする。

```bash
python tools/misc/download_dataset.py --dataset-name coco2017 --save-dir ${HOME}/data/coco --unzip --delete
```

MMDetectionの既存configは`mmdetection/data/`下を見にいくよう書かれているため、ダウンロード先が見えるようシンボリックリンクを張る。

```bash
# データセットごとにシンボリックリンクを張る場合
mkdir data
ln -s ${HOME}/data/coco data/coco
# 面倒なのでdataディレクトリごとシンボリックリンクを張る場合
ln -s ${HOME}/data data
```

### テスト

```bash
CONFIG_FILE=configs/rtmdet/rtmdet_tiny_8xb32-300e_coco.py
CHECKPOINT_FILE=work_dirs/rtmdet/rtmdet_tiny_8xb32-300e_coco_20220902_112414-78e30dcc.pth
GPU_NUM=1
bash tools/dist_test.sh ${CONFIG_FILE} ${CHECKPOINT_FILE} ${GPU_NUM}
```

RTMDet-tinyの場合、[COCO APが41.1](https://github.com/open-mmlab/mmdetection/tree/main/configs/rtmdet)のため、`coco/bbox_mAP: 0.4110`と表示されればテスト精度再現成功。

### 訓練

```bash
CONFIG_FILE=configs/rtmdet/rtmdet_tiny_8xb32-300e_coco.py
GPU_NUM=1
bash tools/dist_train.sh ${CONFIG_FILE} ${GPU_NUM}
```

訓練が始まりlossが下がっていけばOK。
RTMDetは訓練エポック数300であり、`eta: 7 days`などと表示されるため、訓練精度再現確認には向かない（そもそも8 GPU用のconfigであるため、そのままでは精度再現しない）。

### 訓練精度再現確認

理想的には訓練精度の再現も確認する（MMDetectionでここまでやる人は稀だと思う）。
例えば、ATSSを8 GPUで訓練する。

```bash
CONFIG_FILE=configs/atss/atss_r50_fpn_1x_coco.py
GPU_NUM=8
bash tools/dist_train.sh ${CONFIG_FILE} ${GPU_NUM}
```

## 補足：WSL-Ubuntu用CUDA Toolkitのインストール（WSL2のUbuntu上）

[MMDetectionのドキュメント](https://mmdetection.readthedocs.io/en/latest/get_started.html)のNOTEにも書いてある通り、MMDetectionを使う上で必須ではないため、必要になった場合のみインストールすればよい。

基本的には[NVIDIAのドキュメント](https://docs.nvidia.com/cuda/wsl-user-guide/index.html#cuda-support-for-wsl-2)に従う。
PyTorchインストール時に指定したバージョン（今回の場合11.7）と同じバージョンをインストールするため、[CUDA Toolkit Archive](https://developer.nvidia.com/cuda-toolkit-archive)から辿り、[インストールコマンド](https://developer.nvidia.com/cuda-11-7-1-download-archive?target_os=Linux&target_arch=x86_64&Distribution=WSL-Ubuntu&target_version=2.0&target_type=deb_local)を得る。

```bash
sudo apt-key del 7fa2af80
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-wsl-ubuntu.pin
sudo mv cuda-wsl-ubuntu.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/11.7.1/local_installers/cuda-repo-wsl-ubuntu-11-7-local_11.7.1-1_amd64.deb
sudo dpkg -i cuda-repo-wsl-ubuntu-11-7-local_11.7.1-1_amd64.deb
sudo cp /var/cuda-repo-wsl-ubuntu-11-7-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda
rm cuda-repo-wsl-ubuntu-11-7-local_11.7.1-1_amd64.deb
```

```bash
find /usr/ -name 'nvcc'
# /usr/local/cuda-11.7/bin/nvcc
```

## 補足：メモリ不足の対処（Windows上）

WSL2側のメモリが不足したり、逆にWSL2側がメモリを食いすぎてWindows側のメモリが枯渇する場合は、[ユーザーフォルダを開き](https://forest.watch.impress.co.jp/docs/serial/yajiuma/1306709.html)、`.wslconfig`を作成し編集する。
筆者の場合、64GB中、WSL2側最大48GB、Windows側最小16GBとなるよう、以下のように設定した。

```:%USERPROFILE%\.wslconfig
[wsl2]
memory=48GB
swap=0
```

参考：

- https://zenn.dev/karaage0703/articles/d38e17bd6efbaa
- https://qiita.com/yoichiwo7/items/e3e13b6fe2f32c4c6120#%E7%B5%90%E8%AB%96-%E6%9A%AB%E5%AE%9A%E5%AF%BE%E5%87%A6%E3%81%A7%E3%83%A1%E3%83%A2%E3%83%AA%E3%82%B5%E3%82%A4%E3%82%BA%E3%82%92%E5%9B%BA%E5%AE%9A%E3%81%99%E3%82%8B

## 補足：ExplorerPatcher使用時の注意（Windows上）

筆者はExplorerPatcherを使用しており、以下の設定でWindows 11のUIを直している。

- Taskbar > Taskbar style: Windows 10
- File Explorer > Disable the Windows 11 context menu

タスクバースタイルを変更すると、スタートボタン右クリックで「Windows PowerShell (管理者)」が出てくるのだが、クリックしても[管理者として実行されない](https://github.com/valinet/ExplorerPatcher/discussions/1125)。
何という罠。もしやこの罠のせいで`wsl --install`でエラーが出たのでは......。「ターミナル (管理者)」の方を使いましょう。
