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
    - [5.1. Option A: Transcode -> Download](#51-option-a-transcode---download)
        - [5.1.1. Pros](#511-pros)
        - [5.1.2. Cons](#512-cons)
    - [5.2. Option B: Curated hand-placed files](#52-option-b-curated-hand-placed-files)
        - [5.2.1. Pros](#521-pros)
        - [5.2.2. Cons](#522-cons)
    - [5.3. Option 1. Curated video library of companion files for download, outside of Jellyfin library](#53-option-1-curated-video-library-of-companion-files-for-download-outside-of-jellyfin-library)
        - [5.3.1. Example](#531-example)
        - [5.3.2. Pros](#532-pros)
        - [5.3.3. Cons](#533-cons)
    - [5.4. Option 2. Transcode on-the-fly per download request](#54-option-2-transcode-on-the-fly-per-download-request)
        - [5.4.1. Pros](#541-pros)
        - [5.4.2. Cons](#542-cons)
- [6. Proposed Solution](#6-proposed-solution)
- [7. Design & Implementation](#7-design--implementation)

<!-- /TOC -->

## 2. Motivation

Some Jellyfin clients provide the ability to download media for offline play, but in their original formats and filesizes.

Optimised downloading enables downloading media for offline/travel consumption but without the burden of large storage requirements. Libraries may contain high bitrate/resolution files that can't feasibly be stored in large quantities on consumer portable devices such as phones and tablets.

## 3. Constraints

- Video files can come in a variety of container/codec configurations
- Subtitles can either be embedded or external, in a variety of formats
- MP4 file embedded subtitles are restricted to text based (MOV_TEXT)
- Transcoding/re-encoding takes time and resources
- Current transcoding behaviour not equipped to handle download for playback
- 

## 4. Considerations

- Several options for server admins could be made available and are not exclusive
- Options and functionality should be both simple and valuable
- Jellyfin provides transcoding functionality that could be adapted, reducing technical complexity
- Still requires significant technical investment and architectural implementation

## 5. Proposed Options

### 5.1. Option A: Transcode -> Download

Client download requests use server transcoding according to download and user preference quality options, to generate a lower-quality, smaller filesize copy. This will be served to the client in-place of the full-size original. Options for the server admin include mutliple quality tiers with names (e.g. "High", "Standard"), resolution (1080p, 720p) and bitrate (3 Mbps, 800 Kbps) targets. Users could select their preference from the list of quality tiers in their user settings. Optimised files are stored for future download requests and regenerated when needed.

#### 5.1.1. Pros

- Scalable - Server admin is not concerned with managing these optimised downloads.
- Automated - Background processes handle generation and removal.
- Self contained - Everything happens on the server.

#### 5.1.2. Cons

- Real-time cost - Transcoding takes a not-insignificant amount of resources.
- Unpredictable quality - Bitrate and resolution aren't obvious indicators of quality produced, and different hardware can produce different results.
- Unpredictable storage requirements - Storage requirements are unknown, and storage capacity is highly expensive.


### 5.2. Option B: Curated hand-placed files

Server admins generate their own lower-quality files according to their tastes, and store them on the server for the download system to serve. These files would live outside of the library, pointed to in settings, and their directory structure would mirror the library.

#### 5.2.1. Pros

- Highly curated - Server admin has a hand in each file/format.
- Predictable results - Files are chosen by the admin and can be checked for quality issues.
- Tailored quality - Files can be tailored for their requirements, such as darkness, fast-moving action, or high frequency film grain.

#### 5.2.2. Cons

- Time intensive - Manual curation and production takes time
- High storage requirements - Multiple quality tiers for multiple video versions required
- Not scalable - Every new video potentially requires companion optimised versions manually sourced

### 5.3. Option 1. Curated video library of companion files for download, outside of Jellyfin library

Server admins are responsible for curating their own files, and deciding on the format/version available.

Application settings provided for admins to specify locations for a download file lookup. Option provided to enable a new download API/menu option, or use the existing one. Downloadable files must match their video library counterpart folder structure.

#### 5.3.1. Example

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


#### 5.3.2. Pros

- Curated files bring no unpredictability. They're sourced/generated by the admin
- Flexible: admins can offer a single or multiple options for end users
- Mirrors streaming service UX and infrastructure: pre-encoded downloadable versions of content and different quality levels

#### 5.3.3. Cons

- Difficult to scale without automated tools for admins to make use of
- Additional complexity with library management
- Automated 3rd party library tools may need to be adjusted, or possibly unsuitable to integrate


### 5.4. Option 2. Transcode on-the-fly per download request

A downloadable file is generated/streamed (using HLS) to the client device when a download is requested. Quality could be set by the server admin using tiers, like Option 1, for simplicity. Less complicated than users selecting their preferred bitrate.

Technically highly complex and requires not-insignificant integration with client applications. Could downloads be paused? What happens on a connection drop? Are part downloads deleted/cleaned up?

Assuming quality tier options like Option 1, potential for storing the transcoded file at the same time as first serve (Just-In-Time Transcoding with Caching). The first downloading user generates the file, all proceeding requests would download the file without needing transcoding. Although this may prevent HLS muxing.

#### 5.4.1. Pros

- Potentially scalable
- No manual work required of admins

#### 5.4.2. Cons

- Unpredictable, difficult to guarantee the stream ended with success for the user
- Transcoding is expensive. Users may initiate dozens of downloads that may all need to be transcoded (potentially alleviated with a queue system, requires client integration)
- Potentially long first time download


## 6. Proposed Solution

Mixture of both, with a phased approach.

...

## 7. Design & Implementation

Until an implementation to the proposed solution is decided, what follows will be a high-level, logical design. When an implementation is decided, low-level concrete designs will be produced.
