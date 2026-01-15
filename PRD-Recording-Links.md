# Recording Links PRD

Last updated: January 14, 2026
Status: Prototype Complete

<aside>

**TLDR**: Recording Links lets users capture screen recordings with optional voiceover directly from a shareable link. The lightweight flow opens a Picture-in-Picture control window, records the user's screen, and provides a review step before submission. This enables frictionless bug reports, feedback, and async communication without requiring app installation.

</aside>

# **Problem Alignment**

## **Customer or User Problem**

<aside>
Description of the customer (segment) or user problem on a high level. Containing who, what, when and why.
</aside>

| Who | End users submitting bug reports, feedback, or async updates to teams using Jam |
| --- | --- |
| What | Need to capture and share screen recordings quickly without installing software or navigating complex interfaces |
| When | When encountering a bug, providing feedback on a feature, or communicating something that's easier to show than describe |
| Why | Traditional screen recording requires installed apps, multiple steps, and manual file sharing. Users abandon the process or resort to lengthy text descriptions that lack context |

## **Goal**

<aside>
Enable users to capture high-quality screen recordings with optional voice narration in under 30 seconds from first click to submission, with zero software installation required.
</aside>

### **Defining success**

*How can we determine if we've solved this problem for users, and how will we measure whether we've achieved our goals?*

| **Topline Metric** | Recording completion rate (% of users who start recording and successfully submit) |
| --- | --- |
| **Counter Metric** | Average recording duration (ensure we're not just getting accidental/empty recordings) |

---

# **Solution Alignment**

## **Open Issues, Questions & Key Decisions**

| Issue/Question | Status | Decision/Notes |
| --- | --- | --- |
| Should we support camera (facecam) in addition to screen? | Deferred | V1 focuses on screen + voice only. Camera can be added later |
| How do we handle browser compatibility? | Decided | Requires Chrome 116+ for Document Picture-in-Picture API. Show graceful fallback message for unsupported browsers |
| Should recordings auto-upload or require explicit submit? | Decided | Explicit submit with review step - gives users confidence and ability to redo |
| Maximum recording duration? | Open | Need to determine based on upload/storage constraints |
| Where do submitted recordings go? | Open | Integration with Jam's existing artifact system TBD |

## **Use cases or Key Functionality**

*Give an overview of what we're building.*

**Committing to:**

- **Start Screen**: Landing page with mode selection (Screen + voice / Screen only) and prominent start button
- **Recording Mode Selection**: Toggle between screen-only and screen with microphone voiceover
- **Picture-in-Picture Window**: Floating control window showing live preview, recording timer, mute toggle, and stop button
- **Audio Waveform Visualization**: Real-time audio-reactive waveform in stop button when voice mode is active
- **Mute/Unmute Control**: Quick toggle to mute microphone during recording without stopping
- **Review Screen**: Preview recorded video with playback controls, optional context/description field, redo and submit actions
- **Responsive Video Preview**: Dynamic sizing that maintains aspect ratio and always shows rounded corners regardless of recorded content dimensions

**Not Building (V1)**

- Camera/facecam overlay
- Recording pause/resume
- Video trimming/editing
- Direct integrations (Slack, Linear, etc.) - submit goes to Jam first
- Mobile support

## **Key Flows**

### Flow 1: Start Recording

1. User lands on Recording Link URL
2. Sees "Get ready to record" screen with mode selection cards
3. Selects "Screen + voice" or "Screen only"
4. Clicks "Start recording"
5. Browser prompts for screen selection (and mic permission if voice mode)
6. Recording begins, PIP window appears with live preview

### Flow 2: During Recording (PIP Window)

1. PIP window floats above other content showing:
   - Live screen preview with rounded corners
   - Centered recording timer badge with pulsing red dot
   - Mute button (voice mode only) - circular with mic icon
   - Stop button with audio waveform visualization (voice mode)
2. User can:
   - Mute/unmute microphone without stopping
   - See audio levels via waveform animation
   - Stop recording via button or closing PIP window

### Flow 3: Review & Submit

1. Recording stops, PIP window closes
2. Review tab opens with recorded video
3. User sees:
   - Video player with dark overlay and play button
   - Duration badge
   - "Add context" text field (optional)
   - Redo button (returns to start screen)
   - Submit button
4. User reviews playback, optionally adds description
5. Clicks Submit to send recording

## **Design Specifications**

### Start Screen
- Background: Flat gray (#f9f9f9)
- Card: 900x600px white with subtle shadow, 18px border radius
- Mode cards: 172x172px, 18px radius, green accent when selected
- Start button: Full-width green (#7cedc7), 48px height, 12px radius

### PIP Window
- Background: Dark (#0b0c10)
- Video frame: 18px border radius, 3px white border (12% opacity)
- Timer badge: Centered overlay, pill shape, 18px font
- Stop button: Red (#ff4d4d), pill shape with white icon
- Mute button: 44px circular, semi-transparent white, mic icon
- Waveform: White (35% opacity), audio-reactive SVG path

### Review Screen
- Same card dimensions as start screen
- Video player: 554px width, 18px radius, dark overlay when paused
- Play button: 72px circular, white with subtle border
- Context field: Simple textarea, no border
- Action buttons: Redo (outlined), Submit (green fill)

### Footer
- Text: "Powered by Jam Screen Recordings"
- Color: Gray (50% opacity)
- Position: Centered, 64px from bottom

## **Technical Implementation**

### APIs Used
- **Document Picture-in-Picture API** (Chrome 116+): Floating control window
- **MediaDevices.getDisplayMedia()**: Screen capture
- **MediaDevices.getUserMedia()**: Microphone access
- **Web Audio API**: AudioContext, AnalyserNode for waveform visualization
- **MediaRecorder API**: Recording to WebM format
- **BroadcastChannel API**: Cross-tab communication for blob transfer

### Browser Support
- Chrome 116+ required for Document PIP
- Graceful degradation message for unsupported browsers

### Audio Handling
- Microphone and system audio combined via AudioContext
- Real-time analysis using both time-domain and frequency data
- Waveform updates at 60fps via requestAnimationFrame
