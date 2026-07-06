# HoloHub pubsub — Build and Run Guide

## Overview

This guide describes how to build and run the `pubsub` (Holoscan ↔ ROS 2 Publisher/Subscriber) application from the `holohub` repository on **NVIDIA AGX Thor** with **JetPack 7.0** and **Holoscan SDK 3.9.0**.

### Why `pubsub` and not `holoscan_ros2`?

`holoscan_ros2` is the parent directory — it has no run configuration of its own. The runnable sub-applications are:
- `pubsub` — string message pub/sub (C++ and Python)
- `vb1940` — VB1940 Eagle camera pipeline (C++ only)

### Why a separate Dockerfile?

The generic holohub container (`holohub:ngc-v3.9.0-cuda13`) does **not** include ROS 2. The `pubsub` application has its own Dockerfile at `applications/holoscan_ros2/Dockerfile` which installs **ROS 2 Jazzy** and configures `CMAKE_PREFIX_PATH` to include both `/opt/ros/jazzy` and `/opt/nvidia/holoscan`.

---

## Prerequisites

| Requirement | Version |
|---|---|
| Host Device | NVIDIA AGX Thor |
| JetPack | 7.0 |
| Holoscan SDK | 3.9.0 |
| Docker | With NVIDIA Container Toolkit |
| holohub branch | `adcam_ros2` (tag `holoscan-sdk-3.9.0`) |

---

## Step 1 — Exit any existing container

If you are already inside a holohub container, exit it first:

```sh
exit
```

---

## Step 2 — Navigate to the holohub directory

```sh
cd /home/jetsonthor/siva/SPadmana95/holohub
```

---

## Step 3 — Build the container with ROS 2

Use the `pubsub` project name so the CLI picks up `applications/holoscan_ros2/Dockerfile`, which installs ROS 2 Jazzy. Override the base image to match your SDK version:

```sh
./holohub build-container pubsub --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13
```

**What this Dockerfile does:**
- Installs `ros-jazzy-desktop` and `ros-jazzy-ros-base`
- Sets `CMAKE_PREFIX_PATH=/opt/ros/jazzy:/opt/nvidia/holoscan`
- Sets entrypoint to `source /opt/ros/jazzy/setup.bash`

Verify the image was created:

```sh
docker images | grep pubsub
```

---

## Step 4 — Launch the container

```sh
./holohub run-container pubsub --language cpp
```

---

## Step 5 — Verify ROS 2 is available (inside container)

```sh
source /opt/ros/jazzy/setup.bash
ros2 --version
```

Expected output:
```
ros2 cli v0.18.x (or later)
```

---

## Step 6 — Build the pubsub application (inside container)

```sh
./holohub build pubsub --language cpp
```

A successful build produces:
- `holoscan_ros2_simple_publisher`
- `holoscan_ros2_simple_subscriber`

under `./build/pubsub/`.

---

## Step 7 — Run publisher and subscriber

### Terminal 1 — Publisher

Inside the container, run the publisher:

```sh
./holohub run pubsub publisher --language cpp
```

Expected output:
```
Publishing: 'Hello, world! 0'
Publishing: 'Hello, world! 1'
Publishing: 'Hello, world! 2'
...
```

### Terminal 2 — Subscriber

From the **host**, open a second shell into the running container:

```sh
# Find the container ID
docker ps | grep pubsub

# Attach a new shell
docker exec -it <container_id> bash
source /opt/ros/jazzy/setup.bash
```

Then run the subscriber:

```sh
./holohub run pubsub subscriber --language cpp
```

Expected output:
```
I heard: 'Hello, world! 0'
I heard: 'Hello, world! 1'
I heard: 'Hello, world! 2'
...
```

---

## Root Cause Summary

| Problem | Cause | Fix |
|---|---|---|
| `rclcpp` not found during CMake | Generic holohub container has no ROS 2 installed | Use `pubsub` project name to trigger `applications/holoscan_ros2/Dockerfile` |
| Wrong base image in Dockerfile | App Dockerfile hardcodes `holoscan:v3.3.0-dgpu` | Override with `--base-img holoscan:v3.9.0-cuda13` |

---

## Quick Reference — All Commands

```sh
# On host
cd /home/jetsonthor/siva/SPadmana95/holohub

./holohub build-container pubsub --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13

./holohub run-container pubsub --language cpp

# Inside container
source /opt/ros/jazzy/setup.bash
./holohub build pubsub --language cpp

# Terminal 1 (inside container)
./holohub run pubsub publisher --language cpp

# Terminal 2 (docker exec from host)
docker exec -it <container_id> bash
source /opt/ros/jazzy/setup.bash
./holohub run pubsub subscriber --language cpp
```

---

---

# VB1940 Eagle Camera — Build and Run Guide

## Overview

The `vb1940` application demonstrates a full GPU-accelerated camera pipeline using the **VB1940 (Eagle) camera** with Holoscan publishing processed images to a ROS 2 topic (`vb1940/image`) for visualization.

> **Important:** This application requires:
> - Access to NVIDIA's **internal hololink repository** (SSH key required)
> - **VB1940 (Eagle) camera hardware** connected to the host
> - Network reachability to Hololink board at `192.168.0.2`

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Host Device | NVIDIA AGX Thor |
| JetPack | 7.0 |
| Holoscan SDK | 3.9.0 |
| Camera Hardware | VB1940 (Eagle) connected via 10GigE |
| Hololink Board IP | `192.168.0.2` (default) |
| SSH key | Access to NVIDIA internal hololink repo |
| holohub branch | `adcam_ros2` (tag `holoscan-sdk-3.9.0`) |

---

## Network Setup

Before building or running, ensure the Hololink board is reachable:

```sh
ping 192.168.0.2
```

Verify IBV (InfiniBand Verbs) device is available:

```sh
ibv_devinfo
```

---

## Step 1 — Exit any existing container

```sh
exit
```

---

## Step 2 — Navigate to holohub directory

```sh
cd /home/jetsonthor/siva/SPadmana95/holohub
```

---

## Step 3 — Build the container (with ROS 2 + Vulkan + hololink)

The `vb1940` Dockerfile (`applications/holoscan_ros2/vb1940/Dockerfile`) installs:
- ROS 2 Jazzy (`ros-jazzy-desktop`, `ros-jazzy-ros-base`)
- Vulkan support (`vulkan-tools`, `vulkan-validationlayers`, `libvulkan1`) — required for HoloViz
- git (for cloning hololink repo during build)

```sh
./holohub build-container vb1940 --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13
```

Verify the image:

```sh
docker images | grep vb1940
```

---

## Step 4 — Launch the container

```sh
./holohub run-container vb1940 --language cpp
```

---

## Step 5 — Build the vb1940 application (inside container)

The `--build-args="--ssh default"` flag forwards your SSH key into the Docker build for accessing the hololink repository:

```sh
./holohub build vb1940 --language cpp --build-args="--ssh default"
```

A successful build produces:
- `holoscan_ros2_vb1940_publisher`
- `holoscan_ros2_vb1940_subscriber`

under `./build/vb1940/`.

---

## Step 6 — Run the publisher (Terminal 1)

The publisher captures frames from the VB1940 camera, runs them through the GPU pipeline (CSI→Bayer→Demosaic→16-bit→8-bit CUDA kernel), and publishes to ROS 2 topic `vb1940/image`:

```sh
# Default — publisher mode
./holohub run vb1940 publisher --language cpp

# With options:
./holohub run vb1940 publisher --language cpp \
  -- --hololink 192.168.0.2 \
     --camera-mode 0 \
     --frame-limit 100
```

**Publisher options:**

| Option | Default | Description |
|---|---|---|
| `--hololink` | `192.168.0.2` | IP address of the Hololink board |
| `--camera-mode` | `0` | Camera resolution/FPS mode |
| `--frame-limit` | unlimited | Stop after N frames |
| `--ibv-name` | auto | IBV device name |
| `--ibv-port` | `1` | IBV device port |

---

## Step 7 — Run the subscriber (Terminal 2)

The subscriber receives images from the `vb1940/image` ROS 2 topic and visualizes them using **HoloViz**:

```sh
# From the host, open a second shell into the running container
docker exec -it <container_id> bash
source /opt/ros/jazzy/setup.bash

# Run subscriber
./holohub run vb1940 subscriber --language cpp

# With options:
./holohub run vb1940 subscriber --language cpp -- --headless
./holohub run vb1940 subscriber --language cpp -- --fullscreen
```

---

## Pipeline Summary

```
VB1940 Camera (192.168.0.2)
    ↓  RDMA over IBV
RoCE Receiver (roce_receiver_op)
    ↓
CSI-to-Bayer (csi_to_bayer)
    ↓
Image Processor (image_processor)
    ↓
Bayer Demosaic (bayer_demosaic)        — GPU accelerated
    ↓
CUDA Kernel (convert_16bit → 8bit)     — >> 8 per channel
    ↓
PublisherOp<sensor_msgs::msg::Image>
    ↓
ROS 2 topic: vb1940/image
    ↓
SubscriberOp<sensor_msgs::msg::Image>
    ↓
HoloViz visualization
```

---

## Visualizing the `vb1940/image` Topic

There are multiple ways to visualize the published `vb1940/image` (`sensor_msgs/Image`) topic:

### Option 1 — vb1940 Subscriber via HoloViz (Recommended)

The built-in subscriber renders the topic through the GPU-accelerated HoloViz renderer using CUDA/Vulkan for zero-copy rendering:

```sh
./holohub run vb1940 subscriber --language cpp

# Headless (no display required)
./holohub run vb1940 subscriber --language cpp -- --headless

# Fullscreen
./holohub run vb1940 subscriber --language cpp -- --fullscreen
```

---

### Option 2 — RViz2

Inside the container or on the host (with ROS 2 sourced):

```sh
source /opt/ros/jazzy/setup.bash
rviz2
```

In RViz2:
1. Click **Add** → **By topic** → `/vb1940/image` → **Image** → **OK**
2. Or: Add → **Image** display → set **Topic** to `/vb1940/image`

---

### Option 3 — `rqt_image_view` (Lightweight GUI)

```sh
source /opt/ros/jazzy/setup.bash
ros2 run rqt_image_view rqt_image_view /vb1940/image
```

---

### Option 4 — `image_view` (Minimal viewer)

```sh
source /opt/ros/jazzy/setup.bash
ros2 run image_view image_view --ros-args -r image:=/vb1940/image
```

---

### Option 5 — Inspect topic without display

```sh
# Verify topic is publishing
ros2 topic list | grep vb1940

# Check frame rate
ros2 topic hz /vb1940/image

# Check message type and publisher count
ros2 topic info /vb1940/image

# Print metadata (no pixel data)
ros2 topic echo /vb1940/image --no-arr
```

---

### Visualization Method Comparison

| Method | GPU accelerated | Display needed | Package |
|---|---|---|---|
| vb1940 subscriber (HoloViz) | Yes | Yes (or `--headless`) | holohub |
| RViz2 | No | Yes | `ros-jazzy-rviz2` |
| `rqt_image_view` | No | Yes | `ros-jazzy-rqt-image-view` |
| `image_view` | No | Yes | `ros-jazzy-image-view` |
| `topic echo` | No | No | built-in |

> For AGX Thor with a display and GPU, **HoloViz** (Option 1) is the best choice. For quick debugging without the holohub build, **`rqt_image_view`** is the simplest.

---

## Difference from pubsub

| Aspect | pubsub | vb1940 |
|---|---|---|
| Data source | Synthetic string messages | Live VB1940 camera frames |
| Message type | `std_msgs/String` | `sensor_msgs/Image` |
| GPU processing | None | CSI→Bayer→Demosaic→CUDA kernel |
| Hardware required | None | VB1940 camera + Hololink board |
| SSH required for build | No | Yes (hololink internal repo) |
| Visualization | Console log | HoloViz (Vulkan) |
| ROS 2 topic | `topic` | `vb1940/image` |

---

## Quick Reference — All Commands

```sh
# On host
cd /home/jetsonthor/siva/SPadmana95/holohub

./holohub build-container vb1940 --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13

./holohub run-container vb1940 --language cpp

# Inside container
source /opt/ros/jazzy/setup.bash
./holohub build vb1940 --language cpp --build-args="--ssh default"

# Terminal 1 (inside container) — publisher
./holohub run vb1940 publisher --language cpp

# Terminal 2 (docker exec from host) — subscriber
docker exec -it <container_id> bash
source /opt/ros/jazzy/setup.bash
./holohub run vb1940 subscriber --language cpp
```
