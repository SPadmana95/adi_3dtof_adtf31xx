# HoloHub ROS 2 — Build and Run Guide

## Overview

This guide describes how to build and run the `pubsub` and `vb1940` Holoscan ↔ ROS 2 applications from the `holohub` repository on **NVIDIA AGX Thor** with **JetPack 7.0** and **Holoscan SDK 3.9.0**.

Clone the repository before running any commands:

```sh
git clone -b adcam_ros2 https://github.com/SPadmana95/holohub.git
cd holohub
```

All build and run commands in this guide assume your working directory is the `holohub` root.

### Why `pubsub` and not `holoscan_ros2`?

`holoscan_ros2` is the parent directory — it has no run configuration of its own. The runnable sub-applications are:
- `pubsub` — string message pub/sub (C++ and Python)
- `vb1940` — VB1940 Eagle camera pipeline (C++ only)

### Native Build Option (No Docker)

Since **ROS 2 Jazzy is natively installed** on AGX Thor (verified: `echo $ROS_DISTRO` → `jazzy`), you can build and run `pubsub` without Docker:

```sh
cd holohub
source /opt/ros/jazzy/setup.bash

# Build natively
./holohub build pubsub --language cpp

# Run publisher (terminal 1)
./holohub run pubsub publisher --language cpp

# Run subscriber (terminal 2)
./holohub run pubsub subscriber --language cpp
```

> This is faster and simpler than Docker for development on AGX Thor. Docker is recommended for reproducible deployments or when ROS 2 is not natively available on the target machine.

### Why do `pubsub` and `vb1940` have separate Dockerfiles?

Each application has different dependencies and a shared Dockerfile would force unnecessary build time on simpler use cases:

| Dependency | `pubsub` | `vb1940` | Reason |
|---|---|---|---|
| ROS 2 Jazzy | ✓ | ✓ | Required for topic publishing/subscribing |
| Holoscan SDK | ✓ | ✓ | Core pipeline framework |
| Vulkan (`vulkan-tools`, `libvulkan1`) | ✗ | ✓ | HoloViz needs Vulkan for GPU-accelerated image display |
| `holoscan-sensor-bridge` (built from source) | ✗ | ✓ | C++ libraries for VB1940 camera over RDMA/IBV (RoCE receiver, CSI-to-Bayer, image processor) |
| IBV/RDMA support | ✗ | ✓ | VB1940 camera hardware communication |

**`pubsub` Dockerfile** (`applications/holoscan_ros2/Dockerfile`):
- Installs only ROS 2 Jazzy + base Holoscan SDK
- Minimal — no camera hardware stack, no Vulkan
- Fast to build (~2 min)

**`vb1940` Dockerfile** (`applications/holoscan_ros2/vb1940/Dockerfile`):
- Installs ROS 2 Jazzy + Vulkan + `holoscan-sensor-bridge` (cloned at tag `2.5.0` and compiled from source)
- Heavy — includes full camera hardware stack
- Slow to build (~50+ min on first build)

> If they shared one Dockerfile, every developer building the simple `pubsub` string example would wait 50+ minutes for the VB1940 camera SDK to compile — even though no camera hardware is involved.

---

## Prerequisites

| Requirement | Version |
|---|---|
| Host Device | NVIDIA AGX Thor |
| JetPack | 7.0 |
| Holoscan SDK | 3.9.0 |
| Docker | With NVIDIA Container Toolkit |
| holohub branch | `adcam_ros2` (tag `holoscan-sdk-3.9.0`) |
| holoscan-sensor-bridge branch | `demo_HSB_ToF_ROS2` (base tag `2.5.0`) |

---

## Step 1 — Exit any existing container

If you are already inside a holohub container, exit it first:

```sh
exit
```

---

## Step 2 — Navigate to the holohub directory

```sh
cd holohub
```

---

## Step 3 — Build the container with ROS 2

Use the `pubsub` project name so the CLI picks up `applications/holoscan_ros2/Dockerfile`, which installs ROS 2 Jazzy. Override the base image to match your SDK version:

```sh
./holohub build-container pubsub --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13 \
  --build-args="--no-cache"
```

> **Why `--no-cache`?** Both Dockerfiles contain `Acquire::ForceIPv4 "true"` to fix IPv6 network failures. Without `--no-cache`, Docker may reuse a stale cached layer from before this fix was applied.

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

# Check ROS 2 distro
echo $ROS_DISTRO

# Full ROS 2 setup check
ros2 doctor
```

Expected output from `ros2 doctor`:
```
ROS_DISTRO : jazzy
ROS_VERSION : 2
...
1 package(s) checked, no errors found
```

> **Note:** `ros2 --version` is not a valid command in ROS 2. Use `echo $ROS_DISTRO` or `ros2 doctor` instead.

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

> **To stop:** Press `Ctrl+C`. The publisher runs indefinitely until stopped.

> **`ROS_DOMAIN_ID`:** Publisher and subscriber must be on the same ROS 2 domain. If they don't communicate, check: `echo $ROS_DOMAIN_ID` in both terminals — both should show the same value (default is `0`).

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

## Troubleshooting

| Error | Root Cause | Fix |
|---|---|---|
| `rclcpp` not found during CMake | Generic holohub container has no ROS 2 | Use `pubsub` or `vb1940` project name — not `holoscan_ros2` — to pick up the app-specific Dockerfile |
| `holoscan 4.0 not found` (CMake) | vb1940 Dockerfile clones HSB `main` branch (requires SDK 4.0+) | Dockerfile now pins `git checkout tags/2.5.0` — rebuild with `--no-cache` |
| `Network is unreachable` (apt) | Docker `apt` resolves to IPv6 address; AGX Thor has no IPv6 routing | Both Dockerfiles include `Acquire::ForceIPv4 "true"` fix — rebuild with `--no-cache` |
| Old cached layer reused | Docker uses cached layer from before Dockerfile fix | Add `--build-args="--no-cache"` to `build-container` command |
| Wrong base image | App Dockerfile hardcodes `holoscan:v3.3.0-dgpu` | Always pass `--base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13` |
| Publisher and subscriber don't communicate | Different `ROS_DOMAIN_ID` values | Ensure `echo $ROS_DOMAIN_ID` is the same in both terminals (default: `0`) |
| Standalone holohub uses old Dockerfile | Fix applied to submodule but not synced | Run `cd holohub && git pull origin adcam_ros2` |
| `holoscan_ros2` has no run config | `holoscan_ros2` is a parent dir, not a runnable app | Use `pubsub` or `vb1940` as the project name |

---

## Quick Reference — All Commands

```sh
# On host — navigate to holohub
cd holohub --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13 \
  --build-args="--no-cache"

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
| holoscan-sensor-bridge branch | `demo_HSB_ToF_ROS2` (base tag `2.5.0`, SDK 3.9.0 compatible) |

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
cd holohub
```

---

## Step 3 — Build the container (with ROS 2 + Vulkan + hololink)

> **Note:** The vb1940 `README.md` only documents `./holohub build vb1940` (the application build inside the container). The `build-container` step below is **missing from the README** — it was derived from the holohub CLI and `metadata.json`.

### How this command was derived

```
./holohub build-container vb1940 --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13 \
  --build-args="--no-cache"
```

| Part | Source | Reason |
|---|---|---|
| `build-container` | `./holohub build-container --help` | Separate from `build` — this builds the Docker image |
| `vb1940` | `metadata.json` → `"dockerfile": "applications/holoscan_ros2/vb1940/Dockerfile"` | CLI uses this to find the correct Dockerfile |
| `--language cpp` | `metadata.json` → `"language": "C++"` | Required when language must be explicit |
| `--base-img holoscan:v3.9.0-cuda13` | Override for Dockerfile's hardcoded `ARG BASE_IMAGE=holoscan:v3.5.0-dgpu` | Must match your installed SDK (3.9.0) and CUDA version (13 on AGX Thor) |
| `--build-args="--no-cache"` | Docker build cache issue | Forces Docker to re-run `git clone && git checkout tags/2.5.0`; without it, a stale cached layer clones the `main` branch (SDK 4.0+) causing CMake failure |

The `vb1940` Dockerfile (`applications/holoscan_ros2/vb1940/Dockerfile`) installs:
- ROS 2 Jazzy (`ros-jazzy-desktop`, `ros-jazzy-ros-base`)
- Vulkan support (`vulkan-tools`, `vulkan-validationlayers`, `libvulkan1`) — required for HoloViz
- git (for cloning hololink repo during build)

```sh
./holohub build-container vb1940 --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13 \
  --build-args="--no-cache"
```

> **Why `--no-cache`?**  
> The vb1940 Dockerfile clones `holoscan-sensor-bridge` and checks out tag `2.5.0` (SDK 3.9.0 compatible). Without `--no-cache`, Docker may reuse a previously cached layer that cloned the upstream `main` branch (which requires SDK 4.0+), causing a CMake version error.

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

## Step 4b — SSH Key Setup (required before Step 5)

The vb1940 build requires SSH access to the NVIDIA internal hololink repository. Set up SSH agent forwarding **before** entering the container:

```sh
# On host — start ssh-agent and add your key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519      # ed25519 key (use id_rsa if you have RSA key instead)

# Verify key is loaded
ssh-add -l
```

The `./holohub run-container` script automatically forwards the SSH agent into the container when `ssh-agent` is running.

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

> **To stop:** Press `Ctrl+C` in the publisher terminal. The subscriber will exit automatically once the publisher stops. Ensure `ROS_DOMAIN_ID` is the same in both terminals.

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

## Why vb1940 Requires `holoscan-sensor-bridge` as a Dependency

The VB1940 (Eagle) camera does **not** connect like a standard USB or CSI camera. It uses a proprietary high-speed hardware interface that requires the Holoscan Sensor Bridge SDK to function.

### What the VB1940 Camera Is

The VB1940 sensor connects to the host via a **Hololink board** — an FPGA-based hardware bridge that transfers raw camera data over **RDMA (Remote Direct Memory Access)** using **RoCE (RDMA over Converged Ethernet)**. This is a zero-copy, kernel-bypass data path designed for ultra-low latency.

```
VB1940 Sensor
    ↓ MIPI CSI-2 (raw sensor data)
Hololink FPGA Board (192.168.0.2)
    ↓ 10GigE / RDMA / RoCE
AGX Thor Host (IBV device)
    ↓ holoscan-sensor-bridge
vb1940 Application
```

### What `holoscan-sensor-bridge` Provides

The SDK provides the C++ operators that implement each stage of this data path:

| Operator | Source | Purpose |
|---|---|---|
| `RoCE ReceiverOp` | `holoscan-sensor-bridge` | Opens IBV device, receives raw frames from Hololink board via RDMA into GPU memory |
| `CsiToBayerOp` | `holoscan-sensor-bridge` | Converts packed MIPI CSI-2 raw bitstream → Bayer pattern image |
| `ImageProcessorOp` | `holoscan-sensor-bridge` | Applies sensor-level corrections (lens shading, black level) |
| `NativeVb1940Sensor` | `holoscan-sensor-bridge` | Camera configuration API — sets mode, FPS, gain, exposure |
| `Vb1940Mode` | `holoscan-sensor-bridge` | Enumerates supported sensor modes (resolution/FPS combinations) |

Without these operators, the application has **no way to open the camera, receive data, or decode the raw bitstream** — the VB1940 is invisible to standard Linux camera APIs (V4L2, GStreamer, OpenCV `VideoCapture`).

### Why It Must Be Built From Source Inside the Container

The `holoscan-sensor-bridge` C++ libraries link directly against **Holoscan SDK** (`libholoscan`). The version must match exactly:

| holoscan-sensor-bridge tag | Compatible Holoscan SDK |
|---|---|
| `2.5.0` | **3.9.0** ✓ |
| `main` | 4.0+ |

A mismatch causes the CMake error seen earlier:
```
Could not find holoscan compatible with requested version "4.0"
Found: /opt/nvidia/holoscan version: 3.9.0
```

This is why the Dockerfile pins `git checkout tags/2.5.0` after cloning.

### Why pubsub and vb1940 Have Separate Dockerfiles

```
pubsub Dockerfile                  vb1940 Dockerfile
─────────────────                  ─────────────────
Holoscan SDK (from base image)     Holoscan SDK (from base image)
ROS 2 Jazzy                        ROS 2 Jazzy
                                   Vulkan (HoloViz display)
                                   holoscan-sensor-bridge (camera SDK, built from source)
                                   IBV/RDMA support (VB1940 hardware)
```

If they shared one Dockerfile, every developer building the simple `pubsub` example would be forced to wait for `holoscan-sensor-bridge` to clone and compile (~30+ min) and have Vulkan installed — even when no VB1940 camera is present. The separation keeps build times fast for simpler use cases.

### Data Flow Comparison

```
pubsub:
  AGX Thor ──[DDS/ROS 2]──> /topic (std_msgs/String)
  No hardware. No SDK. Just ROS 2.

vb1940:
  VB1940 ──[MIPI CSI-2]──> Hololink FPGA (192.168.0.2)
         ──[10GigE RoCE]──> RoCE ReceiverOp   (holoscan-sensor-bridge)
         ──[GPU memory]──>  CsiToBayer         (holoscan-sensor-bridge)
                        ──> ImageProcessor     (holoscan-sensor-bridge)
                        ──> BayerDemosaic       (Holoscan SDK)
                        ──> CUDA 16→8bit kernel (custom)
                        ──> PublisherOp         (holoscan ROS 2 bridge)
                        ──> /vb1940/image       (sensor_msgs/Image)
```

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
# On host — navigate to holohub
cd holohub

./holohub build-container vb1940 --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13 \
  --build-args="--no-cache"

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

---

---

# Docker Reference — Container Detection and Cache Control

## How to Know If You Are Inside or Outside a Docker Container

### Method 1 — Check for `.dockerenv` file (most reliable)

```sh
ls /.dockerenv && echo "INSIDE container" || echo "OUTSIDE container"
```

### Method 2 — Check the shell prompt

```sh
# Outside (host): username@hostname format with tilde home path
user@myhostname:~/holohub$

# Inside holohub container: root user with short container hash and /workspace path
root@b3f40405e4f6:/workspace/holohub$
```

Inside the container: user is always `root`, hostname is a short container ID hash, working directory is `/workspace/holohub`.

### Method 3 — Check `cgroup`

```sh
cat /proc/1/cgroup | grep -i docker
# Returns output if inside container, empty if on host
```

### Method 4 — Check for Docker-specific environment variables

```sh
env | grep -i docker
# e.g. DOCKER_CONTAINER=1 may be set by the holohub run script
```

### Summary

| Indicator | Host (outside) | Container (inside) |
|---|---|---|
| `ls /.dockerenv` | File not found | File exists |
| Hostname | Your machine hostname | Short container ID hash |
| User | Your login username | `root` |
| Working dir | `~/holohub` | `/workspace/holohub` |
| `cat /proc/1/cgroup` | No `docker` entries | Contains `docker` entries |

---

## Docker Build — With and Without Cache

### Standard build (with cache — default)

Docker reuses cached layers for any Dockerfile step whose instruction has not changed. This makes repeated builds fast.

```sh
# Generic docker
docker build -t my_image .

# holohub wrapper (uses cache by default)
./holohub build-container vb1940 --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13
```

### Build without cache (`--no-cache`)

Forces Docker to re-execute every layer from scratch. Use this when a cached layer is stale (e.g. a `git clone` step cached before a Dockerfile fix).

```sh
# Generic docker
docker build --no-cache -t my_image .

# holohub wrapper
./holohub build-container vb1940 --language cpp \
  --base-img nvcr.io/nvidia/clara-holoscan/holoscan:v3.9.0-cuda13 \
  --build-args="--no-cache"
```

### Clear the Docker build cache manually

```sh
# Remove only dangling/unused build cache
docker builder prune

# Remove ALL build cache (frees maximum disk space)
docker builder prune -a

# Check how much disk cache is being used
docker system df
```

### When to use `--no-cache`

| Situation | Use `--no-cache`? |
|---|---|
| First-ever build (no cache exists) | No — nothing to reuse anyway |
| Dockerfile was updated but Docker still uses old cached `git clone` layer | **Yes** |
| `apt` packages need to be refreshed | **Yes** |
| Build failed with IPv6 error before IPv4 fix was applied | **Yes** |
| SDK version mismatch error occurred before HSB `2.5.0` pin was added | **Yes** |
| Only your application code changed (not the Dockerfile) | No — cache speeds it up |
| Debugging a failed build and want a guaranteed clean state | **Yes** |

### Why `--no-cache` matters for `vb1940`

The `vb1940` Dockerfile contains this step:

```dockerfile
# Before fix (cached layer — clones main branch, SDK 4.0+)
RUN git clone https://github.com/nvidia-holoscan/holoscan-sensor-bridge.git

# After fix (correct — checks out tag 2.5.0, SDK 3.9.0 compatible)
RUN git clone https://github.com/nvidia-holoscan/holoscan-sensor-bridge.git \
    && cd holoscan-sensor-bridge \
    && git checkout tags/2.5.0
```

Even after the Dockerfile is updated, Docker may still serve the **old cached layer** for the `git clone` step because the cache key matches. `--no-cache` forces re-execution of the clone with the new tag checkout.
