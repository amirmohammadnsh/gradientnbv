<p align="center">
  <h1 align="center">Gradient-NBV: Gradient-based Local Next-best-view Planning for Improved Perception of Targeted Plant Nodes</h1>
  <p align="center">
    <strong>Akshay K. Burusa</strong>
    ·
    <strong>Eldert J. van Henten</strong>
    ·
    <strong>Gert Kootstra</strong>
  </p>
</p>

<h2 align="center">
  Paper: 
  <a href="https://ieeexplore.ieee.org/abstract/document/10610397" target="_blank">IEEE</a> | 
  <a href="https://arxiv.org/pdf/2311.16759" target="_blank">ArXiv</a>
</h2>

https://github.com/akshaykburusa/gradientnbv/assets/127020264/dfa1f2a9-f07c-4af0-84ef-7ea20a7cb61b

## About

This repository provides a gradient-based next-best-view (NBV) planner for robotic arm manipulation with an ABB IRB 1200 arm and Intel RealSense L515 camera in Gazebo simulation. It is a re-implementation of the original [gradientnbv](https://github.com/akshaykburusa/gradientnbv) repository for **Ubuntu 22.04** using **ROS Noetic through RoboStack** (conda-based), with compatibility fixes for modern C++ and NumPy versions.

https://github.com/user-attachments/assets/c71a8c2e-26a1-47da-8565-ed78fada5e1c

## Prerequisites

- Ubuntu 22.04
- [Miniconda](https://docs.anaconda.com/miniconda/) (or Anaconda)
- NVIDIA GPU with at least 8 GB VRAM

## Installation

### 1. Create the conda environment

```bash
conda create -n grad_env -c conda-forge -c robostack-noetic ros-noetic-desktop
conda activate grad_env
conda config --env --add channels robostack-noetic
conda config --env --remove channels defaults
```

### 2. Install build tools and dependencies

```bash
conda install -c conda-forge ros-dev-tools compilers cmake=3.26 pkg-config make ninja catkin_tools

conda install -c conda-forge -c robostack-noetic \
  ros-noetic-gazebo-ros-pkgs ros-noetic-ros-control ros-noetic-ros-controllers \
  ros-noetic-moveit ros-noetic-moveit-commander ros-noetic-moveit-setup-assistant \
  ros-noetic-moveit-visual-tools nlopt

conda install -c conda-forge \
  libtiff libffi pillow opencv pyyaml rospkg scipy pytransform3d \
  open3d empy defusedxml pytorch torchvision torchaudio
```

### 3. Create workspace and clone the repository

```bash
mkdir -p ~/grad_ws/src
cd ~/grad_ws/src

git clone https://github.com/akshaykburusa/gradientnbv.git
mv gradientnbv/src/* .
rm -rf gradientnbv

# TRAC-IK kinematics solver (not available in RoboStack's pre-compiled channels)
git clone -b master https://bitbucket.org/traclabs/trac_ik.git
```

### 4. Apply C++14 patch for ABB industrial drivers

The ABB robot drivers in this repo use legacy C++11 flags that conflict with Conda's modern Boost libraries:

```bash
cd ~/grad_ws
find src -type f \( -name "CMakeLists.txt" -o -name "*.cmake" \) -exec sed -i 's/c++11/c++14/g' {} +
find src -type f \( -name "CMakeLists.txt" -o -name "*.cmake" \) -exec sed -i 's/c++0x/c++14/g' {} +
find src -type f \( -name "CMakeLists.txt" -o -name "*.cmake" \) -exec sed -i 's/CXX_STANDARD 11/CXX_STANDARD 14/g' {} +
```

### 5. Apply MoveIt controller fix

The original `ros_controllers.yaml` has an empty controller name that prevents MoveIt from connecting to the Gazebo arm controller:

```bash
cd ~/grad_ws
sed -i 's/name: ""/name: arm_controller/' \
  src/robot/abb_l515_moveit_config/config/ros_controllers.yaml
sed -i 's/action_ns: joint_trajectory_action/action_ns: follow_joint_trajectory/' \
  src/robot/abb_l515_moveit_config/config/ros_controllers.yaml
```

### 6. Apply NumPy compatibility patches to ros_numpy

The shipped `ros_numpy` uses APIs removed in NumPy ≥1.24:

```bash
cd ~/grad_ws
sed -i 's/np\.fromstring(msg\.data/np.frombuffer(msg.data/' \
  src/common/ros_numpy/src/ros_numpy/image.py
sed -i 's/\.tostring()/.tobytes()/g' \
  src/common/ros_numpy/src/ros_numpy/image.py
sed -i 's/\.tostring()/.tobytes()/g' \
  src/common/ros_numpy/src/ros_numpy/point_cloud2.py
```

### 7. Fix arm control service handler

The ROS service callback in `arm_control.cpp` returns `false` on planning failure, which Python interprets as a transport error instead of a response with `success=False`:

```bash
cd ~/grad_ws
sed -i 's/return res.success;/return true;/' \
  src/robot/abb_control/src/arm_control.cpp
```

### 8. Build

```bash
cd ~/grad_ws
catkin build --cmake-args \
  -DCMAKE_CXX_STANDARD=14 \
  -DCMAKE_CXX_FLAGS="-std=c++14 -I$CONDA_PREFIX/include/eigen3" \
  -DCMAKE_BUILD_TYPE=Release
```

### 9. Source the workspace

```bash
source devel/setup.bash
```

### 10. Patch NumPy compatibility in installed cv_bridge

The Conda environment's `cv_bridge` package also calls the removed `.tostring()` API:

```bash
sed -i 's/\.tostring()/.tobytes()/g' \
  $CONDA_PREFIX/lib/python3.12/site-packages/cv_bridge/core.py
```

## Running the experiments

Open two terminals.

**Terminal 1** — Launch the simulation with the ABB arm and L515 camera in Gazebo:

```bash
conda activate grad_env
cd ~/grad_ws
source devel/setup.bash
roslaunch abb_l515_bringup abb_l515_bringup.launch
```

**Terminal 2** — Start the gradient-based NBV planner (wait for Gazebo to fully load first):

```bash
conda activate grad_env
cd ~/grad_ws
source devel/setup.bash
roslaunch viewpoint_planning viewpoint_planning.launch
```

## Citation
```bibtex
@inproceedings{burusa2024gradient,
  title={Gradient-based local next-best-view planning for improved perception of targeted plant nodes},
  author={Burusa, Akshay K and van Henten, Eldert J and Kootstra, Gert},
  booktitle={2024 IEEE International Conference on Robotics and Automation (ICRA)},
  pages={15854--15860},
  year={2024},
  organization={IEEE}
}
```
