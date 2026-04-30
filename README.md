# NexusSense

> Bitstream Intelligence Engine. **Compressed-domain analytics** on H.264/H.265 — motion / health / scene detection straight from NAL units, no decode to RGB. **56 KB**, **33,000 NAL/s** per core, ~0% CPU.

[![status](https://img.shields.io/badge/status-production-15803d?style=flat-square)](#)
[![size](https://img.shields.io/badge/library-56%20KB-1d4ed8?style=flat-square)](#)
[![throughput](https://img.shields.io/badge/throughput-33k%20NAL%2Fs-7c3aed?style=flat-square)](#)
[![cpu](https://img.shields.io/badge/CPU-~0%25-15803d?style=flat-square)](#)
[![lang](https://img.shields.io/badge/lang-C99-475569?style=flat-square)](#)

NexusSense parses the H.264/H.265 bitstream structure and extracts events **without decoding to pixels**. For an NVR running dozens of cameras this means: motion detection, freeze/blackout/scene-change alerts come essentially for free relative to a full decode pipeline.

Part of the [NexusEye](https://github.com/facex-engine) stack.

---

## Where NexusSense lives in the architecture

![analytics tiers](docs/analytics-tiers.svg)

NexusSense is **Tier 0**. When a camera has no built-in AI on its NPU (Tier 2 — Hikvision/Dahua/Axis edge events), and pixel-domain CV (Tier 1) is too expensive for your workload, NexusSense catches motion straight from the bitstream.

## Real-world output

![events flow](docs/events-flow.svg)

```
$ ./nexussense-bench samples/motion_event.h264
nexussense-bench · file=samples/motion_event.h264 · size=4.02 MB
  [t=2333310 us] MOTION_START (0x0100)  conf=0.90  intensity=0.144
  [t=2466642 us] MOTION_STOP  (0x0101)  conf=0.90
  [t=3266634 us] MOTION_START (0x0100)  conf=0.90  intensity=0.117

Processed: 116 NAL units (102 frame events)  in 0.003 s = 33,289 NAL/s
Events emitted:
  motion_start: 2
  motion_stop:  1
```

---

## Capabilities

| Module          | Flag              | What it detects                                       |
|-----------------|-------------------|-------------------------------------------------------|
| Motion          | `BIE_MOD_MOTION`  | size-spike detector on P-frames, EWMA baseline        |
| Tracker         | `BIE_MOD_TRACKER` | object tracking via motion vectors (CABAC walker)     |
| Health          | `BIE_MOD_HEALTH`  | freeze · blackout · whiteout · scene-change · framedrop |
| Forensic        | `BIE_MOD_FORENSIC`| double-compression, GOP breaks, SPS changes           |
| Scene           | `BIE_MOD_SCENE`   | indoor/outdoor · day/night · activity level           |
| Privacy         | `BIE_MOD_PRIVACY` | DCT-domain redaction (no full re-encode)              |

All modules are toggled by a bitmask on `bie_stream_open()` — keep only what you need.

---

## API

```c
#include "bie.h"

/* 1. Initialize */
bie_config_t cfg = bie_config_default();
bie_engine_t* eng = bie_create(&cfg);

bie_set_callback(eng, on_event, /* user_data */ NULL);

/* 2. Open a stream (one engine can manage N streams from different cameras) */
int modules = BIE_MOD_MOTION | BIE_MOD_HEALTH | BIE_MOD_SCENE;
bie_stream_t* str = bie_stream_open(eng, "cam-225", BIE_CODEC_H264, modules);

/* 3. Feed NAL units as they arrive */
while (read_nal(rtsp_session, nal_data, &nal_len)) {
    int r = bie_process_nalu(str, nal_data, nal_len, pts_us);
    if (r > 0) bie_run_modules(eng, str);    /* run analytics on the frame */
}

/* 4. Cleanup */
bie_stream_close(eng, str);
bie_destroy(eng);

/* Callback */
void on_event(bie_event_t* ev, void* user) {
    if (ev->type == BIE_EV_MOTION_START)
        printf("motion! intensity=%.2f conf=%.2f\n",
               ev->data.motion.intensity, ev->confidence);
}
```

---

## Events

```c
typedef enum {
    /* Motion */
    BIE_EV_MOTION_START     = 0x0100,
    BIE_EV_MOTION_STOP      = 0x0101,
    BIE_EV_MOTION_LEVEL     = 0x0102,   /* periodic intensity update */

    /* Object tracker */
    BIE_EV_TRACK_NEW        = 0x0200,
    BIE_EV_TRACK_UPDATE     = 0x0201,
    BIE_EV_TRACK_LOST       = 0x0202,
    BIE_EV_TRACK_LINE_CROSS = 0x0203,

    /* Camera health */
    BIE_EV_HEALTH_BLACKOUT  = 0x0300,
    BIE_EV_HEALTH_WHITEOUT  = 0x0301,
    BIE_EV_HEALTH_FREEZE    = 0x0302,
    BIE_EV_HEALTH_BLUR      = 0x0303,
    BIE_EV_HEALTH_SCENE_CHG = 0x0304,
    BIE_EV_HEALTH_BITRATE   = 0x0305,
    BIE_EV_HEALTH_FRAMEDROP = 0x0306,
    BIE_EV_HEALTH_DEAD      = 0x0307,
    BIE_EV_HEALTH_RECOVER   = 0x0308,

    /* Forensic */
    BIE_EV_FORENSIC_DOUBLE_COMPRESS = 0x0400,
    BIE_EV_FORENSIC_FRAME_GAP       = 0x0401,
    BIE_EV_FORENSIC_GOP_BREAK       = 0x0402,
    BIE_EV_FORENSIC_SPS_CHANGE      = 0x0403,

    /* Scene */
    BIE_EV_SCENE_CLASS              = 0x0500,
} bie_event_type_t;
```

Every event carries `confidence` (0.0–1.0), `timestamp_us`, and a type-specific payload (intensity for motion, baseline+metric for health, etc.).

---

## Sensitivity tuning

```c
typedef struct {
    float motion_threshold_k;        /* default 3.0  — sigma multiplier */
    float motion_ewma_alpha;         /* default 0.02 — adaptation rate */
    int   motion_min_frames;         /* default 3    — warm-up frames */
    int   health_freeze_frames;      /* default 30   — freeze alert delay */
    float health_blackout_ratio;     /* default 0.3  — IDR shrink ratio */
    float health_scene_chg_ratio;    /* default 3.0  — IDR spike ratio */
    int   health_dead_timeout_ms;    /* default 10000 — no-frame alert */
    int   tracker_min_hits;          /* default 3 */
    int   tracker_max_age;           /* default 5 */
    int   tracker_min_area_mb;       /* default 4 — minimum blob in macroblocks */
} bie_config_t;
```

All thresholds are adaptive (EWMA — Exponentially Weighted Moving Average) and self-tune to the scene character. The bundled `bench.c` ships with more sensitive demo values for short clips.

---

## Quick start

### Linux x86-64 (prebuilt binary)

```bash
wget https://github.com/facex-engine/nexussense/releases/latest/download/nexussense-linux-x64.tar.gz
tar xzf nexussense-linux-x64.tar.gz
cd nexussense
./nexussense-bench samples/motion_event.h264
```

### Build from source

```bash
git clone https://github.com/facex-engine/nexussense
cd nexussense
make            # → libnexussense.a (56 KB)
make bench      # → nexussense-bench
```

Requirements: `gcc ≥ 9` or `clang`, `make`, glibc.

---

## Performance

| Sample              | Size      | NAL units | Wall-clock | Throughput      |
|---------------------|-----------|-----------|------------|-----------------|
| `test_idr.h264`     | 339 KB    | 5         | < 1 ms     | —               |
| `motion_event.h264` | 4.02 MB   | 116       | 0.003 s    | **33,289 NAL/s** |

In a real workload (50 cameras @ 25 fps × ~25 NAL/frame ≈ ~31,000 NAL/s) NexusSense covers everything on one core with headroom to spare.

---

## License

MIT — see [LICENSE](LICENSE).

## Author

**Baurzhan Atynov** ([@bauratynov](https://github.com/bauratynov))

## Part of the NexusEye stack

| Component | What it does | Size |
|-----------|--------------|------|
| [NexusDecode](https://github.com/facex-engine/nexusdecode) | H.264 decode without FFmpeg | 497 KB |
| [NexusInfer](https://github.com/facex-engine/nexusinfer) | YOLO inference on CPU, ONNX Runtime replacement | 159 KB |
| **NexusSense** *(you are here)* | Compressed-domain analytics | 56 KB |
| [FaceX](https://github.com/facex-engine/facex) | Face embedding INT8 on CPU | 180 KB |
