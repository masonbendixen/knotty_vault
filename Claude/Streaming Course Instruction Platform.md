---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/2/2026
Version: 0.2
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

This document proposes a three-phase platform for producing, distributing, and licensing fitness instructional content — initially serving Knotty Yoga's own needs, then expanding to serve independent content creators, studios, and eventually a marketplace of subscribers.

The platform addresses a clear gap in the fitness technology landscape: no existing product combines **high-quality produced instructional video**, **interactive live Q&A sessions**, and **B2B studio content licensing** in a single offering. Creators currently cobble together Patreon (payments/community), Zoom (live classes), and YouTube/Vimeo (hosting) — a fragmented experience that produces low-quality results for both creator and student.

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

Before scaling to serve other creators, the platform must first serve Knotty Yoga's own content production needs:

| Content Type | Description | Distribution |
|---|---|---|
| Self-care series | Body maintenance, recovery, mobility work for the general public | Public / free tier |
| Class streaming | Live-streamed studio classes for remote members | Members |
| Acrobatics prep | Strength and conditioning progressions for partner and aerial acrobatics | Members / course |
| Partner massage | Guided partner massage and bodywork sequences | Members / course |
| Teacher training curriculum | Portions of the yoga teacher training and massage certification programs | Enrolled students |

This diverse set of needs — spanning free public content, member-only streaming, structured courses, and accredited certification material — means the platform must support multiple content types and access levels from day one.

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

After recording, the application provides guided post-production:

- **Timeline trimming**: Remove false starts, dead air, off-topic tangents, and retakes from the recorded sources.
- **Voice-over recording**: Record narration over specific sections of the video — for example, adding explanation over a slow-motion replay of a technique.
- **Overlay insertion**: Add text annotations, arrows, safety callouts, or anatomical diagrams over the video at specific timestamps. Critical for acrobatics content where spotting positions and hand placement need highlighting.
- **Slide/presentation insertion**: Insert presentation-style material (anatomy slides, progression charts, theory content) between video segments.
- **Multi-angle compositing**: Choose which camera angle is primary at each point in the timeline. Switch between full-screen, split-screen, and picture-in-picture layouts at scene transition points.
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

### Pricing Model Considerations

The Les Mills model ($500-2,500/program/quarter) is proven but expensive for small studios. A tiered approach may work better:

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

Typical SaaS marketplace take rates range from 15-30%:

| Platform | Take Rate |
|----------|-----------|
| YouTube (ads) | 45% |
| Udemy (organic) | 63% |
| Coursera | 55-70% |
| Spotify | 30-35% |
| App stores | 15-30% |
| **Recommended range** | **20-30%** |

The platform retains 20-30% for infrastructure, development, and marketing. The remaining 70-80% is distributed to creators via the user-centric model.

### Opt-In Structure

Not all creator content should be in the marketplace. Creators should be able to:

- Keep some courses exclusive to their own direct subscribers (premium content, certifications).
- Place selected courses in the platform marketplace for broader reach and discovery.
- Set their own pricing for direct subscriptions while the marketplace has a single platform price.

This gives creators control while the marketplace provides discovery and audience building.

---

## Phase 4 (Potential): Certification and Accreditation

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
| Custom control UI | **Desktop app (Electron or native)** communicating with OBS via WebSocket | Provides a simplified, fitness-focused UI on top of OBS. Instructor sees big buttons: "Start Recording," "Switch to Overhead Camera," "Begin Q&A." The complexity of OBS is hidden. |

**Alternative to consider**: For simpler setups (single camera + screen share), a fully browser-based recording solution using the MediaRecorder API and WebRTC could eliminate the OBS dependency entirely. This trades multi-camera flexibility for simplicity. The custom desktop app could support both modes — OBS-based for professional multi-camera setups and browser-based for simpler recordings.

#### Post-Processing

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Automated post-processing | **FFmpeg** (via command-line or libav* libraries) | The backbone of every video platform's backend. Handles trimming, concatenation, transitions (xfade filter supports 40+ types), overlays, text rendering (drawtext), audio normalization (loudnorm), noise reduction (arnndn/RNNoise), multi-bitrate HLS packaging, and thumbnail extraction. Can be called programmatically from any language. |
| Branded intro/outro generation | **Remotion** (React-based programmatic video) | Define branded templates in React/TypeScript. Pass in class metadata (title, instructor, duration, level) and render unique intros for each video automatically. Integrates into a Node.js or serverless pipeline. |
| Manual editing for premium content | **DaVinci Resolve** (free tier) | Industry-leading color grading, Fusion for motion graphics, Fairlight for audio. The free tier includes full editing, multi-track timeline, and most effects. Has a Python/Lua scripting API for batch operations. Use this for hero content, promotional videos, and certification course material — not for every class recording. |
| Audio cleanup | **FFmpeg audio filters** (loudnorm, arnndn) + **RNNoise** | Automated loudness normalization (EBU R128 standard) and AI-based noise reduction. RNNoise is open source, runs in real-time on CPU, and is available as an FFmpeg filter. Handles HVAC hum, room echo, and ambient noise. |
| Royalty-free music | **Epidemic Sound** or **Artlist** | Both offer commercial licenses covering distribution on your own platform. Epidemic Sound at ~$15/month personal / $49/month commercial. Artlist at ~$17/month annual. Essential for class background music and intro/outro audio. |

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

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Interactive sessions (WebRTC) | **LiveKit** (open source, self-hosted or cloud) | Modern WebRTC SFU (Selective Forwarding Unit). Handles the "raise hand" → promote audience member → host layout control workflow natively via its "stage" pattern. Participants join with audio/video disabled, can be promoted to speaker by the host. Server-side SDKs available in Go, Node, Python, C++, and more. Client SDKs for JavaScript/React, Swift, Android, Flutter. |
| Broadcast to viewers | **LiveKit Egress** → HLS via CDN | LiveKit can simultaneously serve WebRTC to active participants and egress (bridge) the session to HLS for large audiences. Sub-second latency for the host and featured student; 2-6 second latency for the audience via HLS. |
| Session recording | **LiveKit Recording** | Captures all participant tracks as separate files. This is essential for post-production re-editing of Q&A sessions. |

**LiveKit Cloud pricing**: $0.004/participant-minute for audio, $0.016 for video. For a Q&A session with 1 host + 1 featured student + 50 viewers watching 60 minutes: approximately $50-60.

**Alternatives considered:**
- **Janus Gateway** (free, open source, C-based): Lower-level than LiveKit, more development work. Better if you want maximum control and have C expertise.
- **MediaSoup** (free, open source): A library, not a server — you build everything on top. Maximum flexibility but most development effort.
- **Ant Media Server**: Enterprise features (clustering, SRT, hardware encoding) require paid license ($50-2,000/month).
- **AWS IVS (Interactive Video Service)**: Managed service. $2.36/hour per live channel + delivery costs. Good if deeply invested in AWS. Has a Web Broadcast SDK for browser-based streaming without OBS.
- **Mux Live**: $0.07/minute encoding + $0.014/GB delivery. Excellent API but no built-in bidirectional video — better as a supplement to LiveKit for VOD hosting.

#### Video Hosting and Delivery (On-Demand)

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Primary option | **Mux** | Best-in-class API for developers. Automatic adaptive bitrate encoding, signed URLs for content protection, thumbnail/storyboard generation, Mux Player web component, Mux Data for quality-of-experience analytics. Pricing: $0.07/min encoding + $0.007/min/month storage + $0.014/GB delivery. |
| Budget alternative | **Cloudflare Stream** | Simpler pricing ($5/1,000 min stored + $1/1,000 min delivered), Cloudflare's massive global CDN (300+ PoPs). Fewer advanced features than Mux but very cost-effective. Good if already using Cloudflare. |
| Budget alternative | **Bunny.net Stream** | Extremely aggressive pricing ($0.005/min stored + $0.005/GB delivered). Often 5-10x cheaper than competitors. Adequate API and features. Best for cost-conscious early stages. |
| Full control option | **AWS MediaConvert + CloudFront** | Maximum control over encoding pipeline. DRM support via MediaPackage. Many moving parts (S3 + MediaConvert + CloudFront + Lambda). Significant engineering investment but scales well. |

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
| Platform subscription (consumer) | $15-30/month | 20-30% (remainder distributed to creators via user-centric model) |
| Advertising/sponsorship (optional) | CPM-based | 100% or revenue share |

---

## Risk and Mitigation

| Risk | Severity | Mitigation |
|------|----------|------------|
| Cold start: need creators and students simultaneously | High | Start with Knotty Yoga's own content. The platform solves a real internal need regardless of marketplace adoption. |
| Creators already have established Patreon audiences and resist switching | Medium | Position as complementary, not replacement. Offer migration tools. Demonstrate the quality improvement in produced content. |
| Technical complexity of multi-source recording + live WebRTC | Medium | Use established tools (OBS + LiveKit) rather than building from scratch. Phase the build — recording and VOD first, live Q&A second. |
| Les Mills or a large player builds something similar | Low-Medium | Niche focus (acrobatics, partner work, aerial) is unlikely to attract enterprise competitors. The open marketplace model is fundamentally different from closed ecosystems. |
| Video hosting costs scale unpredictably | Medium | Start with cost-effective hosting (Bunny.net or Cloudflare Stream). Move to Mux for premium features only when revenue justifies it. |
| Certification accreditation requirements change | Low | Stay current with Yoga Alliance and state licensing boards. The hybrid model (50% online / 50% in-person) is already conservative relative to what many bodies allow. |

---

## Open Questions

These are questions and decisions that need input before proceeding further:

### Business Model Questions

1. **Platform-first or content-first?** Should Phase 1 focus on building the platform tools (recording app, hosting, Q&A infrastructure) and selling them to other creators immediately? Or should it focus entirely on producing and distributing Knotty Yoga's own content, with the platform features being internal-only until they are proven?

2. **Pricing philosophy**: For studio licensing, is the goal to undercut Les Mills aggressively (making it accessible to small studios) or to price at a premium based on the specialized, niche content?

3. **Revenue sharing split**: For the Phase 3 marketplace, what split feels right? 70/30 (creator/platform) is standard for app stores, but Spotify takes 30-35% and YouTube takes 45%. A 75/25 or 80/20 split would be a differentiator for attracting creators.

4. **Free tier**: Should there be free content on the platform for discovery (like YouTube) or should everything be behind a paywall from day one?

5. **Scope of Phase 1**: How much editing capability should the desktop app have? A full non-linear editor is an enormous engineering effort. An alternative is to focus on guided workflows (trim, reorder, add intro/outro) and leave complex editing to DaVinci Resolve, with the app handling import of the final edited file.

### Technical Questions

6. **Desktop app technology**: Electron (cross-platform, JavaScript/TypeScript, large ecosystem, heavier) vs. native (C++/Qt for performance, matches existing backend expertise, harder cross-platform) vs. Tauri (Rust-based, lighter than Electron, smaller ecosystem)?

7. **Self-hosted vs. cloud for LiveKit**: Self-hosting LiveKit reduces per-session costs but requires infrastructure management. LiveKit Cloud eliminates ops burden but costs ~$50-60 per Q&A session. At what scale does self-hosting become worthwhile?

8. **Video hosting choice**: Start with the cheapest option (Bunny.net) for validation, or invest in Mux from the start for better API, analytics, and player? The cost difference is significant at scale.

9. **OBS dependency**: Is requiring instructors to install and configure OBS (even if controlled via a custom UI) an acceptable user experience? Or should the MVP support browser-based recording (single camera) as the default with OBS as an advanced option?

### Content and Market Questions

10. **Content exclusivity**: Should creators who place content in the Phase 3 marketplace be required to keep it exclusive (not also on Patreon/YouTube)? Or should the platform allow non-exclusive content to reduce the barrier to entry?

11. **Music licensing**: Should the platform handle music licensing centrally (like Les Mills does) for studio-licensed content? This is a significant operational complexity but a major value-add for studios. Without it, studios need their own ASCAP/BMI licenses.

12. **Quality control**: How much curation should the marketplace have? Fully open (anyone can publish, like Udemy — large catalog but variable quality) vs. curated (invitation/application, like Alo Moves — smaller catalog but consistent quality)?

13. **Geographic scope**: Is this a local/regional play (Pacific Northwest acrobatics community) initially, or does the niche nature of the content mean the audience is inherently global from day one?

14. **Certification timeline**: How important is building the certification infrastructure early vs. focusing on content production and distribution first? Certification creates a moat but adds significant regulatory and operational complexity.

### Competitive Positioning Questions

15. **Naming and positioning**: Should the platform be positioned as a "tool for creators" (like Teachable/Kajabi — the creator is the brand) or as a "destination for students" (like Alo Moves — the platform is the brand)? Or can it credibly be both at different phases?

16. **Relationship with Patreon**: Many target creators already have Patreon audiences. Should the platform offer Patreon integration (e.g., sync membership tiers) to ease migration, or position as a clean replacement?

17. **Partnership opportunities**: Are there existing organizations (AcroYoga International, aerial arts associations, Yoga Alliance) that would benefit from endorsing or partnering with the platform?
