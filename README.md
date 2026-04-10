# **3D Gaussian Splatting with OpenSplat on AMtown02 UAV Imagery: Test Report**

## 📋 Table of Contents

1. Executive Summary  
2. Introduction  
3. Methodology  
4. Dataset Description  
5. Implementation Details  
6. Results and Analysis  
7. Challenges Encountered and Solutions  
8. Discussion  
9. Conclusion  
10. References  
11. Appendix  

---

## 📊 Executive Summary

This report documents the complete process of performing 3D Gaussian Splatting (3DGS) using OpenSplat on a subset of the **AMtown02 ROS bag**. UAV images were extracted from the bag file; due to hardware limitations, only the first **200 images** (native resolution 2448×2048) were used for COLMAP sparse reconstruction and OpenSplat training. Training was performed in **CPU mode**, with image downscaling factor `-d 2` (effective input 1224×1024), **2000 iterations**, and checkpoints saved every 500 steps.

**Key Results**

| Metric | Value |
|--------|-------|
| Training iterations | 2,000 |
| Images used (COLMAP) | 200 |
| Initial SfM points | 60,941 |
| Final Gaussian count | 165,253 |
| Output PLY size | ~14 MB (`splat.ply`) |
| Final loss | 0.0233 |
| Total training time (360→2000 steps) | ~10 minutes |

---

## 📖 Introduction

### Background  
3D Gaussian Splatting represents a scene with numerous learnable anisotropic Gaussians, enabling real-time novel view synthesis. OpenSplat is an efficient open‑source implementation of this method, suitable for resource‑constrained environments.

### Objectives  
- Extract compressed image topic `/left_camera/image/compressed` from the ROS bag file.  
- Perform sparse reconstruction with COLMAP to obtain camera poses and initial point cloud.  
- Train a 3D Gaussian scene using OpenSplat, export a PLY model, and record training loss.  
- Validate the entire pipeline in CPU‑only mode.

### Scope  
- Hardware: WSL2 Ubuntu 22.04, CPU only (AMD Ryzen 9 4900HS, 7.5 GiB available memory).  
- Data: First 200 consecutive frames from the bag file, native 2448×2048.  
- Training: 2000 iterations, image downscaling factor 2 (`-d 2`), checkpoint every 500 steps.

---

## 🔬 Methodology

### Rendering and Loss  
OpenSplat uses rasterization‑based splat rendering. The loss function is a weighted sum of L1 and SSIM (default SSIM weight 0.2). Adaptive density control (splitting/pruning) adjusts the number of Gaussians during training.

### Overall Pipeline  
**ROS bag → Extract compressed images → COLMAP sparse reconstruction → OpenSplat training → PLY model + logs**

---

## 📁 Dataset Description

| Property | Value |
|----------|-------|
| Source | `AMtown02.bag` (original file located at `C:\Users\齐\Downloads\`, deleted due to space) |
| Image topic | `/left_camera/image/compressed` |
| Frames used | 200 (first 200 consecutive frames) |
| Native resolution | 2448 × 2048 |
| Training resolution | 1224 × 1024 (`-d 2` downscale) |
| COLMAP output | `sparse/0/` (contains `cameras.txt`, `images.txt`, `points3D.txt`) |
| Initial 3D points | 60,941 |
| Extracted full image directory | `C:\Users\齐\Downloads\extracted_AMtown02` (kept, 7499 images) |

**Directory structure (inside WSL)**  
```
/root/colmap_test/
├── test_images/          # 200 PNG images
├── database.db           # COLMAP database
├── sparse/0/
│   ├── cameras.txt
│   ├── images.txt
│   └── points3D.txt
└── splat.ply             # final model
```

---

## ⚙️ Implementation Details

### System Environment

| Item | Value |
|------|-------|
| CPU | AMD Ryzen 9 4900HS with Radeon Graphics (8 cores, 16 threads) |
| Memory | 7.5 GiB (WSL2 allocation) |
| Framework | OpenSplat (C++) |
| SfM | COLMAP 3.8 |
| Compute device | CPU |
| OS | WSL2 Ubuntu 22.04 |

### Key Commands

#### 1. Extract images from bag (using Docker + `cnspy-rosbag2image`)
```bash
docker run -it --rm -v /mnt/c/Users/齐/Downloads:/data osrf/ros:noetic-desktop-full \
  bash -c "source /opt/ros/noetic/setup.bash && \
           apt update && apt install -y python3-pip && \
           pip install cnspy-rosbag2image && \
           cd /data && \
           rosbag2image -i AMtown02.bag -t /left_camera/image/compressed -o extracted_AMtown02"
```
> This extracted all 7499 compressed images into `extracted_AMtown02`.

#### 2. Take first 200 images for testing
```bash
mkdir test_images
ls extracted_AMtown02/*.png | head -200 | xargs -I {} cp {} test_images/
```

#### 3. COLMAP sparse reconstruction
```bash
colmap feature_extractor --database_path database.db --image_path test_images --ImageReader.single_camera 1
colmap exhaustive_matcher --database_path database.db
colmap mapper --database_path database.db --image_path test_images --output_path sparse
colmap model_converter --input_path sparse/0 --output_path sparse/0 --output_type TXT
```

#### 4. OpenSplat training (2000 steps, CPU, downscale 2)
```bash
/root/colmap_test/opensplat ./sparse/0 -n 2000 -d 2 --cpu
```
> The `opensplat` executable was copied from Docker container `f50311bcc4b3`:  
> `docker cp f50311bcc4b3:/OpenSplat/build/opensplat /home/box717/opensplat`

---

## 📈 Results and Analysis

### Training Loss Progression (excerpt from terminal)

| Step | Loss | Progress |
|------|------|----------|
| 360  | 0.0476 | 18% |
| 365  | 0.0323 | 18% |
| 1997 | 0.0314 | 99% |
| 1998 | 0.0281 | 99% |
| 1999 | 0.0242 | 99% |
| 2000 | **0.0233** | 100% |

**Loss statistics**  
- Early loss (Step 360): ~0.048  
- Final loss: 0.0233  
- Loss reduction: ~51%  

**Training time**  
From Step 360 to Step 2000 (1640 steps) took approximately **10 minutes**, averaging ~40 seconds per 100 steps. Total training time (Step 0 to 2000) is estimated at **15–20 minutes**.

**Output model**  
- File: `splat.ply`  
- Size: 14 MB  
- Gaussian count: **165,253** (from PLY header)

---

## ⚠️ Challenges Encountered and Solutions

During the experiment, several technical difficulties were encountered. The main issues and their resolutions are described below.

### Challenge 1: `offset out of range` error when extracting images with Python `rosbags` library
- **Symptom**: Using the `rosbags` library in WSL to read the compressed image topic repeatedly failed with `offset -654376896 out of range for ...-byte buffer`, unable to decode any frame.
- **Cause**: The `rosbags` library has a bug in handling internal indices of certain ROS1 bag files, especially large files or those with special compression formats.
- **Solution**: Abandoned `rosbags` and switched to **Docker container running ROS Noetic + `cnspy-rosbag2image`**. This tool uses native `rosbag` and `cv_bridge`, which proved stable and reliable. The command successfully extracted all 7499 images.

### Challenge 2: COLMAP reconstruction failed with "No good initial image pair found"
- **Symptom**: Running `colmap mapper` printed "No good initial image pair found" and reconstruction would not start.
- **Cause**: Insufficient overlap or poor feature matching between images prevented COLMAP from finding a suitable initial image pair.
- **Solution**: Forced **sequential matching** instead of exhaustive matching, and relaxed initialization constraints. Executed:
  ```bash
  colmap sequential_matcher --database_path database.db --SequentialMatching.loop_detection 0
  ```
  Then re‑ran `colmap mapper`; reconstruction succeeded.

### Challenge 3: OpenSplat process was `Killed` during training
- **Symptom**: Running `opensplat` resulted in the terminal output "Killed" and the process terminated abnormally.
- **Cause**: In CPU mode, 200 images at native 2448×2048 resolution consumed more memory than the WSL2 default limit (~7.5 GiB), triggering the OOM killer.
- **Solution**: Added downscaling parameter `-d 2` to the OpenSplat command, reducing image size to 1224×1024 and cutting memory usage by roughly 4×. Also limited training to 2000 iterations. After these adjustments, training completed successfully and produced the final model.

---

## 💭 Discussion

### Strengths  
- Successfully demonstrated a complete pipeline from ROS bag to 3D Gaussian model using only CPU.  
- OpenSplat training on 200 images for 2000 iterations took ~15–20 minutes, which is acceptable for testing.  
- All encountered issues were resolved with clear workarounds, making the pipeline reproducible.

### Limitations  
- **Insufficient images**: 200 frames do not fully cover the scene, resulting in holes in the reconstruction.  
- **Resolution loss**: Downscaling (`-d 2`) discards high‑frequency details.  
- **No GPU acceleration**: Using a GPU would allow scaling to 3000+ iterations and processing all 7499 images.  
- **Missing loss log**: A full log was not saved, preventing generation of professional loss curves.

### Future Improvements  
- Train with full resolution (`-d 1`) and more images (e.g., 500).  
- Use CUDA‑enabled versions of COLMAP and OpenSplat to drastically reduce processing time.  
- Save training logs and utilize `plot_amtown_training.py` to generate standardized plots.

---

## 🎯 Conclusion

This test successfully generated a 3D Gaussian splatting model using OpenSplat on a CPU with 200 UAV images. The training loss decreased from ~0.048 to 0.023, indicating good convergence. The resulting PLY model (14 MB, 165,253 Gaussians) can be viewed online, validating the correctness of the pipeline. Although full dataset reconstruction was not possible due to resource constraints, this work provides a solid foundation for larger‑scale scene reconstruction.

---

## 📚 References

1. Kerbl, B., et al. "3D Gaussian Splatting for Real-Time Radiance Field Rendering." SIGGRAPH 2023.  
2. Schönberger, J. L., & Frahm, J. M. "Structure-from-Motion Revisited." CVPR 2016.  
3. OpenSplat GitHub: https://github.com/pierotofy/OpenSplat  

---

## 📎 Appendix

### A. Simplified training command
```bash
./opensplat /path/to/sparse/0 -n 2000 -d 2 --cpu
```

### B. PLY header (excerpt)
```
ply
format binary_little_endian 1.0
comment Generated by opensplat at iteration 2000
element vertex 165253
property float x
property float y
property float z
...
end_header
```

### C. Key paths
- Original bag: `C:\Users\齐\Downloads\AMtown02.bag` (deleted)  
- Extracted full images: `C:\Users\齐\Downloads\extracted_AMtown02` (kept, 7499 images)  
- Test images: `C:\Users\齐\Downloads\test_images` (kept)  
- COLMAP workspace: `/mnt/c/Users/齐/Downloads/colmap_test`  
- Final model: `C:\Users\齐\Desktop\splat.ply`

---
