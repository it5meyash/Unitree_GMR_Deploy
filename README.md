# Unitree_GMR_Deploy

End-to-end pipeline for **sim-to-real motion retargeting and RL-based locomotion deployment** on Unitree humanoid robots (primarily the **G1**). The repository bridges two complementary workflows:

1. **GMR** — General Motion Retargeting: extract human motion from video/live capture → retarget to the robot's joint space in real time.
2. **unitree_rl_mjlab** — MuJoCo-based RL training and sim-to-real deployment: train locomotion/motion-tracking policies in MuJoCo simulation, then deploy to physical hardware.

---

## Repository Structure

```
Unitree_GMR_Deploy/
├── GMR/                    # General Motion Retargeting pipeline
│   └── ...                 # SMPL-X body model integration, retargeting scripts
│                           # Xsens live-streaming, offline video (GVHMR) input
└── unitree_rl_mjlab/       # MuJoCo RL training + sim-to-real deployment
    └── ...                 # Training configs, policy export, hardware controller
```

**Languages used:** Jupyter Notebook (61%) · Python (23%) · C++ (12%) · C (4%) · CMake (0.5%)

---

## Module 1 — GMR (General Motion Retargeting)

### Overview

GMR converts human body motion — captured from monocular video or a live motion capture suit — into robot joint commands, running in real time on CPU. It uses the **SMPL-X** parametric body model as an intermediate representation, then solves per-robot retargeting to produce executable joint trajectories.

### Input Sources

| Source | Tool / Format | Notes |
|--------|--------------|-------|
| Monocular video | [GVHMR](https://github.com/zju3dv/GVHMR) → SMPL-X | Offline; extract poses from YouTube clips or recorded footage |
| Live mocap suit | Xsens MVN Network Streamer | Real-time streaming; MuJoCo preview window |
| OptiTrack FBX export | `fbx_sdk` | Offline FBX motion files |

### Supported Robots

GMR targets a growing list of humanoids. Robots confirmed compatible include Unitree **G1**, **H1**, **H1-2**, Booster T1, Berkeley Humanoid Lite, PND Adam Lite, PAL Talos, and Tienkung.

### Key Files & Scripts

```
GMR/
├── scripts/
│   ├── xsens_live_streaming.py   # Live retargeting via Xsens MVN
│   ├── csv_to_npz.py             # Convert CSV motion captures to NPZ format
│   └── ...
├── assets/
│   └── body_models/smplx/        # SMPL-X model weights (NEUTRAL / FEMALE / MALE .pkl)
└── mjlab/motions/g1/             # Per-robot motion assets
```

### Quick Start — Offline Video Retargeting (GVHMR → GMR)

```bash
# Step 1: Extract SMPL-X poses from video using GVHMR
# (run in GVHMR environment)
python gvhmr_infer.py --video input.mp4 --output smplx_poses.pkl

# Step 2: Activate GMR environment
conda activate gmr

# Step 3: Run retargeting
python scripts/retarget.py \
    --input smplx_poses.pkl \
    --robot g1 \
    --output motions/g1/output.npz
```

### Quick Start — Live Streaming (Xsens)

```bash
conda activate gmr

# Ensure Xsens MVN Network Streamer is active, then:
python scripts/xsens_live_streaming.py
# A MuJoCo preview window opens showing the retargeted G1 in real time.
```

### Dependencies

- Python 3.8+ (Ubuntu 22.04 / 20.04 recommended)
- `smplx` Python package — after installing, change `ext` in `smplx/body_models.py` from `npz` to `pkl` if using `.pkl` model files
- SMPL-X body model weights (download from [SMPL-X project page](https://smpl-x.is.tue.mpg.de/))
- MuJoCo (for visualization)
- `fbx_sdk` (optional, for OptiTrack FBX input — requires a separate conda environment)

---

## Module 2 — unitree_rl_mjlab (RL Training + Sim-to-Real)

### Overview

`unitree_rl_mjlab` is a MuJoCo-based reinforcement learning framework adapted from the official [unitreerobotics/unitree_rl_mjlab](https://github.com/unitreerobotics/unitree_rl_mjlab). It uses **mjlab** (Isaac Lab API powered by MuJoCo-Warp) for lightweight, modular RL training and exports policies directly for hardware deployment.

### Workflow

```
Train (MuJoCo sim) ──► Play (verify policy) ──► Sim2Sim (unitree_mujoco) ──► Sim2Real (physical robot)
```

1. **Train** — Agent interacts with MuJoCo physics; policy optimized via reward maximization.
2. **Play** — Replay trained checkpoint to verify behavior before deployment.
3. **Sim2Sim** — Validate in `unitree_mujoco` before touching hardware (prevents unsafe behavior).
4. **Sim2Real** — Deploy `policy.onnx` to the physical Unitree G1 via Ethernet using `unitree_sdk2`.

### Training

**Velocity tracking (flat terrain):**
```bash
python scripts/train.py Unitree-G1-Flat --env.scene.num-envs=4096
```

**Multi-GPU training:**
```bash
python scripts/train.py Unitree-G1-Flat \
    --gpu-ids 0 1 \
    --env.scene.num-envs=4096
```

**Motion imitation (dance / retargeted sequences):**
```bash
# Step 1: Convert retargeted CSV to NPZ
python scripts/csv_to_npz.py \
    --input-file src/assets/motions/g1/dance1_subject2.csv \
    --output-name dance1_subject2.npz \
    --input-fps 30 \
    --output-fps 50

# Step 2: Train motion tracking policy
python scripts/train.py Unitree-G1-Tracking \
    --motion-file motions/g1/dance1_subject2.npz \
    --env.scene.num-envs=4096
```

Training exports `policy.onnx` and `policy.onnx.data` for hardware deployment.

### Policy Verification (Play)

```bash
python scripts/play.py Unitree-G1-Flat --checkpoint <path/to/checkpoint>
```

### Sim-to-Real Deployment

**Dependencies:**
```bash
# System libraries
sudo apt install -y libyaml-cpp-dev libboost-all-dev libeigen3-dev libspdlog-dev libfmt-dev

# unitree_sdk2
git clone https://github.com/unitreerobotics/unitree_sdk2.git
cd unitree_sdk2 && mkdir build && cd build
cmake .. -DBUILD_EXAMPLES=OFF
sudo make install
```

**Build the hardware controller:**
```bash
cd unitree_rl_mjlab/deploy/robots/g1_23dof   # or g1_29dof
mkdir build && cd build
cmake .. && make
```

**Deploy to robot:**
```bash
# 1. Start sim validation first (recommended)
cd unitree_mujoco/simulate/build
./unitree_mujoco

# 2. Launch policy controller (set network interface in config)
./g1_ctrl

# 3. On robot controller:
#    - Start robot in suspended state
#    - Wait for zero-torque mode
#    - Press L2 + R2 to begin policy execution
#    - Press L2 + Up to command stand-up
```

> **Important:** Always validate in `unitree_mujoco` sim before real hardware. Ensure the `network` field in config matches the Ethernet interface connected to the robot. Robot type (23DOF vs 29DOF) must match the compiled controller — mismatch causes an `Unmatched robot type` critical error.

---

## Real-World Application Context

This pipeline was used to deploy **dance motion retargeting** on a Unitree G1 humanoid:

- Source motion extracted from YouTube video using **GVHMR**
- Retargeted to G1 joint space using **GMR**
- Motion tracking policy trained in **MuJoCo** for ~30,000 iterations
- Policy validated in sim, then deployed to the physical **G1** robot

Demonstrated live at Siemens, EY, Deloitte, Intel, Tata Motors, and multiple IITs/NITs across India.

---

## Requirements Summary

| Component | Requirement |
|-----------|------------|
| OS | Ubuntu 22.04 / 20.04 |
| Python | 3.8+ |
| GPU | NVIDIA (required for RL training) |
| Simulator | MuJoCo |
| Robot SDK | unitree_sdk2 |
| Communication | Ethernet (robot ↔ deployment PC) |
| Motion capture (live) | Xsens MVN (optional) |

---

## References

- [GMR — General Motion Retargeting (ICRA 2026)](https://github.com/YanjieZe/GMR)
- [unitree_rl_mjlab](https://github.com/unitreerobotics/unitree_rl_mjlab)
- [mjlab — Isaac Lab API on MuJoCo](https://github.com/mujocolab/mjlab)
- [GVHMR — Global Video Human Mesh Recovery](https://github.com/zju3dv/GVHMR)
- [SMPL-X Body Model](https://smpl-x.is.tue.mpg.de/)
- [Unitree SDK2](https://github.com/unitreerobotics/unitree_sdk2)

---

## License

See individual submodule licenses. Hardware deployment code follows BSD 3-Clause (unitree_sdk2 terms). The project name and organization name may not be used for promotion per upstream terms.
