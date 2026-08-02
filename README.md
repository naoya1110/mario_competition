# Mario environment for AI Robotics Lab

Windows 11 上でキーボードを使って Super Mario Bros (`gym-super-mario-bros` / `nes-py`) をプレイし、
人間のプレイデータを収集して模倣学習(Behavior Cloning)でCNNモデルを学習、学習済モデルをローカルに
デプロイして自動プレイ・スコア出力するプロジェクト。

- データ収集・データレビュー・デプロイ/スコア出力: ローカル (Windows, Python 3.13)
- モデル学習: Google Colab

## セットアップ (Windows / PowerShell)

1. Python 3.13 をインストール(既に 3.12 系が入っていても問題ない。両方共存できる)

   ```powershell
   winget install --id Python.Python.3.13 -e
   ```

2. インストール確認

   ```powershell
   py -0p
   ```

   `-V:3.13` が一覧に表示されればOK。

3. プロジェクト直下に Python 3.13 の仮想環境を作成し、有効化

   ```powershell
   py -3.13 -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

   実行ポリシーで `Activate.ps1` がブロックされる場合は、現在のセッションだけ許可する:

   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```

4. 依存パッケージをインストール

   ```powershell
   python -m pip install --upgrade pip
   pip install -r requirements.txt
   ```

   `nes-py 9.0.1 requires pyglet<=1.5.21, but you have pyglet 1.5.31` という警告が出るが、これは想定通り
   (`requirements.txt` 内にコメントで理由を記載: pyglet 1.5.21 は Python 3.13 の Windows 環境でウィンドウ
   作成時にクラッシュする既知の不具合があり、1.5.31 で修正されているため意図的に上書きしている)。エラーでは
   ないので無視してよい。

5. VSCodeで `notebooks/*.ipynb` を開き、カーネルとして `.venv` (Python 3.13) を選択する。

6. 動作確認として `notebooks/01_collect_data.ipynb` の最初のコードセルを実行し、ROMがロードでき
   `SIMPLE_MOVEMENT` が表示されることを確認する。

## ファイル構成と使い方

```
mario_test/
├── requirements.txt          # ローカル(Python 3.13)用の依存関係
├── notebooks/
│   ├── 01_collect_data.ipynb  # 人間プレイの記録 (ローカル実行)
│   ├── 02_review_data.ipynb   # 収集データの可視化・確認 (ローカル実行)
│   ├── 03_train_bc.ipynb      # 模倣学習でCNNを学習 (Colabにアップロードして実行)
│   └── 04_playagent.ipynb     # 学習済モデルで自動プレイ+スコア/動画出力 (ローカル実行)
├── data/                      # 収集データ (gitignore対象)
│   ├── episodes/ep_XXXX/      # 1エピソード = 1フォルダ。フレーム画像を連番で保存
│   └── manifest.csv           # エピソード一覧 (score, x_pos_max, flag_get 等)
├── models/checkpoints/        # Colabからダウンロードした学習済 .pt (gitignore対象)
└── results/
    ├── eval_runs/              # 04_playagent.ipynb の実行ログCSV (gitignore対象)
    └── videos/                 # 04_playagent.ipynb が出力するプレイ動画 (gitignore対象)
```

### 1. `notebooks/01_collect_data.ipynb` — データ収集

セルを上から順に実行する。最後のセルを実行するとゲーム画面のウィンドウが開き、キーボードでプレイできる。

**キー操作**

| キー | 動作 |
|---|---|
| `d` | 右 |
| `a` | 左 |
| `s` | しゃがむ (下) |
| `w` | 上 |
| `o` | ジャンプ (A) |
| `p` | ダッシュ / ファイア (B) |
| Enter | スタート |
| Space | セレクト |
| Esc | プレイ終了(その時点までのデータも保存される) |

- 死亡・ゴールすると自動的に次のエピソードが始まる(何度でも連続プレイ可能)。
- 各エピソードのフレームは `data/episodes/ep_XXXX/000000_a1_right.png` のように、
  連番+アクションindex+アクション名をファイル名に含めた画像として保存される。
- プレイ画面のウィンドウはデフォルトで2倍サイズ(`RENDER_SCALE` で変更可)で表示される。
- エピソードごとの要約(スコア・到達距離・クリア有無など)は `data/manifest.csv` に追記される。

### 2. `notebooks/02_review_data.ipynb` — 収集データのレビュー

学習に進む前に、収集したデータの中身を確認するための読み取り専用ノートブック。

- `data/manifest.csv` の一覧表示・要約統計
- 7種類のアクションの出現回数の棒グラフ(クラス不均衡の確認)
- 指定エピソードのフレームサンプル表示・アクション系列の時系列プロット

明らかにおかしいエピソードが見つかった場合は、該当する `data/episodes/ep_XXXX/` フォルダを削除し、
`data/manifest.csv` の対応行も手動で削除する。

### 3. `notebooks/03_train_bc.ipynb` — Colabでの学習

このノートブックは **Google Colab にアップロードして実行する**。

1. ローカルで `data` フォルダをzip化する

   ```powershell
   Compress-Archive -Path data -DestinationPath mario_data.zip
   ```

2. `mario_data.zip` を Google ドライブの適当なフォルダ(例: `MyDrive/mario_bc/`)にアップロードする
3. Colabで `notebooks/03_train_bc.ipynb` を開き、ランタイムのタイプを **GPU** に設定
4. Google Drive マウント・zip展開・学習・チェックポイント書き出しのセルを上から順に実行する
   - 初回はまず少数エピソード・`EPOCHS = 1` で試験実行し、エラーが出ないことを確認してから本学習に進むこと
5. 学習後にダウンロードされる `mario_bc_YYYYMMDD_HHMMSS.pt` をローカルの `models/checkpoints/` に配置する

### 4. `notebooks/04_playagent.ipynb` — デプロイ / スコア出力

`models/checkpoints/` に配置した `.pt` のファイル名をノートブック内の `CHECKPOINT_PATH` に指定して実行する。

- 学習済モデルで自動プレイし、ゲーム画面のウィンドウが観察可能な速度で表示される
- 各エピソードのスコア・到達距離・クリア有無を `results/eval_runs/<timestamp>.csv` に記録
- 各エピソードのプレイ動画を `results/videos/<timestamp>_epNN.mp4` として保存
- 実行後、平均スコア・平均到達距離・クリア率などの要約が表示される
