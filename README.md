# 🚁 Development of a Prototype Closed-Loop Edge-AI UAV System for Flood Response

## 📌 Overview

This repository presents a **prototype end-to-end UAV framework** for **real-time flood detection, segmentation, and GPS-guided navigation** using **Edge AI**. The emphasis is on demonstrating a closed-loop perception-to-action pipeline rather than a fully validated operational system.

The framework integrates:

- **PX4 Autopilot (SITL + embedded hardware)**
- **ROS 2 Humble + MAVROS**
- **Gazebo simulation**
- **Deep learning models (ResNet18, U-Net, DeepLabv3+)**
- **Jetson Nano edge deployment**
- **Real-Time GeoTask Dispatcher (MQTT → PX4 waypoints)**


<img width="1056" height="544" alt="descriptive_fm (1)" src="https://github.com/user-attachments/assets/0b2d72fc-d70f-4ce4-8221-c9e965005fe6" />



The UAV stack can:

- Detect flooded regions in (near) real time
- Segment flood boundaries from UAV imagery
- Convert image-space detections → GPS coordinates
- Dynamically update mission waypoints
- Execute autonomous navigation under controlled conditions with minimal human intervention

---

## 🎯 Key Features

### 🤖 AI-Based Flood Detection

- **ResNet18** → Image-level classification (flood / non-flood)
- **U-Net** → High-precision segmentation in simulation (Gazebo flood-world)
- **DeepLabv3+ (MobileNetV3)** → Segmentation on real-world UAV imagery

### ⚡ Real-Time Edge Inference (Prototype)

- TensorRT FP16 optimization
- GPU acceleration on **Jetson Nano**
- Multi-threaded ROS 2 pipeline
- Zero-copy memory and pre-allocated CUDA buffers

These optimizations reduce per-frame segmentation latency into the sub-20 ms range on Jetson Nano, supporting closed-loop control for short missions.

### 🛰️ Autonomous Navigation

- Dynamic waypoint injection via MQTT
- PX4 mission updates using MAVROS services
- Closed-loop **perception → geospatial mapping → waypoint execution** pipeline

### 🛡️ Fault-Tolerant Control

- MAVLink heartbeat monitoring
- Offboard mode failure detection
- Automatic fallback behaviors:
  - **LAND**
  - **RTL**
  - **Hover / loiter**

---

## 🏗️ System Architecture

```text
UAV Camera → ROS 2 Image Topic
        ↓
Deep Learning Model (Jetson Nano)
        ↓
Flood Detection / Segmentation
        ↓
Pixel → GPS Conversion
        ↓
MQTT (Real-Time GeoTask Dispatcher)
        ↓
PX4 Waypoint Injection (MAVROS)
        ↓
Autonomous UAV Navigation
```

---

## 📊 Dataset

Real-world flood imagery is sourced from:

🔗 https://github.com/sohailahmedkhan/Flood-Detection-from-Images-using-Deep-Learning

Used for:

- ResNet18 (classification)
- DeepLabv3+ (segmentation)

Simulation data (for U-Net) is generated in a Gazebo-based flood-world.

---

## 🌊 Flood UAV Simulation Setup

### 1. System Requirements

- Ubuntu 20.04 or 22.04 (or compatible Linux)
- ROS 2 Humble
- PX4 Autopilot
- Gazebo Classic
- Python 3.8+
- NVIDIA Jetson Nano (for edge deployment)

---

### 2. System Setup Instructions

#### 2.1 Update locale

```bash
sudo apt update
sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```

#### 2.2 Add ROS 2 package repository

```bash
sudo apt update && sudo apt install -y software-properties-common
sudo add-apt-repository universe
sudo apt update && sudo apt install curl -y
```

#### 2.3 Install ROS 2 Humble Desktop

```bash
sudo apt update && sudo apt install ros-humble-desktop
```

#### 2.4 Source ROS 2 environment

```bash
source /opt/ros/humble/setup.bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

#### 2.5 Install Gazebo simulator

```bash
sudo apt update && sudo apt install gazebo
```

#### 2.6 Install ROS 2 Gazebo plugins

```bash
sudo apt install ros-humble-gazebo-ros-pkgs
```

#### 2.7 Verify installation

```bash
gazebo
```

---

### 3. PX4 SITL Setup

Clone and build PX4 (example workspace path):

```bash
cd ~/TEEP/src
git clone --recursive https://github.com/PX4/PX4-Autopilot.git
cd PX4-Autopilot
make px4_sitl gazebo
```

Gazebo models and worlds:

```bash
cd ~/TEEP/src/PX4-Autopilot/Tools/simulation/gazebo-classic
./sitl_gazebo-classic
```

---

### 4. Setup ROS 2 Workspace

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
```

---

### 5. Install Python Dependencies

```bash
pip install torch torchvision opencv-python numpy
pip install paho-mqtt
```

---

## 🧭 Pixel-to-GPS Conversion Pipeline

1. Capture image from UAV camera
2. Run segmentation → binary mask
3. Divide mask into a **4×4 grid**
4. Identify the cell with the highest flood ratio
5. Compute the centroid of that cell
6. Project centroid to 3D using the pinhole camera model and UAV altitude
7. Convert local ENU coordinates to GPS using GeographicLib
8. Publish the resulting waypoint

---

## 🚀 Autonomous UAV Waypoint Injection and Mission Execution

To enable autonomous navigation in flood-affected areas, the framework implements a **real-time waypoint injection pipeline** that integrates:

- Onboard AI perception (Jetson Nano)
- Dynamic mission generation (Real-Time GeoTask Dispatcher)
- PX4 flight control via MAVROS

Traditional waypoint planning is manual and slow. Here, newly detected flood regions are turned into GPS waypoints and injected into the PX4 mission with sub-second end-to-end latency (in our tests).

### 🔄 Workflow

1. **Flood detection (onboard AI)**  
   - Images from the UAV camera are processed on the Jetson Nano.  
   - Deep learning models output segmentation masks and flood centroids.

2. **GPS Conversion**  
   - Flood centroids are converted into GPS coordinates.  
   - Published to ROS 2 topic:
     ```bash
     /next_gps_waypoint   # GeoPoseStamped
     ```

3. **Real-Time GeoTask Dispatcher (`mqtt_to_px4.py`)**  
   - Subscribes to:
     ```bash
     uav/flood_detection   # MQTT
     ```
   - Responsibilities:
     - Convert GPS → PX4-compatible waypoints  
     - Generate QGroundControl mission files  
     - Update PX4 mission dynamically via MAVROS

4. **PX4 Mission Update (MAVROS services)**  
   - Clear mission:
     ```bash
     WaypointClear
     ```
   - Upload mission:
     ```bash
     WaypointPush
     ```

5. **Autonomous Navigation**  
   - PX4 handles attitude, altitude, and velocity control while following the updated mission.

6. **Latency Monitoring**  
   - Timestamps are logged throughout the pipeline to verify real-time behavior.

---

## 🧪 Experiments

### A. Simulation (PX4 SITL + Gazebo)

The closed-loop perception-to-action pipeline is first validated in **PX4 SITL–Gazebo**, including:

- Autonomous takeoff and landing
- Offboard mode activation
- Segmentation-driven waypoint updates
- End-to-end latency measurements

### B. Preliminary Real-World Deployment

After SITL validation, the pipeline is ported to:

- **Pixhawk** flight controller  
- **NVIDIA Jetson Nano** companion computer  

ROS 2 nodes run on the Jetson Nano with minimal changes from the simulation setup. In hardware-in-the-loop:

- Camera → AI detection → GPS conversion  
- Real-time waypoint updates via the GeoTask Dispatcher  
- PX4 missions updated following a basic sequence:

```text
TAKEOFF → WAYPOINTS → RTL
```

These experiments demonstrate feasibility but do **not** constitute large-scale or long-duration field validation.

---

## 📌 Key Observations

- In baseline CPU-only execution, neural network inference dominates (>95% of total latency).
- TensorRT FP16 with GPU acceleration yields **sub-20 ms** segmentation inference on Jetson Nano.
- **DeepLabv3+** is preferred for real-world imagery due to better segmentation robustness.
- **U-Net** is preferred in simulation for stable, high-frequency operation.

<img width="1303" height="1125" alt="image" src="https://github.com/user-attachments/assets/3e60464b-1646-4341-89f0-94bd24baca13" />
Onboard flood detection outputs showing U-Net (top row) and DeepLabv3+ (bottom row) real-time inference logs with latency and FPS overlaid on live UAV camera imagery.

---

## 📸 Results (Prototype)

- Real-time flood segmentation in simulation and real imagery
- Autonomous waypoint navigation driven by perception
- Dynamic mission updates via MQTT → MAVROS
- Stable flight behavior during short test missions
- Minimal operator interaction during closed-loop runs

Demo video (prototype):  
https://drive.google.com/file/d/1uo6OgZUHK_WJVPmG6mCN60SWfuUq5IDi/view?usp=sharing

---

## 🔬 Research Contributions

- Prototype **closed-loop Edge-AI UAV framework** for flood monitoring
- Integration of **AI perception, geospatial mapping, and PX4 mission control**
- **Latency-optimized** edge inference on Jetson Nano
- Real-Time GeoTask Dispatcher for dynamic GPS waypoint injection
- Fault-tolerant control with autonomous fallback behaviors

---

## 👨‍💻 Authors

- Ayush Kumar  
- Huang Po Chun  
- Ayush Pratap  
- Pao-Ann Hsiung  

Department of Computer Science and Information Engineering,  
National Chung Cheng University, Taiwan
