# Proposal: Optimised Video Downloads <!-- omit from toc -->

Provide clients with the ability to download videos from the library in lower quality, smaller file sizes.

---

## Table of Contents <!-- omit from toc -->

- [1. Motivation](#1-motivation)
  - [1.1. Previous requests](#11-previous-requests)
  - [1.2. This Proposal](#12-this-proposal)
    - [1.2.1. AI Assistance](#121-ai-assistance)
- [2. Constraints](#2-constraints)
- [3. Considerations](#3-considerations)
- [4. Proposed Options](#4-proposed-options)
  - [4.1. Common concepts](#41-common-concepts)
  - [4.2. Option A: Transcoding](#42-option-a-transcoding)
    - [4.2.1. Transcoding Spike](#421-transcoding-spike)
    - [4.2.2. Transcoding Tests](#422-transcoding-tests)
    - [4.2.3. On-demand Transcode Download](#423-on-demand-transcode-download)
    - [4.2.4. Hardware vs Software](#424-hardware-vs-software)
    - [4.2.5. Transcoding - Findings](#425-transcoding---findings)
    - [4.2.6. Transcoding - Pros \& Cons](#426-transcoding---pros--cons)
  - [4.3. Option B: Manual Curation](#43-option-b-manual-curation)
    - [4.3.1. Manual Curation MVP](#431-manual-curation-mvp)
    - [4.3.2. Manual Curation - Encoder Comparison](#432-manual-curation---encoder-comparison)
    - [4.3.3. Manual Curation - Findings](#433-manual-curation---findings)
    - [4.3.4. Manual Curation - Pros \& Cons](#434-manual-curation---pros--cons)
  - [4.4. Option C: Do Nothing](#44-option-c-do-nothing)
    - [4.4.1. Do Nothing - Pros \& Cons](#441-do-nothing---pros--cons)
  - [4.5. Recommended Solution](#45-recommended-solution)
- [5. High-Level Design](#5-high-level-design)
  - [5.1. ArchiMate Notation](#51-archimate-notation)
  - [5.2. HLD - Common Designs](#52-hld---common-designs)
    - [5.2.1. Service entry point](#521-service-entry-point)
    - [5.2.2. Downloadable file interaction](#522-downloadable-file-interaction)
    - [5.2.3. Settings interaction](#523-settings-interaction)
    - [5.2.4. Full common design](#524-full-common-design)
  - [5.3. HLD - Transcoding](#53-hld---transcoding)
    - [5.3.1. Transcode Process](#531-transcode-process)
    - [5.3.2. Transcode Cleanup](#532-transcode-cleanup)
    - [5.3.3. Scheduled Pre-transcode](#533-scheduled-pre-transcode)
  - [5.4. HLD - Manual Curation](#54-hld---manual-curation)
- [6. Concrete Designs](#6-concrete-designs)
- [7. Implementation](#7-implementation)
  - [7.1. Common](#71-common)
    - [7.1.1. Global settings screen](#711-global-settings-screen)
    - [7.1.2. Download API](#712-download-api)
    - [7.1.3. ETag](#713-etag)
  - [7.2. Transcoding](#72-transcoding)
    - [7.2.1. Transcoding - Download API](#721-transcoding---download-api)
    - [7.2.2. Transcoding - Global Settings](#722-transcoding---global-settings)
    - [7.2.3. Transcoding - Directory Structure](#723-transcoding---directory-structure)
  - [7.3. Manual Curation](#73-manual-curation)
    - [7.3.1. Manual Curation - Download API](#731-manual-curation---download-api)
    - [7.3.2. Manual Curation - Global Settings](#732-manual-curation---global-settings)
    - [7.3.3. Manual Curation - Directory Structure](#733-manual-curation---directory-structure)
  - [7.4. Open Questions](#74-open-questions)
- [8. Future Phases](#8-future-phases)

<br>

## 1. Motivation

Some Jellyfin clients provide the ability to download media for offline play, but in their original formats and filesizes.

Optimised downloading enables downloading media for offline/travel consumption but without the burden of large storage requirements. Libraries may contain high bitrate/resolution files that can't feasibly be stored in large quantities on consumer portable devices such as phones and tablets.

### 1.1. Previous requests

There have been several requests for this functionality within Jellyfin Core as well as the Android Client. These are either open with no progress, or have been closed either as rejected, part complete, or simply abandoned.

**Meta Discussions**
* [Meta #66 - Proposal for Introducing Transcoding Options to Media Downloads](https://github.com/jellyfin/jellyfin-meta/discussions/66)
  * *nielsvanvelzen* asked why the existing transcoding pipeline (transcode -> serve in packets) can't be used. [Section 4.2.3 - On-demand Transcode Download](#423-on-demand-transcode-download) shows the problems with serving files while they're being transcoded. 
* [Meta #115 - Implementation for transcoded files download](https://github.com/jellyfin/jellyfin-meta/discussions/115)
* [Meta #131 - Transcoded downloads: a job-based, resumable API](https://github.com/jellyfin/jellyfin-meta/discussions/131)
  * This proposal differs to Meta #131 in some significant ways:
    * Transcodes are stored on disk per tier, rather than discarded after download.
    * Server admins define the quality tiers (better UX for end-users, better supports storing for others).
    * Three separate options are appraised and compared.
    * A working MVP and spike were produced and results were measured.

<br>

**Jellyfin Features**
* [Support download of transcoded files](https://features.jellyfin.org/posts/57/support-download-of-transcoded-files)

<br>

**Android**
* [Issue #316 - To be able to download the video content from the server in a specified quality](https://github.com/jellyfin/jellyfin-android/issues/316)
* [Issue #1717 - Add an In-App Download Manager with Quality Options](https://github.com/jellyfin/jellyfin-android/issues/1717)

<br>

---

### 1.2. This Proposal

This is a new proposal that follows *nielsvanvelzen*'s comment in [Meta #66](https://github.com/jellyfin/jellyfin-meta/discussions/66#discussioncomment-15641494), suggesting opening a new discussion rather than continuing the old one. It builds off previous discussions/requests while presenting a solution as an appraisal of three potential options, based on assessments from a proof-of-concept MVP (one in actual use within a real library and device ecosystem) and a feasibility spike, then finally recommending a solution. It also includes transcoding test results from real hardware to provide example data of potential time and resource costs.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 1.2.1. AI Assistance

**MVP & Spike**

Claude assistance was used for the creation of the MVP and during the spike. Most of the code was generated and all generated code was at my direction, as per my design and specification, then manually edited as and when needed.

**Proposal**

Claude assistance was used towards the end of this proposal's creation as a copy-editor (spelling, grammar, missing contexts etc), checking for contradictions between the document and my design notes, and for managing a review backlog.

Claude was also used to carry out some of the encoding tests, and to gather the results.

<br>

## 2. Constraints

* Video files can come in a variety of container/codec configurations.
* Subtitles can either be embedded or external, in a variety of formats.
* MP4 file embedded subtitles are restricted to text based (MOV_TEXT).
* Transcoding/re-encoding takes time and resources.
* Current transcoding behaviour not equipped to handle download for playback without adaptation.

<br>

## 3. Considerations

* Multiple options for server admins could be made available and are not exclusive.
* Options and functionality should be both simple and valuable.
* Jellyfin provides transcoding functionality that could be adapted, reducing technical complexity.
* Still requires significant technical investment and architectural implementation.

<br>

## 4. Proposed Options

What follows is an appraisal of three potential solutions. The two change options could be implemented alongside each other, or only one of them selected. Conceptually they both share these core concepts:

### 4.1. Common concepts

* Simplified end-user experience
* Multiple Quality Tiers
    * Should include options for the server admin to set quality tiers with names (e.g. "High", "Standard") and resolution (1080p, 720p) for better end user UX. [Option A](#42-option-a-transcoding) should also include bitrate targets (3Mbps, 800Kbps) for transcoding.
* Multiple optimised download storage locations
    * Admins should be able to select multiple locations for either transcode download or for manual curation file storage.

<br>

---

### 4.2. Option A: Transcoding

Client download requests use server transcoding according to download and user preference quality options, to generate a lower-quality, smaller filesize copy. This will be served to the client in-place of the full-size original. Optimised files are stored for future download requests and regenerated when needed.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.1. Transcoding Spike

A spike was carried out w/c 5th October 2026 to research feasibility, implementation, scope, and upstream impact. The work was branched off my `optimised-downloads` branch (manual curation MVP) with the manual logic removed.

The results of this work can be viewed as follows: <a href="https://github.com/rwebster85/jellyfin/tree/transcode-downloads">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/transcode-downloads">Web</a>.

*Source Code Notes*

* Encoding is fixed at `mkv h264 1280x720 750k aac 2ch 192k`. See `DownloadTranscoder.cs` <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L56">line 56</a> and <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L249">line 249</a>
* Some features and UI elements may remain from the manual branch but are non-functional

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.2. Transcoding Tests

Transcoding depends entirely on the hardware (or software) solution available on the server. As such, a wide variety of configurations will produce varying results, ranging from potential image quality differences to time-to-transcode delay.

To establish a baseline for cost I tested hardware encoding from two different sources on two different machines. One was a Windows 11 desktop with an RTX 5090 (NVENC), the other a Linux Mint server with an i5-12600 (Intel QuickSync). A separate run with the same sources was tested on the server with hardware acceleration disabled. Each used the `ffmpeg` commands Jellyfin generated on each machine with baseline settings.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.3. On-demand Transcode Download

**Sources:**
* I, Robot (2004) 1080p HEVC Main 10 SDR, 7.2 GB, 1:54:43
* Dune (2021) 4K HEVC DV 8.1 (HDR10 base), 32.6 GB, 2:35:27

<br>

**Command:** H.264 750 kbps, AAC stereo 192k, preset auto.

<br>

*Results Notes*

* Both machines were idle besides the running task, so results will vary on a busy server.
* Each figure is the average of 2 full-film runs.
* Jellyfin caps resolution to gates based on bitrate. This was not adjusted in the spike, but implementation should override that behaviour.

<br>

| | RTX 5090, NVENC (desktop) | i5-12600, QuickSync (server) |
|---|---|---|
| **I, Robot (2004)** - to 960×410 | | |
| Decode / scale / encode | CUDA / `scale_cuda` / `h264_nvenc -preset p1` | VAAPI / `scale_vaapi` / `h264_qsv -preset veryfast` |
| Speed | **80x** (1,922 fps) | **42.9x** (1,030 fps) |
| Time for the whole film | **1 min 26 s** | **2 min 40 s** |
| Output | 771 MiB, 939.5 kbps | 775 MiB, 944.4 kbps |
| **Dune (2021)** - to 960×400 tone-mapped | | |
| Decode / scale / tone-map / encode | CUDA / `scale_cuda` / `tonemap_cuda` (bt2390) / `h264_nvenc -preset p1` | VAAPI / `scale_vaapi` / `tonemap_vaapi` (VPP) / `h264_qsv -preset veryfast` |
| Speed | **32.7x** (783 fps) | **16.5x** (394 fps) |
| Time for the whole film | **4 min 46 s** | **9 min 28 s** |
| Output | 1,042 MiB, 937.6 kbps | 1,040 MiB, 935.6 kbps |

<br>

**Serving a file while it is still being transcoded**

The spike first streamed the transcode to the client as it was written. Testing showed this is unreliable.

| Finding | Impact |
|---|---|
| `ffmpeg` writes an MKV's duration and seek index at the start of the file once encoding finishes. A client streaming the growing file has already received the placeholder. | The downloaded file reports the wrong duration and lacks its seek index. |
| Fragmented MP4 streams intact, but ExoPlayer (used for offline playback on Android) cannot seek it without a `sidx` index, which ffmpeg can only write once encoding finishes. | No container is both streaming-safe and seekable on every client. MP4 would also rule out image-based subtitles. |
| If a transcode fails, or is stopped because its video was removed from the library, the response simply ends. | The client keeps a partial file with no indication it failed. |
| A growing file has no known size and cannot serve byte ranges. | No download progress percentage, and an interrupted download restarts from the beginning. |
| A file that does not exist yet cannot be described (MediaInfo). | Clients that store a description of the download for offline playback cannot get one in advance. |
| The client can only receive what has been encoded so far. | Download speed is capped by encoding speed. |

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.4. Hardware vs Software

The same transcodes on the server with hardware acceleration set to None, to show the cost on a server without it. Jellyfin uses `libx264` / `libx265` with preset auto (`veryfast`), its default CRF (23 / 28) capped at 750 kbps, and `tonemapx` (bt2390) for Dune. HEVC was added for both encoders.

<br>

*Results Notes*

* One full-film run each. The QuickSync H.264 runs were repeated to measure CPU, and were within 3% of the speeds above.
* CPU is the share of all 12 threads busy, sampled every 5 seconds. Peak load is the highest 1-minute load average during the run.
* Software output usually falls below the 750 kbps cap, as Jellyfin's CRF decides the bitrate, so those files are smaller. Quality was not compared.
* Output figures include the 192 kbps stereo AAC audio, so the 750 kbps video cap shows as about 940 kbps.

<br>

| Source | Codec | Encoder | Time for the whole film | CPU busy (mean) | Peak load | Output |
|---|---|---|---|---|---|---|
| I, Robot (2004) | H.264 | QuickSync | **2 min 40 s** (42.9x) | 10% | 1.7 | 775 MiB, 944 kbps |
| | | Software | **5 min 45 s** (19.9x) | 83% | 20.6 | 663 MiB, 807 kbps |
| | HEVC | QuickSync | **4 min 34 s** (25.0x) | 9% | 1.6 | 775 MiB, 944 kbps |
| | | Software | **9 min 53 s** (11.6x) | 78% | 16.1 | 449 MiB, 547 kbps |
| Dune (2021) | H.264 | QuickSync | **9 min 41 s** (16.0x) | 23% | 3.8 | 1,040 MiB, 936 kbps |
| | | Software | **20 min 31 s** (7.5x) | 88% | 25.3 | 575 MiB, 518 kbps |
| | HEVC | QuickSync | **11 min 13 s** (13.8x) | 22% | 3.9 | 1,034 MiB, 930 kbps |
| | | Software | **23 min 36 s** (6.5x) | 90% | 28.0 | 359 MiB, 323 kbps |

<br>

Without hardware encoding, a transcode takes about twice as long and holds 78-90% of the CPU for its whole duration: 8 to 19 times the CPU time of QuickSync. I recommend that transcoded downloads ***should require hardware acceleration***.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.5. Transcoding - Findings

Files should be transcoded in full before they are served (see [5.3.1 - Transcode Process](#531-transcode-process)). The cost is that the first download of a video at a tier and audio track waits for the transcode: 1.5 to 3 minutes for a 1080p film, and 5 to 10 minutes for a 4K HDR one, on the systems above ([4.2.3](#423-on-demand-transcode-download)) with hardware H.264 encoding.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.6. Transcoding - Pros & Cons

* **Pro:** Scalable - Server admin is not concerned with managing these optimised downloads.
* **Pro:** Automated - Background processes handle generation and removal.
* **Pro:** Self contained - Everything happens on the server.
* **Con:** Real-time cost - Transcoding takes a not-insignificant amount of resources, especially in software, and the first download at a tier and audio track waits for it (up to 10 minutes for a 4K film with hardware H.264 encoding, and over 20 in software: see [4.2.4 - Hardware vs Software](#424-hardware-vs-software) above).
* **Con:** Unpredictable quality - Bitrate and resolution aren't obvious indicators of quality produced, and different hardware can produce different results.
* **Con:** Unpredictable storage requirements - Storage requirements are unknown, as each video can have one transcode per tier and audio track (though most downloads use the user's default track), and storage capacity is highly expensive.

<br>

---

### 4.3. Option B: Manual Curation

Server admins generate/source their own lower-quality files according to their tastes, and store them on the server for the download system to serve. These files would live outside of the library, pointed to in settings, and their directory structure would mirror the source folder structure.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.3.1. Manual Curation MVP

Beginning w/c 21st September 2026, an MVP of this feature was implemented in Server as well as Web and Android clients. Some logic is heavily constrained by my own library naming conventions for filename matching and lookup, but aside from that the implementation could broadly match. See the `optimised-downloads` branches in my forks: <a href="https://github.com/rwebster85/jellyfin/tree/optimised-downloads">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/optimised-downloads">Web</a>, <a href="https://github.com/rwebster85/jellyfin-android/tree/optimised-downloads">Android</a>.

This MVP served as an immediate use-case for my own personal use and has been tested in a real-world setting. Optimised downloads were created using a custom encoding workflow tool of my own creation, tailored to my needs, from my own library source videos.

**Testing Notes**

* Client devices tested include a Samsung Galaxy S23 and a Lenovo Y700 Gen 3 (Legion) tablet.
* My own use-case preferred the `Substitute` download link option, rather than `SeparateAction` (this proposal makes it the only behaviour). See <a href="https://github.com/rwebster85/jellyfin/blob/transcode-downloads/MediaBrowser.Model/Configuration/DownloadBehaviour.cs">DownloadBehaviour.cs</a>

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.3.2. Manual Curation - Encoder Comparison

My encoding tool's "High" tier on NVENC, against a QuickSync equivalent (my tool is NVENC-only).

Sources used were the same as section [4.2.3 - On-demand Transcode Download](#423-on-demand-transcode-download).

<br>


*Results Notes*

* VMAF over 300 s from 30:00.
* Each figure is the average of 2 full-film runs.
* Dune scored against a lossless FFV1 reference made through each machine's own tone-map (`tonemap_cuda` on NVENC, `tonemap_opencl` on QuickSync, both bt2390).

<br>

| | RTX 5090, NVENC (desktop) | i5-12600, QuickSync (server) |
|---|---|---|
| Encoder | `hevc_nvenc -preset p4 -tune hq`, CQ 24, 3.5 Mbps ceiling | `hevc_qsv -preset medium`, ICQ 20, no ceiling |
| **I, Robot (2004)** - 1080p SDR, to 1920×822 | | |
| Time for the whole film | **4 min 15 s** (27.0x) | **19 min 3 s** (6.0x) |
| Output | 2.80 GiB, 3.49 Mbps | 2.41 GiB, 3.01 Mbps |
| VMAF vs source (mean / 1st percentile / min) | 95.64 / 90.47 / 88.01 | 94.90 / 88.75 / 85.78 |
| **Dune (2021)** - 4K HDR10, to 1920×802 tone-mapped (bt2390), Main 10 | | |
| Time for the whole film | **5 min 43 s** (27.2x) | **39 min 18 s** (4.0x) |
| Output | 2.03 GiB, 1.87 Mbps | 1.80 GiB, 1.65 Mbps |
| VMAF vs tone-mapped source (mean / 1st percentile / min) | 91.85 / 72.92 / 63.07 | 90.58 / 79.55 / 61.27 |

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.3.3. Manual Curation - Findings

Encoding to a higher-quality tier costs far more than Jellyfin's on-demand transcode: on the same server, about 4x longer for Dune. Curated files can instead be produced on a faster machine, as here.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.3.4. Manual Curation - Pros & Cons

* **Pro:** Highly curated - Server admin has a hand in each file/format.
* **Pro:** Predictable results - Files are chosen by the admin and can be checked for quality issues.
* **Pro:** Tailored quality - Files can be tailored for their requirements, such as darkness, fast-moving action, or high frequency film grain.
* **Con:** Time intensive - Manual curation and production takes time (encoding one 4K film at a high-quality tier took 6 to 39 minutes on the hardware tested: see [4.3.2 - Encoder Comparison](#432-manual-curation---encoder-comparison) above).
* **Con:** High storage requirements - Multiple quality tiers for multiple video versions required.
* **Con:** Not scalable - Every new video potentially requires companion optimised versions manually sourced.

<br>

---

### 4.4. Option C: Do Nothing

Change nothing about the existing system and rely on server admins to make use of the existing `version` behaviour by supplying lower-quality, smaller videos alongside the existing library ones. Users have the option of downloading whichever version they want, but must be mindful of the file sizes (shown in the Media Info page of the item's menu).

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.4.1. Do Nothing - Pros & Cons

* **Pro:** No new code changes needed to review, or for maintainers to own.
* **Pro:** No server transcoding load, and no storage beyond what the admin chooses to add.
* **Pro:** Reuses existing Jellyfin feature set.
* **Con:** Relies on users selecting the lower quality version, which isn't always obvious in the UI.
* **Con:** A small version is a full playback version too, so it shows in every user's version picker when streaming.
* **Con:** Does not address the existing feature requests for smaller downloads.

<br>

---

### 4.5. Recommended Solution

Based on the research and results of the MVP ([4.3.3](#433-manual-curation---findings)) and spike ([4.2.5](#425-transcoding---findings)), the clear choice for friction vs value for server admins is to implement **[Option A: Transcoding](#42-option-a-transcoding)**. It needs less work than manual curation, scales better, and integrates well with the existing systems in Jellyfin.

However, Options A and B both provide value for different use-cases. As such my suggestion would be to implement both options together in phases. [Option A: Transcoding](#42-option-a-transcoding) should be implemented in Phase 1, and [Option B: Manual Curation](#43-option-b-manual-curation) in Phase 2 if there is an appetite for it. Additional phases could also take place to refine this new feature even further (see [section 8](#8-future-phases)).

[Option C: Do Nothing](#44-option-c-do-nothing) remains available to admins today, but leaves the request unaddressed. Option B is essentially Option C with the server choosing the right version for each user based on their preferences, and keeping the smaller files out of the playback versions list.

<br>

## 5. High-Level Design

Until an option to the proposed solution is agreed, what follows will be a high-level logical design focused on system and object interactions. When an option is agreed, low-level concrete designs will be produced for the implementation.

### 5.1. ArchiMate Notation

The high-level design (HLD) uses the ArchiMate enterprise architecture modelling language to represent the different elements within the system and their relationships with each other.

Diagrams were created using [Visual Paradigm Online](https://online.visual-paradigm.com/), and a quirk of the tool is *Access* arrow lines are dashed instead of dotted.

<details>
    <summary>Show colour scheme & notation guide</summary>

<br>

Uses the Open Group provided *alternative* colour scheme available at https://www.opengroup.org/sites/default/files/archimate-alternative-color-set.pdf.

<br>

| Colour | Visual / Hex | Layer | Elements |
|---|---|---|---|
| Yellow | `#EDD779` | Business | Actor, role, collaboration, interface, process, function, interaction, event, service, business object, contract, representation, product |
| Blue | `#B0D0D9` | Application | Component, collaboration, interface, function, interaction, process, event, service, data object |
| Green | `#BCD9AE` | Technology and physical | Node, device, system software, collaboration, interface, path, communication network, function, process, interaction, event, service, artifact, equipment, facility, distribution network, material |
| Orange | `#EFBD5D` | Strategy | Resource, capability, value stream, course of action |
| Purple | `#D1BADC` | Motivation | Stakeholder, driver, assessment, goal, outcome, principle, requirement, constraint, meaning, value |
| Rose | `#F1BBB7` | Implementation and migration | Work package, deliverable, implementation event, plateau, gap |
| Gray | `#E5DFD3` | Composite | Grouping, location |

<br>

Notation comes from the ArchiMate 3.2 specification, provided by Open Group, available at https://www.opengroup.org/sites/default/files/docs/downloads/n221p.pdf

<br>

**Structural relationships**

| Relationship | Line | Arrow | Means |
|---|---|---|---|
| Composition | Solid | Filled diamond at source | Represents that an element consists of one or more other concepts. |
| Aggregation | Solid | Hollow diamond at source | Represents that an element combines one or more other concepts. |
| Assignment | Solid | Ball at source, filled arrowhead at target | Represents the allocation of responsibility, performance of behavior, storage, or execution. |
| Realization | Dotted | Hollow triangle | Represents that an element plays a critical role in the creation, achievement, sustenance, or operation of a more abstract element. |

<br>

**Dependency relationships**

| Relationship | Line | Arrow | Means |
|---|---|---|---|
| Serving | Solid | Open arrowhead | Represents that an element provides its functionality to another element. |
| Access | Dotted* | None, small open arrowhead, or one at both ends | Represents the ability of behavior and active structure elements to observe or act upon passive structure elements. |
| Influence | Dashed | Open arrowhead, labelled `+/-` | Represents that an element affects the implementation or achievement of some motivation element. |
| Association | Solid | None, or a half arrowhead when directed | Represents an unspecified relationship, or one that is not represented by another ArchiMate relationship. |

\**Dashed in diagrams*

<br>

**Dynamic relationships**

| Relationship | Line | Arrow | Means |
|---|---|---|---|
| Triggering | Solid | Filled arrowhead | Represents a temporal or causal relationship between elements. |
| Flow | Dashed | Filled arrowhead | Represents transfer from one element to another. |

<br>

**Other relationships**

| Relationship | Line | Arrow | Means |
|---|---|---|---|
| Specialization | Solid | Hollow triangle | Represents that an element is a particular kind of another element. |

<br>

**Relationship connectors**

| Connector | Line | Symbol | Means |
|---|---|---|---|
| Junction | — | Filled circle (And junction), hollow circle (Or junction) | Used to connect relationships of the same type. |

</details>


<br>

---

### 5.2. HLD - Common Designs

Both transcoding and manual curation would share some design elements.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.2.1. Service entry point

<br>

![Common entry point design](/images/hld-common-entry-point.png)

**Notes**

* "Helper" is a service named as per the Jellyfin convention.
* Clients tested are Web and Android. Others that support downloading should be assessed.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.2.2. Downloadable file interaction

<br>

![Downloadable file interaction design](/images/hld-common-download.png)

**Notes**

* Original file being served when selected, no applicable version available, or the feature is off.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.2.3. Settings interaction

**Note:** Some of this design corresponds with my MVP and spike, in terms of settings and configuration storage. This is subject to change following an agreement on the appraisal and implementation strategy.

<br>

![Common settings design](/images/hld-common-settings.png)

**Notes**

* Helper handles user preferences, backend download options use the backend services.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.2.4. Full common design

<details>
    <summary>See full common design</summary>

<br>

![Full common design diagram](/images/hld-common-all.png)

</details>

<br>

---

### 5.3. HLD - Transcoding

Definitions:

**Stale**

* A transcode file matching a video in the library that was removed/renamed/changed.
* A transcode file that no longer matches the tier resolution and/or bitrate that created it, if the admin opts for that setting. By default, tier changes will not make stale any existing transcodes.
* A transcode whose tier was deleted.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.3.1. Transcode Process

1. Download locations are configured by the admin in settings, or the server default location used.
2. User download request is captured by the controller.
3. If a transcoded file matching the user's quality tier preference (or server default) exists, and is not stale, the file is served.
4. Otherwise, if a transcode job isn't already running, configured against the user's quality tier preference (or server default) and the corresponding resolution and bitrate set by the admin, start a transcode job.
5. Client shows the download file is being prepared in the UI. File transcoded to disk first then served.
6. If a download request is cancelled, the transcoding job continues in the background.
7. If a download request is initiated while the video is being transcoded, the request joins the existing job rather than triggering a new one.
8. If the video is removed from the library during transcode, the job is interrupted and partial file deleted.
9. If the transcode completes successfully, it is recorded as complete. A failed transcode file would be removed.

<br>

<details>
    <summary>Show system diagram</summary>

<br>

![Transcoding / Downloading process design](/images/hld-transcoding-download.png)

</details>

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.3.2. Transcode Cleanup

1. Stale transcoded files are automatically cleaned up.
2. The existing system triggers an event when a library video is removed/renamed/moved. These videos have their transcode counterparts deleted, and transcodes in progress are cancelled.
3. The cleaner also runs on a schedule to remove stale and/or remnant transcodes.

<br>

<details>
    <summary>Show system diagram</summary>

<br>

![Transcoding cleanup design](/images/hld-transcoding-cleaner.png)

</details>

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.3.3. Scheduled Pre-transcode

1. A scheduled task, like Trickplay generation, run manually or on a timer. It generates transcoded download videos without a download request.
2. It produces the same transcoded files (and their sidecar) that a download request would.
3. If a download request is initiated for a video the task is transcoding, it joins that job.
4. All storage, staleness, and cleanup rules apply to pre-made transcodes, as they would a download generated one.

Which videos and tiers the task covers, and how it shares the server with on-demand transcodes, are open questions (see [7.4](#74-open-questions)).

<br>

---

### 5.4. HLD - Manual Curation

1. Download versions of videos are placed on the server by the admin.
2. Download locations are configured by the admin in settings.
3. Download requests are captured by the controller.
4. If an optimised video is found matching the user's quality tier preference (or server default), that video is served.
5. If no optimised video is found, the original file will be served instead.

<details>
    <summary>Show system diagram</summary>

<br>

![Manual curation download design](/images/hld-manual-download.png)

</details>

<br>

## 6. Concrete Designs

To follow once an option is agreed (see [section 5](#5-high-level-design)).

<br>

## 7. Implementation

The following implementation details only reflect what is necessary for appraisal of the options in this proposal. As with the HLD, once an option has been agreed, additional concrete implementation details will be provided.

### 7.1. Common

#### 7.1.1. Global settings screen

A global settings screen will be created for server admins:

* A checkbox to enable/disable this whole feature - default off.
  * When the feature is off, every route would describe and serve the original file, in-line with today's behaviour: `Download?optimized=true` serves the file, and both `Prepare` and `MediaInfo` return a `200` response along with the original file's media info and ETag.
* A global set of locations used to store optimised files.
  * These do not take priority over library settings locations.
* A checkbox to show a "Download Original" link in the UI.
  * The plain route (`GET Items/{itemId}/Download`) is unchanged and still driven by the existing per-user "Allow media downloading" permission.
  * Default is on. Updated clients would have the existing `Download` link as well as `Download Original` while the feature is on.

<br>

The global download settings are stored in the XML config store, like some other settings, as per the MVP and HLD. Storing tiers in the database instead is open to discussion, and may be preferable if we decide to store records of transcode files in the DB (not part of this proposal but potentially desirable).

<br>

**Permissions**

Optimised downloading still requires the existing `Allow media downloading` permissions setting, and transcoding still uses the `Allow video playback that requires transcoding` permissions setting. Even when a transcode of that video already exists, a user who is not permitted to use transcoding will be served the original file: the `Downloads/Behaviour` API response will return `enabled: false`. Nothing is encoded with the Manual Curation option, so `Allow media downloading` is the only permission used. These decisions are open to discussion, with the potential of a new permission being introduced.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 7.1.2. Download API

All clients that support downloading will need to be updated for either Transcoding or Manual Curation.

Before downloading, clients that store media info for offline playback (Android) get a description of the file to be served, and its ETag (see [7.1.3](#713-etag)), from `Prepare`'s ready response (Transcoding) or a new `Items/{itemId}/Download/MediaInfo` route (Manual Curation). The client sends that ETag with the download (`If-Match`). If the file has changed in between, the server refuses with `412` and the client starts again. Clients that hand the file to the browser or operating system (Web) skip this, and send the download without `If-Match`.

The existing download route will require an `optimized=true` parameter, becoming `GET Items/{itemId}/Download?optimized=true`. For backwards compatibility, without the new query parameter the behaviour of `GET Items/{itemId}/Download` remains unchanged: the original file will be served, in line with current behaviour.

When Optimised Downloads is enabled, the existing `Download` link in the client would call `GET Items/{itemId}/Download?optimized=true`, and the new `Download Original` would call the existing plain `GET Items/{itemId}/Download`. When the feature is disabled, `Download Original` is hidden and `Download` simply calls the existing plain route.

The transcoding API uses an optional `audioStreamIndex` for `Prepare` and the optimised download (see [7.2.1](#721-transcoding---download-api)). When not supplied, the audio stream the system would determine to be the user's default would be selected.

<br>

**Download behaviour**

When clients are updated, they determine which links to show by calling the new `GET Downloads/Behaviour` API, as well as the tiers the user could select:

<br>

```json
{
  "enabled": true,
  "showOriginalLink": true,
  "tiers": [
    { "id": "8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50", "name": "High (1080p)" },
    { "id": "1f0e2d3c4b5a69788796a5b4c3d2e1f0", "name": "Standard (720p)" }
  ]
}
```

* `enabled` - Whether the feature is on and the user has permission to transcode (see [7.1.1 Permissions](#711-global-settings-screen)).
* `showOriginalLink` - Whether the `Download Original` link should show.
* `tiers` - The quality tiers, in order from the dashboard - sorted by the admin, the first being the server default. Clients use this list for the user's tier preference dropdown selection.

<br>

For an out-of-date server the route would return a `404`, so clients should treat a `404` response as the feature being off, and only show the existing `Download` link.

<br>

**Response codes**

| Route | Code | Meaning |
|---|---|---|
| `GET Downloads/Behaviour` | `200 OK` | The feature enabled state, whether to show `Download Original`, and the preference tiers. |
| `POST Items/{itemId}/Download/Prepare` <br> *(Transcoding)* | `200 OK` | Ready. Body: media info of the file, and its ETag. |
| | `202 Accepted` | Transcoding, or queued. Body: progress, or position in the queue. A `Retry-After` header suggests the poll interval. |
| | `500 Internal Server Error` | The transcode failed. Body: state `failed`. |
| | `507 Insufficient Storage` | No designated transcode location has storage capacity for the new video. |
| | `404 Not Found` | Item not found, or not a downloadable video. |
| `GET Items/{itemId}/Download/MediaInfo` <br> *(Manual Curation)* | `200 OK` | Media info of the file that would be served (the optimised copy, or the original if there is none), and its ETag. |
| | `404 Not Found` | Item not found. |
| `GET Items/{itemId}/Download?optimized=true` | `200 OK` | The file, with its `ETag` header. |
| | `206 Partial Content` | A range request, for resuming. |
| | `409 Conflict` | *(Transcoding)* No finished copy yet: call `Prepare`. |
| | `412 Precondition Failed` | The `If-Match` ETag does not match: the file has changed, start again. |
| | `404 Not Found` | Item not found. |

<br>

Permission checks ensure that all routes will return a `401 Unauthorized` response if necessary, and the `Items/{itemId}/Download` routes can return a `403 Forbidden` response, in-line with current behaviour.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 7.1.3. ETag

A strong ETag ensures the exact file requested is the exact file being served. It is provided to clients in the initial response, as well as the file download response, so the server can check they match. It is used for transcoded files, manually curated ones, and the original. It is calculated using the following formula:

`ETag = "SHA-256(<file name with extension>|<size in bytes>|<last modified, UTC ticks>)"`

<br>

---

### 7.2. Transcoding

The resulting transcoded video will contain a single audio track (the audio stream as requested in the API, or the user's calculated default audio stream for that video), every embedded subtitle track, and the chapters. External subtitles are not included, but could be added in a future phase (see [section 8](#8-future-phases)).

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 7.2.1. Transcoding - Download API

A new *Prepare* route should be introduced for the *Prepare then Download* workflow: `Items/{itemId}/Download/Prepare`. It starts the transcode job, or joins a running one, and reports its state - ready, queued (with its position), transcoding (with progress), or failed. Prepare returns a response immediately and is then polled (holding a download request open until the transcode finishes would exceed reverse-proxy and browser timeouts).

`Prepare` should include the currently selected audio stream as a query parameter (e.g. `audioStreamIndex=1`). The parameter is optional and without it, the server will select the audio stream to use based on the user's preferences.

Once `Prepare` returns `ready`, the client would send the download request with the same audio stream included: `GET Items/{itemId}/Download?optimized=true&audioStreamIndex=1`. If the optimised video file isn't ready yet (does not exist, or is awaiting transcoding), the server will respond with a `409 Conflict`.

The file transcoded/returned will match the user's quality tier preference (or server default), the selected audio track downmixed to stereo (AAC by default) with no other audio, and all embedded subtitle tracks and chapters from the video.

Simultaneous transcode jobs are limited by a server setting (see [7.2.2](#722-transcoding---global-settings)). A queue is set up when a new job would go over that limit, first in first out. If a request is received for a video already in the queue, it would join that job. Abandoned queued jobs, determined by lack of polling, are cancelled and removed from the queue.

1. Client sends the request: `POST Items/{itemId}/Download/Prepare?audioStreamIndex=1`
    * Starts a job if a file doesn't exist, or joins an existing job if one is running.
    * Returns `queued` along with the position when transcode job is waiting for a free slot
    * Returns `transcoding` with progress when a file is not ready
    * Returns `ready` when the file can be downloaded
    * Returns `failed` on error
2. Client polls `Prepare` until it returns `ready` or `failed`
3. When `Prepare` returns `ready` it includes the media info describing the video, and the ETag.
4. Client sends the download request `GET Items/{itemId}/Download?optimized=true&audioStreamIndex=1` with `If-Match: <ETag from step 3>`. If the server responds with a `412` (the file changed), or a `409` (not ready), the client returns to step 1.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 7.2.2. Transcoding - Global Settings

The admin will be able to create quality tiers that include a display name, resolution, and bitrate. For example:

* Display name: High (1080p) - free text field
* Resolution: 1080p - from a list of resolutions
* Bitrate: 1500 - number field in kbps

<br>

Tiers are displayed in order of preference and their position can be moved up or down the list. Whichever tier is at the top of the list is considered the "server default".

Each tier has a unique ID generated (GUID) along with a version number, incrementing from 1. These are not displayed in the UI. They are used to generate the transcoded video filename (see [7.2.3](#723-transcoding---directory-structure)), and the version number increments when the tier is edited, unless the "invalidate existing transcodes" option detailed below is disabled.

There will also be a checkbox option where the admin can opt to invalidate existing transcodes when they make changes to the quality tiers. By default this will be off. When off, tier version numbers will not be incremented, preserving existing transcodes when staleness checks are performed. When on and a tier is amended, its version number increments, invalidating all existing transcodes generated with the previous tier values.

When deleting a tier any transcodes generated using it will be deleted. Existing user preferences that had it set as preferred would default to server default.

<br>

The server admin sets the number of simultaneous transcode downloads, which defaults to 1. Transcode download requests beyond this amount will be added to a queue (see [7.2.1](#721-transcoding---download-api)). This limit applies to download transcodes only and is not impacted by playback transcodes.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 7.2.3. Transcoding - Directory Structure

**Transcode Locations**

The admin should have the ability to select multiple storage locations for transcoded files. The UI would re-use the folder picker used by the Library editor, and locations will be set on a *per library* basis, as well as a global collection. For writing, the library set location will be used first, followed by the global locations. Within each collection, the first location is primary, second is secondary etc. Each location has a free-space amount allocation. When the limit is reached, the next location is checked, then the next, and so on. It would be a number field in `GB` that represents the minimum free space to leave on the disk *drive*. When multiple locations share the same drive, the smallest amount set will be used, effectively giving them all the same value. An estimate of the resulting transcoded file will be used to determine how much space is needed.

The unpredictability of the required storage space for as-yet-to-be transcoded files means the system must enable resilience in the event one storage location may be at capacity.

If all given locations are full, the `Prepare` API call will return a `507 Insufficient Storage` response.

<br>

**Locations Example**

Given a `Films` library, that library's settings screen would include an element for selecting multiple locations for storing transcodes. Given a `TV Shows` library, that library's settings screen would have its own. On the Downloads settings screen, a separate list of locations can be set. This allows server admins to have distinct locations for individual libraries, use a single global set of locations, or a mix of both.

<br>

**Locating a Transcode**

Transcode file paths are determined by the video version ID, tier ID, tier version, and the selected audio track (see *Filename/Directory Structure* below). The server can check each location for that file, starting with the library, then the global locations. The first video found that has its finished sidecar matching the expected filepath is served. Any location that is unavailable is skipped entirely.

<br>

**Filename/Directory Structure**

The directory structure used for storing transcoded files, inside of the admin selected root locations, will be similar to the schema used by the Trickplay system. This is to protect against OS path length limits that may be triggered when using something else such as library, film, or video filename, or combination of all.

Resulting extension will be determined by the source video and its compatibility with audio/subtitles.

| Source | Transcode | Why |
|---|---|---|
| `.mkv` | `.mkv` | Supports any subtitle format. |
| `.mp4`, `.m4v`, `.mov` | `.mp4` | They only support `mov_text` subtitles, which are supported in MP4. |
| `.webm` | `.mkv` | WebM only supports VP8/VP9/AV1 video codecs and Opus/Vorbis audio codecs. |
| `.avi` | `.mkv` | Lacks support for modern audio codecs. |
| `.ts`, `.m2ts`, `.mts` | `.mkv` | `ffmpeg` can't write PGS into MPEG-TS, so a new container is required. |
| `.wmv`, `.asf`, `.flv`, `.mpg`, `.mpeg`, `.vob`, `.ogv` and other video types | `.mkv` | Legacy containers with limited media codec and subtitle support. |
| `.iso`, DVD / Blu-ray folders | `.mkv` | No playable extension to use. |

<br>

The formula for determining the transcoded video's `filename` would be as follows:

`filename = <tierId>_v<tierVersion>_a<audioStreamIndex>.<ext>`

* `<tierId>` - Lowercase GUID with no hyphens (`Guid.ToString("N")`).
* `<tierVersion>` - The change version ID of the tier, starts at 1, increments when the tier is edited when *invalidate existing transcodes* ticked.
* `<audioStreamIndex>` - The audio stream index supplied by the client, or the user's calculated default for that video. With Jellyfin's index across all streams, the first audio track is usually 1.
* `<ext>` - See above table.

<br>

Example result:

`8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50_v2_a1.mkv`

<br>

For the directory names, the first two characters of the video version ID (`MediaSourceInfo.Id`) - the `itemId` used by the existing `Download` route - will be used as the parent directory, then the full video version ID as a child directory, within which the transcoded video would be placed.


```
<location>/<first 2 chars of the version id>/<version id>/<filename>
```

Using the video Version ID GUID also solves the issue of a film/episode containing multiple video file versions.

<br>

Example:

```
/path/to/downloads/
└── a1/
    └── a1b2c3d4e5f64a7b8c9d0e1f2a3b4c5d/
        └── 8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50_v2_a1.mkv
```

<br>

Transcoded files should have a companion sidecar `.json` file, describing the encoding details and the details of the source. This is used for staleness checking, and is only created upon successful video transcode. A transcode that lacks a sidecar is either failed or incomplete so is removed by the cleanup.

Example:

```
/path/to/downloads/
└── a1/
    └── a1b2c3d4e5f64a7b8c9d0e1f2a3b4c5d/
        ├── 8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50_v2_a1.mkv
        └── 8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50_v2_a1.mkv.json
```

<br>

---

### 7.3. Manual Curation

#### 7.3.1. Manual Curation - Download API

Manual Curation makes use of the same download links as Transcoding (see [7.1.2](#712-download-api)): the existing `Download` link sends requests to `GET Items/{itemId}/Download?optimized=true`, and the `Download Original` link, when shown, the unchanged route. Curated files already exist so there is no `Prepare` step in the workflow.

The file returned will match the user's quality tier preference (or server default), or if no optimised file exists for the item, the original is served.

1. Client calls `GET Items/{itemId}/Download/MediaInfo`, which returns the description of the file to be served (the optimised copy, or the original if there is none), and its ETag.
2. Client sends the download request `GET Items/{itemId}/Download?optimized=true` with `If-Match: <ETag from step 1>`. If the server responds with a `412` (the file changed), the client returns to step 1.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 7.3.2. Manual Curation - Global Settings

The admin will be able to create quality tiers that include a display name and suffix. For example:

* Display name: "High (1080p)"
* Suffix: "high"

Manually placed files should use the suffix as set in the UI.

A fixed *no suffix* tier is always present for optimised videos named per the original (e.g. `Film (1986).mkv`). It communicates to the admin that suffixes aren't necessary if they only want to store/provide a single optimised version of each video. The suffix field is always empty and read-only, it cannot be removed, but its name can be edited. Besides that, it is a tier like the others.

Note: If both options are implemented, whether Transcoding and Manual Curation share one list of tiers (each tier carrying the fields its option needs) or keep two separate lists is open to discussion.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 7.3.3. Manual Curation - Directory Structure

**Video Locations**

The admin should have the ability to select multiple storage locations for optimised files. The UI would re-use the folder picker used by the Library editor. The first location is primary, second is secondary etc.

The folder layout would mirror the source folder structure, and the download lookup would perform a file system search to locate an optimised video. The Tier suffix the admin sets in the dashboard would be appended to the video filename. Files are named `<source file name> - <suffix>.<ext>`, and names are matched case-insensitively on every operating system, so `Film (1986) - High.mkv` matches the suffix `high`. If tiered videos are found, but not matching the user's preference (or server default), they would be served the next available per the ordering. The *no suffix* tier (see [7.3.2](#732-manual-curation---global-settings)) matches files in the following order:

1. A file named exactly as the source video, e.g. `Film (1986).mkv`.
2. A file whose name starts with the source video's name followed by ` - `, when exactly one does, e.g. `Film (1986) - Extended - 720p.mkv` for the source `Film (1986) - Extended.mkv`, where `720p` is not a defined suffix. If the library holds several versions whose names start the same way (`Film (1986).mkv` and `Film (1986) - Extended.mkv`), the version with the longest matching name claims the file, so each cut is served its own copy and never another cut's.
3. The only video in the folder, when there is just one.

If no tier matches, the original is served.

For example, a movie might be located in a library like so:

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
└── Film (1986).mkv <-- optimised file
```

A download request for that movie would discover the optimised file, and serve that one.

If the admin includes tier-appended files, the tier matching the preference would be served.

```
/Path/To/Downloads/Movies
└── Film (1986)
    ├── Film (1986) - High.mkv
    └── Film (1986) - Standard.mkv
```

<br>

---

### 7.4. Open Questions

Points not yet decided, where a maintainer's view would settle the design:

1. **Tier lists.** If both options are implemented, should Transcoding and Manual Curation share one tier list, or have their own? I would suggest shared, but since fields like bitrate don't apply to curated, perhaps two lists would be better.
2. **Tier storage.** Proposal and MVP store them in XML. If we want to record transcoded videos in the database, should tier lists be stored in the database as well? (See [7.1.1](#711-global-settings-screen).)
3. **Permission for transcoded downloads.** Using "Allow video playback that requires transcoding" is simplest, but should we just have a new user permission for transcode downloads? (See [7.1.1](#711-global-settings-screen).)
4. **Users without that permission.** Serve them the original video, like today, or refuse download?
5. **Audio codec.** The audio is downmixed to stereo: is AAC a suitable default?
6. **A failed transcode.** Should `Prepare` return an error response (`500`) or an ok (`200`) with the state as `failed`? (See [7.1.2](#712-download-api).)
7. **Scheduled pre-transcode.** Which videos, tiers, audio tracks, and should it count towards the limit etc. (See [5.3.3](#533-scheduled-pre-transcode).)
8. **Hardware acceleration.** I recommended transcoded downloads should be unavailable without hardware acceleration (see [4.2.4 - Hardware vs Software](#424-hardware-vs-software)). Should we allow it regardless, with a warning if no hardware acceleration is set?

<br>

## 8. Future Phases

Looking beyond Phase 1 (Transcoding) and the potential of adding Manual Curation in Phase 2, later phases could refine this feature further:

* **Total transcode storage.** A limit to how much space transcodes can use, removing the least recently used first to make room for new.
* **External subtitle files.** Include external subtitle files with the transcode (embed when transcoding).
* **Transcode records in the database.** Store transcode download records in the database, like Trickplay, instead of using a `.json` sidecar.
* **"Ask each time".** A user preference option to always ask which download tier they want when requesting a download.
* **Regenerate transcodes.** A checkbox in Refresh Metadata to regenerate an item's transcodes.
