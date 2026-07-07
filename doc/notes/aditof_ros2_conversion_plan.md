# ADI ToF → ROS 2 Publisher/Subscriber — Design Notes

Source: `holoscan-sensor-bridge/examples/aditof/cpp/` (branch `demo_HSB_ToF_ROS2` @ HSB 2.5.0)

---

## Overview

This document explains how to adapt the `adcam_player` Holoscan application
(ADI ADTF3175 ToF camera viewer) into a **ROS 2 publisher/subscriber** by
referencing the `holohub/applications/holoscan_ros2/vb1940/` examples as the
ROS 2 integration pattern.

---

## Does This Require InfiniBand (IBV)?

**No — InfiniBand is NOT required for this use case on AGX Thor + JP 7.0 + HSB 2.5.0.**

The `adcam_player.cpp` supports **two receiver paths**:

| Receiver | IBV required? | JP 7.0 support | Used when |
|---|---|---|---|
| `LinuxReceiverOp` | ❌ No | ✅ Yes | `ibv_name` is empty (auto-selected on JP 7.0) |
| `RoceReceiverOp` | ✅ Yes | ❌ No (added in HSB 2.6-EA2, JP 7.1) | `ibv_name` is set |

The application **auto-detects** IBV at startup:
```cpp
// adcam_player.cpp
auto devices = hololink::infiniband_devices();
ibv_name = devices.size() > 0 ? devices[0] : "";
// If no IBV found → ibv_name is empty → LinuxReceiverOp is used
```

On JP 7.0 + HSB 2.5.0, `ibv_devinfo` returns "No IB devices found" →
`ibv_name` is empty → `LinuxReceiverOp` is automatically selected.

---

## aditof Data Path (HSB hardware — not USB)

The `adcam_player` uses the **Holoscan Sensor Bridge (HSB) hardware**, which
is different from the `adi_3dtof_adtf31xx` ROS 2 package that uses USB + libaditof.

```
ADI ADTF3175 ToF Sensor
    ↓  MIPI CSI-2 (raw phase/amplitude data)
ADSD3500 Dual Depth Processor
    ↓  Processes: Phase → Depth, calibration, filtering
    ↓  Outputs: Depth (16-bit), AB (16-bit), Confidence (8-bit) via MIPI
HSB FPGA (Holoscan Sensor Bridge @ 192.168.0.2)
    ↓  Packetizes frames into UDP packets over 10GigE
LinuxReceiverOp   (JP 7.0, no IBV needed)
    ↓
CsiToBayerOp      (CSI-2 raw → Bayer pattern, GPU)
    ↓
ADTFUnpackOp      (ADI 5-byte/pixel format → 3 planes: Depth / AB / Conf)
    ↓
HolovizOp         (visualization — LEFT: Depth | CENTER: AB | RIGHT: Conf)
```

---

## Current Operator Graph (`adcam_player.cpp`)

```cpp
// adcam_player.cpp compose() — operator connections
add_flow(receiver_operator,   csi_to_bayer_operator, {{"output", "input"}});
add_flow(csi_to_bayer_operator, ADIToF_data,         {{"output", "input"}});
add_flow(ADIToF_data,           visualizer,          {{"output", "receivers"}});
```

| Step | Operator | Output | Notes |
|---|---|---|---|
| 1 | `LinuxReceiverOp` / `RoceReceiverOp` | Raw CSI frame (GPU) | Auto-selected by IBV availability |
| 2 | `CsiToBayerOp` | Bayer pattern (GPU) | `BlockMemoryPool`, 2 blocks |
| 3 | `ADTFUnpackOp` | 3 tensors: `Depth`, `ActiveBrightness`, `Conf` | 5-byte/pixel → 16-bit planes |
| 4 | `HolovizOp` | Display (3 panels side-by-side) | Replace with ROS 2 publishers |

---

## Plan: Converting to ROS 2 Publisher

### What to change

Replace **`HolovizOp`** (step 4) with **three `PublisherOp` instances** from
the holohub `holoscan_ros2` operator package:

```cpp
// NEW: ROS 2 bridge resource
auto bridge = make_resource<holoscan::ros2::Bridge>(
    "adcam_ros2_bridge", "adcam_ros2_node");

// NEW: Three publishers — one per output plane
auto depth_publisher = make_operator<DepthPublisherOp>(
    "depth_pub",
    holoscan::Arg("ros2_bridge", bridge),
    holoscan::Arg("topic_name", std::string("/adcam/depth_image")),
    holoscan::Arg("qos", holoscan::ros2::QoS(10)));

auto ab_publisher = make_operator<ABPublisherOp>(
    "ab_pub",
    holoscan::Arg("ros2_bridge", bridge),
    holoscan::Arg("topic_name", std::string("/adcam/ab_image")),
    holoscan::Arg("qos", holoscan::ros2::QoS(10)));

auto conf_publisher = make_operator<ConfPublisherOp>(
    "conf_pub",
    holoscan::Arg("ros2_bridge", bridge),
    holoscan::Arg("topic_name", std::string("/adcam/conf_image")),
    holoscan::Arg("qos", holoscan::ros2::QoS(10)));
```

### Updated operator graph

```
LinuxReceiverOp
    ↓
CsiToBayerOp
    ↓
ADTFUnpackOp (outputs: Depth, ActiveBrightness, Conf tensors)
    ├──→ DepthPublisherOp    → ROS 2 topic: /adcam/depth_image
    ├──→ ABPublisherOp       → ROS 2 topic: /adcam/ab_image
    └──→ ConfPublisherOp     → ROS 2 topic: /adcam/conf_image
```

### ROS 2 Published Topics

| Topic | Type | Content |
|---|---|---|
| `/adcam/depth_image` | `sensor_msgs/Image` (16-bit) | Depth in mm |
| `/adcam/ab_image` | `sensor_msgs/Image` (16-bit) | Active Brightness (IR intensity) |
| `/adcam/conf_image` | `sensor_msgs/Image` (16-bit) | Confidence/reliability score |
| `/adcam/camera_info` | `sensor_msgs/CameraInfo` | Calibration (optional) |

---

## Reference: vb1940 vs aditof Comparison

| Aspect | `vb1940_publisher.cpp` (holohub) | `adcam_player.cpp` (HSB aditof) |
|---|---|---|
| Camera sensor | VB1940 Eagle (RGB) | ADI ADTF3175 (ToF) |
| Output type | RGB image (8-bit, 1 plane) | Depth + AB + Conf (16-bit, 3 planes) |
| Receiver | `RoceReceiverOp` (IBV required) | `LinuxReceiverOp` OR `RoceReceiverOp` (auto-select) |
| IBV on JP 7.0 | ❌ Fails — no IBV | ✅ Works — falls back to Linux receiver |
| Unpacking | `convert_16bit_to_8bit` CUDA kernel | `ADTFUnpackOp` (5-byte/pixel → 3 planes) |
| ROS 2 message | `sensor_msgs/Image` (1 topic) | `sensor_msgs/Image` (3 topics) |
| ROS 2 integration | `PublisherOp<sensor_msgs::msg::Image>` | Same pattern × 3 |

---

## Key Files

| File | Purpose |
|---|---|
| `examples/aditof/cpp/adcam_player.cpp` | Main application — full pipeline |
| `examples/aditof/cpp/adcam_unpack_op.cpp` | ADTFUnpackOp — 5-byte/pixel → Depth/AB/Conf |
| `examples/aditof/cpp/adcam_unpack_op.hpp` | ADTFUnpackOp header |
| `examples/aditof/cpp/adcam_lib.cpp` | ADSD3500 I2C control (mode config, streaming) |
| `examples/aditof/cpp/adcam_lib.hpp` | ADSD3500 command constants + class |
| `holohub/operators/holoscan_ros2/cpp/holoscan/ros2/operators/publisher.hpp` | `PublisherOp<T>` template — ROS 2 bridge publisher |
| `holohub/operators/holoscan_ros2/cpp/holoscan/ros2/bridge.hpp` | `Bridge` resource — manages `rclcpp::Node` |

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Host Device | NVIDIA AGX Thor |
| JetPack | 7.0 (LinuxReceiverOp path) |
| HSB | 2.5.0 (`demo_HSB_ToF_ROS2` branch) |
| Holoscan SDK | 3.9.0 |
| holohub | `adcam_ros2` branch (tag `holoscan-sdk-3.9.0`) |
| IBV device | NOT required (LinuxReceiverOp auto-selected on JP 7.0) |
| HSB board | Hololink FPGA @ `192.168.0.2` |
| ToF sensor | ADI ADTF3175 / ADSD3100 connected via HSB |
