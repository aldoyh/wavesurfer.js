# Audio-to-MP4 Visualizer - Complete Implementation Reference

## Quick Start: Minimal Working Example

### Project Structure
```
audio-visualizer/
├── src/
│   ├── index.ts                 # Main entry point
│   ├── audioProcessor.ts        # Audio decoding & analysis
│   ├── frameRenderer.ts         # Canvas rendering
│   ├── videoEncoder.ts          # FFmpeg integration
│   └── visualizerPipeline.ts    # Orchestration
├── package.json
├── tsconfig.json
├── ffmpeg.sh                    # FFmpeg script
└── README.md
```

### Setup Instructions

#### 1. Initialize Project
```bash
mkdir audio-visualizer
cd audio-visualizer
npm init -y
npm install --save-dev typescript ts-node @types/node
npm install wavesurfer.js canvas fluent-ffmpeg
```

#### 2. TypeScript Configuration
**tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020"],
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

---

## Core Implementation Files

### 1. Audio Processor (`audioProcessor.ts`)

```typescript
import { promises as fs } from 'fs'
import path from 'path'

interface AudioData {
  sampleRate: number
  duration: number
  channels: Float32Array[]
  peaks: PeakData[]
}

interface PeakData {
  min: number
  max: number
  rms: number // Root Mean Square for amplitude
}

export class AudioProcessor {
  private audioContext: AudioContext
  
  constructor() {
    this.audioContext = new (typeof AudioContext !== 'undefined' 
      ? AudioContext 
      : (globalThis as any).webkitAudioContext)()
  }
  
  /**
   * Decode audio file to PCM data
   */
  async decodeAudio(filePath: string): Promise<AudioData> {
    console.log(`📊 Decoding audio: ${filePath}`)
    
    try {
      const buffer = await fs.readFile(filePath)
      const arrayBuffer = buffer.buffer.slice(
        buffer.byteOffset,
        buffer.byteOffset + buffer.byteLength
      )
      
      const audioBuffer = await this.audioContext.decodeAudioData(arrayBuffer)
      
      return {
        sampleRate: audioBuffer.sampleRate,
        duration: audioBuffer.duration,
        channels: Array.from(
          { length: audioBuffer.numberOfChannels },
          (_, i) => audioBuffer.getChannelData(i)
        ),
        peaks: this.extractPeaks(audioBuffer)
      }
    } catch (error) {
      throw new Error(`Failed to decode audio: ${(error as Error).message}`)
    }
  }
  
  /**
   * Extract peak data for visualization
   * Divides audio into time-based buckets and finds min/max/RMS
   */
  private extractPeaks(
    audioBuffer: AudioBuffer,
    peaksPerSecond: number = 60
  ): PeakData[] {
    const peaks: PeakData[] = []
    const samplesPerPeak = audioBuffer.sampleRate / peaksPerSecond
    
    // Use first channel for now (can be enhanced for multi-channel)
    const channelData = audioBuffer.getChannelData(0)
    
    for (let i = 0; i < channelData.length; i += samplesPerPeak) {
      const sliceEnd = Math.min(i + samplesPerPeak, channelData.length)
      const slice = channelData.slice(i, sliceEnd)
      
      let min = 0
      let max = 0
      let sum = 0
      
      for (let sample of slice) {
        if (sample < min) min = sample
        if (sample > max) max = sample
        sum += sample * sample
      }
      
      const rms = Math.sqrt(sum / slice.length)
      peaks.push({ min, max, rms })
    }
    
    return peaks
  }
  
  /**
   * Extract frequency spectrum using FFT
   */
  async extractFrequencySpectrum(
    audioBuffer: AudioBuffer,
    fftSize: number = 2048
  ): Promise<Uint8Array[]> {
    const spectrumFrames: Uint8Array[] = []
    const channelData = audioBuffer.getChannelData(0)
    const hop = audioBuffer.sampleRate / 60 // 60 fps
    
    // Note: This is a simplified approach
    // For production, use FFT.js or similar
    
    return spectrumFrames
  }
  
  /**
   * Normalize peak data to 0-1 range
   */
  normalizePeaks(peaks: PeakData[]): PeakData[] {
    const maxAbs = Math.max(
      ...peaks.map(p => Math.max(Math.abs(p.min), Math.abs(p.max)))
    )
    
    return peaks.map(p => ({
      min: p.min / maxAbs,
      max: p.max / maxAbs,
      rms: p.rms / maxAbs
    }))
  }
}
```

### 2. Frame Renderer (`frameRenderer.ts`)

```typescript
import { createCanvas, loadImage } from 'canvas'
import { promises as fs } from 'fs'
import path from 'path'

interface RendererConfig {
  width: number
  height: number
  fps: number
  waveformColor: string
  backgroundColor: string
  backgroundImagePath?: string
  waveformLineWidth: number
  waveformScale: number
}

export class FrameRenderer {
  private config: RendererConfig
  private backgroundImage: any = null
  
  constructor(config: RendererConfig) {
    this.config = {
      waveformLineWidth: 2,
      waveformScale: 0.8,
      ...config
    }
  }
  
  async initialize(): Promise<void> {
    if (this.config.backgroundImagePath) {
      try {
        this.backgroundImage = await loadImage(this.config.backgroundImagePath)
      } catch (error) {
        console.warn('Failed to load background image:', error)
      }
    }
  }
  
  /**
   * Render single frame
   */
  async renderFrame(
    frameIndex: number,
    peaks: PeakData[],
    totalFrames: number
  ): Promise<Buffer> {
    const canvas = createCanvas(this.config.width, this.config.height)
    const ctx = canvas.getContext('2d')
    
    // Draw background
    this.drawBackground(ctx)
    
    // Draw waveform
    const framePeaksEnd = Math.floor((frameIndex / totalFrames) * peaks.length)
    const framePeaks = peaks.slice(0, framePeaksEnd)
    this.drawWaveform(ctx, framePeaks)
    
    // Draw playhead
    this.drawPlayhead(ctx, frameIndex, totalFrames)
    
    // Export as PNG
    return canvas.toBuffer('image/png')
  }
  
  private drawBackground(ctx: CanvasRenderingContext2D): void {
    if (this.backgroundImage) {
      // Draw scaled background
      ctx.drawImage(
        this.backgroundImage,
        0,
        0,
        this.config.width,
        this.config.height
      )
      
      // Optional: Add overlay
      ctx.fillStyle = 'rgba(0, 0, 0, 0.3)'
      ctx.fillRect(0, 0, this.config.width, this.config.height)
    } else {
      // Solid background
      ctx.fillStyle = this.config.backgroundColor
      ctx.fillRect(0, 0, this.config.width, this.config.height)
    }
  }
  
  private drawWaveform(
    ctx: CanvasRenderingContext2D,
    peaks: PeakData[]
  ): void {
    const { width, height } = this.config
    const centerY = height / 2
    const scale = this.config.waveformScale
    
    ctx.strokeStyle = this.config.waveformColor
    ctx.lineWidth = this.config.waveformLineWidth
    ctx.lineCap = 'round'
    ctx.lineJoin = 'round'
    
    // Draw top waveform (positive samples)
    ctx.beginPath()
    for (let i = 0; i < peaks.length; i++) {
      const x = (i / peaks.length) * width
      const y = centerY - peaks[i].max * height * scale / 2
      
      if (i === 0) {
        ctx.moveTo(x, y)
      } else {
        ctx.lineTo(x, y)
      }
    }
    ctx.stroke()
    
    // Draw bottom waveform (negative samples)
    ctx.beginPath()
    for (let i = 0; i < peaks.length; i++) {
      const x = (i / peaks.length) * width
      const y = centerY + Math.abs(peaks[i].min) * height * scale / 2
      
      if (i === 0) {
        ctx.moveTo(x, y)
      } else {
        ctx.lineTo(x, y)
      }
    }
    ctx.stroke()
    
    // Optional: Fill waveform area
    this.drawWaveformFill(ctx, peaks, centerY, scale)
  }
  
  private drawWaveformFill(
    ctx: CanvasRenderingContext2D,
    peaks: PeakData[],
    centerY: number,
    scale: number
  ): void {
    const { width, height } = this.config
    
    ctx.fillStyle = this.hexToRgba(this.config.waveformColor, 0.2)
    ctx.beginPath()
    
    // Top path
    for (let i = 0; i < peaks.length; i++) {
      const x = (i / peaks.length) * width
      const y = centerY - peaks[i].max * height * scale / 2
      if (i === 0) ctx.moveTo(x, y)
      else ctx.lineTo(x, y)
    }
    
    // Bottom path (reverse)
    for (let i = peaks.length - 1; i >= 0; i--) {
      const x = (i / peaks.length) * width
      const y = centerY + Math.abs(peaks[i].min) * height * scale / 2
      ctx.lineTo(x, y)
    }
    
    ctx.closePath()
    ctx.fill()
  }
  
  private drawPlayhead(
    ctx: CanvasRenderingContext2D,
    frameIndex: number,
    totalFrames: number
  ): void {
    const x = (frameIndex / totalFrames) * this.config.width
    
    ctx.strokeStyle = '#FF0000'
    ctx.lineWidth = 3
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, this.config.height)
    ctx.stroke()
    
    // Draw playhead label
    ctx.fillStyle = '#FF0000'
    ctx.font = 'bold 12px Arial'
    ctx.fillText(
      `${(frameIndex / this.totalFrames * 100).toFixed(0)}%`,
      x + 5,
      20
    )
  }
  
  private hexToRgba(hex: string, alpha: number): string {
    const r = parseInt(hex.slice(1, 3), 16)
    const g = parseInt(hex.slice(3, 5), 16)
    const b = parseInt(hex.slice(5, 7), 16)
    return `rgba(${r}, ${g}, ${b}, ${alpha})`
  }
  
  private totalFrames: number = 0
}
```

### 3. Video Encoder (`videoEncoder.ts`)

```typescript
import { spawn } from 'child_process'
import path from 'path'
import { promises as fs } from 'fs'

export class VideoEncoder {
  /**
   * Encode frame sequence to MP4 with audio
   */
  async encodeVideo(
    framesDir: string,
    audioPath: string,
    outputPath: string,
    fps: number = 30,
    bitrate: string = '5000k',
    crf: number = 23
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const ffmpegArgs = [
        // Input frames
        '-framerate', fps.toString(),
        '-i', path.join(framesDir, 'frame_%06d.png'),
        
        // Input audio
        '-i', audioPath,
        
        // Video codec options
        '-c:v', 'libx264',
        '-pix_fmt', 'yuv420p',
        '-b:v', bitrate,
        '-crf', crf.toString(),
        '-preset', 'medium',
        
        // Audio codec options
        '-c:a', 'aac',
        '-b:a', '128k',
        
        // Sync audio with video
        '-shortest',
        
        // Output
        outputPath
      ]
      
      console.log('🎬 Starting FFmpeg encoding...')
      console.log('ffmpeg', ffmpegArgs.join(' '))
      
      const ffmpeg = spawn('ffmpeg', ffmpegArgs, {
        stdio: ['pipe', 'pipe', 'pipe']
      })
      
      let progressBuffer = ''
      
      ffmpeg.stderr.on('data', (data) => {
        const output = data.toString()
        progressBuffer += output
        
        // Parse progress
        const match = output.match(/frame=\s*(\d+)/)
        if (match) {
          process.stdout.write(
            `\r  Encoded frame ${match[1]}...`
          )
        }
      })
      
      ffmpeg.on('close', (code) => {
        console.log('\n')
        if (code === 0) {
          console.log(`✅ Video encoded: ${outputPath}`)
          resolve()
        } else {
          console.error('❌ FFmpeg error output:', progressBuffer)
          reject(new Error(`FFmpeg exited with code ${code}`))
        }
      })
      
      ffmpeg.on('error', (error) => {
        reject(new Error(`Failed to start FFmpeg: ${error.message}`))
      })
    })
  }
  
  /**
   * Get video information
   */
  async getVideoInfo(videoPath: string): Promise<any> {
    return new Promise((resolve, reject) => {
      const ffprobe = spawn('ffprobe', [
        '-v', 'quiet',
        '-print_format', 'json',
        '-show_format',
        '-show_streams',
        videoPath
      ])
      
      let output = ''
      
      ffprobe.stdout.on('data', (data) => {
        output += data.toString()
      })
      
      ffprobe.on('close', (code) => {
        if (code === 0) {
          try {
            resolve(JSON.parse(output))
          } catch (error) {
            reject(error)
          }
        } else {
          reject(new Error(`ffprobe failed with code ${code}`))
        }
      })
      
      ffprobe.on('error', reject)
    })
  }
}
```

### 4. Main Pipeline (`visualizerPipeline.ts`)

```typescript
import path from 'path'
import { promises as fs } from 'fs'
import { AudioProcessor, PeakData } from './audioProcessor'
import { FrameRenderer } from './frameRenderer'
import { VideoEncoder } from './videoEncoder'

export interface VisualizerConfig {
  audioFile: string
  imageFile: string
  outputFile: string
  width?: number
  height?: number
  fps?: number
  waveformColor?: string
  backgroundColor?: string
  bitrate?: string
  crf?: number
}

const DEFAULTS: Partial<VisualizerConfig> = {
  width: 1920,
  height: 1080,
  fps: 30,
  waveformColor: '#0099FF',
  backgroundColor: '#000000',
  bitrate: '5000k',
  crf: 23
}

export class AudioVisualizerPipeline {
  private tempDir: string
  private config: Required<VisualizerConfig>
  
  constructor(userConfig: VisualizerConfig) {
    this.config = { ...DEFAULTS, ...userConfig } as Required<VisualizerConfig>
    this.tempDir = path.join(process.cwd(), '.temp_frames_' + Date.now())
  }
  
  async generate(): Promise<void> {
    try {
      await this.validateInputs()
      
      console.log('\n🎬 Starting audio visualizer generation...\n')
      
      // Step 1: Process audio
      const audioData = await this.processAudio()
      
      // Step 2: Prepare rendering
      const renderer = new FrameRenderer({
        width: this.config.width,
        height: this.config.height,
        fps: this.config.fps,
        waveformColor: this.config.waveformColor,
        backgroundColor: this.config.backgroundColor,
        backgroundImagePath: this.config.imageFile,
        waveformLineWidth: 2,
        waveformScale: 0.8
      })
      await renderer.initialize()
      
      // Step 3: Render frames
      const totalFrames = Math.ceil(audioData.duration * this.config.fps)
      await this.renderFrames(renderer, audioData.peaks, totalFrames)
      
      // Step 4: Encode video
      const encoder = new VideoEncoder()
      await encoder.encodeVideo(
        this.tempDir,
        this.config.audioFile,
        this.config.outputFile,
        this.config.fps,
        this.config.bitrate,
        this.config.crf
      )
      
      // Step 5: Cleanup
      await this.cleanup()
      
      console.log('\n✅ Done!\n')
    } catch (error) {
      await this.cleanup()
      throw error
    }
  }
  
  private async validateInputs(): Promise<void> {
    const { audioFile, imageFile } = this.config
    
    console.log('📋 Validating inputs...')
    
    try {
      await fs.access(audioFile)
    } catch {
      throw new Error(`Audio file not found: ${audioFile}`)
    }
    
    try {
      await fs.access(imageFile)
    } catch {
      throw new Error(`Image file not found: ${imageFile}`)
    }
    
    if (this.config.width < 320 || this.config.height < 240) {
      throw new Error('Minimum resolution is 320x240')
    }
    
    console.log('✅ Validation passed\n')
  }
  
  private async processAudio(): Promise<any> {
    console.log('📊 Processing audio...')
    const processor = new AudioProcessor()
    const audioData = await processor.decodeAudio(this.config.audioFile)
    const normalized = processor.normalizePeaks(audioData.peaks)
    
    console.log(
      `✅ Audio loaded: ${audioData.duration.toFixed(2)}s, ` +
      `${audioData.channels.length} channel(s)\n`
    )
    
    return { ...audioData, peaks: normalized }
  }
  
  private async renderFrames(
    renderer: FrameRenderer,
    peaks: PeakData[],
    totalFrames: number
  ): Promise<void> {
    console.log(`🎨 Rendering ${totalFrames} frames...`)
    
    // Create temp directory
    await fs.mkdir(this.tempDir, { recursive: true })
    
    // Render frames
    for (let i = 0; i < totalFrames; i++) {
      if (i % 30 === 0) {
        const progress = ((i / totalFrames) * 100).toFixed(1)
        console.log(`  ${progress}% (${i}/${totalFrames})`)
      }
      
      const frameBuffer = await renderer.renderFrame(i, peaks, totalFrames)
      const framePath = path.join(
        this.tempDir,
        `frame_${String(i).padStart(6, '0')}.png`
      )
      
      await fs.writeFile(framePath, frameBuffer)
    }
    
    console.log('✅ Frames rendered\n')
  }
  
  private async cleanup(): Promise<void> {
    try {
      await fs.rm(this.tempDir, { recursive: true })
      console.log('✅ Cleaned up temporary files')
    } catch (error) {
      console.warn('⚠️  Failed to cleanup temp directory:', error)
    }
  }
}
```

### 5. Entry Point (`index.ts`)

```typescript
import { AudioVisualizerPipeline } from './visualizerPipeline'
import path from 'path'

async function main() {
  const args = process.argv.slice(2)
  
  if (args.length < 2) {
    console.log(`
Usage: ts-node index.ts <audio-file> <image-file> [output-file] [options]

Examples:
  ts-node index.ts music.mp3 cover.jpg output.mp4
  ts-node index.ts song.wav album.png result.mp4 --width 1920 --fps 60
  
Options:
  --width <number>           Video width (default: 1920)
  --height <number>          Video height (default: 1080)
  --fps <number>            Frames per second (default: 30)
  --waveform-color <hex>    Waveform color (default: #0099FF)
  --bg-color <hex>          Background color (default: #000000)
  --bitrate <string>        Video bitrate (default: 5000k)
  --crf <number>           Quality 0-51 (default: 23)
    `)
    process.exit(1)
  }
  
  // Parse arguments
  const audioFile = args[0]
  const imageFile = args[1]
  const outputFile = args[2] || 'output.mp4'
  
  const config: any = {
    audioFile,
    imageFile,
    outputFile
  }
  
  // Parse options
  for (let i = 3; i < args.length; i++) {
    if (args[i].startsWith('--')) {
      const key = args[i].slice(2)
      const value = args[i + 1]
      
      if (key === 'width' || key === 'height' || key === 'fps' || key === 'crf') {
        config[key] = parseInt(value)
      } else if (key === 'waveform-color') {
        config.waveformColor = value
      } else if (key === 'bg-color') {
        config.backgroundColor = value
      } else if (key === 'bitrate') {
        config.bitrate = value
      }
      
      i++ // Skip next argument
    }
  }
  
  try {
    const pipeline = new AudioVisualizerPipeline(config)
    await pipeline.generate()
  } catch (error) {
    console.error('❌ Error:', (error as Error).message)
    process.exit(1)
  }
}

main()
```

---

## Package Configuration

### `package.json`
```json
{
  "name": "audio-to-mp4-visualizer",
  "version": "1.0.0",
  "description": "Convert audio + image to animated MP4 video using wavesurfer.js",
  "main": "dist/index.js",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "ts-node src/index.ts",
    "dev": "ts-node src/index.ts",
    "test": "jest"
  },
  "dependencies": {
    "wavesurfer.js": "^8.0.0",
    "canvas": "^2.11.2"
  },
  "devDependencies": {
    "typescript": "^5.9.3",
    "@types/node": "^20.0.0",
    "ts-node": "^10.9.1"
  }
}
```

---

## Usage Examples

### Basic Usage
```bash
npm run dev music.mp3 cover.jpg output.mp4
```

### Full Example with Options
```bash
npm run dev song.wav album.jpg result.mp4 \
  --width 1920 \
  --height 1080 \
  --fps 60 \
  --waveform-color "#00FF00" \
  --bg-color "#1A1A1A" \
  --bitrate 8000k \
  --crf 20
```

### Programmatic Usage
```typescript
import { AudioVisualizerPipeline } from './src/visualizerPipeline'

const pipeline = new AudioVisualizerPipeline({
  audioFile: 'music.mp3',
  imageFile: 'cover.jpg',
  outputFile: 'result.mp4',
  width: 1920,
  height: 1080,
  fps: 30
})

await pipeline.generate()
```

---

## Advanced Customizations

### Custom Waveform Style

```typescript
// In FrameRenderer.drawWaveform()
ctx.strokeStyle = '#FF00FF'
ctx.lineWidth = 1
ctx.globalAlpha = 0.8

// Add gradient
const gradient = ctx.createLinearGradient(0, 0, width, height)
gradient.addColorStop(0, '#FF00FF')
gradient.addColorStop(1, '#00FFFF')
ctx.strokeStyle = gradient
```

### Frequency-Based Coloring

```typescript
private drawFrequencyBars(
  ctx: CanvasRenderingContext2D,
  frequencyData: Uint8Array
): void {
  const barWidth = this.config.width / frequencyData.length
  
  for (let i = 0; i < frequencyData.length; i++) {
    const value = frequencyData[i]
    const hue = (i / frequencyData.length) * 360
    
    ctx.fillStyle = `hsl(${hue}, 100%, ${50 + value / 5}%)`
    ctx.fillRect(
      i * barWidth,
      this.config.height - (value / 255) * this.config.height,
      barWidth,
      (value / 255) * this.config.height
    )
  }
}
```

---

## Troubleshooting

### "ffmpeg: command not found"
```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
sudo apt-get install ffmpeg

# Windows (if using Chocolatey)
choco install ffmpeg
```

### "Out of memory" error
- Reduce resolution: `--width 1280 --height 720`
- Reduce bitrate: `--bitrate 2000k`
- Process in chunks (implement frame batching)

### Audio not syncing
- Ensure audio duration matches frame count
- Use `-shortest` flag in FFmpeg (already included)
- Check if audio file is VBR (convert to CBR)

### Waveform not visible
- Adjust waveform color to contrast with background
- Increase `waveformScale` in config
- Check audio levels aren't clipping

---

## Performance Tips

1. **GPU Acceleration**: Use `--preset fast` in FFmpeg for faster encoding
2. **Parallel Encoding**: Use Python script to process multiple files
3. **Caching**: Cache decoded audio for multiple renders
4. **Memory**: Use streaming for large files instead of loading entirely

---

## Testing

```typescript
describe('AudioVisualizerPipeline', () => {
  test('should generate MP4 from audio and image', async () => {
    const pipeline = new AudioVisualizerPipeline({
      audioFile: './test/sample.mp3',
      imageFile: './test/sample.jpg',
      outputFile: './test/output.mp4'
    })
    
    await pipeline.generate()
    
    const exists = fs.existsSync('./test/output.mp4')
    expect(exists).toBe(true)
  })
})
```

---

**Ready for agent implementation!** 🚀
