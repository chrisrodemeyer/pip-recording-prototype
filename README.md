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

## Files

- `index.html` - Complete prototype (start screen, PIP window, review screen)
- `HANDOFF-PIP-Window.md` - Detailed technical handoff doc
