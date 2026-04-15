# ros2_unity

Stream ROS2 point cloud data from Ubuntu to Unity on Windows over WebSocket.

Two bridge approaches are documented here — both have been tested and confirmed working on this machine with ROS2 Jazzy:

| Approach | Port | Install method | Notes |
|---|---|---|---|
| [rosbridge_suite (built from source)](#approach-a--rosbridge-suite-built-from-source) | 9090 | `colcon build` in workspace | Required because the apt binary has a FastCDR 1.x vs 2.x ABI mismatch on Jazzy |
| [foxglove-bridge](#approach-b--foxglove-bridge) | 8765 | `sudo apt install` | Single apt install, simpler setup |

---

## Architecture

```
Ubuntu (ROS2 Jazzy)                         Windows (Unity)
─────────────────────────────────           ─────────────────────────────
ros2 bag play → /velodyne_points            ROS# library (WebSocket client)
         ↕                                           ↕
  rosbridge_websocket node              subscribes to /velodyne_points_throttled
  — or —                                parses PointCloud2 → renders in Unity
  foxglove_bridge node
  (WebSocket server)
         ↕
  topic_tools throttle
  (rate-limits to 2 Hz to reduce bandwidth)
```

---

## Common Prerequisites

### Ubuntu

```bash
sudo apt install ros-jazzy-topic-tools
```

### Windows / Unity

- Unity 2021.3 LTS or newer
- **ROS#** library — download the `.unitypackage` from the latest release:
  https://github.com/siemens/ros-sharp/releases

---

## Approach A — rosbridge_suite (built from source)

### Why build from source?

The apt package `ros-jazzy-rosbridge-suite` was compiled against FastCDR 1.x. ROS Jazzy ships
FastCDR 2.x — the symbol `Cdr::serialize(unsigned int)` was removed in 2.x, causing a fatal
symbol lookup error at launch. Building from source compiles against the FastCDR version
actually installed on the system.

### One-time build

```bash
cd ~/projects/RMC_2.0/ros2_ws/src
git clone https://github.com/RobotWebTools/rosbridge_suite.git
cd ~/projects/RMC_2.0/ros2_ws
colcon build --packages-select rosbridge_msgs rosbridge_library rosapi rosapi_msgs rosbridge_server --symlink-install
```

### Run

```bash
# Terminal 1 — source workspace and start bridge
source /opt/ros/jazzy/setup.bash
source ~/projects/RMC_2.0/ros2_ws/install/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
# WebSocket server starts on port 9090
```

Allow through firewall (one-time):

```bash
sudo ufw allow 9090/tcp
```

### Unity connection URL

```
ws://<ubuntu-ip>:9090
```

---

## Approach B — foxglove-bridge

### Install

```bash
sudo apt install ros-jazzy-foxglove-bridge
```

### Run

```bash
# Terminal 1
source /opt/ros/jazzy/setup.bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
# WebSocket server starts on port 8765
```

Allow through firewall (one-time):

```bash
sudo ufw allow 8765/tcp
```

### Unity connection URL

```
ws://<ubuntu-ip>:8765
```

---

## Ubuntu — Bag Playback and Throttle (same for both approaches)

### Find your Ubuntu LAN IP

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

Example: `inet 192.168.1.50/24` → IP is `192.168.1.50`

### Play the bag

```bash
# Terminal 2
source /opt/ros/jazzy/setup.bash
ros2 bag play ~/projects/RMC_2.0/ros2_ws/src/ros2_point_cloud/ros2_bags/record_2025-03-19-09-26-37_0 --clock
```

### Throttle the point cloud (recommended)

Full VLP-16 at 10 Hz is heavy over WebSocket. Throttle to 2 Hz:

```bash
# Terminal 3
source /opt/ros/jazzy/setup.bash
ros2 run topic_tools throttle messages /velodyne_points 2.0 /velodyne_points_throttled
```

Unity subscribes to `/velodyne_points_throttled`. Increase the rate if your network allows it.

---

## Unity Setup (Windows) — same for both approaches

### 1. Import ROS#

1. Download `RosSharp.unitypackage` from https://github.com/siemens/ros-sharp/releases
2. In Unity: **Assets → Import Package → Custom Package** → select the file
3. Accept all default imports

### 2. Configure the ROS connection

Create a `RosConnector` GameObject:

1. **GameObject → Create Empty**, name it `RosConnector`
2. Add component: **RosConnector**
3. Set **Ros Bridge Server Url**:
   - rosbridge: `ws://192.168.1.50:9090`
   - foxglove:  `ws://192.168.1.50:8765`
4. Set **Protocol** to `Web Socket Sharp`

### 3. Subscribe to the point cloud topic

Create a new C# script `PointCloudSubscriber.cs` and attach it to a GameObject:

```csharp
using UnityEngine;
using RosSharp.RosBridgeClient;
using RosSharp.RosBridgeClient.MessageTypes.Sensor;

public class PointCloudSubscriber : MonoBehaviour
{
    private RosConnector rosConnector;
    private ParticleSystem pointCloudParticles;
    private PointCloud2 latestMsg;
    private bool newMessageReceived = false;
    private readonly object msgLock = new object();

    void Start()
    {
        rosConnector = FindObjectOfType<RosConnector>();
        pointCloudParticles = GetComponent<ParticleSystem>();

        rosConnector.RosSocket.Subscribe<PointCloud2>(
            "/velodyne_points_throttled",
            OnPointCloudReceived
        );
    }

    void OnPointCloudReceived(PointCloud2 msg)
    {
        lock (msgLock)
        {
            latestMsg = msg;
            newMessageReceived = true;
        }
    }

    void Update()
    {
        lock (msgLock)
        {
            if (!newMessageReceived) return;
            newMessageReceived = false;
            RenderPointCloud(latestMsg);
        }
    }

    void RenderPointCloud(PointCloud2 msg)
    {
        // PointCloud2 field offsets for Velodyne VLP-16 (x, y, z as float32)
        int pointStep = msg.point_step;   // bytes per point (typically 22 for VLP-16)
        int numPoints = (int)(msg.width * msg.height);
        byte[] data = msg.data;

        var particles = new ParticleSystem.Particle[numPoints];
        int xOffset = 0;   // x field at byte 0
        int yOffset = 4;   // y field at byte 4
        int zOffset = 8;   // z field at byte 8

        for (int i = 0; i < numPoints; i++)
        {
            int baseIdx = i * pointStep;
            float x = System.BitConverter.ToSingle(data, baseIdx + xOffset);
            float y = System.BitConverter.ToSingle(data, baseIdx + yOffset);
            float z = System.BitConverter.ToSingle(data, baseIdx + zOffset);

            // ROS uses X-forward, Y-left, Z-up — Unity uses X-right, Y-up, Z-forward
            particles[i].position = new Vector3(-y, z, x);
            particles[i].startSize = 0.05f;
            particles[i].startColor = Color.white;
        }

        pointCloudParticles.SetParticles(particles, numPoints);
    }
}
```

### 4. Set up the ParticleSystem for rendering

1. Create a **GameObject → Effects → Particle System**
2. Attach the `PointCloudSubscriber.cs` script to it
3. Configure the Particle System:
   - **Emission → Rate over Time**: `0` (script drives it, not the emitter)
   - **Renderer → Render Mode**: `Billboard` or `Point`
   - **Max Particles**: `200000` (VLP-16 outputs ~70k points per scan)
   - **Start Lifetime**: `0.6` (slightly longer than the 0.5s throttle period so points persist between frames)
   - **Simulation Space**: `World`

---

## Field Offsets for Velodyne VLP-16

The `PointCloud2` binary layout for Velodyne data is:

| Field | Offset | Type | Size |
|---|---|---|---|
| x | 0 | float32 | 4 bytes |
| y | 4 | float32 | 4 bytes |
| z | 8 | float32 | 4 bytes |
| intensity | 12 | float32 | 4 bytes |
| ring | 16 | uint16 | 2 bytes |
| time | 18 | float32 | 4 bytes |
| **Total** | | | **22 bytes/point** |

Verify the offsets for your specific bag with:

```bash
ros2 topic echo /velodyne_points --once | grep -A 30 "fields:"
```

---

## Coordinate Frame Conversion

ROS and Unity use different coordinate systems:

| Axis | ROS | Unity |
|---|---|---|
| Forward | +X | +Z |
| Left | +Y | -X |
| Up | +Z | +Y |

Conversion applied in the script:
```
Unity.x = -ROS.y
Unity.y =  ROS.z
Unity.z =  ROS.x
```

---

## Troubleshooting

| Issue | Fix |
|---|---|
| `symbol lookup error: undefined symbol Cdr::serialize` | Do not use apt rosbridge — build from source (see Approach A) |
| Unity cannot connect | Check Ubuntu IP, correct port open in firewall, both on same LAN |
| No points appear in Unity | Verify bag is playing: `ros2 topic hz /velodyne_points_throttled` |
| Point cloud appears rotated/flipped | Check coordinate frame conversion in `RenderPointCloud()` |
| High latency / dropped frames | Reduce throttle rate: change `2.0` to `1.0` Hz |
| rosbridge launch not found after build | Run `source ~/projects/RMC_2.0/ros2_ws/install/setup.bash` first |
| topic_tools not found | `sudo apt install ros-jazzy-topic-tools` |

---

## Quick Start — Approach A (rosbridge from source)

```bash
# Terminal 1 — bridge server
source /opt/ros/jazzy/setup.bash
source ~/projects/RMC_2.0/ros2_ws/install/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# Terminal 2 — bag playback
source /opt/ros/jazzy/setup.bash
ros2 bag play ~/projects/RMC_2.0/ros2_ws/src/ros2_point_cloud/ros2_bags/record_2025-03-19-09-26-37_0 --clock

# Terminal 3 — throttle
source /opt/ros/jazzy/setup.bash
ros2 run topic_tools throttle messages /velodyne_points 2.0 /velodyne_points_throttled
```

Unity URL: `ws://<ubuntu-ip>:9090`

## Quick Start — Approach B (foxglove-bridge)

```bash
# Terminal 1 — bridge server
source /opt/ros/jazzy/setup.bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml

# Terminal 2 — bag playback
source /opt/ros/jazzy/setup.bash
ros2 bag play ~/projects/RMC_2.0/ros2_ws/src/ros2_point_cloud/ros2_bags/record_2025-03-19-09-26-37_0 --clock

# Terminal 3 — throttle
source /opt/ros/jazzy/setup.bash
ros2 run topic_tools throttle messages /velodyne_points 2.0 /velodyne_points_throttled
```

Unity URL: `ws://<ubuntu-ip>:8765`
