# Mario environment for AI Robotics Lab

## Overview

In this competition, each team develops an AI model that plays Super Mario Bros (Stage 1-1).
You collect your own gameplay data and train a CNN on it with imitation learning (Behavior Cloning).
The trained model is then deployed to play the game automatically, and teams compete on its score.

- Data collection, data review, and deployment/scoring: local PC (Windows / Ubuntu, Python 3.13)
- Model training: Google Colab

## Scoring

Each run of the AI (one episode) is evaluated with a **Skill score** from 0 to 100.
When you run `04_playagent.ipynb`, the score is recorded in the `skill` column of `episodes.csv` in your submission.

```
progress = min(x_pos_max / 3161, 1)                         # progress rate (3161 = x position of the goal)
clear    = 1 if flag_get else 0                             # whether Mario cleared the stage
t_used   = 400 - time_remaining                             # in-game time used
speed    = clear × clip((260 - t_used) / (260 - 110), 0, 1)  # speed
points   = clear × min(score / 5000, 1)                     # in-game score

Skill = 100 × (0.40·progress + 0.10·clear + 0.20·speed + 0.30·points)
```

| Component | Max points | Condition for full points |
|---|---|---|
| Progress | 40 | Reach the goal |
| Clear | 10 | Touch the flagpole |
| Speed | 20 | Clear within 110 in-game time (0 points at 260 or more) |
| Points | 30 | Clear with an in-game score of 5000 or more |

- **A plain clear is worth 50 points** (progress 40 + clear 10). The remaining 50 points depend on speed and in-game score.
- **Speed and points are only awarded for cleared episodes.** An episode that does not clear is scored on progress alone, up to 40 points.
- The in-game score increases with coins (200 each), defeated enemies (100 or more per stomp, increasing for consecutive stomps),
  broken blocks (50), and power-ups such as mushrooms, flowers, and stars (1000).
  The episode ends as soon as Mario touches the flagpole, so the flagpole bonus and the time bonus are not included.
- Time is measured with the in-game timer at the top right of the screen, which counts down from 400.

**Examples**

| Result | Skill |
|---|---|
| Died just before the goal (98% progress) | 39.2 |
| Cleared using 300 in-game time, in-game score 0 | 50.0 |
| Cleared using 112 in-game time, in-game score 900 | 75.1 |
| Cleared using 100 in-game time, in-game score 6000 | 100.0 |


## Setup (Windows / PowerShell)

1. Install Python 3.13. (An existing Python 3.12 installation is fine; both versions can coexist.)

   ```powershell
   winget install --id Python.Python.3.13 -e
   ```

2. Verify the installation.

   ```powershell
   py -0p
   ```

   It is installed correctly if `-V:3.13` appears in the list.

3. Create a Python 3.13 virtual environment in the project root and activate it.

   ```powershell
   py -3.13 -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

   If the execution policy blocks `Activate.ps1`, allow scripts for the current session only:

   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```

4. Install the dependencies. **This is done in two steps. Always run them in this order.**

   ```powershell
   python -m pip install --upgrade pip
   pip install -r requirements1.txt
   pip install -r requirements2.txt
   ```

   The second step overrides the pyglet version. `nes-py 9.0.1` in `requirements1.txt` requires
   `pyglet<=1.5.21`, but pyglet 1.5.21 has a known bug that crashes when it creates a window
   on Windows with Python 3.13. Version 1.5.31 fixes this without changing the pyglet 1.x API that
   nes-py uses, so `requirements2.txt` intentionally installs 1.5.31 over it.

   As a result, the second command prints the warning
   `nes-py 9.0.1 requires pyglet<=1.5.21, but you have pyglet 1.5.31`.
   This is expected and is not an error, so you can ignore it.

5. Open `notebooks/*.ipynb` in VS Code and select `.venv` (Python 3.13) as the kernel.

6. To check the setup, run the first code cell of `notebooks/01_collect_data.ipynb`.
   The setup works if the ROM loads and `SIMPLE_MOVEMENT` is printed.

## Setup (Ubuntu / bash)

These steps have been tested on Ubuntu 24.04 LTS (x86_64). `nes-py 9.0.1` requires **Python 3.13 or later**
(`requires_python >=3.13` on PyPI), so dependency resolution fails with Ubuntu 24.04's default Python 3.12.

1. Install Python 3.13. (The system Python 3.12 is kept; both versions can coexist.)

   ```bash
   sudo add-apt-repository ppa:deadsnakes/ppa
   sudo apt update
   sudo apt install python3.13 python3.13-venv
   ```

2. Verify the installation.

   ```bash
   python3.13 -V
   ```

   It is installed correctly if `Python 3.13.x` is shown.

3. Install the libraries needed to display the game window.

   ```bash
   sudo apt install libgl1 libglu1-mesa libfontconfig1 libfreetype6 libxext6 libxrender1
   ```

4. Create a Python 3.13 virtual environment in the project root and activate it.

   ```bash
   python3.13 -m venv .venv
   source .venv/bin/activate
   ```

5. Install the dependencies. **This is done in two steps. Always run them in this order.**

   ```bash
   python -m pip install --upgrade pip
   pip install -r requirements1.txt
   pip install -r requirements2.txt
   ```

   The reason is the same as in Windows step 4. The pyglet 1.5.21 crash only happens on Windows,
   but Ubuntu uses the same steps so that everyone has an identical environment.
   The warning `nes-py 9.0.1 requires pyglet<=1.5.21, but you have pyglet 1.5.31` from the
   second command is also expected and can be ignored.

6. Open `notebooks/*.ipynb` in VS Code and select `.venv` (Python 3.13) as the kernel.

7. To check the setup, run the first code cell of `notebooks/01_collect_data.ipynb`.
   The setup works if the ROM loads and `SIMPLE_MOVEMENT` is printed.


## Project structure and workflow

```
mario_competition/
├── requirements1.txt          # local dependencies (Python 3.13)
├── requirements2.txt          # installed afterwards to override pyglet with 1.5.31
├── notebooks/
│   ├── 01_collect_data.ipynb  # record human gameplay (local)
│   ├── 02_review_data.ipynb   # visualize and check the collected data (local)
│   ├── 03_train_bc.ipynb      # train the CNN with imitation learning (upload to Colab)
│   ├── 04_playagent.ipynb     # run the trained model and create the submission files (local)
│   └── competition_config.py  # competition settings shared by 01, 02, and 04 (DO NOT EDIT)
├── data/                      # collected data (gitignored)
│   ├── episodes/ep_YYYYMMDD_HHMMSS/  # one folder per episode (video.mp4 + actions.csv)
│   └── manifest.csv           # episode list (score, x_pos_max, flag_get, etc.)
├── models/                    # put trained .pt files downloaded from Colab here (contents gitignored)
└── results/                   # submission files created by 04_playagent.ipynb (gitignored)
```

### Workflow

For detailed instructions, see the description at the top of each notebook.

| Step | Notebook | Runs on | What to do | Output |
|---|---|---|---|---|
| 1 | `01_collect_data.ipynb` | Local | Play Mario with the keyboard and record your gameplay | `data/` |
| 2 | `02_review_data.ipynb` | Local | Check the collected data and Skill scores, and delete unusable episodes | None |
| 3 | `03_train_bc.ipynb` | Colab | Upload `data/` to Google Drive as a zip file and train the CNN | `.pt` checkpoint (download it to `models/`) |
| 4 | `04_playagent.ipynb` | Local | Set `CHECKPOINT_PATH` and `TEAM_ID`, then run it to let the AI play | `results/` |

**The file to submit is `results/submission_<TEAM_ID>_<timestamp>.zip`.** Each run of 04 plays one episode
and creates a complete set of submission files (gameplay video, logs, Skill score, and the checkpoint that played).

**The same checkpoint always plays the same episode.** The model always picks its highest-scoring action, and
the game starts from the same state every time, so running 04 again with the same `.pt` gives exactly the same
result. To improve your score, improve the model (data in 01, training in 03). Because the play is reproducible,
the organizers can replay the checkpoint in your zip to check the result.

**Do not edit `notebooks/competition_config.py`.** It contains the rules shared by all teams: the frame rate
for recording and inference (`TARGET_FPS`) and the constants and formula of the Skill score.
Notebooks 01, 02, and 04 import their settings from this file.

Hyperparameters that you may change to improve your model, such as `STACK_SIZE`, are in the notebooks.
`STACK_SIZE` only needs to be set in 03; notebook 04 reads it from the checkpoint.
