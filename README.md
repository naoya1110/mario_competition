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

4. 依存パッケージをインストール(**2段階に分かれている。必ずこの順番で実行すること**)

   ```powershell
   python -m pip install --upgrade pip
   pip install -r requirements1.txt
   pip install -r requirements2.txt
   ```

   2つに分けているのは pyglet のバージョンを上書きするため。`requirements1.txt` の `nes-py 9.0.1` は
   依存として `pyglet<=1.5.21` を要求するが、pyglet 1.5.21 は Python 3.13 の Windows 環境で
   ウィンドウ作成時にクラッシュする既知の不具合がある(ctypes/COM の非互換)。1.5.31 で修正されており
   nes-py の `ImageViewer` が使う pyglet 1.x のAPIも変わらないため、`requirements2.txt` で
   意図的に 1.5.31 を上書きインストールしている。

   そのため2つ目のコマンドで
   `nes-py 9.0.1 requires pyglet<=1.5.21, but you have pyglet 1.5.31` という警告が出るが、
   これは想定通り。エラーではないので無視してよい。

5. VSCodeで `notebooks/*.ipynb` を開き、カーネルとして `.venv` (Python 3.13) を選択する。

6. 動作確認として `notebooks/01_collect_data.ipynb` の最初のコードセルを実行し、ROMがロードでき
   `SIMPLE_MOVEMENT` が表示されることを確認する。

## ファイル構成と使い方

```
mario_test/
├── requirements1.txt         # ローカル(Python 3.13)用の依存関係
├── requirements2.txt         # 同上(pyglet を 1.5.31 に上書きするため後から入れる分)
├── notebooks/
│   ├── 01_collect_data.ipynb  # 人間プレイの記録 (ローカル実行)
│   ├── 02_review_data.ipynb   # 収集データの可視化・確認 (ローカル実行)
│   ├── 03_train_bc.ipynb      # 模倣学習でCNNを学習 (Colabにアップロードして実行)
│   └── 04_playagent.ipynb     # 学習済モデルで自動プレイ+提出用ファイル作成 (ローカル実行)
├── data/                      # 収集データ (gitignore対象)
│   ├── episodes/ep_YYYYMMDD_HHMMSS/  # 1エピソード = 1フォルダ (video.mp4 + actions.csv)
│   └── manifest.csv           # エピソード一覧 (score, x_pos_max, flag_get 等)
├── models/checkpoints/        # Colabからダウンロードした学習済 .pt (gitignore対象)
└── results/submissions/       # 04_playagent.ipynb が出力する提出物 (gitignore対象)
```

### 1. `notebooks/01_collect_data.ipynb` — データ収集

セルを上から順に実行する。最後のセルを実行するとゲーム画面のウィンドウが開き、キーボードでプレイできる。

**キー操作**

| キー | 動作 | 変更 |
|---|---|---|
| `d` | 右 | 可 |
| `a` | 左 | 可 |
| `o` | ジャンプ (A) | 可 |
| `p` | ダッシュ / ファイア (B) | 可 |
| Space | エピソード開始 (記録スタート) | 不可 |
| Esc | データ収集を終了(その時点までのデータも保存される) | 不可 |

ゲーム操作のキーはプレイループのセルにある `KEY_BINDINGS` で変更できる
(例: `"right": "l"` に書き換えれば右移動が `l` キーになる)。変更したらそのセルを実行し直すこと。
画面下部に表示される操作説明も `KEY_BINDINGS` から生成されるので表示は自動的に追従する。
Space と Esc は収集の制御用に固定で、変更できない。

`SIMPLE_MOVEMENT` は7種類の行動(何もしない / 右 / 右+A / 右+B / 右+A+B / A / 左)しか持たないため、
しゃがみ(下)や上入力に割り当てるキーはない。

**収集の流れ**

1. ウィンドウが開くと**待機状態**になり、画面に操作方法が表示される。この間ゲームは進まない。
2. **Space** を押すとそのエピソードが開始され、記録も同時に始まる。
3. 死亡・ゴールするとエピソードが自動保存され、画面中央にそのエピソードのサマリー
   (到達距離・コイン・スコア・消費時間・フレーム数と、0〜100点のSkillスコア)が表示される。
   **Space** で次のエピソードを開始できる(何度でも連続プレイ可能)。
4. **Esc** を押すとデータ収集全体が終了する。プレイ途中だった場合、その時点までのデータも
   `truncated=True` として保存される。

- 各エピソードの出来は0〜100点の**Skillスコア**で評価される(到達距離45% + クリア30% + 速さ15% +
  コイン10%)。式の詳細と重みの根拠は `02_review_data.ipynb` にある。
- エピソードID(`ep_YYYYMMDD_HHMMSS`)は記録開始時刻から作られるため一意になる。ノートブック/カーネルを
  再起動して複数回に分けてプレイしても、既存データを上書きすることはない。
- プレイ自体はNES本来の60fpsのまま行われるが、**保存するフレームは30fpsに間引かれる**(`TARGET_FPS` で変更可。
  データ量を抑えるための措置で、プレイの操作感には影響しない)。
- 各エピソードは `data/episodes/ep_YYYYMMDD_HHMMSS/video.mp4`(間引き後のフレームをmp4で保存)と
  同フォルダの `actions.csv`(フレームごとのアクション)のペアとして保存される。
- プレイ画面のウィンドウはデフォルトで2倍サイズ(`RENDER_SCALE` で変更可)で表示される。
- エピソードごとの要約(スコア・到達距離・クリア有無など)は `data/manifest.csv` に追記される。

**`TARGET_FPS`(間引き設定)を変更する場合は、`notebooks/04_playagent.ipynb` の同じ値も必ず変更すること。**
学習データのフレーム間隔と、デプロイ時にモデルがアクションを決定する頻度を一致させるため
(`04_playagent.ipynb` は `RECORD_EVERY_N` フレームごとに1回だけ新しいアクションを決め、その間は
同じアクションを保持するフレームスキップ方式になっている)。

### 2. `notebooks/02_review_data.ipynb` — 収集データのレビュー

学習に進む前に、収集したデータの中身を確認するための読み取り専用ノートブック。

- `data/manifest.csv` の一覧表示・要約統計
- **Skillスコア**(0〜100点)の算出と、その分布・エピソード順の推移プロット
  (プレイを重ねてうまくなっているか、学習データとして残す価値があるかの確認)
- 7種類のアクションの出現回数の棒グラフ(クラス不均衡の確認)
- 指定エピソードのフレームサンプル表示・アクション系列の時系列プロット

Skillスコアの定数と計算式は `01_collect_data.ipynb` と一致させてあるので、
片方を変更したらもう片方も必ず合わせること。

明らかにおかしいエピソード(スコアが極端に低い、記録ミスなど)が見つかった場合は、該当する
`data/episodes/ep_YYYYMMDD_HHMMSS/` フォルダを削除し、`data/manifest.csv` の対応行も手動で削除する。

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
5. 学習後にダウンロードされる `mario_bc_sN_YYYYMMDD_HHMMSS.pt` をローカルの `models/checkpoints/` に配置する
   (`sN` の `N` は後述の `STACK_SIZE`。複数の設定で学習したものを並べて置ける)

**エピソード冒頭の NOOP は自動的に除外される (`TRIM_LEADING_NOOP`)**

人間は開始後キーに手を伸ばすまでに間があるため、各エピソードの先頭には必ず NOOP が並ぶ。
この区間の画像は「開始位置に立ったままのマリオ」で、最初に `right` を押したフレームとも
画素単位でほぼ同一なので、そのまま学習させると同じ画像が矛盾したラベルで与えられ、しかも
NOOP が多数派になる。結果としてモデルは開始画面で NOOP を返し、NOOP では画面が変わらないため
次も同じ入力・同じ出力となって、**マリオが一歩も動かないまま固まる**。

`03` はこの冒頭 NOOP を除外してからデータセットを作る。捨てるのは全体の2%程度。
エピソード途中の NOOP は「静止しているのが正しい判断」の場合があるため残す。

**演習: `STACK_SIZE` を 1〜4 で変えて比較する**

`STACK_SIZE` はモデルが一度に見る連続フレーム数で、共有ブロックのセルで設定する。
これがこのノートブックの主要な調整項目。

| 値 | 入力 | 意味 |
|---|---|---|
| 1 | `[1, 84, 84]` | 1枚の静止画からアクションを推論する、普通の画像分類モデル |
| 2〜3 | `[N, 84, 84]` | 中間 |
| 4 | `[4, 84, 84]` | 既定値。Atari系の研究で標準的に使われる値 |

1枚の静止画からは**速度が読めない**(ジャンプの上昇中か落下中か、走っているか止まっているかが
原理的に判別できない)。フレームを重ねるとそれが観測可能になるが、入力が大きくなり
メモリ使用量も `STACK_SIZE` 倍になる。どこまで必要かを実際に確かめるのが演習の狙い。

設定するのは `03` だけでよい。`04_playagent.ipynb` はチェックポイントに記録された値を読み取って
モデルを組み直すので、手で合わせる必要はない。

**判断は検証精度 (`val_acc`) ではなく `04` の到達距離とクリア率で行うこと。**
人間のプレイには「どちらでもよい場面」が大量にある一方、着地の1フレームを外すと穴に落ちて即終了する。
val_acc は「人間の軌跡上での正解率」しか測っておらず、プレイの質とはあまり一致しない。

### 4. `notebooks/04_playagent.ipynb` — デプロイ / 提出用ファイルの作成

`models/checkpoints/` に配置した `.pt` のファイル名をノートブック内の `CHECKPOINT_PATH` に指定し、
**`STUDENT_ID` を自分の学籍番号に書き換えてから**実行する(提出ファイル名になる)。

**AIのプレイと提出物の作成は同じ実行で行われる。1回の実行 = 1エピソード = 提出物1式。**
動画はプレイ中に提出用フォルダへ直接書き込まれるので、別途コピーする手順はない。

- `STACK_SIZE` はチェックポイントから読み取るため、`03` で 1〜4 のどの値で学習した場合でも
  `CHECKPOINT_PATH` を差し替えるだけでよい(値が食い違ってテンソル形状エラーになることはない)
- 学習済モデルで自動プレイし、ゲーム画面のウィンドウが観察可能な速度で表示される
- 実行後、到達距離・クリア有無・Skillスコアと、提出するzipのパスが表示される

```
results/submissions/
├── <STUDENT_ID>_<timestamp>/
│   ├── ep00.mp4        # AIのプレイ動画
│   ├── frames.csv      # フレームごとのログ (mp4のフレームと1対1で対応)
│   └── episodes.csv    # エピソードごとの集計 + Skillスコア
└── submission_<STUDENT_ID>_<timestamp>.zip   ← これを提出する
```

`frames.csv` の列: `episode`, `frame_idx`, `action`, `action_name`, `x_pos`, `y_pos`,
`x_pos_max`, `coins`, `score`, `time_remaining`, `flag_get`, `status`, `life`

Skillスコアは `episodes.csv` に入っているが、`frames.csv` だけからでも再計算できる。
算出式と定数は `01` / `02` と一致させてあるので、人間のプレイと同じ基準で比較できる。

何度か挑戦して良いものを提出したい場合は、プレイのセルから実行し直せばよい。
実行のたびに別のフォルダとzipが作られるので、前の結果が上書きされることはない。
ただし1エピソードの結果は運の影響が大きいため、`STACK_SIZE` どうしを比較するような場合は
`EPISODES` を増やすか、複数回実行して平均を見ること。
