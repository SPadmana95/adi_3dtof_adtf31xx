# ADCam ROS 2 Publisher / Subscriber — Design and Code Explanation

Source: `holohub/applications/holoscan_ros2/aditof/cpp/`

---

## Overview

The `adcam_ros2_publisher.cpp` and `adcam_ros2_subscriber.cpp` are derived from `adcam_player.cpp` (the original standalone ADI ToF player in `holoscan-sensor-bridge/examples/aditof/cpp/`).

The original `adcam_player.cpp` captures frames from the ADI ADTF3175 sensor and displays them directly using HolovizOp in a single application. The ROS 2 versions split this into two separate applications:

- **Publisher** — captures sensor data and publishes to ROS 2 topics
- **Subscriber** — subscribes to those topics and displays using HolovizOp

---

## Structural Comparison

```
adcam_player.cpp              adcam_ros2_publisher.cpp      adcam_ros2_subscriber.cpp
─────────────────             ────────────────────────      ─────────────────────────
Single self-contained app     Publisher half                 Subscriber half
(capture + display)           (capture only)                 (display only)
```

---

## Pipeline Comparison

| Step | `adcam_player.cpp` | `adcam_ros2_publisher.cpp` | `adcam_ros2_subscriber.cpp` |
|---|---|---|---|
| Sensor connection | ✅ LinuxReceiverOp / RoceReceiverOp | ✅ Same | ❌ Not needed |
| `CsiToBayerOp` | ✅ | ✅ Same | ❌ Not needed |
| `ADTFUnpackOp` | ✅ | ✅ Same | ❌ Not needed |
| `HolovizOp` | ✅ — displays directly | ❌ Removed | ✅ — displays received data |
| ROS 2 Bridge | ❌ None | ✅ `holoscan::ros2::Bridge` | ✅ `holoscan::ros2::Bridge` |
| Publisher | ❌ None | ✅ `AdiTofPublisherOp` × 3 topics | ❌ None |
| Subscriber | ❌ None | ❌ None | ✅ `AdiTofSubscriberOp` × 3 topics |

---

## What Changed: `adcam_player.cpp` → `adcam_ros2_publisher.cpp`

**One operator replaced:** `HolovizOp` → `AdiTofPublisherOp`

```cpp
// adcam_player.cpp — ORIGINAL end of compose()
auto visualizer = make_operator<holoscan::ops::HolovizOp>("holoviz", ...);
add_flow(ADIToF_data, visualizer, {{"output", "receivers"}});

// adcam_ros2_publisher.cpp — REPLACED with:
auto ros2_bridge    = make_resource<holoscan::ros2::Bridge>(...);
auto ros2_publisher = make_operator<AdiTofPublisherOp>(...);
add_flow(adtf_unpack, ros2_publisher, {{"output", "input"}});
```

`AdiTofPublisherOp` does what HolovizOp would have done with the named tensors —
but instead of rendering to screen, it reads each named tensor
(`"Depth"`, `"ActiveBrightness"`, `"Conf"`), copies GPU → CPU,
wraps in `sensor_msgs/Image` (`rgb8`), and publishes to ROS 2.

---

## What `adcam_ros2_subscriber.cpp` Mirrors from `adcam_player.cpp`

The subscriber **exactly reproduces** the `adcam_player` HolovizOp display layout — same three panels, same tensor names, same viewport positions.

```cpp
// adcam_player.cpp HolovizOp setup:
HolovizOp::InputSpec left_spec   {"Depth",            HolovizOp::InputType::COLOR};
HolovizOp::InputSpec center_spec {"ActiveBrightness", HolovizOp::InputType::COLOR};
HolovizOp::InputSpec right_spec  {"Conf",             HolovizOp::InputType::COLOR};
left_spec.views_   = {{0.00f, 0.0f, 0.33f, 1.0f}};  // Left panel
center_spec.views_ = {{0.33f, 0.0f, 0.33f, 1.0f}};  // Center panel
right_spec.views_  = {{0.66f, 0.0f, 0.34f, 1.0f}};  // Right panel

// adcam_ros2_subscriber.cpp — IDENTICAL HolovizOp setup
// Feeds it from 3 ROS 2 subscribers instead of directly from ADTFUnpackOp
```

---

## Full Data Flow Diagram

```
adcam_player.cpp (all-in-one):
  LinuxReceiverOp
      → CsiToBayerOp
      → ADTFUnpackOp      (5-byte/pixel ADI format → Depth/AB/Conf RGB tensors)
      → HolovizOp         (Left: Depth | Center: AB | Right: Conf)

Split into two ROS 2 applications:

adcam_ros2_publisher.cpp:
  LinuxReceiverOp
      → CsiToBayerOp
      → ADTFUnpackOp
      → AdiTofPublisherOp
            ├─ /aditof/depth_image   (sensor_msgs/Image, rgb8, Jet colormap)
            ├─ /aditof/ab_image      (sensor_msgs/Image, rgb8, Grayscale AB)
            └─ /aditof/conf_image    (sensor_msgs/Image, rgb8, Grayscale Conf)
                        ↓ ROS 2 DDS
adcam_ros2_subscriber.cpp:
  AdiTofSubscriberOp ("Depth")            → subscribes /aditof/depth_image
  AdiTofSubscriberOp ("ActiveBrightness") → subscribes /aditof/ab_image
  AdiTofSubscriberOp ("Conf")             → subscribes /aditof/conf_image
      → HolovizOp   (Left: Depth | Center: AB | Right: Conf)
                     identical layout to adcam_player.cpp
```

---

## ADTFUnpackOp Output Format

`ADTFUnpackOp` unpacks the ADI 5-byte/pixel raw ToF data and outputs **3 separate named GXF tensors** — each already colorized as RGB for visualization:

| Tensor name | Shape | dtype | Content |
|---|---|---|---|
| `"Depth"` | `{H, W, 3}` | `uint8_t` | **Jet colormap** — depth distance |
| `"ActiveBrightness"` | `{H, W, 3}` | `uint8_t` | **Grayscale as RGB** — IR intensity (normalized to 4096) |
| `"Conf"` | `{H, W, 3}` | `uint8_t` | **Grayscale as RGB** — confidence score (normalized to 255) |

The publisher reads these GPU tensors directly, does `cudaMemcpy DeviceToHost`, and publishes as `rgb8` images. The subscriber receives `rgb8` images, does `cudaMemcpy HostToDevice`, and recreates matching GXF tensors for HolovizOp.

---

## Published ROS 2 Topics

| Topic | Type | Encoding | Content |
|---|---|---|---|
| `/aditof/depth_image` | `sensor_msgs/Image` | `rgb8` | Jet colormap depth visualization |
| `/aditof/ab_image` | `sensor_msgs/Image` | `rgb8` | Grayscale active brightness |
| `/aditof/conf_image` | `sensor_msgs/Image` | `rgb8` | Grayscale confidence |

---

## Subscriber Startup Order Handling

The subscriber handles both startup orderings:

| Scenario | Behaviour |
|---|---|
| **Subscriber starts before publisher** | `wait_for(500ms)` loop — waits silently until publisher sends |
| **Publisher starts before subscriber** | Message already in `message_queue_` — `receive()` resolves immediately |
| Publisher stops mid-run | 500ms retry loop resumes when publisher restarts |
| ROS 2 shutdown | `rclcpp::ok()` check exits `compute()` cleanly |

---

## Running

```sh
# Terminal 1 — Publisher (captures from ADI ADTF3175 sensor)
./holohub run aditof publisher --language cpp -- --resetAdcam 1

# Terminal 2 — Subscriber (displays in HolovizOp)
./holohub run aditof subscriber --language cpp

# Alternative — RViz2 (add 3 Image displays, set Reliability to "Reliable")
rviz2
# Add: /aditof/depth_image, /aditof/ab_image, /aditof/conf_image
```

---

## InfiniBand / RDMA Requirement

Unlike the `vb1940` example, **the `aditof` publisher does NOT require InfiniBand**.

| Example | Receiver | IBV required | Works on JP 7.0 |
|---|---|---|---|
| `vb1940_publisher` | `RoceReceiverOp` (RDMA) | ✅ Yes (JP 7.1+) | ❌ No |
| `adcam_ros2_publisher` | `LinuxReceiverOp` | ❌ No | ✅ Yes |

The publisher auto-detects IBV devices at startup:
- If IBV found → uses `RoceReceiverOp` (faster, RDMA)
- If no IBV found → falls back to `LinuxReceiverOp` (Linux network, works on JP 7.0)
