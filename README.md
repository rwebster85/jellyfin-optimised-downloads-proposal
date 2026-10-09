<!-- TOC ignore:true -->
# Proposal: Optimised Video Downloads

Provide clients with the ability to download videos from the library in lower quality, smaller file sizes.

<!-- TOC ignore:true -->

## 1. Table of Contents

<!-- TOC depthfrom:2 orderedlist:true -->

- [1. Table of Contents](#1-table-of-contents)
- [2. Motivation](#2-motivation)
- [3. Constraints](#3-constraints)
- [4. Considerations](#4-considerations)
- [5. Proposed Options](#5-proposed-options)
    - [5.1. Common concepts](#51-common-concepts)
    - [5.2. Option A: Transcode -> Download](#52-option-a-transcode---download)
        - [5.2.1. Transcode -> Download Spike](#521-transcode---download-spike)
            - [5.2.1.1. Transcoding results](#5211-transcoding-results)
            - [5.2.1.2. On-demand download transcode: I, Robot (2004) - BluRay 1080p source](#5212-on-demand-download-transcode-i-robot-2004---bluray-1080p-source)
            - [5.2.1.3. Pros](#5213-pros)
            - [5.2.1.4. Cons](#5214-cons)
    - [5.3. Option B: Curated hand-placed files](#53-option-b-curated-hand-placed-files)
        - [5.3.1. Manual Curation MVP](#531-manual-curation-mvp)
            - [5.3.1.1. Encoder comparison: Dune (2021), BluRay 1080p source - same 300-second segment at 1920×804 - My re-encoding tool, for comparison](#5311-encoder-comparison-dune-2021-bluray-1080p-source---same-300-second-segment-at-1920%C3%97804---my-re-encoding-tool-for-comparison)
            - [5.3.1.2. Pros](#5312-pros)
            - [5.3.1.3. Cons](#5313-cons)
    - [5.4. Recommended Solution](#54-recommended-solution)
- [6. Research (MVP & Spike)](#6-research-mvp--spike)
        - [6.1. Example](#61-example)
        - [6.2. Pros](#62-pros)
        - [6.3. Cons](#63-cons)
    - [5.5. Option 2. Transcode on-the-fly per download request](#55-option-2-transcode-on-the-fly-per-download-request)
        - [5.5.1. Pros](#551-pros)
        - [5.5.2. Cons](#552-cons)
- [7. Proposed Solution](#7-proposed-solution)
- [8. Design & Implementation](#8-design--implementation)

<!-- /TOC -->

## 2. Motivation

Some Jellyfin clients provide the ability to download media for offline play, but in their original formats and filesizes.

Optimised downloading enables downloading media for offline/travel consumption but without the burden of large storage requirements. Libraries may contain high bitrate/resolution files that can't feasibly be stored in large quantities on consumer portable devices such as phones and tablets.

## 3. Constraints

- Video files can come in a variety of container/codec configurations.
- Subtitles can either be embedded or external, in a variety of formats.
- MP4 file embedded subtitles are restricted to text based (MOV_TEXT).
- Transcoding/re-encoding takes time and resources.
- Current transcoding behaviour not equipped to handle download for playback without adaptation.

## 4. Considerations

- Multiple options for server admins could be made available and are not exclusive.
- Options and functionality should be both simple and valuable.
- Jellyfin provides transcoding functionality that could be adapted, reducing technical complexity.
- Still requires significant technical investment and architectural implementation.

## 5. Proposed Options

What follows is an appraisal of two potential solutions. They could be implemented alongside eachother, or only one of them selected. Conceptually they both share these core concepts:

### 5.1. Common concepts

- Simplified end-user epxerience
- Multiple Quality Tiers
    - Should include options for the server admin to set quality tiers with names (e.g. "High", "Standard") and resolution (1080p, 720p) for better end user UX. [Option A](#51-option-a-transcode---download) should also include bitrate targets (3Mbps, 800Kbps) for transcoding.
- Multiple optimised download storage locations
    - Admins should be able to select multiple locations for either transcode download or for manual curation file storage.


### 5.2. Option A: Transcode -> Download

Client download requests use server transcoding according to download and user preference quality options, to generate a lower-quality, smaller filesize copy. This will be served to the client in-place of the full-size original. Optimised files are stored for future download requests and regenerated when needed.

#### 5.2.1. Transcode -> Download Spike

A spike was carried out w/c 5th October 2026 to research feasibility, implementation, scope, and upstream impact. The work was branched off my `optimised-downloads` branch (manual curation MVP) with the manual logic removed.

The results of this work can be viewed as follows: <a href="https://github.com/rwebster85/jellyfin/tree/transcode-downloads" target="_blank">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/transcode-downloads" target="_blank">Web</a>.

**Source Code Notes**

- Encoding is fixed at `mkv h264 1280x720 750k aac 2ch 192k`. See `DownloadTranscoder.cs` <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L56" target="_blank">line 56</a> and <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L249" target="_blank">line 249</a>
- Some features and UI elements may remain from the manual branch but are non-functional
- Unlike the MVP, the spike did not cover Android, but implementation would be broadly similar.

##### 5.2.1.1. Transcoding results

Transcoding depends entirely on the hardware (or software) solution available on the server. As such, a wide variety of configurations will produce varying results, ranging from potential image quality differences to time-to-transcode delay.

To establish a baseline for cost I tested encoding from two different sources on two different machines. One machine used a Windows 11 machine RTX 5090 with NVENC encoding, the other used a Linux Mint system with an i5-12600 and Intel Quicksync hardware transcoding. Both runs used `ffmpeg` commands as per what Jellyfin would generate with baseline settings.

**Results Notes**

- Both encoders were idle besides the running task, so real-world result will vary.
- Jellyfin caps resolution to gates based on bitrate. This was not adjusted in the spike, but implementation should override that behaviour.

##### 5.2.1.2. On-demand download transcode: I, Robot (2004) - BluRay 1080p source

Source: 1:54:43, HEVC Main 10 SDR, 1920×822, 7.2 GB.
command: H.264 at 960×410, 750 kbps video, AAC stereo at 192 kbps, with the encoder preset on `auto`.

| | RTX 5090, NVENC (desktop) | i5-12600, QuickSync (server) |
|---|---|---|
| Decode / scale / encode | CUDA / `scale_cuda` / `h264_nvenc -preset p1` | VAAPI / `scale_vaapi` / `h264_qsv -preset veryfast` |
| Speed | **80x** (1,922 fps, average of 2 runs) | **42.9x** (1,030 fps) |
| Time for the whole film | **1 min 26 s** | **2 min 40 s** |
| Output | 771 MiB, 939.5 kbps | 773 MiB, 941.6 kbps |

##### 5.2.1.3. Pros

- Scalable - Server admin is not concerned with managing these optimised downloads.
- Automated - Background processes handle generation and removal.
- Self contained - Everything happens on the server.

##### 5.2.1.4. Cons

- Real-time cost - Transcoding takes a not-insignificant amount of resources.
- Unpredictable quality - Bitrate and resolution aren't obvious indicators of quality produced, and different hardware can produce different results.
- Unpredictable storage requirements - Storage requirements are unknown, and storage capacity is highly expensive.


### 5.3. Option B: Curated hand-placed files

Server admins generate/source their own lower-quality files according to their tastes, and store them on the server for the download system to serve. These files would live outside of the library, pointed to in settings, and their directory structure would mirror the library.

#### 5.3.1. Manual Curation MVP

Beginning w/c 21st September, an MVP of this feature was implemented in Server as well as Web and Android clients. Some logic is heavily constrained by my own library naming conventions for filename matching and lookup, but aside from that the implementation could broadly match. See the `optimised-downloads` branches in my forks: <a href="https://github.com/rwebster85/jellyfin/tree/optimised-downloads" target="_blank">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/optimised-downloads" target="_blank">Web</a>, <a href="https://github.com/rwebster85/jellyfin-android/tree/optimised-downloads" target="_blank">Android</a>.

This MVP served as an immediate use-case for my own personal use and has been tested in a real-world setting. Optimised downloads were created using a custom encoding workflow tool of my own creation, tailored to my needs, from my own library source videos.

**Testing Notes**

- Client devices tested incldue a Samsung Galaxy S23 and a Lenovo Y700 Gen 3 (Legion) tablet
- My own use-case preferred the `Substitute` download link option, rather than `SeparateAction`. See <a href="https://github.com/rwebster85/jellyfin/blob/transcode-downloads/MediaBrowser.Model/Configuration/DownloadBehaviour.cs" target="_blank">DownloaderBehaviour.cs</a>

##### 5.3.1.1. Encoder comparison: Dune (2021), BluRay 1080p source - same 300-second segment at 1920×804 - My re-encoding tool, for comparison

| Encoder | Setting | Speed | Bitrate | VMAF vs source |
|---|---|---|---|---|
| RTX 5090, NVENC | CQ 24 | 25.6x | 1.626 Mbps | – |
| i5-12600, QuickSync | ICQ 20, about the same quality | 6.7x | 1.403 Mbps | – |
| RTX 5090, NVENC | 3 Mbps target | – | 2.558 Mbps | 82.93 |
| i5-12600, QuickSync | 3 Mbps target | – | 2.998 Mbps | 83.40 |

##### 5.3.1.2. Pros

- Highly curated - Server admin has a hand in each file/format.
- Predictable results - Files are chosen by the admin and can be checked for quality issues.
- Tailored quality - Files can be tailored for their requirements, such as darkness, fast-moving action, or high frequency film grain.

##### 5.3.1.3. Cons

- Time intensive - Manual curation and production takes time
- High storage requirements - Multiple quality tiers for multiple video versions required.
- Not scalable - Every new video potentially requires companion optimised versions manually sourced.


### 5.4. Recommended Solution

Both options provide value for different use-cases. As such I think both options should be implemented together in phases. [Option A - Transcoding](#51-option-a-transcode---download) should be implemented in Phase 1, and if accepted, [Option B - Manual Curation](#52-option-b-curated-hand-placed-files) in phase 2.

## 6. Research (MVP & Spike)





#### 6.1. Example

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

In all cases, if a download can't be matched, the user would be served the original file (perhaps with a confirmation prompt).


#### 6.2. Pros

- Curated files bring no unpredictability. They're sourced/generated by the admin
- Flexible: admins can offer a single or multiple options for end users
- Mirrors streaming service UX and infrastructure: pre-encoded downloadable versions of content and different quality levels

#### 6.3. Cons

- Difficult to scale without automated tools for admins to make use of
- Additional complexity with library management
- Automated 3rd party library tools may need to be adjusted, or possibly unsuitable to integrate


### 5.5. Option 2. Transcode on-the-fly per download request

A downloadable file is generated/streamed (using HLS) to the client device when a download is requested. Quality could be set by the server admin using tiers, like Option 1, for simplicity. Less complicated than users selecting their preferred bitrate.

Technically highly complex and requires not-insignificant integration with client applications. Could downloads be paused? What happens on a connection drop? Are part downloads deleted/cleaned up?

Assuming quality tier options like Option 1, potential for storing the transcoded file at the same time as first serve (Just-In-Time Transcoding with Caching). The first downloading user generates the file, all proceeding requests would download the file without needing transcoding. Although this may prevent HLS muxing.

#### 5.5.1. Pros

- Potentially scalable
- No manual work required of admins

#### 5.5.2. Cons

- Unpredictable, difficult to guarantee the stream ended with success for the user
- Transcoding is expensive. Users may initiate dozens of downloads that may all need to be transcoded (potentially alleviated with a queue system, requires client integration)
- Potentially long first time download


## 7. Proposed Solution

Mixture of both, with a phased approach.

...

## 8. Design & Implementation

Until an implementation to the proposed solution is decided, what follows will be a high-level, logical design. When an implementation is decided, low-level concrete designs will be produced.
