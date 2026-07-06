# Camera Sensor Mode — Deep Dive

## Overview

Camera Sensor Mode (`arg_input_sensor_mode:=0`) is the **live hardware mode** where the ROS 2 node directly interfaces with the EVAL-ADTF3175D-NXZ sensor over USB or SSH. The node reads raw depth, AB (Active Brightness), confidence, and XYZ frames in real time and publishes them as ROS 2 topics.

---

## Code Flow Diagram

```mermaid
flowchart TD
    A([User runs launch command\narg_input_sensor_mode:=0]) --> B

    subgraph LAUNCH["launch/adi_3dtof_adtf31xx_launch.py"]
        B[Parse & declare launch arguments] --> C[Start 4 ROS 2 nodes]
    end

    C --> D
    C --> TF1[static_transform_publisher\nmap → cam1_adtf31xx]
    C --> TF2[static_transform_publisher\ncam1_adtf31xx → cam1_adtf31xx_optical]
    C --> RV[rviz2\nadi_3dtof_adtf31xx.rviz]

    subgraph INIT["Node Initialization — ADI3DToFADTF31xx constructor"]
        D[Declare ROS 2 parameters] --> E
        E["InputSensorFactory::getInputSensor(0)\n→ creates InputSensorADTF31XX"] --> F

        subgraph OPEN["openSensor()  —  input_sensor_adtf31xx.cpp"]
            F["libaditof: system.getCameraList()\nDiscover USB camera"] --> G
            G[Select cameras.front] --> H
            H[Register ADSD3500 interrupt callback\nshutdown node on hardware error] --> I
            I["camera_->initialize(config.json)\nProgram ADSD3500 depth processor"]
        end

        I --> J

        subgraph CFG["configureSensor(camera_mode)  —  input_sensor_adtf31xx.cpp"]
            J["camera_->getAvailableModes()"] --> K
            K["camera_->setMode(camera_mode)\ne.g. lr-qnative = 3"] --> L
            L["camera_->getDetails()\nRead calibration from sensor\nfx, fy, cx, cy, k1–k6, p1, p2"] --> M
            M[Build camera matrix K\nand distortion vector D]
        end

        M --> N[Spawn Input Thread\nreadInput]
        M --> O[Spawn Output Thread\nprocessOutput]
    end

    subgraph IT["Input Thread — adi_3dtof_adtf31xx_input_thread.cpp"]
        N --> P{abort?}
        P -- No --> Q["Allocate ADI3DToFADTF31xxFrameInfo\n(depth, AB, conf, XYZ buffers)"]
        Q --> R["input_sensor_->readNextFrame()\nlibaditof frame capture"]
        R --> S[Get frame timestamp]
        S --> T{Input queue full?}
        T -- No --> U[Push frame to input_frames_queue_]
        T -- Yes --> V[Overwrite last buffer\nwith newest frame]
        U --> P
        V --> P
    end

    subgraph MAIN["Main Timer — adi_3dtof_adtf31xx.cpp  readNextFrame"]
        U --> W[updateTunableParameters\nAB threshold, confidence threshold]
        W --> X[adtf31xxSensorGetNextFrame\npop from input_frames_queue_]
        X --> Y{compression\nenabled?}
        Y -- Yes --> Z["RVL CompressRVL(depth)\nrvl_codec.cpp"]
        Y -- No --> AA
        Z --> AA["memcpy depth/AB/conf/XYZ\ninto ADI3DToFADTF31xxOutputInfo"]
        AA --> AB[adtf31xxSensorPushOutputNode\npush to output_node_queue_]
    end

    subgraph OT["Output Thread — adi_3dtof_adtf31xx_output_thread.cpp"]
        AB --> AC{abort?}
        AC -- No --> AD{Output queue\nnot empty?}
        AD -- Yes --> AE[Pop from output_node_queue_]
        AE --> AF{compression\nenabled?}
        AF -- Yes --> AG["RVL CompressRVL\n(AB + confidence)"]
        AF -- No --> AH
        AG --> AH[publishImageAndCameraInfo]
        AH --> AI{publish\nflags}
        AI --> AJ["/cam1/depth_image\n16-bit"]
        AI --> AK["/cam1/ab_image\n16-bit"]
        AI --> AL["/cam1/conf_image\n16-bit"]
        AI --> AM["/cam1/camera_info"]
        AI --> AN["/cam1/depth_image/compressedDepth\nRVL compressed"]
        AI --> AO["/cam1/ab_image/compressedDepth\nRVL compressed"]
        AI --> AP["/cam1/point_cloud\nPointCloud2"]
        AH --> AC
        AD -- No --> AC
    end
```

---


```bash
ros2 launch adi_3dtof_adtf31xx adi_3dtof_adtf31xx_launch.py arg_input_sensor_mode:=0
```

> **Note:** The default value of `arg_input_sensor_mode` in the launch file is `3` (network mode). You must explicitly pass `arg_input_sensor_mode:=0` to activate camera sensor mode.

---

## Function Call Reference Table

| Step | Description | Function | File | Arguments | Returns |
|------|-------------|----------|------|-----------|---------|
| **1** | Parse and declare all launch arguments; start 4 ROS 2 nodes | `generate_launch_description()` | `launch/adi_3dtof_adtf31xx_launch.py` | — | `LaunchDescription` (4 nodes) |
| **2** | Construct main ROS 2 node; declare all parameters and create all publishers | `ADI3DToFADTF31xx()` | `include/adi_3dtof_adtf31xx.h` | ROS 2 node params via `rclcpp::Node` | Node object |
| **3** | Factory selects `InputSensorADTF31XX` based on `input_sensor_mode = 0` | `InputSensorFactory::getInputSensor(int)` | `include/input_sensor_factory.h` | `input_sensor_type = 0` | `IInputSensor*` → `InputSensorADTF31XX*` |
| **4** | Open the physical sensor — wraps all libaditof SDK init steps | `InputSensorADTF31XX::openSensor(...)` | `src/input_sensor_adtf31xx.cpp` | `sensor_name, image_width, image_height, config_file_name, sensor_ip` | `void` |
| **4a** | Discover all cameras connected via USB | `system.getCameraList(cameras)` | libaditof SDK | `cameras` (output vector) | `void` — populates camera list over USB |
| **4b** | Register a callback that shuts down the node if ADSD3500 raises a hardware error interrupt | `sensor->adsd3500_register_interrupt_callback(cb)` | libaditof SDK | Hardware interrupt callback lambda | `Status` |
| **4c** | Load JSON config file and program the ADSD3500 depth processor | `camera_->initialize(config_file_name)` | libaditof SDK | `config_file_name` — path to JSON config | `Status` — programs ADSD3500 |
| **5** | Configure the imaging mode and read calibration data from the sensor | `InputSensorADTF31XX::configureSensor(int)` | `src/input_sensor_adtf31xx.cpp` | `camera_mode` (e.g. `3` = lr-qnative) | `void` |
| **5a** | Verify the requested camera mode is available on the connected sensor | `camera_->getAvailableModes(modes)` | libaditof SDK | `available_modes` (output vector) | `void` |
| **5b** | Read calibration intrinsics from sensor: `fx, fy, cx, cy, k1–k6, p1, p2` | `camera_->getDetails(camera_details)` | libaditof SDK | `camera_details` (output struct) | `void` — reads `fx, fy, cx, cy, k1–k6, p1, p2` |
| **5c** | Set the active imaging mode (e.g. short-range, long-range, native, mixed) | `camera_->setMode(camera_mode)` | libaditof SDK | `camera_mode` (int) | `Status` |
| **6** | Spawn the input thread that continuously captures frames from the sensor | `std::thread(&ADI3DToFADTF31xx::readInput, ...)` | `src/adi_3dtof_adtf31xx.cpp` (main) | `adi_3dtof_adtf31xx` shared_ptr | `std::thread` — input thread |
| **7** | Spawn the output thread that dequeues and publishes frames to ROS topics | `std::thread(&ADI3DToFADTF31xx::processOutput, ...)` | `src/adi_3dtof_adtf31xx.cpp` (main) | `adi_3dtof_adtf31xx` shared_ptr | `std::thread` — output thread |
| **8** | Input thread entry — runs in a loop until `read_input_thread_abort_` is set | `ADI3DToFADTF31xx::readInput()` | `src/adi_3dtof_adtf31xx_input_thread.cpp` | — (continuous loop) | `void` |
| **8-alloc** | Allocate per-frame buffers for depth, AB, confidence, and XYZ data | `new ADI3DToFADTF31xxFrameInfo(image_width_, image_height_)` | `src/adi_3dtof_adtf31xx_input_thread.cpp` | `image_width_`, `image_height_` | `ADI3DToFADTF31xxFrameInfo*` — allocates depth, AB, conf, XYZ buffers |
| **8a** | Capture one frame from the sensor via libaditof SDK | `input_sensor_->readNextFrame(depth, ab, conf, xyz)` | `src/input_sensor_adtf31xx.cpp` | `unsigned short* depth, ab, conf`; `short* xyz` | `bool` — `true` on success |
| **8b** | Read and attach sensor hardware timestamp to the frame | `input_sensor_->getFrameTimestamp(timestamp)` | `src/input_sensor_adtf31xx.cpp` | `rclcpp::Time*` | `void` |
| **8c** | Push the captured frame onto the thread-safe input queue for the main thread to consume | `input_frames_queue_.push(new_frame)` | `src/adi_3dtof_adtf31xx_input_thread.cpp` | `ADI3DToFADTF31xxFrameInfo*` | `void` — thread-safe push |
| **9** | Main timer callback — bridges input queue to output queue; called by `rclcpp::spin` | `ADI3DToFADTF31xx::readNextFrame()` | `src/adi_3dtof_adtf31xx.cpp` | — (called by `rclcpp::spin` timer) | `bool` |
| **9a** | Synchronise any runtime parameter changes (AB threshold, confidence threshold, JBLF filter, etc.) | `updateTunableParameters()` | `src/adi_3dtof_adtf31xx.cpp` | — | `void` — applies AB/confidence thresholds at runtime *(called first)* |
| **9b** | Pop the next available frame from the input queue | `adtf31xxSensorGetNextFrame()` | `src/adi_3dtof_adtf31xx_input_thread.cpp` | — | `ADI3DToFADTF31xxFrameInfo*` — pops from `input_frames_queue_` |
| **9c** | RVL-compress the depth buffer to reduce bandwidth before publishing | `rvl.CompressRVL(input, output, numPixels)` | `src/ros-perception/.../rvl_codec.cpp` | `depth_frame*, compressed_buf*, image_width × image_height` | `int` — compressed byte size |
| **9d-copy** | Copy raw depth, AB, confidence, and XYZ data into the output struct | `memcpy(depth/AB/conf/XYZ)` | `src/adi_3dtof_adtf31xx.cpp` | `ADI3DToFADTF31xxOutputInfo*` dest, frame src, size | `void` — copies raw frame data into output struct |
| **9e** | Push the filled output struct onto the output queue for the output thread | `adtf31xxSensorPushOutputNode(node)` | `src/adi_3dtof_adtf31xx_output_thread.cpp` | `ADI3DToFADTF31xxOutputInfo*` | `void` — pushes to `output_node_queue_` |
| **10** | Output thread entry — runs in a loop until `process_output_thread_abort_` is set | `ADI3DToFADTF31xx::processOutput()` | `src/adi_3dtof_adtf31xx_output_thread.cpp` | — (continuous loop) | `void` |
| **10-deq** | Dequeue the next output frame from the output queue | `output_node_queue_.front()` + `.pop()` | `src/adi_3dtof_adtf31xx_output_thread.cpp` | — | `ADI3DToFADTF31xxOutputInfo*` — dequeues next output frame |
| **10a** | RVL-compress the AB buffer (only if compression is enabled) | `rvl.CompressRVL(ab, output, numPixels)` | `src/ros-perception/.../rvl_codec.cpp` | `ab_frame*, compressed_buf*, image_width × image_height` | `int` — compressed byte size |
| **10b** | RVL-compress the confidence buffer (only if compression is enabled) | `rvl.CompressRVL(conf, output, numPixels)` | `src/ros-perception/.../rvl_codec.cpp` | `conf_frame*, compressed_buf*, image_width × image_height` | `int` — compressed byte size |
| **10c** | Orchestrate publishing of all enabled ROS topics for one frame | `publishImageAndCameraInfo(frame)` | `src/adi_3dtof_adtf31xx_output_thread.cpp` | `ADI3DToFADTF31xxOutputInfo*` | `void` — orchestrates all publish calls below |
| **10c-1** | Always publish camera calibration info (K, D, R, P matrices) | `fillAndPublishCameraInfo(camera_link_, publisher)` | `include/adi_3dtof_adtf31xx.h` | `frame_id`, `CameraInfo publisher` | `void` — publishes `/cam1/camera_info` (always) |
| **10c-2** | Publish uncompressed 16-bit image (depth / AB / conf) as `sensor_msgs/Image` | `publishImageAsRosMsg(img, encoding, frame_id, publisher)` | `include/adi_3dtof_adtf31xx.h` | `cv::Mat`, encoding string, frame_id, `Image publisher` | `void` — publishes uncompressed depth / AB / conf |
| **10c-3** | Publish RVL-compressed image with `ConfigHeader` prepended as `sensor_msgs/CompressedImage` | `publishRVLCompressedImageAsRosMsg(buf, size, encoding, frame_id, publisher)` | `include/adi_3dtof_adtf31xx.h` | compressed buffer, size, encoding, frame_id, `CompressedImage publisher` | `void` — publishes `/compressedDepth` topics |
| **10c-4** | Convert XYZ short values (mm) to float metres and publish as `sensor_msgs/PointCloud2` | `publishPointCloud(xyz_frame)` | `include/adi_3dtof_adtf31xx.h` | `short* xyz_frame` | `void` — converts mm→m floats, publishes `/cam1/point_cloud` |
| **Shutdown** | Signal input thread to exit its capture loop | `readInputAbort()` | `src/adi_3dtof_adtf31xx_input_thread.cpp` | — | `void` — sets `read_input_thread_abort_ = true` |
| **Shutdown** | Signal output thread to exit its publish loop | `processOutputAbort()` | `src/adi_3dtof_adtf31xx_output_thread.cpp` | — | `void` — sets `process_output_thread_abort_ = true` |

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Hardware | EVAL-ADTF3175D-NXZ connected via USB Type-C to Type-A (5 Gbps) |
| Firmware | Sensor firmware version **≥ 5.2.5.0** |
| SDK | `libaditof` built in the same ROS 2 workspace |
| Build flag | `SENSOR_CONNECTED=True` passed to `colcon build` |
| SSH | Node runs on the sensor itself — SSH into `analog@10.43.0.1` first |

---

## What Happens Internally

When you run the launch command with `arg_input_sensor_mode:=0`, the following sequence occurs:

### 1. InputSensorFactory selects `InputSensorADTF31XX`

In `input_sensor_factory.h`, the factory checks the mode value:

```cpp
case 0:
    input_sensor = new InputSensorADTF31XX;  // Live USB/direct sensor
```

This creates an instance of `InputSensorADTF31XX`, which wraps the `libaditof` SDK.

### 2. `openSensor()` — Discovers and initializes the camera

`InputSensorADTF31XX::openSensor()` (in `src/input_sensor_adtf31xx.cpp`) performs:

1. **Camera discovery** — Calls `system.getCameraList(cameras)` via `libaditof` to find all connected cameras over USB.
2. **Selects the first camera** — Uses `cameras.front()`.
3. **Registers a hardware interrupt callback** — Monitors ADSD3500 hardware interrupt status. If an error interrupt fires, the node shuts down via `rclcpp::shutdown()`.
4. **Initializes the camera** — Calls `camera_->initialize(config_file_name)` using the JSON config file (e.g., `config_adsd3500_adsd3100.json`), which programs the ADSD3500 depth processor.

### 3. `configureSensor()` — Sets camera mode and reads calibration

After initialization:

1. **Sets the imaging mode** — Calls `camera_->setMode(camera_mode)` with the value from `arg_camera_mode` (default: `3` = `lr-qnative`).
2. **Reads camera intrinsics** from the sensor's stored calibration data:
   - Focal lengths: `fx`, `fy`
   - Principal point: `cx`, `cy`
   - Radial distortion: `k1`, `k2`, `k3`, `k4`, `k5`, `k6`
   - Tangential distortion: `p1`, `p2`
3. **Builds the camera matrix**:

$$K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$$

These intrinsics are read directly from the sensor — not from any hardcoded defaults.

### 4. Input Thread — Continuous frame capture

`ADI3DToFADTF31xx::readInput()` (in `src/adi_3dtof_adtf31xx_input_thread.cpp`) runs in a dedicated thread:

- Allocates a new `ADI3DToFADTF31xxFrameInfo` buffer for each frame.
- Calls `input_sensor_->readNextFrame(depth, ab, conf, xyz)` which calls the `libaditof` frame capture API.
- Pushes the filled frame onto the **input queue** (thread-safe, mutex-protected).
- If the queue is full, the oldest frame is dropped to avoid memory buildup.

### 5. Output Thread — Processing and Publishing

`ADI3DToFADTF31xx::processOutput()` (in `src/adi_3dtof_adtf31xx_output_thread.cpp`) runs in a second dedicated thread:

- Dequeues frames from the input queue.
- Optionally applies **RVL lossless compression** to depth and AB frames (if `arg_enable_depth_ab_compression:=True`).
- Publishes enabled topics to ROS 2.

---

## ROS 2 Nodes Started by the Launch File

The launch file starts **four nodes** in the `cam1` namespace:

| Node | Package | Purpose |
|---|---|---|
| `adi_3dtof_adtf31xx_node` | `adi_3dtof_adtf31xx` | Main sensor node — captures and publishes frames |
| `cam1_adtf31xx_optical_tf` | `tf2_ros` | Static TF: `cam1_adtf31xx` → `cam1_adtf31xx_optical` (optical rotation −90° roll, −90° yaw) |
| `cam1_adtf31xx_tf` | `tf2_ros` | Static TF: `map` → `cam1_adtf31xx` (position from `arg_camera_height_from_ground_in_mtr`) |
| `rviz2` | `rviz2` | Visualization with the pre-configured `adi_3dtof_adtf31xx.rviz` layout |

### TF Tree

```
map
 └── cam1_adtf31xx          (position: x=0, y=0, z=arg_camera_height_from_ground_in_mtr)
      └── cam1_adtf31xx_optical   (rotation: roll=-1.57, pitch=0, yaw=-1.57)
```

---

## Published Topics (under `/cam1/` namespace)

| Topic | Type | Condition |
|---|---|---|
| `/cam1/depth_image` | `sensor_msgs/Image` (16-bit) | `arg_enable_depth_publish:=True` |
| `/cam1/ab_image` | `sensor_msgs/Image` (16-bit) | `arg_enable_ab_publish:=True` |
| `/cam1/conf_image` | `sensor_msgs/Image` (16-bit) | `arg_enable_conf_publish:=True` |
| `/cam1/camera_info` | `sensor_msgs/CameraInfo` | Always |
| `/cam1/depth_image/compressedDepth` | `sensor_msgs/CompressedImage` | `arg_enable_depth_ab_compression:=True` |
| `/cam1/ab_image/compressedDepth` | `sensor_msgs/CompressedImage` | `arg_enable_depth_ab_compression:=True` |
| `/cam1/point_cloud` | `sensor_msgs/PointCloud2` | `arg_enable_point_cloud_publish:=True` |

---

## Topic Details

All topics use the same QoS profile:

| QoS Field | Value |
|---|---|
| History | `KEEP_LAST` |
| Depth | 10 |
| Reliability | `RELIABLE` |
| Durability | `VOLATILE` |

---

### `/cam1/depth_image` — 16-bit Depth Image

| Field | Detail |
|---|---|
| ROS message type | `sensor_msgs/msg/Image` |
| Publisher | `depth_image_publisher_` |
| Encoding | `mono16` or `16UC1` (set by `arg_encoding_type`) |
| OpenCV format | `CV_16UC1` — single-channel 16-bit unsigned |
| Image size | `image_width_ × image_height_` (default 1024×1024 for ADSD3100, 640×512 for ADSD3030) |
| Pixel value | Distance from camera in **millimetres** (raw sensor output, 16-bit range: 0–65535 mm) |
| frame_id | `cam1_adtf31xx_optical` |
| Timestamp | `curr_frame_timestamp_` from sensor |
| Enabled by | `arg_enable_depth_publish:=True` (default: True) |
| Disabled when | `arg_enable_depth_ab_compression:=True` — replaced by `/compressedDepth` variant |

> **Note:** Pixels that fail the AB threshold (`arg_ab_threshold`) or confidence threshold (`arg_confidence_threshold`) are invalidated by the ADSD3500 depth processor and will appear as `0`.

---

### `/cam1/ab_image` — 16-bit Active Brightness (IR) Image

| Field | Detail |
|---|---|
| ROS message type | `sensor_msgs/msg/Image` |
| Publisher | `ab_image_publisher_` |
| Encoding | `mono16` or `16UC1` (set by `arg_encoding_type`) |
| OpenCV format | `CV_16UC1` |
| Image size | Same as depth image |
| Pixel value | Intensity of reflected infrared light — higher value = stronger IR return |
| frame_id | `cam1_adtf31xx_optical` |
| Timestamp | `curr_frame_timestamp_` (same as depth, captured together) |
| Enabled by | `arg_enable_ab_publish:=True` (default: True) |
| Use case | Texture overlay on point cloud, scene understanding, obstacle detection in low-light |

> **Note:** AB = Active Brightness. It is the amplitude of the reflected modulated IR signal, analogous to an IR intensity image. Bright objects reflect more IR → higher pixel value.

---

### `/cam1/conf_image` — 16-bit Confidence Image

| Field | Detail |
|---|---|
| ROS message type | `sensor_msgs/msg/Image` |
| Publisher | `conf_image_publisher_` |
| Encoding | `mono16` or `16UC1` |
| OpenCV format | `CV_16UC1` |
| Image size | Same as depth image |
| Pixel value | Confidence/reliability score of the depth measurement per pixel — higher = more reliable |
| frame_id | `cam1_adtf31xx_optical` |
| Timestamp | `curr_frame_timestamp_` |
| Enabled by | `arg_enable_conf_publish:=True` (default: True) |
| Use case | Mask out low-confidence depth pixels before processing — filter by threshold |

---

### `/cam1/camera_info` — Camera Calibration Info

| Field | Detail |
|---|---|
| ROS message type | `sensor_msgs/msg/CameraInfo` |
| Publisher | `depth_info_publisher_` |
| Always published | Yes — regardless of compression or publish flags |
| Distortion model | `RATIONAL_POLYNOMIAL` (8 coefficients: k1, k2, p1, p2, k3, k4, k5, k6) |

**Message fields populated:**

| Field | Content |
|---|---|
| `width`, `height` | Sensor image resolution |
| `K[9]` | Camera matrix — `[fx, 0, cx, 0, fy, cy, 0, 0, 1]` from sensor calibration |
| `D[]` | Distortion coefficients — `[k1, k2, p1, p2, k3, k4, k5, k6]` |
| `R[9]` | Rotation matrix from extrinsics |
| `P[12]` | Projection matrix — includes translation `[Tx, Ty, Tz]` from extrinsics |
| `header.frame_id` | `cam1_adtf31xx_optical` |
| `header.stamp` | `curr_frame_timestamp_` |

---

### `/cam1/depth_image/compressedDepth` — RVL Compressed Depth

| Field | Detail |
|---|---|
| ROS message type | `sensor_msgs/msg/CompressedImage` |
| Publisher | `compressed_depth_image_publisher_` |
| Compression algorithm | **RVL (Run-Length Variable-Length)** lossless codec — `rvl_codec.cpp` |
| Format string | `"mono16;compressedDepth rvl"` |
| Enabled by | `arg_enable_depth_ab_compression:=True` (default: False) |
| Header prepended | `ConfigHeader` with `format = INV_DEPTH` + `image_width_` + `image_height_` (8 bytes + sizeof ConfigHeader) |
| Worst-case size | ~1.5× input (1.5 × width × height × 2 bytes) — RVL guarantees this upper bound |
| Typical compression | 2×–4× for typical ToF depth data with large uniform regions |
| Compatible with | `image_transport` compressed depth subscriber — can be decompressed automatically |

**Compressed packet layout:**

```
[ ConfigHeader (depthParam[0], depthParam[1], format=INV_DEPTH) ]
[ image_width (4 bytes) ]
[ image_height (4 bytes) ]
[ RVL compressed pixel data ]
```

---

### `/cam1/ab_image/compressedDepth` — RVL Compressed AB Image

| Field | Detail |
|---|---|
| ROS message type | `sensor_msgs/msg/CompressedImage` |
| Publisher | `compressed_ab_image_publisher_` |
| Compression algorithm | **RVL lossless** — same codec as depth |
| Enabled by | `arg_enable_depth_ab_compression:=True` |
| Source data | `out_frame->ab_frame_` (16-bit unsigned short array) |
| Same packet layout | As `/depth_image/compressedDepth` |

---

### `/cam1/point_cloud` — 3D Point Cloud

| Field | Detail |
|---|---|
| ROS message type | `sensor_msgs/msg/PointCloud2` |
| Publisher | `xyz_image_publisher_` |
| Enabled by | `arg_enable_point_cloud_publish:=True` (default: False) |
| Fields | `x`, `y`, `z` (float32 each) |
| Organised | Yes — `width × height` structured grid (same layout as depth image) |
| `is_dense` | `false` — invalid points (depth = 0) are included as NaN-equivalent |
| `is_bigendian` | `false` |
| Units | **metres** — raw XYZ shorts from sensor are in mm; divided by 1000.0 before publishing |
| Source | `out_frame->xyz_frame_` — pre-computed XYZ array from `ImageProcUtils` range-to-3D LUT |
| frame_id | `cam1_adtf31xx_optical` |

**XYZ conversion:**
```cpp
*iter_x = (float)(*xyz_sensor_buf++) / 1000.0f;  // mm → metres
*iter_y = (float)(*xyz_sensor_buf++) / 1000.0f;
*iter_z = (float)(*xyz_sensor_buf++) / 1000.0f;
```

The XYZ values are computed using a pre-generated Look-Up Table (LUT) in `ImageProcUtils` that maps each pixel `(u, v)` and its depth `d` to a 3D point using the camera intrinsics.

---



All parameters can be overridden on the command line:

```bash
ros2 launch adi_3dtof_adtf31xx adi_3dtof_adtf31xx_launch.py \
  arg_input_sensor_mode:=0 \
  arg_camera_mode:=1 \
  arg_ab_threshold:=20 \
  arg_confidence_threshold:=15 \
  arg_enable_point_cloud_publish:=True \
  arg_enable_depth_ab_compression:=True \
  arg_camera_height_from_ground_in_mtr:=0.25
```

### Key Parameters for Camera Sensor Mode

| Parameter | Recommended Value | Notes |
|---|---|---|
| `arg_input_sensor_mode` | `0` | Must be `0` for direct USB/sensor mode |
| `arg_camera_mode` | `1` (lr-native) or `3` (lr-qnative) | See Camera Modes table |
| `arg_config_file_name_of_tof_sdk` | `config_adsd3500_adsd3100.json` | Use `adsd3030` variant for VGA sensor |
| `arg_ab_threshold` | `10`–`30` | Higher value filters more noisy pixels |
| `arg_confidence_threshold` | `10`–`25` | Higher value keeps only high-confidence pixels |
| `arg_camera_height_from_ground_in_mtr` | Measure and set | Affects TF and point cloud ground position |

---

## Camera Sensor Mode vs Other Modes

| Aspect | Camera Sensor Mode (`0`) | File-IO Mode (`2`) | Network Mode (`3`) |
|---|---|---|---|
| Data source | Live USB sensor | Pre-recorded `.bin` file | Sensor over LAN/USB-network |
| Requires hardware | Yes | No | Yes (over network) |
| Requires `libaditof` | Yes | No | Yes |
| Real-time | Yes | Replay speed | Yes |
| Calibration source | Read from sensor | Hardcoded defaults | Read from sensor |
| Typical use | On-sensor deployment | Host PC evaluation | Host PC with live sensor |
