# H.264/DASH Rate-Distortion Curve Experiment

This repository contains an end-to-end video streaming experiment for analysing **H.264/AVC DASH-style representations** using rate-distortion curves. The project studies how bitrate and resolution affect objective video quality, and how these measurements can be used to design a practical adaptive streaming bitrate ladder.

The experiment was implemented in a Python notebook using FFmpeg, FFprobe, Pandas and Matplotlib.

## Project Overview

Adaptive streaming systems such as DASH deliver multiple encoded versions of the same video at different bitrates and resolutions. During playback, the client can switch between these representations depending on available bandwidth, buffer level and device capability.

In this project, I created multiple H.264 encoded representations of a 1920×1080 source clip and measured their rate-distortion behaviour.

The experiment covers three resolutions:

* 1920×1080
* 1280×720
* 854×480

For each resolution, I encoded the source clip at multiple target bitrates using single-pass CBR-style H.264 encoding. I then measured the actual bitrate and PSNR for each encoded output, plotted RD curves and computed the upper convex hull to identify efficient representation candidates.

## Objectives

The main objectives of this project were:

* Generate DASH-style H.264 video representations at multiple resolutions.
* Use single-pass CBR encoding for each target bitrate.
* Measure actual achieved bitrate instead of relying only on target bitrate.
* Compute PSNR for each encoded representation.
* Plot rate-distortion curves for 480p, 720p and 1080p representations.
* Compute the Pareto frontier and upper hull of the RD points.
* Use the RD results to choose practical bitrate-ladder candidates for consumer distribution.


## Methodology

### 1. Source Video Inspection

The source clip is uploaded into the notebook runtime and inspected using `ffprobe`. This confirms key metadata such as resolution, codec, pixel format, duration and frame rate.

This step is important because the experiment assumes a 1920×1080 source clip. All lower-resolution representations are generated from the same source so that the comparison remains consistent.

### 2. Representation Ladder Definition

The notebook defines three resolution groups:

| Label | Resolution |
| ----- | ---------: |
| 1080p |  1920×1080 |
| 720p  |   1280×720 |
| 480p  |    854×480 |

The tested target bitrates are:

| Resolution | Target bitrates                                 |
| ---------- | ----------------------------------------------- |
| 1920×1080  | 5000, 8000, 12000, 15000, 22000 kbps            |
| 1280×720   | 800, 1200, 2000, 6000, 12000, 17000, 20000 kbps |
| 854×480    | 500, 750, 1200, 1800, 2400, 4800, 7200 kbps     |

This gives 19 total encoded representations.

### 3. Reference Video Creation

A high-quality 1920×1080 reference file is generated using FFmpeg. This reference is encoded using lossless H.264 settings and is used for PSNR measurement.

The notebook uses the `upsampled_to_source` PSNR mode. This means that every encoded representation, including 720p and 480p outputs, is upscaled back to 1920×1080 before being compared with the 1080p reference.

This makes the PSNR comparison display-oriented because all representations are evaluated at the same final viewing resolution.

### 4. H.264 Encoding

Each representation is encoded using FFmpeg with `libx264`.

The main encoding settings are:

* H.264/AVC encoding through `libx264`
* Lanczos scaling to the target resolution
* `yuv420p` output format
* single-pass CBR-style rate control
* target bitrate, minimum bitrate and maximum bitrate set to the selected bitrate
* buffer size set to twice the target bitrate
* audio disabled for clean video-only analysis

Each encoded file becomes one point on the RD curve.

### 5. Actual Bitrate Measurement

Actual bitrate is measured using the encoded file size and duration:

```text
actual bitrate = file size in bits / duration
```

This is more accurate than relying only on the target bitrate because an encoder may not produce a file with exactly the requested bitrate.

### 6. PSNR Measurement

PSNR is measured using FFmpeg’s `psnr` filter.

For each encoded output:

1. The encoded video is upscaled to 1920×1080.
2. It is compared against the 1080p reference.
3. The average PSNR is extracted from FFmpeg’s output.
4. The result is stored as one RD point.

Each RD point contains:

* representation label
* width and height
* target bitrate
* actual bitrate
* measured PSNR
* encoded file path

### 7. Pareto Frontier and Upper Hull

After collecting all RD points, the notebook computes the Pareto frontier. A point is considered dominated if another representation provides equal or better PSNR at equal or lower bitrate.

The notebook also computes the upper hull of the RD curve. The upper hull identifies the most efficient representation candidates and helps decide which bitrate-resolution combinations are worth including in a bitrate ladder.

This is useful because a real streaming ladder should not include every possible encode. It should include a compact set of efficient representations.

## Results Summary

The experiment produced RD curves for 480p, 720p and 1080p representations.

The main observations were:

* PSNR generally increases with bitrate.
* Lower resolutions can be more efficient at lower bitrates.
* Higher resolutions become more useful when enough bitrate is available.
* Some high-bitrate lower-resolution encodes become inefficient because they give limited quality improvement compared with switching to a higher resolution.
* The upper hull helps identify the most useful representation candidates.

## Recommended 3-Representation Ladder

Based on the measured RD behaviour, I selected the following three representations for consumer distribution:

| Resolution | Target bitrate | Actual bitrate |       PSNR | Role                        |
| ---------- | -------------: | -------------: | ---------: | --------------------------- |
| 854×480    |      1200 kbps |   ~1225.9 kbps | ~39.255 dB | Low / fallback tier         |
| 1280×720   |      6000 kbps |   ~6144.1 kbps | ~40.927 dB | Medium / standard HD tier   |
| 1920×1080  |     15000 kbps |  ~15387.7 kbps | ~42.314 dB | High / premium Full HD tier |

This gives a practical low-medium-high bitrate ladder.

The 480p 1200 kbps point provides a strong low-tier option without wasting bitrate on diminishing returns. The 720p 6000 kbps point provides a useful middle tier with good visual quality. The 1080p 15000 kbps point provides a high-quality Full HD representation without the very high bitrate cost of the 22000 kbps point.

## Recommended 2-Representation Ladder

If only two representations are allowed, I would use:

| Resolution | Target bitrate | Actual bitrate |       PSNR | Role                         |
| ---------- | -------------: | -------------: | ---------: | ---------------------------- |
| 1280×720   |      1200 kbps |   ~1225.8 kbps | ~39.317 dB | Base / general-consumer tier |
| 1920×1080  |     15000 kbps |  ~15387.7 kbps | ~42.314 dB | Premium / high-quality tier  |

The 720p 1200 kbps representation provides a better baseline than a 480p option because it gives a higher spatial resolution while still remaining relatively accessible in bitrate. The 1080p 15000 kbps representation provides strong Full HD quality without the large additional bitrate cost of the 22000 kbps point.

## Key Outputs

The notebook generates:

* `rd_results.csv` — table of all measured RD points
* `rd_curves_with_hull.png` — RD plot with upper hull
* `rd_curves_with_hull.pdf` — high-quality PDF version of the plot

## Tools Used

* Python
* Google Colab
* FFmpeg
* FFprobe
* H.264/AVC / libx264
* Pandas
* NumPy
* Matplotlib

## Skills Demonstrated

* Video encoding workflow design
* Adaptive streaming representation analysis
* DASH bitrate-ladder experimentation
* H.264/AVC encoding
* FFmpeg command-line automation
* PSNR-based quality measurement
* Actual bitrate extraction
* Rate-distortion curve generation
* Pareto frontier analysis
* Upper hull computation
* Data analysis using Pandas
* Technical visualisation using Matplotlib

## Notes

The original source video and generated encoded video files are not included in this repository to avoid uploading large media files and potentially copyrighted content. The notebook is designed so that the experiment can be reproduced by uploading the required source clip and rerunning the cells.
