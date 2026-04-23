<!--
====================================================================
  COVER IMAGE PLACEHOLDER
  Drop your cover image here. Recommended size: 1600 x 500 px.
  Example:
  ![Cover](docs/cover.png)
====================================================================
-->

<p align="center">
  <!-- Replace the line below with your cover image -->
  <img src="docs/cover.png" alt="Ouster LiDAR SLAM Workflow" width="100%"/>
</p>

<br/>

# Ouster LiDAR End-to-End Workflow
### Network Setup, Firmware Update, SDK Installation, PCAP Recording, Offline & Online SLAM, and BIM-Ready Point Cloud Export

A complete, beginner-friendly, step-by-step tutorial for working with **Ouster OS1 and OS0 LiDAR sensors** on **Ubuntu 22.04** using the **Ouster SDK**.

This guide covers the full pipeline — from connecting the sensor for the very first time to generating a BIM-ready 3D point cloud. It is designed to work transparently across multiple robotic platforms (e.g. **Clearpath Jackal**, **Boston Dynamics Spot**, or a standalone desktop) by relying on the sensor hostname rather than fragile static IP addresses.

> Target use cases: robotics, indoor/outdoor mapping, digital twins, nuclear-environment surveys, and any workflow that needs a reliable offline LiDAR pipeline before — or instead of — moving to ROS 2.

---

## Table of Contents

1. [Tested Configuration](#tested-configuration)
2. [Workflow Overview](#workflow-overview)
3. [Step 1 — Connect to the Sensor (Network Setup)](#step-1--connect-to-the-sensor-network-setup)
4. [Step 2 — Update the Firmware](#step-2--update-the-firmware)
5. [Step 3 — Install the Ouster SDK](#step-3--install-the-ouster-sdk)
6. [Step 4 — Retrieve Sensor Metadata](#step-4--retrieve-sensor-metadata)
7. [Step 5 — Record a PCAP Dataset](#step-5--record-a-pcap-dataset)
8. [Step 6 — Run SLAM (Offline and Online)](#step-6--run-slam-offline-and-online)
9. [Step 7 — Visualize the Reconstructed Map](#step-7--visualize-the-reconstructed-map)
10. [Step 8 — Export a BIM-Ready Point Cloud](#step-8--export-a-bim-ready-point-cloud)
11. [Recommended Settings Summary](#recommended-settings-summary)
12. [Monitor System Resources](#monitor-system-resources)
13. [Project Structure](#project-structure)
14. [Official Resources](#official-resources)
15. [License](#license)

---

## Tested Configuration

### Sensors
- **Ouster OS1 — Rev6 / 64 channels**
- **Ouster OS0 — Rev7 / 128 channels**

### Host system
- **Ubuntu 22.04 LTS**
- **Python 3.8 – 3.13**
- **Ouster SDK 0.16.1**
- (Optional) **ROS 2 Humble** for downstream integration

> Throughout this guide, replace `<SENSOR_HOSTNAME>` with the hostname of your sensor (see Step 1). Anywhere you see `os-XXXXXXXXXXXX.local`, substitute your own 12-digit serial number.

---

## Workflow Overview

```text
Network Setup → Firmware → SDK → Metadata → PCAP → SLAM → OSF → Visualization → PLY Export
```

1. Connect to the sensor (IP vs. hostname)
2. Update the firmware
3. Install and verify the Ouster SDK
4. Retrieve sensor metadata
5. Record a `.pcap` dataset
6. Run SLAM (offline from PCAP or online from the live sensor)
7. Visualize the result
8. Export a `.ply` point cloud for BIM / digital twin workflows

---

## Step 1 — Connect to the Sensor (Network Setup)

The most common pitfall when moving a LiDAR between robots (for example a **Clearpath Jackal** on the `192.168.131.x` subnet vs. a **Boston Dynamics Spot** on the `192.168.50.x` subnet) is an IP address mismatch. This step explains how to avoid that problem entirely.

### 1.1 Default factory configuration (DHCP)

Out of the box, Ouster sensors **do not have a static IP address**. They are configured as **DHCP clients**:

- If connected to a DHCP server (a router, a Spot payload port, the Jackal internal switch…), they receive a dynamic IP.
- If connected **directly to a PC** without a DHCP server, they fall back to a **Link-Local** address in the `169.254.x.x` range.

### 1.2 The bulletproof method — use the sensor hostname

Ouster sensors continuously advertise their hostname on the local network via **mDNS** (Bonjour / Avahi). This means you can reach the sensor **by name**, no matter which subnet it landed on.

The hostname is always:

```text
os-<12_DIGIT_SERIAL_NUMBER>.local
```

> The 12-digit serial number is printed on a label on top of the sensor.

**Example (placeholder — replace with your own serial):**

```text
os-XXXXXXXXXXXX.local
```

**Test the connection from your Ubuntu PC:**

```bash
ping -4 os-XXXXXXXXXXXX.local
```

If the ping fails on a **direct point-to-point** connection, open your Ubuntu network settings and set the wired IPv4 configuration to **"Link-Local Only"** (instead of "Automatic (DHCP)"). Then replug the cable and retry.

> On Ubuntu, mDNS resolution requires `avahi-daemon`, which is installed and enabled by default on desktop images. If needed: `sudo apt install avahi-daemon libnss-mdns`.

### 1.3 Recommended strategy for multi-robot labs

**Do not set a static IP** if you frequently move the sensor between a Jackal, a Spot, and a desktop workstation. Keep the sensor in DHCP mode and **always use the hostname** in your commands and launch files. Your ROS 2 and `ouster-cli` workflows will then work on every platform without modification.

### 1.4 Open the sensor web interface

Once the ping succeeds, open a browser and navigate to:

```text
http://os-XXXXXXXXXXXX.local
```

You will land on Ouster's built-in dashboard showing firmware version, temperature, network configuration, and a live preview.

---

## Step 2 — Update the Firmware

Before collecting data, verify that the sensor is running the correct firmware family for its hardware revision.

### 2.1 Check the current firmware version

From the web interface (Step 1.4), the firmware version is shown on the dashboard. From the terminal:

```bash
curl "http://os-XXXXXXXXXXXX.local/api/v1/system/firmware" | python3 -m json.tool
```

### 2.2 Recommended firmware families

| Sensor | Hardware revision | Recommended firmware family |
|--------|-------------------|-----------------------------|
| OS1    | Rev6              | 2.x                         |
| OS1    | Rev7              | 3.x                         |
| OS0    | Rev7              | 3.x                         |

> **Important:** never mix Rev6 and Rev7 firmware families — each hardware revision has its own firmware lineage.

### 2.3 Download the firmware

Download the correct firmware package from the official Ouster downloads page:
[https://ouster.com/downloads](https://ouster.com/downloads)

### 2.4 Update via the web interface (recommended method)

This is the **official, supported** way to flash an Ouster sensor:

1. Open `http://os-XXXXXXXXXXXX.local`
2. Go to **Settings**
3. Open **Firmware Update**
4. Upload the `.img` file you downloaded
5. Wait for the sensor to reboot (≈ 1–2 minutes)
6. Refresh the page and confirm the new version

> ⚠️ **Do not power-cycle the sensor during the update.** A failed flash can leave the device in recovery mode.

Compatibility note: as of Ouster SDK 0.13.0 and later, the SDK is no longer compatible with firmware versions older than **2.1.0**. Official support for firmware 2.2 and 2.3 ended in June 2025.

---

## Step 3 — Install the Ouster SDK

The recommended installation uses a dedicated Python virtual environment.

### 3.1 Create a working directory and a virtual environment

```bash
mkdir -p ~/ouster_project
cd ~/ouster_project

python3 -m venv .venv
source .venv/bin/activate
```

### 3.2 Upgrade pip and install the SDK

```bash
python3 -m pip install --upgrade pip setuptools
python3 -m pip install "ouster-sdk[examples]"
```

### 3.3 Verify the installation

```bash
python3 -m pip show ouster-sdk
python3 -m pip list | grep ouster
ouster-cli --help
```

You should see Ouster SDK `0.16.1` (or newer) and a working `ouster-cli` entry point.

> Requirements: Python **3.8 – 3.13**, pip **≥ 19.0**, and `glibc ≥ 2.28` on Linux.

---

## Step 4 — Retrieve Sensor Metadata

The sensor metadata JSON file is **required** when replaying a PCAP offline. It describes the beam angles, lidar mode, UDP profile, intrinsic calibration, etc.

### 4.1 Download the metadata

```bash
curl "http://os-XXXXXXXXXXXX.local/api/v1/sensor/metadata" -o sensor_metadata.json
```

### 4.2 Inspect the metadata

```bash
cat sensor_metadata.json | python3 -m json.tool | head -40
```

Or open it directly in a browser:

```text
http://os-XXXXXXXXXXXX.local/api/v1/sensor/metadata
```

---

## Step 5 — Record a PCAP Dataset

Use `ouster-cli` to record raw packets from the live sensor. Note that we use the **hostname** — not a fragile IP address.

### 5.1 Start a recording

```bash
ouster-cli source os-XXXXXXXXXXXX.local record capture.pcap
```

This produces two files in the current directory:

- `capture.pcap` — raw LiDAR + IMU packet stream
- `capture.json` — sensor metadata required for offline playback

Press **Ctrl+C** to stop the recording.

### 5.2 Practical recommendations

- Move the robot **slowly and smoothly** during acquisition — avoid sharp yaws.
- Prefer trajectories with **clear loop overlap** for better SLAM convergence.
- Use **Gigabit Ethernet**, especially for the OS0-128 (higher packet rate).
- Monitor CPU, RAM, and disk I/O during recording (see [Section 12](#monitor-system-resources)).

---

## Step 6 — Run SLAM (Offline and Online)

The `ouster-cli ... slam` command uses the built-in **KISS-ICP** algorithm to estimate the trajectory and fuse the point clouds into a globally consistent map.

Two input sources are supported:

| Mode    | Source                        | Typical use case                           |
|---------|-------------------------------|--------------------------------------------|
| Offline | `capture.pcap` (+ metadata)   | Post-processing a recorded dataset         |
| Online  | `os-XXXXXXXXXXXX.local` (live) | Real-time preview during data acquisition |

### 6.1 Parameter overview

| Parameter                   | Purpose                                                             |
|-----------------------------|---------------------------------------------------------------------|
| `--voxel-size`              | Controls map resolution and computational load (see table below)    |
| `filter RANGE X:Y`          | Removes points outside the useful range                             |
| `filter REFLECTIVITY 1:1`   | Helps suppress unstable reflections on glass or shiny surfaces      |

**Voxel-size rules of thumb** (from Ouster's official guidance):

- **Outdoor:** `1.4 – 2.2`
- **Large indoor:** `1.0 – 1.8`
- **Small indoor:** `0.4 – 0.8`
- **Best possible detail** (cost: more CPU/RAM): `0.25 – 0.5`

> Since SDK 0.16, `ouster-cli` can **auto-calculate** the voxel size if you omit `--voxel-size`. It is often a good starting point; set a manual value only when you need tighter control.

---

### 6.2 Offline SLAM — OS1-64 (Rev6)

**Indoor mapping:**

```bash
ouster-cli source capture.pcap slam \
  --voxel-size 0.25 \
  filter RANGE 0.5m:20m \
  filter REFLECTIVITY 1:1 \
  save map_os1_indoor.osf
```

**Outdoor mapping:**

```bash
ouster-cli source capture.pcap slam \
  --voxel-size 0.8 \
  filter RANGE 1.0m:100m \
  filter REFLECTIVITY 1:1 \
  save map_os1_outdoor.osf
```

---

### 6.3 Offline SLAM — OS0-128 (Rev7)

**Indoor mapping:**

```bash
ouster-cli source capture.pcap slam \
  --voxel-size 0.30 \
  filter RANGE 0.3m:20m \
  filter REFLECTIVITY 1:1 \
  save map_os0_indoor.osf
```

**Outdoor mapping:**

```bash
ouster-cli source capture.pcap slam \
  --voxel-size 1.0 \
  filter RANGE 1.0m:50m \
  filter REFLECTIVITY 1:1 \
  save map_os0_outdoor.osf
```

---

### 6.4 Replay a PCAP with an explicit metadata file

If the metadata JSON was not generated automatically, pass it with `--meta`:

```bash
ouster-cli source --meta sensor_metadata.json capture.pcap slam \
  --voxel-size 0.25 \
  filter RANGE 0.5m:20m \
  filter REFLECTIVITY 1:1 \
  save map.osf
```

---

### 6.5 Online SLAM (live sensor)

You can also run SLAM **directly on the live stream** — useful for quick field checks:

```bash
ouster-cli source os-XXXXXXXXXXXX.local slam viz --accum-num 100
```

Add filters and saving just like the offline case:

```bash
ouster-cli source os-XXXXXXXXXXXX.local slam \
  --voxel-size 0.25 \
  filter RANGE 0.5m:20m \
  filter REFLECTIVITY 1:1 \
  viz --accum-num 100 \
  save live_map.osf
```

Press **Ctrl+C** (or close the viewer) to stop.

---

## Step 7 — Visualize the Reconstructed Map

Before exporting, validate the trajectory and map quality:

```bash
ouster-cli source map_os0_indoor.osf viz \
  --accum-num 20 \
  --accum-every-m 2.0 \
  --map \
  -e stop
```

**Why this configuration:** it provides a clear global view of the reconstructed environment without overloading the GPU or system memory.

> Avoid aggressive settings such as very large `--map-size` or high `--map-ratio` values unless you specifically need them for close inspection and your workstation has sufficient graphics memory.

---

## Step 8 — Export a BIM-Ready Point Cloud

Once the SLAM result is validated, export the map as a `.ply` point cloud:

```bash
ouster-cli source map_os0_indoor.osf convert final_map.ply
```

The resulting file can be opened or post-processed in:

- **CloudCompare**
- **Open3D**
- **MeshLab**
- **Blender**
- Any BIM / digital twin processing pipeline (Revit, Navisworks, Autodesk ReCap, etc.)

---

## Recommended Settings Summary

| Sensor   | Environment | Voxel size | Range filter   | Expected result                        |
|----------|-------------|------------|----------------|----------------------------------------|
| OS1-64   | Indoor      | 0.25       | 0.5 m – 20 m   | High-detail mapping                    |
| OS1-64   | Outdoor     | 0.80       | 1.0 m – 100 m  | Stable large-scale mapping             |
| OS0-128  | Indoor      | 0.30       | 0.3 m – 20 m   | Dense indoor reconstruction            |
| OS0-128  | Outdoor     | 1.00       | 1.0 m – 50 m   | Efficient short-range outdoor mapping  |

---

## Monitor System Resources

SLAM can be computationally heavy, especially with dense OS0-128 datasets.

```bash
btop
# or
htop
```

Watch closely:

- **RAM usage**
- **Swap usage**
- **CPU saturation**
- **Disk throughput**

> If swap usage starts growing significantly, stop the process (**Ctrl+C**) and rerun with a **larger `--voxel-size`**.

---

## Project Structure

A suggested layout for a clean working directory:

```text
ouster_project/
├── .venv/
├── captures/
│   ├── capture.pcap
│   └── capture.json
├── maps/
│   ├── map_os1_indoor.osf
│   └── map_os0_outdoor.osf
└── exports/
    └── final_map.ply
```

---

## Output Files

| Extension | Description                                        |
|-----------|----------------------------------------------------|
| `.pcap`   | Raw LiDAR packet capture                           |
| `.json`   | Sensor metadata                                    |
| `.osf`    | SLAM result with trajectory and fused point cloud  |
| `.ply`    | Exported point cloud for BIM / post-processing     |

---

## Official Resources

- [Ouster Downloads](https://ouster.com/downloads)
- [Ouster SDK Documentation](https://static.ouster.dev/sdk-docs/)
- [Ouster Sensor Documentation](https://static.ouster.dev/sensor-docs/)
- [Ouster HTTP API Reference](https://static.ouster.dev/sensor-docs/image_route1/image_route2/common_sections/API/http-api-v1.html)
- [Ouster Community Forum](https://community.ouster.com/)
- [Optimal Sensor and SLAM Configuration for "Crisp" Mapping — Ouster Blog](https://ouster.com/insights/blog/sensor-config-for-crisp-mapping)

---

## License

This tutorial is provided as a technical reference for robotics, mapping, and digital twin workflows using Ouster LiDAR sensors and the Ouster SDK. Feel free to adapt it to your own projects — attribution appreciated.
