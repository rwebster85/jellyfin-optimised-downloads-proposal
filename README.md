<!-- TOC ignore:true -->
# Proposal: Optimised Video Downloads <a id="proposal"></a>

Provide clients with the ability to download videos from the library in lower quality, smaller file sizes.

---

<!-- TOC ignore:true -->
**Table of Contents** <a id="toc"></a>

<!-- TOC depthfrom:2 depthto:4 orderedlist:true -->

- [1. Motivation](#1-motivation)
- [2. Constraints](#2-constraints)
- [3. Considerations](#3-considerations)
- [4. Proposed Options](#4-proposed-options)
    - [4.1. Common concepts](#41-common-concepts)
    - [4.2. Option A: Transcoding](#42-option-a-transcoding)
        - [4.2.1. Transcoding Spike](#421-transcoding-spike)
        - [4.2.2. Transcoding Results](#422-transcoding-results)
        - [4.2.3. Pros & Cons](#423-pros--cons)
    - [4.3. Option B: Manual Curation](#43-option-b-manual-curation)
        - [4.3.1. Manual Curation MVP](#431-manual-curation-mvp)
        - [4.3.2. Pros & Cons](#432-pros--cons)
    - [4.4. Recommended Solution](#44-recommended-solution)
- [5. High-Level Design](#5-high-level-design)
    - [5.1. ArchiMate Notation](#51-archimate-notation)
    - [5.2. Common Designs](#52-common-designs)
        - [5.2.1. Service entry point](#521-service-entry-point)
        - [5.2.2. Downloadable file interaction](#522-downloadable-file-interaction)
        - [5.2.3. Settings interaction](#523-settings-interaction)
        - [5.2.4. Full common design](#524-full-common-design)
    - [5.3. Transcoding](#53-transcoding)
        - [5.3.1. Transcode -> Download process](#531-transcode---download-process)
        - [5.3.2. Transcode Cleanup](#532-transcode-cleanup)
    - [5.4. Manual Curation](#54-manual-curation)
- [6. Implementation](#6-implementation)
    - [6.1. Common](#61-common)
    - [6.2. Transcoding](#62-transcoding)
        - [6.2.1. Directory Structure](#621-directory-structure)
    - [6.3. Manual Curation](#63-manual-curation)
        - [6.3.1. Directory Structure](#631-directory-structure)

<!-- /TOC -->

## 1. Motivation <a id="motivation"></a>

Some Jellyfin clients provide the ability to download media for offline play, but in their original formats and filesizes.

Optimised downloading enables downloading media for offline/travel consumption but without the burden of large storage requirements. Libraries may contain high bitrate/resolution files that can't feasibly be stored in large quantities on consumer portable devices such as phones and tablets.

<br>

## 2. Constraints <a id="constraints"></a>

- Video files can come in a variety of container/codec configurations.
- Subtitles can either be embedded or external, in a variety of formats.
- MP4 file embedded subtitles are restricted to text based (MOV_TEXT).
- Transcoding/re-encoding takes time and resources.
- Current transcoding behaviour not equipped to handle download for playback without adaptation.

<br>

## 3. Considerations <a id="considerations"></a>

- Multiple options for server admins could be made available and are not exclusive.
- Options and functionality should be both simple and valuable.
- Jellyfin provides transcoding functionality that could be adapted, reducing technical complexity.
- Still requires significant technical investment and architectural implementation.

<br>

## 4. Proposed Options <a id="options"></a>

What follows is an appraisal of two potential solutions. They could be implemented alongside eachother, or only one of them selected. Conceptually they both share these core concepts:

### 4.1. Common concepts <a id="common"></a>

- Simplified end-user epxerience
- Multiple Quality Tiers
    - Should include options for the server admin to set quality tiers with names (e.g. "High", "Standard") and resolution (1080p, 720p) for better end user UX. [Option A](#option-a) should also include bitrate targets (3Mbps, 800Kbps) for transcoding.
- Multiple optimised download storage locations
    - Admins should be able to select multiple locations for either transcode download or for manual curation file storage.

<br>

---

### 4.2. Option A: Transcoding <a id="option-a"></a>

Client download requests use server transcoding according to download and user preference quality options, to generate a lower-quality, smaller filesize copy. This will be served to the client in-place of the full-size original. Optimised files are stored for future download requests and regenerated when needed.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.1. Transcoding Spike

A spike was carried out w/c 5th October 2026 to research feasibility, implementation, scope, and upstream impact. The work was branched off my `optimised-downloads` branch (manual curation MVP) with the manual logic removed.

The results of this work can be viewed as follows: <a href="https://github.com/rwebster85/jellyfin/tree/transcode-downloads">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/transcode-downloads">Web</a>.

*Source Code Notes*

- Encoding is fixed at `mkv h264 1280x720 750k aac 2ch 192k`. See `DownloadTranscoder.cs` <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L56">line 56</a> and <a href="https://github.com/rwebster85/jellyfin/blob/d748fc95051c1d2eac464ac665d5dfafe462f450/Jellyfin.Api/Helpers/DownloadTranscoder.cs#L249">line 249</a>
- Some features and UI elements may remain from the manual branch but are non-functional
- Unlike the MVP, the spike did not cover Android, but implementation would be broadly similar.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.2. Transcoding Results

Transcoding depends entirely on the hardware (or software) solution available on the server. As such, a wide variety of configurations will produce varying results, ranging from potential image quality differences to time-to-transcode delay.

To establish a baseline for cost I tested encoding from two different sources on two different machines. One machine used a Windows 11 machine RTX 5090 with NVENC encoding, the other used a Linux Mint system with an i5-12600 and Intel Quicksync hardware transcoding. Both runs used `ffmpeg` commands as per what Jellyfin would generate with baseline settings.

*Results Notes*

- Both encoders were idle besides the running task, so real-world result will vary.
- Jellyfin caps resolution to gates based on bitrate. This was not adjusted in the spike, but implementation should override that behaviour.

<br>

**On-demand transcode download**

Source: 1920×822, 7.2 GB, 1:54:43, HEVC Main 10 SDR.

Command: H.264 at 960×410, 750 kbps video, AAC stereo at 192 kbps, with the encoder preset on `auto`.

| | RTX 5090, NVENC (desktop) | i5-12600, QuickSync (server) |
|---|---|---|
| Decode / scale / encode | CUDA / `scale_cuda` / `h264_nvenc -preset p1` | VAAPI / `scale_vaapi` / `h264_qsv -preset veryfast` |
| Speed | **80x** (1,922 fps, average of 2 runs) | **42.9x** (1,030 fps) |
| Time for the whole film | **1 min 26 s** | **2 min 40 s** |
| Output | 771 MiB, 939.5 kbps | 773 MiB, 941.6 kbps |

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.2.3. Pros & Cons

* **Pro:** Scalable - Server admin is not concerned with managing these optimised downloads.
* **Pro:** Automated - Background processes handle generation and removal.
* **Pro:** Self contained - Everything happens on the server.
* **Con:** Real-time cost - Transcoding takes a not-insignificant amount of resources.
* **Con:** Unpredictable quality - Bitrate and resolution aren't obvious indicators of quality produced, and different hardware can produce different results.
* **Con:** Unpredictable storage requirements - Storage requirements are unknown, and storage capacity is highly expensive.

<br>

---

### 4.3. Option B: Manual Curation <a id="option-b"></a>

Server admins generate/source their own lower-quality files according to their tastes, and store them on the server for the download system to serve. These files would live outside of the library, pointed to in settings, and their directory structure would mirror the library.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.3.1. Manual Curation MVP

Beginning w/c 21st September 2026, an MVP of this feature was implemented in Server as well as Web and Android clients. Some logic is heavily constrained by my own library naming conventions for filename matching and lookup, but aside from that the implementation could broadly match. See the `optimised-downloads` branches in my forks: <a href="https://github.com/rwebster85/jellyfin/tree/optimised-downloads">Server</a>, <a href="https://github.com/rwebster85/jellyfin-web/tree/optimised-downloads">Web</a>, <a href="https://github.com/rwebster85/jellyfin-android/tree/optimised-downloads">Android</a>.

This MVP served as an immediate use-case for my own personal use and has been tested in a real-world setting. Optimised downloads were created using a custom encoding workflow tool of my own creation, tailored to my needs, from my own library source videos.

**Testing Notes**

- Client devices tested incldue a Samsung Galaxy S23 and a Lenovo Y700 Gen 3 (Legion) tablet
- My own use-case preferred the `Substitute` download link option, rather than `SeparateAction`. See <a href="https://github.com/rwebster85/jellyfin/blob/transcode-downloads/MediaBrowser.Model/Configuration/DownloadBehaviour.cs">DownloaderBehaviour.cs</a>

<br>

**Encoder comparison**

Dune (2021), BluRay 1080p source - same 300-second segment at 1920×804 - My re-encoding tool, for comparison

| Encoder | Setting | Speed | Bitrate | VMAF vs source |
|---|---|---|---|---|
| RTX 5090, NVENC | CQ 24 | 25.6x | 1.626 Mbps | – |
| i5-12600, QuickSync | ICQ 20, about the same quality | 6.7x | 1.403 Mbps | – |
| RTX 5090, NVENC | 3 Mbps target | – | 2.558 Mbps | 82.93 |
| i5-12600, QuickSync | 3 Mbps target | – | 2.998 Mbps | 83.40 |

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 4.3.2. Pros & Cons

* **Pro:** Highly curated - Server admin has a hand in each file/format.
* **Pro:** Predictable results - Files are chosen by the admin and can be checked for quality issues.
* **Pro:** Tailored quality - Files can be tailored for their requirements, such as darkness, fast-moving action, or high frequency film grain.
* **Con:** Time intensive - Manual curation and production takes time
* **Con:** High storage requirements - Multiple quality tiers for multiple video versions required.
* **Con:** Not scalable - Every new video potentially requires companion optimised versions manually sourced.

<br>

---

### 4.4. Recommended Solution <a id="recommendation"></a>

Based on the research and results of the MVP and spike, **[Option A: Transcode -> Download](#option-a)** is the clear choice for friction vs value for server admins. It has less work compared to manual curation, scales better, and integrates well with the existing systems in Jellyfin.

However, both options provide value for different use-cases. As such my suggestion would be to implement both options together in phases. [Option A - Transcoding](#option-a) should be implemented in Phase 1, and if accepted, [Option B - Manual Curation](#option-b) in Phase 2.

<br>

## 5. High-Level Design <a id="hld"></a>

Until an option to the proposed solution is agreed, what follows will be a high-level logical design focused on system and object interactions. When an option is agreed, low-level concrete designs will be produced for the implementation.

### 5.1. ArchiMate Notation

The high-level design (HLD) uses the ArchiMate enterprise architecture modelling language to represent the different elements within the system and their relationships with eachother.

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

### 5.2. Common Designs

Both transcoding and manual curation would share some design elements.

#### 5.2.1. Service entry point

<br>

![Common entry point design](/images/hld-common-entry-point.png)

**Notes**

- "Helper" is a service named as per the Jellyfin convention.
- Clients tested are Web and Android. Others that support downloading should be assessed.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.2.2. Downloadable file interaction

<br>

![Downloadable file interaction design](/images/hld-common-download.png)

**Notes**

- Original file being served when selected or no applicable version available.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.2.3. Settings interaction

**Note:** Some of this design corresponds with my MVP and spike, in terms of settings and configuration storage. This is subject to change following an agreement on the appraisal and implementation strategy.

<br>

![Common settings design](/images/hld-common-settings.png)

**Notes**

- Helper handles user preferences, backend download options use the backend services.

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

### 5.3. Transcoding

Definitions:

**Stale**

* A transcode file matching a video in the library that was removed/renamed/changed.
* A transcode file that no longer matches the tier resolution and/or bitrate that created it, if the admin opts for that setting. By default, tier changes will not make stale any existing transcodes.

<br>

<p align="left">─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─</p>

#### 5.3.1. Transcode -> Download process

1. Download locations are configured by the admin in settings, or the server default location used.
2. User download request is captured by the controller.
3. If a transcoded file matching the user's quality tier preference (or server default) exists, and is not stale, the file is served.
4. Otherwise, if a transcode job isn't already running, configured against the user's quality tier preference (or server default) and the corresponding resolution and bitrate set by the admin, start a transcode job.
5. File is streamed in packets to the client until completed.
6. If a download request is cancelled, the transcoding job continues in the background.
7. If a download request is initiated while the video is being transcoded, the request joins the existing job and is streamed the file from the start.
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

---

### 5.4. Manual Curation

1. Download versions of videos are placed on the server by the admin.
2. Download locations are configured by the admin in settings.
3. Download requests are captured by the controler.
4. If an optimised video is found matching the user's quality tier preference (or server default), that video is served.
5. If no optimised video is found, the original file will be served instead.

<details>
    <summary>Show system diagram</summary>

<br>

![Manual curation download design](/images/hld-manual-download.png)

</details>

<br>

## 6. Implementation  <a id="implementation"></a>

The following implementation details only reflect what is necessary for appraisal of the options in this proposal. As with the HLD, once an option has been agreed, additional concrete implementation details will be provided.

### 6.1. Common

**Download API**

The admin could be provided with the option of having all requests go through the existing API (`Items/{itemId}/Download`), or having "Optimised Downloads" use a secondary API/link. This proposal suggests the former, which matches the `Substitute` download type in my forks. This would mean a smaller upstream impact to existing Jellyfin clients, and would be the better option from a UX perspective.

**Clients**



### 6.2. Transcoding

#### 6.2.1. Directory Structure

**Transcode Locations**

The admin should have the ability to select multiple storage locations for transcoded files. The UI would re-use the folder picker used by the Library editor. The first location is primary, second is secondary etc. We would need to enable a remaining storage limit amount for each location, which when met would then store files in the second location, then the third etc.

The unpredictability of the required storage space for as-yet-to-be transcoded files means the system must enable resiliancy in the event one storage location may be at capacity.

**Directory Structure**

The directory structure used for storing transcoded files, inside of the admin selected root locations, should match the schema used by the Trickplay system. This is to protext against OS path length limits that may be triggered when using something else such as library, film, or video filename, or combination of all.

Using the video Version ID GUID also solves the issue of a film/episode containing multiple video file versions.

For example:

```
/path/to/downloads/
└── a1/
    └── a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d/           <- one video (one version of a film)
        ├── 8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50.mkv        <- its transcode at one tier
```

<br>

Transcoded files should have a companion side-car `.json` file, describing it's encoded details. This is used for staleness checking, and is only created upon successful video transcode.

For example:

```
/path/to/downloads/
└── a1/
    └── a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d/           <- one video (one version of a film)
        ├── 8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50.mkv        <- its transcode at one tier
        └── 8d3a6f1e7c4b4a2d9e5f1b0c2d3e4f50.json       <- "finished" record for that transcode
```

<br>

### 6.3. Manual Curation

#### 6.3.1. Directory Structure

**Video Locations**

The admin should have the ability to select multiple storage locations for optimised files. The UI would re-use the folder picker used by the Library editor. The first location is primary, second is secondary etc.

The folder layout would mirror the library, and the download lookup would perform a file system search to locate an optimised video. Tier names the admin sets in the dashboard would be appended to the video filename. However, if no files are found matching the user's preferred tier (or server default), but there is a single video file found, that one would be served.

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
