# PIP Recording Window

A floating Picture-in-Picture window for screen recording. Users can navigate freely while recording controls stay visible.

**Live prototype**: https://zesty-kitsune-a231bd.netlify.app

## How it works

Uses the [Document Picture-in-Picture API](https://developer.chrome.com/docs/web-platform/document-picture-in-picture/) (Chrome 116+) to open a floating window with:
- Live video preview with rounded corners
- Recording timer badge (centered)
- Mute button (voice mode only)
- Stop button with audio waveform visualization

## Key technical bits

### Opening the PIP window

```javascript
const pipWindow = await documentPictureInPicture.requestWindow({
  width: 400,
  height: 300,
});

// Inject your styles and DOM
pipWindow.document.head.appendChild(styleElement);
pipWindow.document.body.appendChild(container);
```

### Video frame sizing

The frame container needs explicit sizing to maintain rounded corners regardless of aspect ratio. Can't just use `object-fit`.

```javascript
function resizeVideoFrame() {
  const aspectRatio = pipVideo.videoWidth / pipVideo.videoHeight;
  // Calculate frame dimensions based on available space
  // Then set explicit width/height on the frame container
}

pipVideo.addEventListener('loadedmetadata', resizeVideoFrame);
pipWindow.addEventListener('resize', resizeVideoFrame);
setInterval(resizeVideoFrame, 500); // Catch screen resize during recording
```

### Audio waveform

The waveform animation runs in the **main window** and updates the PIP window's DOM via cross-window access. This avoids MediaStream issues in the PIP context.

```javascript
// Main window: analyze audio and update PIP
const analyser = audioContext.createAnalyser();
source.connect(analyser);

function updateWaveform() {
  const wavePath = pipWindow.document.getElementById("pipWavePath");
  // Get audio levels, generate SVG path, update wavePath
  requestAnimationFrame(updateWaveform);
}
```

### Combining mic + system audio

```javascript
const audioContext = new AudioContext();
const dest = audioContext.createMediaStreamDestination();

// Mix mic and system audio
micSource.connect(dest);
screenSource.connect(dest);

// Use combined stream for recording
const recordingStream = new MediaStream([
  ...screenStream.getVideoTracks(),
  ...dest.stream.getAudioTracks()
]);
```

Audio Waveform Visualization
When voice mode is active, the stop button shows an audio-reactive waveform that responds to microphone input levels.

Structure
<button class="pip-stop-btn">
  <div class="pip-waveform">
    <svg viewBox="0 0 100 100" preserveAspectRatio="none">
      <path id="pipWavePath" fill="rgba(255,255,255,0.35)" />
    </svg>
  </div>
  <div class="pip-stop-icon"></div>
  <span>Stop</span>
</button>
Waveform Styling
.pip-stop-btn {
  position: relative;
  overflow: hidden;
  /* ... other stop button styles */
}

.pip-waveform {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 100%;
  pointer-events: none;
}

.pip-waveform svg {
  width: 100%;
  height: 100%;
}

/* Ensure icon and text appear above waveform */
.pip-stop-icon, .pip-stop-btn span {
  position: relative;
  z-index: 1;
}
Audio Analysis (from main window)
The audio analysis runs in the main window and updates the PIP window's DOM via cross-window access. This avoids MediaStream access issues in the PIP context.

// Create audio context and analyser
const audioContext = new AudioContext();
const analyser = audioContext.createAnalyser();
analyser.fftSize = 256;
analyser.smoothingTimeConstant = 0.3;
analyser.minDecibels = -90;
analyser.maxDecibels = -10;

// Connect mic stream
const source = audioContext.createMediaStreamSource(micStream);
source.connect(analyser);

const bufferLength = analyser.frequencyBinCount;
const frequencyData = new Uint8Array(bufferLength);
const timeDomainData = new Uint8Array(bufferLength);

function updateWaveform() {
  if (!pipWindow) return;

  const wavePath = pipWindow.document.getElementById("pipWavePath");
  if (!wavePath) {
    requestAnimationFrame(updateWaveform);
    return;
  }

  // Get both time domain (responsive) and frequency data
  analyser.getByteTimeDomainData(timeDomainData);
  analyser.getByteFrequencyData(frequencyData);

  // Calculate volume from time domain
  let maxAmplitude = 0;
  for (let i = 0; i < bufferLength; i++) {
    const amplitude = Math.abs(timeDomainData[i] - 128);
    if (amplitude > maxAmplitude) maxAmplitude = amplitude;
  }

  // Also get frequency-based volume
  let freqSum = 0;
  for (let i = 2; i < 48; i++) {
    freqSum += frequencyData[i];
  }
  const freqAvg = freqSum / 46;

  // Use higher of the two for responsiveness
  const timeVolume = maxAmplitude / 128;
  const freqVolume = freqAvg / 180;
  const volume = Math.min(Math.max(timeVolume, freqVolume) * 1.5, 1);

  // Generate wave path
  // volume=0 → wave at bottom (y=85)
  // volume=1 → wave at top (y=20)
  const baseY = 85 - (volume * 65);
  const time = Date.now() / 1000;

  let d = `M0,100 L0,${baseY}`;
  for (let i = 0; i <= 12; i++) {
    const x = (i / 12) * 100;
    const waveOffset = Math.sin(time * 3 + i * 0.8) * (3 + volume * 15);
    const y = Math.max(20, Math.min(95, baseY + waveOffset));
    // Quadratic curves for smooth wave
    d += ` Q${x},${y} ${x + 4},${y}`;
  }
  d += ` L100,100 Z`;

  wavePath.setAttribute("d", d);
  requestAnimationFrame(updateWaveform);
}

updateWaveform();
Audio Stream Handling
When "Screen + voice" is selected, combine microphone audio with any system audio from the screen capture.

// Get mic stream first
const micStream = await navigator.mediaDevices.getUserMedia({
  audio: { echoCancellation: true, noiseSuppression: true }
});

// Get screen capture (may include system audio)
const screenStream = await navigator.mediaDevices.getDisplayMedia({
  video: { frameRate: 30 },
  audio: true
});

// Combine audio streams
const audioContext = new AudioContext();
const dest = audioContext.createMediaStreamDestination();

// Add mic
const micSource = audioContext.createMediaStreamSource(micStream);
micSource.connect(dest);

// Add system audio if present
const screenAudioTracks = screenStream.getAudioTracks();
if (screenAudioTracks.length > 0) {
  const screenAudioStream = new MediaStream([screenAudioTracks[0]]);
  const screenSource = audioContext.createMediaStreamSource(screenAudioStream);
  screenSource.connect(dest);
}

// Combined stream for recording
const recordingStream = new MediaStream([
  ...screenStream.getVideoTracks(),
  ...dest.stream.getAudioTracks()
]);
Cleanup
Ensure all resources are properly released on stop or PIP window close.

function cleanup() {
  // Stop animation frames
  cancelAnimationFrame(waveformAnimationFrame);

  // Close audio contexts
  audioContext?.close();
  waveformAudioContext?.close();

  // Stop all tracks
  micStream?.getTracks().forEach(t => t.stop());
  screenStream?.getTracks().forEach(t => t.stop());
  recordingStream?.getTracks().forEach(t => t.stop());

  // Close PIP window
  pipWindow?.close();
}
Reference
Working prototype: /Users/chrisrodemeyer/pip/index.html

Key sections in prototype:

PIP styles: lines 812-953
openDocumentPip() function: lines 957-1131
Audio waveform setup: lines 1304-1426

## Files

- `index.html` - Complete prototype (start screen, PIP window, review screen)
- `HANDOFF-PIP-Window.md` - Detailed technical handoff doc
