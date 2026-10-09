<!-- TOC ignore:true -->
# Proposal: Optimised Video Downloads <a id="proposal"></a>

Provide clients with the ability to download videos from the library in lower quality, smaller file sizes.

<!-- TOC ignore:true -->
## 1. Table of Contents <a id="toc"></a>

<!-- TOC depthfrom:2 depthto:3 orderedlist:true -->

- [2. Motivation](#2-motivation)
- [3. Constraints](#3-constraints)
- [4. Considerations](#4-considerations)
- [5. Proposed Options](#5-proposed-options)
    - [5.1. Common concepts](#51-common-concepts)
    - [5.2. Option A: Transcode -> Download](#52-option-a-transcode---download)
    - [5.3. Option B: Curated hand-placed files](#53-option-b-curated-hand-placed-files)
    - [5.4. Recommended Solution](#54-recommended-solution)
- [6. High-Level Logical Design](#6-high-level-logical-design)

<!-- /TOC -->

## 2. Motivation <a id="motivation"></a>

Some Jellyfin clients provide the ability to download media for offline play, but in their original formats and filesizes.

Optimised downloading enables downloading media for offline/travel consumption but without the burden of large storage requirements. Libraries may contain high bitrate/resolution files that can't feasibly be stored in large quantities on consumer portable devices such as phones and tablets.

## 3. Constraints <a id="constraints"></a>

- Video files can come in a variety of container/codec configurations.
- Subtitles can either be embedded or external, in a variety of formats.
- MP4 file embedded subtitles are restricted to text based (MOV_TEXT).
- Transcoding/re-encoding takes time and resources.
- Current transcoding behaviour not equipped to handle download for playback without adaptation.

## 4. Considerations <a id="considerations"></a>

- Multiple options for server admins could be made available and are not exclusive.
- Options and functionality should be both simple and valuable.
- Jellyfin provides transcoding functionality that could be adapted, reducing technical complexity.
- Still requires significant technical investment and architectural implementation.

## 5. Proposed Options <a id="options"></a>

What follows is an appraisal of two potential solutions. They could be implemented alongside eachother, or only one of them selected. Conceptually they both share these core concepts:

### 5.1. Common concepts <a id="common"></a>

- Simplified end-user epxerience
- Multiple Quality Tiers
    - Should include options for the server admin to set quality tiers with names (e.g. "High", "Standard") and resolution (1080p, 720p) for better end user UX. [Option A](#option-a) should also include bitrate targets (3Mbps, 800Kbps) for transcoding.
- Multiple optimised download storage locations
    - Admins should be able to select multiple locations for either transcode download or for manual curation file storage.

---

### 5.2. Option A: Transcode -> Download <a id="option-a"></a>

Client download requests use server transcoding according to download and user preference quality options, to generate a lower-quality, smaller filesize copy. This will be served to the client in-place of the full-size original. Optimised files are stored for future download requests and regenerated when needed.

**Transcode -> Download Spike**

A spike was carried out w/c 5th October 2026 to research feasibility, implementation, scope, and upstream impact. The work was branched off my `optimised-downloads` branch (manual curation MVP) with the manual logic removed.

The results of this work can be viewed as follows: <a href="https://github.com/rwebster85/jellyfin/tree/transcode-downloads">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/transcode-downloads">Web</a>.

*Source Code Notes*

- Encoding is fixed at `mkv h264 1280x720 750k aac 2ch 192k`. See `DownloadTranscoder.cs` <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L56">line 56</a> and <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L249">line 249</a>
- Some features and UI elements may remain from the manual branch but are non-functional
- Unlike the MVP, the spike did not cover Android, but implementation would be broadly similar.

**Transcoding results**

Transcoding depends entirely on the hardware (or software) solution available on the server. As such, a wide variety of configurations will produce varying results, ranging from potential image quality differences to time-to-transcode delay.

To establish a baseline for cost I tested encoding from two different sources on two different machines. One machine used a Windows 11 machine RTX 5090 with NVENC encoding, the other used a Linux Mint system with an i5-12600 and Intel Quicksync hardware transcoding. Both runs used `ffmpeg` commands as per what Jellyfin would generate with baseline settings.

*Results Notes*

- Both encoders were idle besides the running task, so real-world result will vary.
- Jellyfin caps resolution to gates based on bitrate. This was not adjusted in the spike, but implementation should override that behaviour.

**On-demand download transcode: I, Robot (2004) - BluRay 1080p source**

Source: 1:54:43, HEVC Main 10 SDR, 1920×822, 7.2 GB.
command: H.264 at 960×410, 750 kbps video, AAC stereo at 192 kbps, with the encoder preset on `auto`.

| | RTX 5090, NVENC (desktop) | i5-12600, QuickSync (server) |
|---|---|---|
| Decode / scale / encode | CUDA / `scale_cuda` / `h264_nvenc -preset p1` | VAAPI / `scale_vaapi` / `h264_qsv -preset veryfast` |
| Speed | **80x** (1,922 fps, average of 2 runs) | **42.9x** (1,030 fps) |
| Time for the whole film | **1 min 26 s** | **2 min 40 s** |
| Output | 771 MiB, 939.5 kbps | 773 MiB, 941.6 kbps |

**Pros & Cons**

* **Pro:** Scalable - Server admin is not concerned with managing these optimised downloads.
* **Pro:** Automated - Background processes handle generation and removal.
* **Pro:** Self contained - Everything happens on the server.
* **Con:** Real-time cost - Transcoding takes a not-insignificant amount of resources.
* **Con:** Unpredictable quality - Bitrate and resolution aren't obvious indicators of quality produced, and different hardware can produce different results.
* **Con:** Unpredictable storage requirements - Storage requirements are unknown, and storage capacity is highly expensive.


### 5.3. Option B: Curated hand-placed files <a id="option-b"></a>

Server admins generate/source their own lower-quality files according to their tastes, and store them on the server for the download system to serve. These files would live outside of the library, pointed to in settings, and their directory structure would mirror the library.

**Manual Curation MVP**

Beginning w/c 21st September 2026, an MVP of this feature was implemented in Server as well as Web and Android clients. Some logic is heavily constrained by my own library naming conventions for filename matching and lookup, but aside from that the implementation could broadly match. See the `optimised-downloads` branches in my forks: <a href="https://github.com/rwebster85/jellyfin/tree/optimised-downloads">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/optimised-downloads">Web</a>, <a href="https://github.com/rwebster85/jellyfin-android/tree/optimised-downloads">Android</a>.

This MVP served as an immediate use-case for my own personal use and has been tested in a real-world setting. Optimised downloads were created using a custom encoding workflow tool of my own creation, tailored to my needs, from my own library source videos.

**Testing Notes**

- Client devices tested incldue a Samsung Galaxy S23 and a Lenovo Y700 Gen 3 (Legion) tablet
- My own use-case preferred the `Substitute` download link option, rather than `SeparateAction`. See <a href="https://github.com/rwebster85/jellyfin/blob/transcode-downloads/MediaBrowser.Model/Configuration/DownloadBehaviour.cs">DownloaderBehaviour.cs</a>

**Encoder comparison: Dune (2021), BluRay 1080p source - same 300-second segment at 1920×804 - My re-encoding tool, for comparison**

| Encoder | Setting | Speed | Bitrate | VMAF vs source |
|---|---|---|---|---|
| RTX 5090, NVENC | CQ 24 | 25.6x | 1.626 Mbps | – |
| i5-12600, QuickSync | ICQ 20, about the same quality | 6.7x | 1.403 Mbps | – |
| RTX 5090, NVENC | 3 Mbps target | – | 2.558 Mbps | 82.93 |
| i5-12600, QuickSync | 3 Mbps target | – | 2.998 Mbps | 83.40 |

**Pros & Cons**
* **Pro:** Highly curated - Server admin has a hand in each file/format.
* **Pro:** Predictable results - Files are chosen by the admin and can be checked for quality issues.
* **Pro:** Tailored quality - Files can be tailored for their requirements, such as darkness, fast-moving action, or high frequency film grain.
* **Con:** Time intensive - Manual curation and production takes time
* **Con:** High storage requirements - Multiple quality tiers for multiple video versions required.
* **Con:** Not scalable - Every new video potentially requires companion optimised versions manually sourced.


### 5.4. Recommended Solution <a id="recommendation"></a>

Based on the research and results of the MVP and spike, **[Option A: Transcode -> Download](#option-a)** is the clear choice for friction vs value for server admins. It has less work compared to manual curation, scales better, and integrates well with the existing systems in Jellyfin.

However, both options provide value for different use-cases. As such my suggestion would be to implement both options together in phases. [Option A - Transcoding](#option-a) should be implemented in Phase 1, and if accepted, [Option B - Manual Curation](#option-b) in Phase 2.


## 6. High-Level Logical Design <a id="hld"></a>

Until an implementation to the proposed solution is decided, what follows will be a high-level, logical design. When an implementation is decided, low-level concrete designs will be produced.


#### 4.1. Example

A movie might be located in a library like so:

```
/Path/To/Movies
└── Film (1986)
    └── Film (1986).mkv <-- 4K file
```

A download location to scan set to:

`/Path/To/Downloads/Movies`

Within that directory:

```
Film (1986)
└── Film (1986).mkv <-- 720p file
```

With Optimised Downloads enabled in the server dashboard, a download folder location set to 
`/Path/To/Downloads/Movies`, and a folder/file named for the film like the above example, when the user triggers a download, they would be served the 720p file.

Additional options provided to server admins to create quality tiers. For example, a High (1080p) and a Standard (720p). Users can set their preferred tier in their own preferences. Downloads should be appended with the tier name. For example:

```
/Path/To/Downloads/Movies
└── Film (1986)
    ├── Film (1986) - High.mkv
    └── Film (1986) - Standard.mkv
```

Users would be served the quality tier matching their preference.

Potential for a fallback, if quality tiers are enabled but only one download file is present, user is served that download.

