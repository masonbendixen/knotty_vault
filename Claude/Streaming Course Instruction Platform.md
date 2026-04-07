---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/7/2026
Version: 0.3
tags: 
---
# Overview

I am working on an idea for a business proposal and want help fleshing it out. I am getting ready to open:

- A fitness business
	- Strength training
	- Flexibility
	- Yoga
	- Aerial acrobatics
	- Partner acrobatics
- A spa with service like massage and self care
- A combo yoga teacher training / massage therapist certification program

---

# Streaming Course Instruction Platform

## Executive Summary

This document proposes a multi-phase platform for producing, distributing, and licensing fitness instructional content. The MVP phase will serve Knotty Yoga's own content needs alongside 1-2 beta creator partners, proving the product before scaling to additional content creators, studios, and eventually a marketplace of subscribers. The platform targets a global audience from day one, with initial beta creators including partners in Canada.

The platform addresses a clear gap in the fitness technology landscape: no existing product combines **high-quality produced instructional video**, **interactive live Q&A sessions**, and **B2B studio content licensing** in a single offering. Creators currently cobble together Patreon (payments/community), Zoom (live classes), and YouTube/Vimeo (hosting) — a fragmented experience that produces low-quality results for both creator and student. This platform positions as a clean replacement for Patreon — not an integration or supplement, but a superior alternative that creators migrate to.

The specialized modalities at play — aerial acrobatics, partner acrobatics, flexibility, and yoga — have virtually no dedicated digital infrastructure. This is both a niche opportunity and a natural moat: the content requires genuine expertise, safety training, and multi-angle demonstration that generic fitness platforms cannot replicate.

---

## The Problem

### For Content Creators

Athletic and movement content producers today rely on a patchwork of tools:

- **Patreon** for payment collection — but it is fundamentally a blog platform. Videos are buried in a chronological feed with no curriculum structure, no progress tracking, and no search. Video hosting is basic. There is no live streaming capability.
- **Zoom** for live classes — but recordings are low quality, storage is limited (creators must delete recordings within 1-2 weeks), and the resulting dead-link graveyard in Patreon posts degrades the entire content library. Videos are unbranded, unprofessional, and excessively long because instructional content and student Q&A are interleaved into a single session.
- **YouTube/Vimeo** for hosting — but these offer no paywall integration without third-party plugins, no curriculum structure for progressive learning, and no interactivity.

The result: creators spend enormous effort producing content but deliver it through tools that make it look amateurish, provide no structured learning path, and lose archival value within weeks.

### For Students

- Content is hard to navigate — you cannot browse a structured curriculum or track what you have completed.
- Live classes mix instruction and Q&A, making them 2-3x longer than necessary for the instructional content they contain.
- Recordings disappear due to storage limits, so you must watch immediately or lose access.
- No way to prepare questions in advance, leading to lower-quality live interactions.
- Video quality is poor (Zoom compression, single camera angle, no overlays or multi-angle views).

### For Studios

- Studios want fresh programming and teacher development but cannot afford Les Mills-style licensing ($8,000-$20,000+/year per program suite).
- Independent instructors have great content but no distribution infrastructure to reach studios.
- There is no marketplace connecting content creators with studios that want to license and teach their material.
- Studio instructors lack a structured continuing education pathway for specialized modalities like acrobatics.

---

## Market Context

### Industry Landscape

| Metric | Value |
|--------|-------|
| Global fitness industry (2023) | $87-96 billion |
| Digital fitness market (2023) | $16-20 billion |
| Digital fitness CAGR through 2030 | 15-20% |
| Hybrid member retention vs. single-channel | 2-3x higher |
| Studios offering some digital content (post-COVID) | 30-40% |
| Creator economy (total, 2023) | ~$250 billion |

The hybrid fitness model — combining in-person and digital — is the fastest-growing segment. Gyms offering hybrid models report 15-25% higher member retention. Pure digital streaming (the Peloton model) has shown high churn and difficult unit economics, but digital as a value-add to physical studios is highly viable.

### Competitive Landscape

**No existing platform combines all three pillars**: produced instructional video + interactive live Q&A + B2B studio licensing.

| Platform | Produced Video | Live Interactive | Payments | B2B Licensing | Creator Tools |
|----------|:-:|:-:|:-:|:-:|:-:|
| Patreon | Basic feed | No | Yes | No | Yes |
| Uscreen | Excellent | Broadcast only | Yes | No | Yes |
| Teachable / Thinkific | Course structure | No | Yes | No | Yes |
| Kajabi | Course structure | Limited | Yes | No | Yes |
| Zoom | No | Yes (2-way) | No | No | No |
| Les Mills | Licensed | Virtual (screen) | B2B | Yes (own content) | Closed |
| Alo Moves / Glo | Excellent | No | Sub only | No | Closed |
| Mindbody | Minimal | Zoom embed | Yes | No | No |
| **This Platform** | **Yes** | **Yes (WebRTC)** | **Yes** | **Yes (open)** | **Yes** |

**Key gaps this platform fills:**

1. **Live interactive + on-demand in one product.** Uscreen comes closest to a full creator video platform, but its live streaming is broadcast-only (no two-way video for form correction or Q&A where you see the student). No platform truly combines produced on-demand content with low-latency interactive sessions where the host can bring audience members on screen.

2. **B2B studio content licensing for independent creators.** Les Mills proved the model — licensing choreographed programs to 21,000+ studios globally at $500-2,500/program/quarter. But it is a closed ecosystem: only Les Mills' own programs. There is no open marketplace where an independent acrobatics instructor can license content to studios.

3. **Specialized modality support.** Yoga, acrobatics, and partner work need multiple camera angles, progression tracking, safety notes, and spotting demonstrations. No platform is designed for these modalities.

4. **Structured learning paths for physical skills.** Course platforms (Teachable, Thinkific) have sequential lesson structures designed for information delivery, not physical skill acquisition. No platform understands progressive skill gating appropriate for movement arts ("you must be able to do X before attempting Y").

---

## Content Needs (Knotty Yoga)

The MVP phase focuses on Knotty Yoga's own content while also onboarding 1-2 beta creator partners to validate the platform against real external needs:

| Content Type | Description | Distribution |
|---|---|---|
| Self-care series | Body maintenance, recovery, mobility work for the general public | Public / free tier |
| Class streaming | Live-streamed studio classes for remote members | Members |
| Acrobatics prep | Strength and conditioning progressions for partner and aerial acrobatics | Members / course |
| Partner massage | Guided partner massage and bodywork sequences | Members / course |
| Teacher training curriculum | Portions of the yoga teacher training and massage certification programs | Enrolled students |

This diverse set of needs — spanning free public content, member-only streaming, structured courses, and accredited certification material — means the platform must support multiple content types and access levels from day one. A **free tier** will be available for discovery and marketing purposes — select content from creators will be publicly accessible to drive awareness and conversion to paid tiers.

---

## Phase 1: Creator Content Production and Distribution

### Core Concept

Build a desktop application that automates the creation of professional instructional videos, then pair those videos with structured live Q&A sessions delivered through the web platform.

The key innovation is **separating instruction from interaction**. Instead of a single live session where the instructor teaches and answers questions simultaneously (the current Zoom model), the workflow becomes:

1. **Record** → Instructor records the instructional content at their own pace, with multiple takes if needed.
2. **Edit** → The app assists with post-production: trimming, voice-overs, overlays, multi-angle compositing, branded intro/outro.
3. **Publish** → The polished instructional video is published to the platform.
4. **Watch** → Students watch the video on their own time, try the material, and come prepared with questions.
5. **Q&A** → A scheduled live Q&A session is conducted with high-quality interactive video. Students ask informed questions because they have already studied and attempted the material.
6. **Archive** → Both the instructional video and the Q&A session (re-edited for clarity) become permanent library content.

This produces better outcomes for everyone: instructors create higher-quality content, students come prepared with better questions, and the archived material is more useful and professional.

### Recording Capabilities

The recording application must support:

- **Multi-source capture**: Synchronize recording from multiple camera angles (front, side, overhead for aerial work) plus separate audio from a dedicated microphone. All sources captured at full local quality regardless of any simultaneous streaming.
- **Scene management**: Pre-configured layouts (full-screen single camera, split-screen two cameras, picture-in-picture) with one-click switching during recording. Essential for acrobatics where you need both a wide shot and a close-up of hand/foot placement simultaneously.
- **Live stream + high-quality local capture**: Stream a lower-bitrate version to viewers while simultaneously recording full-quality local files. The stream lets remote viewers watch live; the local files are the source material for the edited product.
- **Audio isolation**: Separate audio tracks for the instructor's microphone, ambient room audio, and any music — allowing independent mixing and cleanup in post-production.

### Editing Capabilities

The editing mode is **not a full non-linear editor** — that would be an enormous engineering effort better left to tools like DaVinci Resolve. Instead, it is a focused, guided post-production workflow built around the platform's core strength: multi-source video management.

**Multi-source layout editing** (the primary editing capability):
- All captured video/audio sources from a recording session are available on a synchronized timeline, aligned by their timestamp information.
- The creator can scrub through the timeline and set **layout transition points** — choosing at each point which source arrangement to display:
  - **Full-screen**: Any single source fills the frame (e.g., overhead camera, front camera, or an audience member's video stream during Q&A).
  - **Picture-in-picture**: A primary source fills the frame with a secondary source inset in a corner (e.g., instructor full-screen with audience member PIP, or wide shot with close-up PIP).
  - **Split-screen**: Two sources displayed side-by-side (e.g., front and side camera angles simultaneously).
- During a live stream, the creator toggles between these layouts in real time. But crucially, **all individual streams are saved** so the creator can go back in edit mode and re-decide every layout transition with the benefit of hindsight. The live switching decisions are just the initial defaults — everything is re-editable.
- This same workflow applies to both instructional recordings (multiple camera angles) and Q&A sessions (host stream + multiple audience member streams).

**Voice-over and PIP commentary**:
- The creator can record new voice-over narration at any point on the timeline — for example, adding explanation over a slow-motion replay of a technique.
- The creator can record **PIP commentary** — a new video+audio recording of themselves that is overlaid as a picture-in-picture on the existing footage. This allows the creator to "commentate" on their own performance, point out details, or add teaching notes after the fact.

**Additional editing features**:
- **Timeline trimming**: Remove false starts, dead air, off-topic tangents, and retakes.
- **Overlay insertion**: Add text annotations, arrows, safety callouts, or anatomical diagrams at specific timestamps. Critical for acrobatics content where spotting positions and hand placement need highlighting.
- **Slide/presentation insertion**: Insert presentation-style material (anatomy slides, progression charts, theory content) between video segments.
- **Branded intro/outro**: Automatically prepend and append branded intro and outro sequences with the class title, instructor name, level, and duration dynamically inserted.
- **Audio post-processing**: Noise reduction, loudness normalization, and optional background music mixing.

### Live Q&A Session

The Q&A session is delivered through the web platform and operates differently from a standard video call:

- **Host broadcasts via WebRTC** with low latency to all connected viewers.
- **Audience members watch via HLS** (adaptive bitrate streaming through CDN) for scalability.
- **"Raise hand" mechanism**: An audience member clicks to request to ask a question. The host sees a queue of requests.
- **Audience promotion**: When the host accepts a question, the audience member is promoted to a WebRTC connection with the host. Their video and audio are included in the broadcast.
- **Layout control**: The host can switch between layouts for the promoted audience member:
  - Host full-screen (audience audio only)
  - Audience member full-screen (host audio only)
  - Split-screen (side by side)
  - Host full-screen with audience PIP (picture-in-picture)
  - Audience full-screen with host PIP
- **Session ending**: The host can end the audience member's active connection at any time and promote the next person in the queue.
- **Source capture**: The host video and every audience member's video stream are captured as separate files. This enables full re-editing of the Q&A session after the fact.

### Q&A Post-Production

The recorded Q&A session can be re-edited using the same editing tools:

- All participant streams are available as separate video sources.
- The host can re-choose layout transitions (when to go split-screen vs. PIP vs. full-screen) with the benefit of hindsight.
- Redundant or low-quality questions can be trimmed out.
- The result is a polished Q&A companion video that pairs with the instructional video.

---

## Phase 2: Studio Licensing (B2B)

### Core Concept

Expand the platform from serving individual students to serving **studios**. Content creators produce material and make it available for studio licensing. Studios subscribe, enroll their instructors, and use the content in their classes.

This is the Les Mills model — but open to independent creators rather than locked to a single company's programs.

### How It Works

1. **Content creator produces a course** — a structured series of instructional videos with progression (e.g., "6-Week Partner Acrobatics Foundations").
2. **Studios subscribe** to access the content library or specific courses.
3. **Studio instructors** are enrolled by the studio and review the content independently — watching the instructional videos, attempting the material, and studying the teaching methodology.
4. **Instructor Q&A sessions** — The content creator runs Q&A sessions specifically for studio instructors (not students). These sessions use the same interactive format from Phase 1. Instructors ask about teaching methodology, common student mistakes, progression decisions, and safety considerations.
5. **In-studio class delivery** — The local studio instructor teaches classes using the content creator's material:
   - Plays the content creator's video progressions on a studio screen
   - Pauses playback at key points
   - Demonstrates locally and provides hands-on corrections
   - Guides students through the material with in-person support
6. **Relationship building** — Students become familiar with the content creator through the videos. This creates opportunities for:
   - In-person workshops by the content creator at the studio
   - Student-tier platform memberships (lower price, video-only access)
   - Direct enrollment in the creator's own classes or training programs

### Studio Benefits

- **Fresh programming** without the cost of sending instructors to expensive external trainings.
- **Structured teacher development** — instructors learn from experts in specialized modalities they could not otherwise access.
- **Reduced liability** — content from credentialed creators with safety protocols built into the material.
- **Marketing differentiation** — "We offer [Creator Name]'s Aerial Foundations program" is a compelling studio differentiator.

### Creator Benefits

- **Recurring B2B revenue** that is more stable than individual subscriptions.
- **Reach** — one creator can have their content taught in dozens of studios simultaneously.
- **Workshop pipeline** — established studio relationships create warm audiences for premium in-person events.
- **Brand building** — students in multiple cities see and learn from the creator's content.

### Pricing Model

**Decision: prioritize accessibility.** Most acrobatics studios are small operations that cannot afford Les Mills-style pricing ($500-2,500/program/quarter). The initial pricing must be low enough that a small studio can try it without significant financial risk. As the platform scales to higher-volume modalities like general yoga, pricing can be revisited upward.

| Tier | Target | Price Range | Includes |
|------|--------|-------------|----------|
| Single Course | Small studio trying one program | $50-150/month | 1 course, 1-2 instructor seats, scheduled Q&A access |
| Studio Library | Medium studio wanting variety | $200-500/month | Full library access, 3-5 instructor seats, priority Q&A |
| Enterprise | Multi-location or franchise | Custom | Unlimited, custom branding, API access, dedicated support |

---

## Phase 3: Platform Marketplace

### Core Concept

After building a critical mass of content creators and studios, open a **platform-wide subscription** for individual consumers. Subscribers pay a single monthly fee for access to content from all participating creators. Revenue is shared with creators based on viewership.

### Revenue Sharing Model

The industry standard is the **pro-rata model** (Spotify, YouTube): all revenue goes into a pool and is distributed based on each creator's share of total platform viewership. This systematically disadvantages niche creators.

A **user-centric payment model** (Deezer's approach since 2023, SoundCloud's "Fan-Powered Royalties") is fairer for specialized content: each subscriber's payment is distributed only to the creators that specific subscriber actually watched. A subscriber who watches exclusively acrobatics content would have their entire subscription payment distributed among acrobatics creators, not diluted into mainstream yoga.

**Recommended: user-centric model.** This is a competitive differentiator and aligns incentives — creators are rewarded for building loyal audiences, not just racking up casual views.

### Platform Take Rate

The goal is to **strongly appeal to content creators** — a generous split is a key differentiator for attracting and retaining the best creators in a niche market. The current thinking is between 70/30 and 80/20 (creator/platform).

| Platform | Take Rate | Creator Gets |
|----------|-----------|-------------|
| YouTube (ads) | 45% | 55% |
| Udemy (organic) | 63% | 37% |
| Coursera | 55-70% | 30-45% |
| Spotify | 30-35% | 65-70% |
| App stores | 15-30% | 70-85% |
| **Option A** | **30%** | **70%** |
| **Option B** | **20%** | **80%** |

An **80/20 split** would be a headline differentiator — "creators keep 80%" is a compelling pitch against Patreon (which takes 5-12% but offers far less). The trade-off is less platform revenue to fund infrastructure and growth. A **70/30 split** is still very competitive (better than Spotify, YouTube, Coursera, and Udemy) and leaves more room for platform investment.

**Open decision** — this needs further discussion based on projected infrastructure costs and growth targets. See Open Questions below.

### Opt-In Structure and Content Exclusivity

**Decision: no exclusivity requirements.** Creators are not locked in and are not required to remove content from other platforms. They will likely need to coexist on Patreon during a transition period (or indefinitely). YouTube and Instagram remain critical for discovery and driving traffic to the platform.

Creators can:

- Keep some courses exclusive to their own direct subscribers (premium content, certifications).
- Place selected courses in the platform marketplace for broader reach and discovery.
- Set their own pricing for direct subscriptions while the marketplace has a single platform price.
- Continue to publish free/teaser content on YouTube and Instagram for audience building.

This gives creators control and reduces the switching cost. The platform wins by being genuinely better, not by locking people in.

### Positioning and Branding

**Decision: creator-first branding.** In the early phases, the creator is the brand — each creator's platform page is their storefront with their branding, their domain (or subdomain), and their identity. The platform is the infrastructure, not the destination.

As the creator base grows and the Phase 3 marketplace launches, the platform develops its own brand identity as a curated destination. But the initial pitch to creators is: "this is *your* platform, with *your* brand."

### Quality Control

**Decision: curated marketplace.** Creators will be vetted for quality — this is not an open marketplace where anyone can publish. Initially, creators are hand-picked by invitation and referral. As the platform grows, an application and review process will be established. This ensures a consistent quality bar (the Alo Moves model, not the Udemy model).

---

## Phase 4 (Future): Certification and Accreditation

**Decision: defer to later phases.** The technical platform comes first. Certification will initially be handled entirely by referral and hand-picked invitation, not through platform infrastructure. The features below describe the eventual vision once the core platform is proven.

### Opportunity

Given Knotty Yoga's plan for a yoga teacher training / massage therapist certification program, the platform could eventually support structured certification delivery.

### Regulatory Landscape

**Yoga teacher training (Yoga Alliance)**:
- Up to 50% of contact hours can be delivered online (must be synchronous/live — asynchronous content is supplementary only).
- Remaining 50% must be in-person.
- Practical assessment must be in-person.
- The platform's Q&A sessions (live, interactive, with video) qualify as synchronous contact hours.

**Massage therapy certification**:
- State-regulated, typically requiring 500-1,000+ hours of training.
- Most states allow 25-40% of didactic (theory) hours online.
- Hands-on hours must be 100% in-person.
- The platform could deliver the theory portion, but in-person intensives are mandatory.

### How the Platform Supports This

The Phase 1 and Phase 2 features naturally support a hybrid certification model:

1. **Asynchronous theory** — Produced instructional videos for anatomy, philosophy, history, business.
2. **Synchronous live sessions** — Q&A sessions with video interaction count as contact hours under Yoga Alliance rules.
3. **In-person intensives** — Scheduled at the studio for hands-on work, practical assessment, and practice teaching.
4. **Progress tracking** — The platform tracks which content a student has completed, which live sessions they attended, and assessment results.
5. **Video submission** — Students submit recorded practice teaching for remote assessment.
6. **Certification issuance** — Upon completion, the platform issues a credential and records continuing education.

This hybrid model makes certification programs accessible to students who cannot relocate for full-time training, while maintaining the in-person standards required by accrediting bodies.

### Insurance and Credentialing Moat

For acrobatics and aerial work — modalities with real injury risk — a credible certification creates a natural business moat:

- If insurance companies recognize the certification as risk-reducing, studios have a financial incentive to hire only certified instructors.
- This creates demand for the certification program, which creates demand for the platform content, which creates demand for the studio licensing tier.
- The entire ecosystem reinforces itself.

---

## Technical Architecture

### Recommended Technology Stack

Based on extensive evaluation of available tools and services:

#### Recording and Capture

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Recording engine | **OBS Studio** (standalone, controlled via obs-websocket) | Industry standard. The WebSocket API (built-in since v28) provides comprehensive remote control: scene switching, source management, recording start/stop, filter configuration. Embedding libobs directly is technically possible but impractical — controlling the standalone app via WebSocket from a custom UI is far simpler and more maintainable. |
| Multi-camera sync | **NDI over LAN** (via obs-ndi plugin) | NDI (Network Device Interface) enables sending camera feeds from dedicated capture devices to OBS over the local network. Better sync than USB cameras. Works with professional cameras via HDMI-to-NDI converters. |
| Custom control UI | **Qt/C++ desktop app** communicating with OBS via WebSocket | A native Qt/C++ application provides a simplified, fitness-focused UI on top of OBS. Instructor sees big buttons: "Start Recording," "Switch to Overhead Camera," "Begin Q&A." The complexity of OBS is hidden. Qt/C++ targets Windows and Mac cleanly (Linux support is straightforward if needed later). This also matches the existing backend expertise on the project. |

**OBS as a dependency**: OBS Studio has a standard installer for Windows (`.exe` via the OBS website) and Mac (`.dmg`). It is a one-time install with no configuration needed by the end user — our Qt app would handle all OBS configuration via the WebSocket API, including setting up sources, scenes, and recording settings. The creator never needs to open or interact with OBS directly. The installer is lightweight (~150-250 MB) and widely trusted (open source, millions of users). This is a low barrier to entry — comparable to installing any other desktop application. The Qt app could even check for OBS on launch and guide the user through installation if it is missing.

#### Post-Processing

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Automated post-processing | **FFmpeg** (via command-line or libav* libraries) | The backbone of every video platform's backend. Handles trimming, concatenation, transitions (xfade filter supports 40+ types), overlays, text rendering (drawtext), audio normalization (loudnorm), noise reduction (arnndn/RNNoise), multi-bitrate HLS packaging, and thumbnail extraction. Can be called programmatically from any language. |
| Branded intro/outro generation | **Remotion** (React-based programmatic video) | Define branded templates in React/TypeScript. Pass in class metadata (title, instructor, duration, level) and render unique intros for each video automatically. Integrates into a Node.js or serverless pipeline. |
| Manual editing for premium content | **DaVinci Resolve** (free tier) | Industry-leading color grading, Fusion for motion graphics, Fairlight for audio. The free tier includes full editing, multi-track timeline, and most effects. Has a Python/Lua scripting API for batch operations. Use this for hero content, promotional videos, and certification course material — not for every class recording. |
| Audio cleanup | **FFmpeg audio filters** (loudnorm, arnndn) + **RNNoise** | Automated loudness normalization (EBU R128 standard) and AI-based noise reduction. RNNoise is open source, runs in real-time on CPU, and is available as an FFmpeg filter. Handles HVAC hum, room echo, and ambient noise. |
| Royalty-free music | **Epidemic Sound** or **Artlist** | Both offer commercial licenses covering distribution on your own platform. Epidemic Sound at ~$15/month personal / $49/month commercial. Artlist at ~$17/month annual. Essential for class background music and intro/outro audio. **Decision: the platform will handle music licensing centrally** — creators and studios get access to a licensed music library as part of the platform, eliminating the need for separate ASCAP/BMI licenses. This is a significant value-add (Les Mills bundles music licensing and studios cite it as one of the top reasons they license). |

**Alternative audio tools worth evaluating:**
- **NVIDIA Broadcast SDK (Maxine)**: Real-time AI noise removal for NVIDIA GPU users. Excellent quality, free.
- **ElevenLabs**: For generating narration or voice-over programmatically. High-quality AI voices with voice cloning. $5-330/month based on usage.
- **SoX (Sound eXchange)**: Open-source command-line audio processing. Useful for batch operations like silence detection/removal and format conversion.

**Alternative video tools worth evaluating:**
- **Kdenlive**: Free, open-source non-linear video editor. Good alternative to Resolve for manual editing.
- **Blender**: Free, open-source 3D suite with motion graphics capabilities. Overkill for simple branded intros but powerful if 3D animated intros are desired. Full Python API for automation.
- **Motion Canvas**: Open-source TypeScript library for creating animated videos programmatically. Better suited for explanatory/educational content than Remotion's more general-purpose approach.
- **Cavalry ($30/month)**: Purpose-built for 2D motion graphics and data-driven animation. Worth evaluating if complex motion graphics are important.

#### Live Streaming and Interactive Video

**What problem does this solve?** The Q&A sessions described in Phase 1 require real-time, low-latency, bidirectional video — the host needs to see and hear the audience member asking a question, and the audience member needs to see and hear the host, with minimal delay (under 500ms). This is fundamentally different from one-way video streaming (like watching a YouTube live stream). Standard streaming protocols like HLS/RTMP have 3-30 seconds of latency, which makes conversation impossible. **WebRTC** is the browser-native protocol for real-time bidirectional video (it is what powers Zoom, Google Meet, and Discord video calls).

**Why do we need a media server?** WebRTC works peer-to-peer for 1-on-1 calls, but our Q&A sessions have a host, a featured audience member, and potentially dozens or hundreds of viewers. A **media server** sits in the middle and efficiently routes video streams: it receives the host's video once and distributes it to all connected clients, rather than requiring the host to send a separate copy to each viewer. Without a media server, the host's upload bandwidth would be overwhelmed with more than a few viewers.

**LiveKit** is the recommended media server — it is open source, modern, and purpose-built for exactly this use case. It provides:
- A "stage" pattern where participants join with audio/video off and can be promoted to active speakers by the host (the "raise hand" → accept workflow).
- Simultaneous WebRTC for active participants (low latency) and HLS egress for passive viewers (scalable via CDN).
- Server-side recording of all participant tracks as separate files — essential for the post-production re-editing workflow.
- Server SDKs in Go, Node, Python, and C++. Client SDKs for JavaScript/React, Swift, Android, and Flutter.

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Interactive sessions (WebRTC) | **LiveKit** (open source, self-hosted or cloud) | Handles the "raise hand" → promote audience member → host layout control workflow natively. |
| Broadcast to viewers | **LiveKit Egress** → HLS via CDN | Sub-second latency for active participants; 2-6 second latency for passive viewers via HLS. |
| Session recording | **LiveKit Recording** | Captures all participant tracks as separate files for post-production re-editing. |

**Deployment options**:
- **Self-hosted** (free): Run LiveKit on your own server (a Linux VPS). No per-session cost beyond the server itself. Requires infrastructure management. A single $20-40/month VPS can handle small-to-medium sessions.
- **LiveKit Cloud** (managed): $0.004/participant-minute for audio, $0.016 for video. A Q&A session with 1 host + 1 featured student + 50 viewers for 60 minutes costs approximately $50-60. Zero ops burden. Best for starting out — switch to self-hosted later if session volume makes it cost-effective.

**Alternatives considered:**
- **Janus Gateway** (free, open source, C-based): Lower-level than LiveKit, more development work. Good if you want maximum control and have C expertise. Has been around longer, well-proven.
- **MediaSoup** (free, open source): A library, not a server — you build all room management, recording, and signaling on top. Maximum flexibility but most development effort.
- **Ant Media Server**: Enterprise features (clustering, SRT, hardware encoding) require paid license ($50-2,000/month).
- **AWS IVS (Interactive Video Service)**: Managed service. $2.36/hour per live channel + delivery costs. Has a Web Broadcast SDK for browser-based streaming.

Given the existing C++ expertise on this project, **Janus Gateway** is also a strong contender — it is written in C, highly performant, and offers more low-level control. The trade-off is more development work vs. LiveKit's more batteries-included approach.

#### Video Hosting and Delivery (On-Demand)

**What problem does this solve?** Once an instructional video or Q&A session has been edited and finalized, it needs to be stored, encoded into multiple quality levels (so it plays smoothly on both fast WiFi and slow mobile connections), and delivered to viewers worldwide through a CDN (Content Delivery Network — a global network of servers that serves video from a location near the viewer to minimize buffering).

You *could* do all of this yourself: store videos on a server, use FFmpeg to encode them into multiple bitrates, generate HLS (HTTP Live Streaming) playlists, and serve them through a CDN like CloudFront. But video hosting services handle all of this automatically — upload a video and they return a player-ready URL with adaptive bitrate streaming, content protection (signed URLs so only paying subscribers can watch), thumbnail generation, and analytics.

**The trade-off is cost vs. engineering effort.** A video hosting service costs money per minute of video stored and per GB delivered, but saves significant development and operations work. Self-hosting is cheaper at scale but requires building and maintaining the entire encoding/delivery pipeline.

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Primary option | **Mux** | Best-in-class developer API. Automatic adaptive bitrate encoding, signed URLs for content protection, thumbnail/storyboard generation, built-in web player component, engagement analytics. Pricing: $0.07/min encoding + $0.007/min/month storage + $0.014/GB delivery. |
| Budget alternative | **Cloudflare Stream** | Simpler pricing ($5/1,000 min stored + $1/1,000 min delivered), massive global CDN (300+ PoPs). Fewer advanced features than Mux but very cost-effective. |
| Budget alternative | **Bunny.net Stream** | Extremely aggressive pricing ($0.005/min stored + $0.005/GB delivered). Often 5-10x cheaper than competitors. Adequate API and features. Best for cost-conscious early stages. |
| Full control option | **AWS MediaConvert + CloudFront** | Maximum control over the encoding pipeline. DRM support. Many moving parts to manage. Significant engineering investment but scales well and has the lowest per-unit cost at volume. |

**Cost example** — a library of 100 hours of content viewed by 500 subscribers averaging 5 hours/month each:
| Provider | Monthly storage | Monthly delivery (~2,500 hours viewed, ~1.5TB) | Total/month |
|----------|----------------|--------------------------------------------------|-------------|
| Bunny.net | ~$1.50 | ~$7.50 | ~$9 |
| Cloudflare Stream | $30 | $150 | ~$180 |
| Mux | $42 | $21 | ~$63 + encoding |

Recommendation: **Start with Bunny.net or Cloudflare Stream** for the MVP to keep costs minimal. Migrate to Mux when the analytics, player, and API features justify the higher cost — likely when you have enough revenue that the cost difference is negligible relative to subscription income.

#### Payments and Subscriptions

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Payment processing | **Stripe** | Industry standard for online payments. Already widely used in SaaS and marketplaces. |
| Marketplace splits | **Stripe Connect** (Express or Custom) | Handles multi-party payments and revenue splitting between platform and content creators. Express accounts provide Stripe-hosted onboarding for creators. Fees: 2.9% + $0.30/transaction + 0.25% + $0.25/payout. |
| Subscription management | **Stripe Billing** | Manages recurring subscriptions, plan changes, proration, trial periods, and dunning (failed payment recovery). |
| Tax compliance | **Stripe Tax** | Automated sales tax and VAT calculation across jurisdictions. |

Note: Knotty Yoga currently uses Square for in-person payments. Square has marketplace capabilities but they are less mature than Stripe Connect for online subscription and marketplace use cases. The recommendation is to use Stripe for the platform's online payments while continuing Square for in-person POS. The two can coexist.

---

## Additional Ideas and Opportunities

### Progressive Skill Gating

Unlike information courses where you can skip ahead freely, physical skills must be learned sequentially. An "Intermediate Partner Acrobatics" course should not be accessible until "Foundations" is complete, and ideally should include a self-assessment or video submission checkpoint. This is a differentiator no existing platform offers for movement arts.

### AI-Assisted Features (Future)

- **Pose estimation**: Use computer vision to analyze a student's form from their webcam during a live session and provide automated feedback. Libraries like MediaPipe (Google) or MoveNet (TensorFlow) can do real-time pose estimation in the browser.
- **Automatic highlight extraction**: Use audio analysis (applause, laughter, emphasis) and transcript analysis to identify the most valuable moments in a Q&A session for automatic clip generation.
- **Smart camera switching**: During multi-camera recordings, use AI to determine which camera angle is most relevant based on the instructor's movement and position.
- **Automatic chapter markers**: Generate chapter points from speech-to-text transcription, allowing students to jump to specific topics within a long video.

### Community Features

- **Discussion threads per video**: Students can ask asynchronous questions about specific timestamps in a video. Other students or the instructor can answer. This builds a searchable knowledge base.
- **Student progress dashboards**: Track completed videos, attended Q&A sessions, hours practiced, and progression through skill trees.
- **Peer matching**: Connect students at similar skill levels for practice partners (especially valuable for partner acrobatics).

### Mobile Experience

- **Offline download**: Let students download videos for offline practice (essential for gyms with poor WiFi).
- **Practice mode**: A simplified player that loops specific video segments for follow-along practice, with adjustable playback speed.
- **Mirrored playback**: Flip the video horizontally so the instructor appears to mirror the student — standard for follow-along fitness content.

### Analytics for Creators

- **Engagement heatmaps**: Show creators which sections of their videos students rewatch, skip, or drop off from. This directly informs content improvement.
- **Q&A effectiveness metrics**: Track whether students who watch the instructional video before Q&A ask higher-quality questions and have better outcomes.
- **Retention and completion rates**: By course, by video, by student cohort. Help creators identify where students disengage.

### White-Label Option for Large Studios

Studios or franchise operations could get a white-labeled version of the platform (their branding, their domain) while still accessing the content marketplace. This is the "Mighty Pro" model (Mighty Networks charges custom pricing for white-label apps).

### Workshop and Event Integration

Content creators frequently do in-person workshops at studios. The platform could support:

- **Workshop listing and registration** integrated with the creator's platform profile.
- **Pre-workshop preparation**: Assign prerequisite videos that attendees must complete before the workshop.
- **Post-workshop follow-up**: Share recorded workshop segments, supplementary content, and continuing practice material.

---

## Revenue Model Summary

### Phase 1 Revenue (Creator Direct-to-Consumer)

| Source | Pricing | Revenue To Platform |
|--------|---------|---------------------|
| Creator subscriptions to use platform tools | $29-99/month | 100% |
| Student subscriptions to creator content | Creator sets price (typically $10-50/month) | 10-20% transaction fee |
| Pay-per-view for individual courses | Creator sets price | 10-20% transaction fee |

### Phase 2 Revenue (B2B Studio Licensing)

| Source | Pricing | Revenue To Platform |
|--------|---------|---------------------|
| Studio subscriptions | $50-500/month depending on tier | 20-30% (remainder to creator) |
| Instructor certification fees | $200-500/module | 20-30% (remainder to creator) |
| Workshop booking commissions | % of ticket price | 10-15% |

### Phase 3 Revenue (Platform Marketplace)

| Source | Pricing | Revenue To Platform |
|--------|---------|---------------------|
| Platform subscription (consumer) | $15-30/month | 20-30% (under discussion — see Open Questions) |
| Free tier | $0 (select content for discovery) | $0 (marketing/conversion funnel) |

---

## Risk and Mitigation

| Risk | Severity | Mitigation |
|------|----------|------------|
| Cold start: need creators and students simultaneously | High | Start with Knotty Yoga's own content. The platform solves a real internal need regardless of marketplace adoption. |
| Creators already have established Patreon audiences and resist switching | Medium | Position as a clean replacement that is demonstrably better. Allow coexistence during transition — no exclusivity requirement. The quality improvement in produced content and the elimination of the Zoom/Patreon/YouTube patchwork is the pitch. |
| Technical complexity of multi-source recording + live WebRTC | Medium | Use established tools (OBS + LiveKit) rather than building from scratch. Phase the build — recording and VOD first, live Q&A second. |
| Les Mills or a large player builds something similar | Low-Medium | Niche focus (acrobatics, partner work, aerial) is unlikely to attract enterprise competitors. The open marketplace model is fundamentally different from closed ecosystems. |
| Video hosting costs scale unpredictably | Medium | Start with cost-effective hosting (Bunny.net or Cloudflare Stream). Move to Mux for premium features only when revenue justifies it. |
| Certification accreditation requirements change | Low | Stay current with Yoga Alliance and state licensing boards. The hybrid model (50% online / 50% in-person) is already conservative relative to what many bodies allow. |

---

## Resolved Decisions

Summary of decisions made (for reference):

| # | Question | Decision |
|---|----------|----------|
| 1 | Platform-first or content-first? | MVP focused on Knotty Yoga + 1-2 beta creator partners |
| 2 | Studio licensing pricing | Accessible pricing for small acrobatics studios; revisit when scaling to yoga |
| 3 | Revenue sharing split | Under discussion — between 70/30 and 80/20 (creator/platform) |
| 4 | Free tier | Yes — free content for discovery and marketing |
| 5 | Editing scope | Multi-source layout switching (full/PIP/split) + voice-over + PIP commentary; not a full NLE |
| 6 | Desktop app technology | Qt/C++ (Windows + Mac, Linux easy to add) |
| 7 | LiveKit | Explained in Technical Architecture; decision on self-hosted vs. cloud deferred |
| 8 | Video hosting | Explained in Technical Architecture; start cheap (Bunny.net/Cloudflare), upgrade later |
| 9 | OBS dependency | Keep OBS; standard installer, low barrier; Qt app hides complexity |
| 10 | Content exclusivity | No exclusivity — creators can coexist on Patreon and use YouTube/Instagram for discovery |
| 11 | Music licensing | Platform handles centrally via royalty-free music licensing |
| 12 | Quality control | Curated — vet for quality, hand-picked initially, application process later |
| 13 | Geographic scope | Global from day one |
| 14 | Certification timeline | Defer — focus on technical first; certification by referral initially |
| 15 | Branding/positioning | Creator is the brand initially; develop platform brand later |
| 16 | Patreon relationship | Clean replacement, not integration |
| 17 | Partnership opportunities | Future phase — not for MVP |

---

## Open Questions

### Revenue Sharing: 70/30 vs 80/20

This is the most significant open business model question. Here are the trade-offs:

**80/20 (creator gets 80%)**:
- Headline differentiator: "creators keep 80%" — much better than YouTube (55%), Spotify (65-70%), Coursera (30-45%), or Udemy (37%).
- Comparable to Patreon (88-95% after fees) — but creators get dramatically better tools, so the slightly lower percentage is justified by the value provided.
- **Risk**: At early scale with few subscribers, the platform's 20% may not cover infrastructure costs (video hosting, LiveKit sessions, CDN, payment processing fees). Stripe alone takes ~3% + fixed fees, so the effective platform margin is closer to 17%.
- Best if the platform can keep infrastructure costs low and grow through volume.

**70/30 (creator gets 70%)**:
- Still very competitive — better than every major platform except Patreon and app stores.
- More headroom for platform investment in marketing, features, and infrastructure.
- 30% is the "standard" marketplace take rate (Apple/Google app stores, many SaaS marketplaces).
- **Risk**: Creators comparing against Patreon's 5-12% fee may see 30% as steep, even though the platform provides far more value.

**Possible hybrid approach**: Start at 80/20 to attract the initial creator base, with a published plan to move to 75/25 or 70/30 as the platform scales and adds more value (studio licensing, marketplace reach, analytics, tools). Early creators could be grandfathered at the better rate. This rewards early adopters and creates urgency to join early.

**Input needed**: What are the projected infrastructure costs per creator per month? This will determine whether 20% is viable or if 30% is necessary to sustain the platform.

Mason- This is where I'm not sure and could use helping calculate. I was thinking of starting with unlisted YouTube videos to avoid the storage / CDN fees. Payment processing is a given. I'm not sure about LiveKit. There are obvious disadvantages to YouTube unlisted but the cost is great. 

### Additional Questions for Future Discussion

1. **Beta creator selection**: Who are the 1-2 friends/colleagues you're considering for the beta? Understanding their specific content types and current workflows will help shape the MVP feature set.
	- Mason- One is Jenn Bruyer. She is an aerialist and all of her content is aerial. She publishes several videos a week that are recorded zoom meetings published on Patreon. PJ Perry is another. She does rope, straps conditioning, pilates, and handstands. He main appeal to most people is probably just rope though. She goes way deeper into rope and has a strong anatomy component.

2. **MVP timeline and sequencing**: Should the MVP build recording/editing first and Q&A live sessions second? Or are both needed from day one? The recording workflow is technically simpler and delivers immediate value even without live streaming.
	- Mason- We really need to do both. Both people currently really lean on the live Q&A thing to do a virtual "class" format. That has pluses and minuses, in my opinion, and I think the split model of putting out content and then doing a Q&A session after people have looked at it would be better but it is a different model and I know that they will want to support the interactive format as well.

3. **Free tier boundaries**: What content belongs in the free tier? Options include:
   - First lesson of every course free (the "preview" model)
   - Select full courses permanently free (loss leaders)
   - Creator-controlled: each creator decides what to make free
   - Time-limited free trials (first 7 days of any course)
	- Mason- Creator should decide. Might do teasers or trailers or content that is interesting but mainly piques interest.

1. **Centralized music licensing scope**: Royalty-free music services (Epidemic Sound, Artlist) cover distribution on your own platform. But if studios play content in a commercial setting (in-class), the licensing requirements may differ from online distribution. This needs investigation — specifically whether Epidemic Sound / Artlist commercial licenses cover in-studio public performance, or if a separate blanket license is needed.
	- Mason- Okay, let's dive into this. I more am thinking of light duty background sound and opening video title / trailer stuff than actual music during the class. Honestly, AI generated psuedo music / sound would be fine. We aren't going to need commercial music.

2. **Multi-platform content strategy**: If creators will use YouTube/Instagram for discovery, should the platform offer tools to generate short-form clips (Reels, Shorts, TikToks) from full-length content? This would be a compelling feature: upload a full class, and the platform auto-generates 30-60 second highlight clips with branding for social media distribution.
	- Mason- that does sound like a nice thing to do but definitely not a MVP or must have or should have. More of a nice to have or stretch goal. In other words, I love the idea and would like to do it eventually.

3. **Platform naming**: Does this platform have a working name, or should one be developed? The name will shape how creators and students perceive and remember it.
	- Mason- Yes, we need to think of a name but I don't want to block the progress on that.
