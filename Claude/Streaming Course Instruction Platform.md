---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/1/2026
Version: 0.1
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

I have a number of needs for creating video content:

- A self care series for people to take care of their bodies
- Streaming some classes
- Prep series for acrobatics
	- Strengthening for partner acrobatics
	- Strengthening for aerial acrobatics
- Partner massage work
- Some of the massage school / yoga teacher training curriculum

I need to create a platform to deliver this content. For managing recording of sessions, OBS studio seems to be the gold standard. It is available as a standalone app and can be embedded in an app with libobs. Even the standalone app can be driven through websockets. FFMPEG seems to be a great library for post processing and doing trimming, transitions, overlays, splicing in title / credit sequences, and so forth. DaVinci resolve seems like a good tool for generating video effects for free. Open to other suggestions for helping with this and doing things like audio and available video effects or generating things like intro video.

In creating this platform and consuming media from people I do video lessons from, I feel like there is a business opportunity here since I'm already developing a platform for me. The athletic content producers generally use Patreon. Patreon is mostly a blog site that collects payment. The content producers generally post a schedule for the week with Zoom links for their online classes. People attend the online classes and both learn the material and also do the material and ask questions. After the class is over, the Zoom recording is posted. 

There are several issues with this. The Zoom video recordings are low quality. Zoom has limited storage space so the people generally can only keep videos up for a week or two and delete them so you have to download the videos quickly or they will be gone leaving a blog full of dead links. The videos do not look professional and have no branding or anything like that. They videos also tend to be very long for the amount of information present in them because so much of it is student questions.

I would like to create a platform to help tackle this issue. Phase one would be to create a program that automates creating high quality instructional videos and then running Q&A sessions that are also recorded and distributed. For creating the instructional video, the app would:

- Sync recording of multiple input sources like different camera angles and audio input via a microphone and capture the input sources locally.
- Support an edit mode where the person goes through and:
	- Trims sections out of the stream so they don't appear in the final product
	- Do voice overs over parts of the video stream
	- Allow the insertion of overlays on top of the video stream
	- Allow the insertion of slides / presentation style material
	- Allow taking the multiple video inputs and choosing split screen and PIP at various scene transition points
	- Inserting intro video
	- Inserting a trailer
- Support live stream and higher quality local capture while recording

The idea is that people would distribute these instructional videos ahead of a Q&A session that is scheduled at a later date. People could watch these videos and become familiar with them and even try the video and come prepared with high quality questions for the Q&A session. The Q&A session would be live streamed via the website. People can click, ask a video question. The host would see the question and choose to accept a question. This would cause the client to stream their video via WebRTC and get a lower latency WebRTC connection to the host. The host could choose between staying full screen, putting the audience full screen, making the audience splitscreen, making the audience PIP with the host full screen, or making the audience full screen with the host PIP. The host could end this audience session at any time and also switch to another.

The host video and client streams would all be saved as separate video sources. Later editing would have all the features of the creational video support but also capture all the client streams separately and allow the host to later remix and choose when to make people full screen / split screen / PIP / etc as well as being able to trim.

Phase 

# Place plan here