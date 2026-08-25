# Audio-to-MP4 Video Visualizer Implementation Guide

## Overview

This comprehensive guide enables the creation of an **animated MP4 video** that combines:
- **Audio file** (MP3, WAV, OGG, etc.)
- **Static image** (background/cover art)
- **Dynamic waveform visualization** (using wavesurfer.js)

The resulting MP4 can be used for music platforms, social media, streaming services, or multimedia applications.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Input Processing                        │
├─────────────────────────────────────────────────────────────┤
│  Audio File  │  Image File  │  Configuration              │
└───────┬───────────┬──────────────┬─────────────────────────┘
        │           │              │
        ▼           ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│            Audio Analysis & Waveform Extraction            │
├─────────────────────────────────────────────────────────────┤
│  • Decode audio to PCM                                     │
│  • Extract peak data for each frame                        │
│  • Calculate frequency spectrum (FFT)                      │
│  • Generate waveform visualization data                    │
└───────┬─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│            Frame Generation (Canvas Rendering)             │
├─────────────────────────────────────────────────────────────┤
│  • Create canvas with background image                     │
│  • Render animated waveform for current frame              │
│  • Apply visual effects (gradients, colors, animations)    │
│  • Export frame as PNG/JPEG                                │
└───────┬─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│              Video Encoding (FFmpeg Integration)           │
├─────────────────────────────────────────────────────────────┤
│  • Combine rendered frames into video stream               │
│  • Merge audio track with video frames                     │
│  • Encode to MP4 (H.264/H.265)                            │
│  • Apply quality/bitrate settings                          │
└───────┬─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                  Output: Final MP4 File                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Stack Components

### 1. **Wavesurfer.js** (Audio Visualization)
- Decodes audio files using Web Audio API
- Generates waveform peak data
- Provides real-time audio analysis

### 2. **Canvas API** (Frame Rendering)
- Renders waveform visualization per frame
- Applies image background
- Creates animated graphics

### 3. **FFmpeg** (Video Encoding)
- Combines rendered frames into video
- Encodes to MP4 format
- Merges audio track

### 4. **Node.js Runtime Environment**
- Coordinates the workflow
- Manages file I/O and processes
- Orchestrates frame generation

---

## Implementation Strategy

### Phase 1: Audio Processing

#### 1.1 Audio Decoding
```typescript
// Decode audio file to PCM data
import WaveSurfer from 'wavesurfer.js'

async function decodeAudio(audioFilePath: string) {
  // Load audio file
  const audioBuffer = await fetch(audioFilePath)
    .then(res => res.arrayBuffer())
  
  // Use Web Audio API to decode
  const audioContext = new AudioContext()
  const decoded = await audioContext.decodeAudioData(audioBuffer)
  
  return {
    sampleRate: decoded.sampleRate,
    duration: decoded.duration,
    channelData: [
      decoded.getChannelData(0),  // Left channel
      decoded.getChannelData(1)   // Right channel (if stereo)
    ]
  }
}
```

#### 1.2 Waveform Peak Extraction
```typescript
// Extract peak data for visualization
function extractPeaks(
  channelData: Float32Array,
  sampleRate: number,
  peaksPerSecond: number = 60
) {
  const peakData = []
  const samplesPerPeak = sampleRate / peaksPerSecond
  
  for (let i = 0; i < channelData.length; i += samplesPerPeak) {
    const sliceEnd = Math.min(i + samplesPerPeak, channelData.length)
    const slice = channelData.slice(i, sliceEnd)
    
    // Find min/max in this slice
    let min = 0, max = 0
    for (let sample of slice) {
      if (sample < min) min = sample
      if (sample > max) max = sample
    }
    
    peakData.push({ min, max })
  }
  
  return peakData
}
```

#### 1.3 Frequency Spectrum Analysis (Optional)
```typescript
// Extract frequency spectrum using FFT
function computeFrequencySpectrum(
  channelData: Float32Array,
  fftSize: number = 2048
) {
  const offlineContext = new OfflineAudioContext(
    2,
    channelData.length,
    44100
  )
  
  const source = offlineContext.createBufferSource()
  const analyser = offlineContext.createAnalyser()
  
  analyser.fftSize = fftSize
  source.connect(analyser)
  analyser.connect(offlineContext.destination)
  
  // Extract frequency data
  const frequencyData = new Uint8Array(analyser.frequencyBinCount)
  analyser.getByteFrequencyData(frequencyData)
  
  return frequencyData
}
```

---

### Phase 2: Canvas Rendering

#### 2.1 Frame Renderer Setup
```typescript
interface FrameRenderConfig {
  width: number           // Output video width (e.g., 1920)
  height: number          // Output video height (e.g., 1080)
  fps: number            // Frames per second (e.g., 30 or 60)
  waveformColor: string  // Hex color (#FF0000)
  backgroundColor: string // Hex color or 'transparent'
  backgroundImage?: string // Path to background image
  waveformScale: number  // Height multiplier for waveform
}

class FrameRenderer {
  canvas: HTMLCanvasElement
  ctx: CanvasRenderingContext2D
  config: FrameRenderConfig
  
  constructor(config: FrameRenderConfig) {
    this.config = config
    this.canvas = new OffscreenCanvas(config.width, config.height)
    this.ctx = this.canvas.getContext('2d')!
  }
  
  async renderFrame(
    frameIndex: number,
    peakData: Array<{min: number, max: number}>,
    backgroundImage: HTMLImageElement
  ): Promise<Buffer> {
    const ctx = this.ctx
    
    // 1. Draw background
    if (backgroundImage) {
      ctx.drawImage(backgroundImage, 0, 0, this.config.width, this.config.height)
    } else {
      ctx.fillStyle = this.config.backgroundColor
      ctx.fillRect(0, 0, this.config.width, this.config.height)
    }
    
    // 2. Draw waveform
    this.drawWaveform(frameIndex, peakData)
    
    // 3. Export frame as PNG buffer
    return await this.canvas.convertToBlob()
      .then(blob => blob.arrayBuffer())
      .then(buf => Buffer.from(buf))
  }
  
  private drawWaveform(
    frameIndex: number,
    peakData: Array<{min: number, max: number}>
  ) {
    const ctx = this.ctx
    const width = this.config.width
    const height = this.config.height
    const centerY = height / 2
    const scale = this.config.waveformScale
    
    ctx.strokeStyle = this.config.waveformColor
    ctx.lineWidth = 2
    ctx.beginPath()
    
    // Calculate how many frames of peaks to display
    const peaksToShow = Math.min(width, peakData.length)
    const peakStep = Math.max(1, Math.floor(peakData.length / peaksToShow))
    
    // Draw waveform
    for (let i = 0; i < peaksToShow; i++) {
      const peakIdx = i * peakStep
      if (peakIdx >= peakData.length) break
      
      const peak = peakData[peakIdx]
      const x = (i / peaksToShow) * width
      
      // Top half (positive samples)
      const topY = centerY - (peak.max * height * scale / 2)
      
      if (i === 0) {
        ctx.moveTo(x, topY)
      } else {
        ctx.lineTo(x, topY)
      }
    }
    
    ctx.stroke()
    
    // Draw bottom half for stereo
    ctx.beginPath()
    for (let i = 0; i < peaksToShow; i++) {
      const peakIdx = i * peakStep
      if (peakIdx >= peakData.length) break
      
      const peak = peakData[peakIdx]
      const x = (i / peaksToShow) * width
      
      // Bottom half (negative samples)
      const bottomY = centerY + (Math.abs(peak.min) * height * scale / 2)
      
      if (i === 0) {
        ctx.moveTo(x, bottomY)
      } else {
        ctx.lineTo(x, bottomY)
      }
    }
    
    ctx.stroke()
  }
}
```

#### 2.2 Animation Effects
```typescript
// Animate waveform progression through video
function animateWaveformProgression(
  allPeakData: Array<{min: number, max: number}>,
  totalFrames: number,
  framesPerSecond: number
) {
  const framesToPeaksRatio = allPeakData.length / totalFrames
  
  return (frameIndex: number) => {
    // Calculate which portion of peaks to show
    const endPeakIndex = Math.floor(frameIndex * framesToPeaksRatio)
    return allPeakData.slice(0, endPeakIndex)
  }
}

// Color gradient animation
function getAnimatedColor(frameIndex: number, totalFrames: number): string {
  const hue = (frameIndex / totalFrames) * 360
  return `hsl(${hue}, 100%, 50%)`
}
```

---

### Phase 3: Video Encoding with FFmpeg

#### 3.1 Frame Sequence to Video
```bash
# Basic FFmpeg command to convert PNG frames to MP4
ffmpeg -framerate 30 \
  -i frame_%06d.png \
  -c:v libx264 \
  -pix_fmt yuv420p \
  -crf 23 \
  output.mp4
```

#### 3.2 Audio & Video Merge
```bash
# Combine video frames with audio track
ffmpeg -framerate 30 \
  -i frame_%06d.png \
  -i audio.mp3 \
  -c:v libx264 \
  -c:a aac \
  -pix_fmt yuv420p \
  -shortest \
  output.mp4
```

#### 3.3 Node.js FFmpeg Integration
```typescript
import { spawn } from 'child_process'
import fs from 'fs'
import path from 'path'

class VideoEncoder {
  async encodeFramesToVideo(
    framesDir: string,
    audioPath: string,
    outputPath: string,
    fps: number = 30,
    bitrate: string = '5000k'
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const ffmpegArgs = [
        '-framerate', fps.toString(),
        '-i', path.join(framesDir, 'frame_%06d.png'),
        '-i', audioPath,
        '-c:v', 'libx264',
        '-c:a', 'aac',
        '-pix_fmt', 'yuv420p',
        '-b:v', bitrate,
        '-shortest',
        outputPath
      ]
      
      const ffmpeg = spawn('ffmpeg', ffmpegArgs)
      
      ffmpeg.stderr.on('data', (data) => {
        console.log(`FFmpeg: ${data}`)
      })
      
      ffmpeg.on('close', (code) => {
        if (code === 0) {
          resolve()
        } else {
          reject(new Error(`FFmpeg exited with code ${code}`))
        }
      })
      
      ffmpeg.on('error', reject)
    })
  }
}
```

---

### Phase 4: Complete Orchestration

#### 4.1 Main Pipeline
```typescript
interface VisualizerInput {
  audioFile: string           // Path to audio file
  imageFile: string           // Path to background image
  outputFile: string          // Path to output MP4
  width: number               // Video width (default: 1920)
  height: number              // Video height (default: 1080)
  fps: number                 // Frames per second (default: 30)
  waveformColor: string       // Waveform color (default: #0099ff)
  backgroundColor: string     // BG color (default: #000000)
  bitrate: string             // Video bitrate (default: 5000k)
}

class AudioVisualizerPipeline {
  private tempDir: string
  
  constructor() {
    this.tempDir = path.join(process.cwd(), '.temp_frames')
  }
  
  async generate(input: VisualizerInput): Promise<void> {
    console.log('🎬 Starting audio visualizer generation...')
    
    try {
      // Step 1: Decode audio
      console.log('📊 Decoding audio...')
      const audioData = await this.decodeAudio(input.audioFile)
      const peaks = this.extractPeaks(audioData)
      
      // Step 2: Load background image
      console.log('🖼️  Loading background image...')
      const bgImage = await this.loadImage(input.imageFile)
      
      // Step 3: Generate frames
      console.log('🎨 Rendering frames...')
      const totalFrames = Math.ceil(audioData.duration * input.fps)
      await this.generateFrames(peaks, bgImage, input, totalFrames)
      
      // Step 4: Encode video
      console.log('🎬 Encoding video...')
      await this.encodeVideo(input, totalFrames)
      
      // Step 5: Cleanup
      console.log('🧹 Cleaning up temporary files...')
      this.cleanup()
      
      console.log(`✅ Video created: ${input.outputFile}`)
    } catch (error) {
      console.error('❌ Error:', error)
      this.cleanup()
      throw error
    }
  }
  
  private async generateFrames(
    peaks: Array<{min: number, max: number}>,
    bgImage: HTMLImageElement,
    input: VisualizerInput,
    totalFrames: number
  ): Promise<void> {
    // Create temp directory
    if (!fs.existsSync(this.tempDir)) {
      fs.mkdirSync(this.tempDir, { recursive: true })
    }
    
    const renderer = new FrameRenderer({
      width: input.width,
      height: input.height,
      fps: input.fps,
      waveformColor: input.waveformColor,
      backgroundColor: input.backgroundColor,
      waveformScale: 1.0
    })
    
    // Render each frame
    for (let i = 0; i < totalFrames; i++) {
      if (i % 30 === 0) {
        console.log(`  Frame ${i}/${totalFrames}`)
      }
      
      // Get peaks for current frame
      const framePeaksEnd = Math.floor((i / totalFrames) * peaks.length)
      const framePeaks = peaks.slice(0, framePeaksEnd)
      
      // Render and save
      const frameBuffer = await renderer.renderFrame(i, framePeaks, bgImage)
      const framePath = path.join(
        this.tempDir,
        `frame_${String(i).padStart(6, '0')}.png`
      )
      fs.writeFileSync(framePath, frameBuffer)
    }
  }
  
  private async encodeVideo(
    input: VisualizerInput,
    totalFrames: number
  ): Promise<void> {
    const encoder = new VideoEncoder()
    await encoder.encodeFramesToVideo(
      this.tempDir,
      input.audioFile,
      input.outputFile,
      input.fps,
      input.bitrate
    )
  }
  
  private cleanup(): void {
    if (fs.existsSync(this.tempDir)) {
      fs.rmSync(this.tempDir, { recursive: true })
    }
  }
  
  private async decodeAudio(audioPath: string): Promise<AudioData> {
    // Implementation from Phase 1
  }
  
  private extractPeaks(audioData: AudioData): Array<{min: number, max: number}> {
    // Implementation from Phase 1
  }
  
  private async loadImage(imagePath: string): Promise<HTMLImageElement> {
    // Load image for canvas rendering
  }
}
```

#### 4.2 CLI Usage
```bash
# Example: Create visualizer from audio and image
node visualizer.js \
  --audio music.mp3 \
  --image cover.jpg \
  --output result.mp4 \
  --width 1920 \
  --height 1080 \
  --fps 30 \
  --waveform-color "#00FF00" \
  --background-color "#000000" \
  --bitrate 5000k
```

---

## Advanced Features

### 1. **Frequency-Based Coloring**
```typescript
function getColorFromFrequency(frequencyData: Uint8Array, index: number): string {
  const value = frequencyData[index]
  // Map frequency magnitude to color
  const hue = (value / 255) * 360
  return `hsl(${hue}, 100%, 50%)`
}
```

### 2. **Multiple Waveform Layers**
```typescript
// Draw layered waveforms with different scales and colors
function drawLayeredWaveform(peaks: PeakData[], colors: string[]) {
  for (let layerIdx = 0; layerIdx < colors.length; layerIdx++) {
    ctx.strokeStyle = colors[layerIdx]
    ctx.globalAlpha = 1 / (layerIdx + 1)
    
    // Draw waveform at different scale
    const scale = 1 / Math.pow(2, layerIdx)
    drawWaveform(peaks, scale)
  }
}
```

### 3. **Spectogram Overlay**
```typescript
// Add frequency spectrum visualization on top
function drawSpectrogram(frequencyData: Uint8Array, x: number, y: number) {
  for (let i = 0; i < frequencyData.length; i++) {
    const value = frequencyData[i]
    const barHeight = (value / 255) * height
    
    ctx.fillStyle = `hsl(${(i / frequencyData.length) * 360}, 100%, 50%)`
    ctx.fillRect(x + i, y + height - barHeight, 1, barHeight)
  }
}
```

### 4. **Animated Playhead Indicator**
```typescript
function drawPlayhead(
  ctx: CanvasRenderingContext2D,
  frameIndex: number,
  totalFrames: number,
  width: number,
  height: number
) {
  const x = (frameIndex / totalFrames) * width
  ctx.strokeStyle = '#FF0000'
  ctx.lineWidth = 3
  ctx.beginPath()
  ctx.moveTo(x, 0)
  ctx.lineTo(x, height)
  ctx.stroke()
}
```

---

## Performance Optimization

### 1. **Frame Batching**
```typescript
// Process frames in batches to manage memory
async function renderFramesBatch(
  startFrame: number,
  batchSize: number,
  peakData: PeakData[]
): Promise<void> {
  const batch = []
  for (let i = startFrame; i < startFrame + batchSize; i++) {
    const frame = await renderFrame(i, peakData)
    batch.push(frame)
  }
  // Save batch to disk
  await saveBatch(batch)
  // Clear memory
  batch.length = 0
}
```

### 2. **Downsampling for Preview**
```typescript
// Generate low-res preview quickly
function generatePreview(
  peakData: PeakData[],
  downscaleFactor: number = 4
) {
  const previewRenderer = new FrameRenderer({
    width: 480,
    height: 270,
    // ... other config
  })
  // Renders at 1/16th the frames initially
}
```

### 3. **GPU Acceleration**
```typescript
// Use WebGL for faster rendering
class WebGLRenderer {
  private gl: WebGLRenderingContext
  
  // Shader programs for waveform rendering
  // Much faster for large datasets
}
```

---

## Supported Audio Formats

| Format | Extension | Browser Support |
|--------|-----------|-----------------|
| MP3    | .mp3      | ✅ Universal    |
| WAV    | .wav      | ✅ Universal    |
| OGG    | .ogg      | ✅ Modern       |
| M4A    | .m4a      | ✅ Modern       |
| FLAC   | .flac     | ⚠️ Limited      |
| AIFF   | .aiff     | ⚠️ Limited      |

---

## Supported Image Formats

| Format | Extension | Recommendation |
|--------|-----------|-----------------|
| JPEG   | .jpg      | ✅ Recommended  |
| PNG    | .png      | ✅ Recommended  |
| WebP   | .webp     | ✅ Modern       |
| GIF    | .gif      | ⚠️ Limited      |

---

## Quality Settings

### Resolution Presets
```typescript
const PRESETS = {
  '1080p': { width: 1920, height: 1080, fps: 30, bitrate: '5000k' },
  '720p': { width: 1280, height: 720, fps: 30, bitrate: '2500k' },
  '480p': { width: 854, height: 480, fps: 30, bitrate: '1000k' },
  '4K': { width: 3840, height: 2160, fps: 60, bitrate: '15000k' }
}
```

### Encoding Speed vs Quality
```
CRF (Constant Rate Factor): 0-51
- 0: Lossless (largest file)
- 18-23: Visually lossless (recommended)
- 28: Default quality
- 51: Lowest quality (smallest file)
```

---

## Error Handling

```typescript
class VisualizerError extends Error {
  constructor(
    message: string,
    public phase: 'decoding' | 'rendering' | 'encoding',
    public originalError?: Error
  ) {
    super(message)
  }
}

// Validation
function validateInputs(input: VisualizerInput): void {
  if (!fs.existsSync(input.audioFile)) {
    throw new VisualizerError(
      `Audio file not found: ${input.audioFile}`,
      'decoding'
    )
  }
  
  if (!fs.existsSync(input.imageFile)) {
    throw new VisualizerError(
      `Image file not found: ${input.imageFile}`,
      'rendering'
    )
  }
  
  if (input.width < 320 || input.height < 240) {
    throw new VisualizerError(
      'Minimum resolution is 320x240',
      'rendering'
    )
  }
}
```

---

## Dependencies

### Required
```json
{
  "wavesurfer.js": "^8.0.0",
  "canvas": "^2.11.2"
}
```

### System Requirements
- Node.js 16+
- FFmpeg 4.0+
- 4GB RAM minimum
- 500MB disk space (for temp files)

### Installation
```bash
npm install wavesurfer.js canvas
# macOS
brew install ffmpeg
# Ubuntu/Debian
sudo apt-get install ffmpeg
# Windows
choco install ffmpeg
```

---

## Testing & Validation

### Unit Tests
```typescript
describe('FrameRenderer', () => {
  test('renders frame with correct dimensions', async () => {
    const renderer = new FrameRenderer({
      width: 1920,
      height: 1080,
      fps: 30,
      waveformColor: '#0099ff',
      backgroundColor: '#000000'
    })
    
    const peaks = generateMockPeaks()
    const image = await loadTestImage()
    
    const frame = await renderer.renderFrame(0, peaks, image)
    
    expect(frame).toBeDefined()
    expect(frame.length).toBeGreaterThan(0)
  })
})
```

### Integration Tests
```typescript
describe('AudioVisualizerPipeline', () => {
  test('generates complete MP4 from audio and image', async () => {
    const pipeline = new AudioVisualizerPipeline()
    
    await pipeline.generate({
      audioFile: 'test_audio.mp3',
      imageFile: 'test_image.jpg',
      outputFile: 'test_output.mp4',
      width: 1920,
      height: 1080,
      fps: 30,
      waveformColor: '#0099ff',
      backgroundColor: '#000000',
      bitrate: '5000k'
    })
    
    expect(fs.existsSync('test_output.mp4')).toBe(true)
  })
})
```

---

## Workflow Summary

1. **Input Preparation**
   - Audio file (MP3/WAV/OGG)
   - Background image (JPG/PNG)
   - Configuration parameters

2. **Audio Analysis**
   - Decode audio to PCM
   - Extract peak data
   - Optional: Compute frequency spectrum

3. **Frame Rendering**
   - Initialize Canvas rendering context
   - For each frame:
     - Draw background image
     - Render waveform
     - Apply animations/effects
     - Export as PNG

4. **Video Encoding**
   - Combine frame sequence with FFmpeg
   - Merge audio track
   - Encode to H.264 MP4
   - Clean temporary files

5. **Output**
   - Final MP4 video file
   - Ready for distribution

---

## Example Implementations

### Minimal Example
```typescript
const visualizer = new AudioVisualizerPipeline()
await visualizer.generate({
  audioFile: 'music.mp3',
  imageFile: 'cover.jpg',
  outputFile: 'result.mp4'
})
```

### Advanced Example with Custom Styling
```typescript
await visualizer.generate({
  audioFile: 'music.mp3',
  imageFile: 'cover.jpg',
  outputFile: 'result.mp4',
  width: 3840,
  height: 2160,
  fps: 60,
  waveformColor: '#FF00FF',
  backgroundColor: 'rgba(0,0,0,0.5)',
  bitrate: '15000k'
})
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| FFmpeg not found | Install FFmpeg: `brew install ffmpeg` |
| Audio decode fails | Ensure audio format is supported |
| Out of memory | Reduce resolution or use frame batching |
| Video codec error | Install libx264: `apt-get install libx264-dev` |
| Waveform not visible | Adjust waveformScale and colors |
| Audio sync issues | Check audio duration matches frame count |

---

## Performance Benchmarks

| Task | Time (1920x1080 @ 30fps) |
|------|-------------------------|
| Audio decoding | 2-5 seconds |
| Frame rendering | 5-15 minutes |
| Video encoding | 10-30 minutes |
| **Total** | **20-50 minutes** |

*Times vary based on audio length, CPU, and disk speed*

---

## License & Attribution

This guide builds upon:
- [Wavesurfer.js](https://github.com/katspaugh/wavesurfer.js) - BSD-3-Clause
- [FFmpeg](https://ffmpeg.org/) - LGPL 2.1+
- [Canvas API](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API)

---

## References

- [Wavesurfer.js Documentation](https://wavesurfer.xyz/docs/)
- [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [Canvas API Reference](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264)

---

**Version:** 1.0  
**Last Updated:** 2026-08-25  
**Author:** Audio Visualizer Implementation Guide
