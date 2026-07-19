I'll design a comprehensive architecture for your AI Interview Flutter application that avoids MediaProjection by recording **internally within the Flutter render pipeline** rather than capturing the device screen. This is the most robust approach for your requirements.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           FLUTTER LAYER                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │  Interview  │  │   Camera    │  │  Transcript │  │    Timer    │   │
│  │    Screen   │  │   Preview   │  │   Widget    │  │   Widget    │   │
│  │  (AI Avatar)│  │  (Texture)  │  │             │  │             │   │
│  └──────┬──────┘  └──────┬──────┘  └─────────────┘  └─────────────┘   │
│         │                │                                              │
│         └────────────────┼──────────────────────────────────────────────┘
│                          │                                              │
│                   ┌──────▼──────┐                                       │
│                   │  Flutter    │  ← Renders to SurfaceTexture         │
│                   │  Renderer   │     (Offscreen FBO)                   │
│                   │  (Custom    │                                       │
│                   │  Painter)   │                                       │
│                   └──────┬──────┘                                       │
│                          │                                              │
│         ┌────────────────┼────────────────┐                            │
│         │                │                │                            │
│    ┌────▼────┐    ┌─────▼─────┐   ┌─────▼─────┐                       │
│    │  TTS    │    │  STT/Speech│   │  Recording│                       │
│    │ Service │    │  Recognition│   │  Controller│                       │
│    │ (AI Voice)│   │  (User Mic) │   │             │                       │
│    └────┬────┘    └─────┬─────┘   └─────┬─────┘                       │
│         │               │               │                              │
│         └───────────────┼───────────────┘                              │
│                         │                                              │
│                   ┌─────▼─────┐                                         │
│                   │  Method   │  ← Platform Channels                    │
│                   │  Channel  │                                         │
│                   └─────┬─────┘                                         │
└────────────────────────┼──────────────────────────────────────────────┘
                         │
┌────────────────────────┼──────────────────────────────────────────────┐
│                   NATIVE LAYER                                         │
│                         │                                              │
│    ┌────────────────────┼────────────────────┐                        │
│    │                    │                    │                        │
│ ┌──▼────────┐    ┌─────▼──────┐    ┌──────▼──────┐                  │
│ │  ANDROID  │    │   SHARED   │    │    iOS      │                  │
│ │  LAYER    │    │   AUDIO    │    │   LAYER     │                  │
│ │           │    │   MIXER    │    │             │                  │
│ └───────────┘    │  (C++ FFI) │    └─────────────┘                  │
│                  └────────────┘                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Core Strategy: Internal App Recording (No MediaProjection)

Instead of capturing the **device screen** (MediaProjection), we record the **Flutter app's own render output** using platform-specific APIs that capture only our application's content:

| Approach | Android | iOS |
|----------|---------|-----|
| **Screen Capture** | MediaProjection ❌ | ReplayKit ❌ |
| **Internal Recording** | MediaCodec + Surface ✅ | AVAssetWriter + CALayer ✅ |
| **Flutter Rendering** | TextureRegistry / SurfaceTexture ✅ | FlutterTexture ✅ |

---

## 1. Flutter Layer Architecture

### 1.1 Interview Screen Layout

```dart
// lib/screens/interview_screen.dart
class InterviewScreen extends StatefulWidget {
  @override
  _InterviewScreenState createState() => _InterviewScreenState();
}

class _InterviewScreenState extends State<InterviewScreen> {
  // Controllers
  late CameraController _cameraController;
  late InterviewRecorder _recorder;
  late TTSManager _ttsManager;
  late SpeechToText _sttManager;
  
  // State
  String _transcript = '';
  Duration _elapsed = Duration.zero;
  bool _isRecording = false;
  
  // Key: CustomPainter for capturing Flutter UI
  final GlobalKey _renderKey = GlobalKey();
  
  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      key: _renderKey,  // Used for frame capture
      child: Scaffold(
        backgroundColor: Colors.black,
        body: Stack(
          children: [
            // Layer 1: AI Avatar (Video/Animated)
            Positioned.fill(
              child: AIAvatarWidget(
                videoUrl: _currentQuestion?.avatarUrl,
                isSpeaking: _ttsManager.isSpeaking,
              ),
            ),
            
            // Layer 2: User Camera Preview (Picture-in-Picture)
            Positioned(
              right: 16,
              top: 16,
              width: 120,
              height: 160,
              child: ClipRRect(
                borderRadius: BorderRadius.circular(12),
                child: CameraPreview(_cameraController),
              ),
            ),
            
            // Layer 3: Transcript Overlay
            Positioned(
              bottom: 120,
              left: 16,
              right: 16,
              child: TranscriptWidget(text: _transcript),
            ),
            
            // Layer 4: Timer
            Positioned(
              top: 16,
              left: 16,
              child: TimerWidget(elapsed: _elapsed),
            ),
            
            // Layer 5: Controls
            Positioned(
              bottom: 32,
              left: 0,
              right: 0,
              child: InterviewControls(
                onStart: _startInterview,
                onStop: _stopInterview,
                isRecording: _isRecording,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### 1.2 Custom Render Capture (The Critical Piece)

```dart
// lib/core/render_capture.dart
/// Captures Flutter UI frames at 30fps and sends to native encoder
class FlutterRenderCapture {
  final MethodChannel _channel = 
      const MethodChannel('com.interview.recorder');
  
  final GlobalKey _repaintKey;
  final int _targetFps;
  Timer? _frameTimer;
  
  FlutterRenderCapture(this._repaintKey, {int fps = 30}) 
      : _targetFps = fps;
  
  void startCapture() {
    final interval = Duration(milliseconds: 1000 ~/ _targetFps);
    
    _frameTimer = Timer.periodic(interval, (_) async {
      final image = await _captureFrame();
      if (image != null) {
        // Send raw RGBA bytes to native encoder
        final byteData = await image.toByteData(
          format: ImageByteFormat.rawRgba,
        );
        _channel.invokeMethod('onFrame', {
          'width': image.width,
          'height': image.height,
          'bytes': byteData?.buffer.asUint8List(),
          'timestamp': DateTime.now().millisecondsSinceEpoch,
        });
      }
    });
  }
  
  Future<ui.Image?> _captureFrame() async {
    final boundary = _repaintKey.currentContext?.findRenderObject() 
        as RenderRepaintBoundary?;
    if (boundary == null) return null;
    
    // Capture at 720p for performance
    return boundary.toImage(pixelRatio: 1.5);
  }
  
  void stopCapture() => _frameTimer?.cancel();
}
```

**Performance Optimization**: Instead of `RepaintBoundary.toImage()` (slow), we use a **custom Flutter Texture** backed by an OpenGL FBO shared with native code.

---

## 2. Native Android Layer (MediaCodec)

### 2.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    ANDROID LAYER                         │
│                                                          │
│  ┌─────────────────────────────────────────────────┐  │
│  │           Flutter Activity/Fragment                │  │
│  │  ┌─────────────┐      ┌─────────────────────┐   │  │
│  │  │  Flutter    │◄────►│  PlatformView         │   │  │
│  │  │  Engine     │      │  (Camera Texture)     │   │  │
│  │  │  (Surface)  │      └─────────────────────┘   │  │
│  │  └──────┬──────┘                                  │  │
│  │         │                                           │  │
│  │  ┌──────▼─────────────────────────────────────┐   │  │
│  │  │   CustomFlutterSurface (OpenGL ES FBO)       │   │  │
│  │  │   - Renders Flutter UI to offscreen texture   │   │  │
│  │  │   - Shared with MediaCodec input surface      │   │  │
│  │  └──────┬─────────────────────────────────────┘   │  │
│  │         │                                           │  │
│  │  ┌──────▼─────────────────────────────────────┐   │  │
│  │  │   MediaCodec (H.264/AAC Encoder)             │   │  │
│  │  │   ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │  │
│  │  │   │ Video   │  │  Audio  │  │  Muxer  │   │   │  │
│  │  │   │ Encoder │  │ Encoder │  │(MP4)    │   │   │  │
│  │  │   │(Surface)│  │(AudioRecord)│           │   │   │  │
│  │  │   └────┬────┘  └────┬────┘  └────┬────┘   │   │  │
│  │  │        │            │            │          │   │  │
│  │  │        └────────────┴────────────┘          │   │  │
│  │  │                   ▼                          │   │  │
│  │  │            ┌─────────────┐                  │   │  │
│  │  │            │  MediaMuxer │                  │   │  │
│  │  │            │  (MP4 Output)│                  │   │  │
│  │  │            └─────────────┘                  │   │  │
│  │  └─────────────────────────────────────────────┘   │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  AudioMixer (C++ via JNI / Oboe)             │  │  │
│  │  │  - Mixes AI TTS + User Mic                    │  │  │
│  │  │  - Applies noise suppression                  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Implementation: Custom Flutter Surface + MediaCodec

```kotlin
// android/app/src/main/kotlin/com/interview/recorder/FlutterRecorder.kt
class FlutterRecorder(
    private val context: Context,
    private val flutterEngine: FlutterEngine
) {
    // OpenGL ES 3.0 context for offscreen rendering
    private var eglContext: EGLContext? = null
    private var eglDisplay: EGLDisplay? = null
    private var eglSurface: EGLSurface? = null
    
    // MediaCodec for H.264 encoding
    private var videoEncoder: MediaCodec? = null
    private var audioEncoder: MediaCodec? = null
    private var mediaMuxer: MediaMuxer? = null
    
    // Shared surface between Flutter and MediaCodec
    private var sharedSurface: Surface? = null
    
    // Audio components
    private var audioRecord: AudioRecord? = null
    private var audioMixer: AudioMixer? = null
    
    companion object {
        const val VIDEO_WIDTH = 1280
        const val VIDEO_HEIGHT = 720
        const val VIDEO_BITRATE = 4_000_000  // 4 Mbps
        const val FRAME_RATE = 30
        const val AUDIO_SAMPLE_RATE = 48000
        const val AUDIO_CHANNELS = 2
    }
    
    /**
     * Initialize OpenGL ES context and shared surface
     * This surface will be used by Flutter to render offscreen
     */
    fun initialize() {
        setupEgl()
        setupVideoEncoder()
        setupAudioEncoder()
        setupMuxer()
        setupAudioCapture()
    }
    
    private fun setupEgl() {
        eglDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY)
        val version = IntArray(2)
        EGL14.eglInitialize(eglDisplay, version, 0)
        
        val attribList = intArrayOf(
            EGL14.EGL_RED_SIZE, 8,
            EGL14.EGL_GREEN_SIZE, 8,
            EGL14.EGL_BLUE_SIZE, 8,
            EGL14.EGL_ALPHA_SIZE, 8,
            EGL14.EGL_RENDERABLE_TYPE, EGLExt.EGL_OPENGL_ES3_BIT_KHR,
            EGL14.EGL_NONE
        )
        
        val config = arrayOfNulls<EGLConfig>(1)
        val numConfigs = IntArray(1)
        EGL14.eglChooseConfig(eglDisplay, attribList, 0, config, 0, 1, numConfigs, 0)
        
        val contextAttribs = intArrayOf(
            EGL14.EGL_CONTEXT_CLIENT_VERSION, 3,
            EGL14.EGL_NONE
        )
        
        eglContext = EGL14.eglCreateContext(
            eglDisplay, config[0], EGL14.EGL_NO_CONTEXT, contextAttribs, 0
        )
        
        // Create offscreen pbuffer surface for Flutter rendering
        val surfaceAttribs = intArrayOf(
            EGL14.EGL_WIDTH, VIDEO_WIDTH,
            EGL14.EGL_HEIGHT, VIDEO_HEIGHT,
            EGL14.EGL_NONE
        )
        
        eglSurface = EGL14.eglCreatePbufferSurface(
            eglDisplay, config[0], surfaceAttribs, 0
        )
    }
    
    /**
     * Setup MediaCodec video encoder with Surface input
     * This is KEY: MediaCodec provides a Surface that Flutter renders into
     */
    private fun setupVideoEncoder() {
        val format = MediaFormat.createVideoFormat(MediaFormat.MIMETYPE_VIDEO_AVC, 
            VIDEO_WIDTH, VIDEO_HEIGHT).apply {
            setInteger(MediaFormat.KEY_BIT_RATE, VIDEO_BITRATE)
            setInteger(MediaFormat.KEY_FRAME_RATE, FRAME_RATE)
            setInteger(MediaFormat.KEY_COLOR_FORMAT, 
                MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface)
            setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, 1)
        }
        
        videoEncoder = MediaCodec.createEncoderByType(MediaFormat.MIMETYPE_VIDEO_AVC)
        videoEncoder?.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE)
        
        // CRITICAL: Get input surface from encoder
        // Flutter will render directly into this surface
        sharedSurface = videoEncoder?.createInputSurface()
        videoEncoder?.start()
    }
    
    /**
     * Setup AAC audio encoder
     */
    private fun setupAudioEncoder() {
        val format = MediaFormat.createAudioFormat(MediaFormat.MIMETYPE_AUDIO_AAC,
            AUDIO_SAMPLE_RATE, AUDIO_CHANNELS).apply {
            setInteger(MediaFormat.KEY_BIT_RATE, 128_000)
            setInteger(MediaFormat.KEY_AAC_PROFILE, 
                MediaCodecInfo.CodecProfileLevel.AACObjectHE)
        }
        
        audioEncoder = MediaCodec.createEncoderByType(MediaFormat.MIMETYPE_AUDIO_AAC)
        audioEncoder?.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE)
        audioEncoder?.start()
    }
    
    /**
     * Initialize MediaMuxer for MP4 output
     */
    private fun setupMuxer() {
        val outputFile = File(context.cacheDir, "interview_${System.currentTimeMillis()}.mp4")
        mediaMuxer = MediaMuxer(outputFile.absolutePath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4)
    }
    
    /**
     * Setup audio capture from microphone + TTS mixing
     */
    private fun setupAudioCapture() {
        val minBufferSize = AudioRecord.getMinBufferSize(
            AUDIO_SAMPLE_RATE,
            AudioFormat.CHANNEL_IN_STEREO,
            AudioFormat.ENCODING_PCM_16BIT
        )
        
        audioRecord = AudioRecord(
            MediaRecorder.AudioSource.MIC,
            AUDIO_SAMPLE_RATE,
            AudioFormat.CHANNEL_IN_STEREO,
            AudioFormat.ENCODING_PCM_16BIT,
            minBufferSize * 2
        )
        
        // Initialize C++ audio mixer via JNI
        audioMixer = AudioMixer(AUDIO_SAMPLE_RATE, AUDIO_CHANNELS)
    }
    
    /**
     * Called by Flutter for every frame (via MethodChannel)
     * Instead of copying pixels, we redirect Flutter's render to our shared surface
     */
    fun onFlutterFrame(textureId: Long, transformMatrix: FloatArray) {
        // Make our EGL context current
        EGL14.eglMakeCurrent(eglDisplay, eglSurface, eglSurface, eglContext)
        
        // Render Flutter's texture to our shared surface
        // This is done via OpenGL ES shader that applies the transform
        renderFlutterTextureToEncoder(textureId, transformMatrix)
        
        // Signal MediaCodec that a new frame is available
        // MediaCodec internally reads from the shared surface
        videoEncoder?.signalEndOfInputStream() // Only for final frame
        // For continuous: use eglSwapBuffers with sharedSurface
    }
    
    /**
     * Main recording loop
     */
    fun startRecording() {
        audioRecord?.startRecording()
        
        // Start encoding threads
        Thread { videoEncodeLoop() }.start()
        Thread { audioEncodeLoop() }.start()
    }
    
    private fun videoEncodeLoop() {
        val bufferInfo = MediaCodec.BufferInfo()
        var videoTrackIndex = -1
        
        while (isRecording) {
            val outputBufferId = videoEncoder?.dequeueOutputBuffer(bufferInfo, 10_000) ?: break
            if (outputBufferId >= 0) {
                val encodedBuffer = videoEncoder?.getOutputBuffer(outputBufferId)
                
                if (bufferInfo.flags and MediaCodec.BUFFER_FLAG_CODEC_CONFIG != 0) {
                    // SPS/PPS data for video track format
                    val format = videoEncoder?.outputFormat
                    videoTrackIndex = mediaMuxer?.addTrack(format!!) ?: -1
                    if (audioTrackIndex >= 0) mediaMuxer?.start()
                } else {
                    encodedBuffer?.let {
                        mediaMuxer?.writeSampleData(videoTrackIndex, it, bufferInfo)
                    }
                }
                videoEncoder?.releaseOutputBuffer(outputBufferId, false)
            }
        }
    }
    
    private fun audioEncodeLoop() {
        val buffer = ByteBuffer.allocateDirect(4096)
        val bufferInfo = MediaCodec.BufferInfo()
        
        while (isRecording) {
            // Read mixed audio (AI TTS + User Mic)
            val readSize = audioMixer?.readMixedAudio(buffer, buffer.capacity()) ?: 0
            
            if (readSize > 0) {
                // Feed to AAC encoder
                val inputBufferId = audioEncoder?.dequeueInputBuffer(10_000) ?: break
                val inputBuffer = audioEncoder?.getInputBuffer(inputBufferId)
                inputBuffer?.clear()
                inputBuffer?.put(buffer)
                
                audioEncoder?.queueInputBuffer(
                    inputBufferId, 0, readSize, 
                    System.nanoTime() / 1000, 0
                )
            }
            
            // Drain encoder output
            val outputBufferId = audioEncoder?.dequeueOutputBuffer(bufferInfo, 10_000) ?: break
            if (outputBufferId >= 0) {
                // Similar to video: write to muxer
            }
        }
    }
}
```

### 2.3 Audio Mixer (C++ via JNI)

```cpp
// android/app/src/main/cpp/audio_mixer.cpp
#include <oboe/Oboe.h>
#include <jni.h>

class AudioMixer : public oboe::AudioStreamDataCallback {
public:
    AudioMixer(int sampleRate, int channels) 
        : sampleRate_(sampleRate), channels_(channels) {
        
        // Setup Oboe output stream for mixing
        oboe::AudioStreamBuilder builder;
        builder.setDirection(oboe::Direction::Output)
               ->setPerformanceMode(oboe::PerformanceMode::LowLatency)
               ->setSharingMode(oboe::SharingMode::Exclusive)
               ->setFormat(oboe::AudioFormat::I16)
               ->setSampleRate(sampleRate)
               ->setChannelCount(channels)
               ->setDataCallback(this)
               ->openStream(mixStream_);
    }
    
    // Called by Oboe when audio buffer needs data
    oboe::DataCallbackResult onAudioReady(
        oboe::AudioStream *stream, 
        void *audioData, 
        int32_t numFrames
    ) override {
        auto* outBuffer = static_cast<int16_t*>(audioData);
        int32_t totalSamples = numFrames * channels_;
        
        // Zero buffer
        memset(outBuffer, 0, totalSamples * sizeof(int16_t));
        
        // Mix AI TTS audio (pulled from buffer)
        if (aiAudioBuffer_.available() > 0) {
            int16_t aiSamples[totalSamples];
            size_t aiRead = aiAudioBuffer_.read(aiSamples, totalSamples);
            for (int i = 0; i < aiRead; i++) {
                outBuffer[i] += aiSamples[i] / 2; // 50% volume
            }
        }
        
        // Mix User microphone audio
        if (micAudioBuffer_.available() > 0) {
            int16_t micSamples[totalSamples];
            size_t micRead = micAudioBuffer_.read(micSamples, totalSamples);
            for (int i = 0; i < micRead; i++) {
                // Apply noise suppression before mixing
                int16_t cleanSample = noiseSuppressor_.process(micSamples[i]);
                outBuffer[i] += cleanSample / 2; // 50% volume
            }
        }
        
        // Prevent clipping
        for (int i = 0; i < totalSamples; i++) {
            if (outBuffer[i] > 32767) outBuffer[i] = 32767;
            if (outBuffer[i] < -32767) outBuffer[i] = -32767;
        }
        
        // Also write to recording buffer
        recordBuffer_.write(outBuffer, totalSamples);
        
        return oboe::DataCallbackResult::Continue;
    }
    
    // Called by Java layer to read mixed audio for encoding
    int readMixedAudio(JNIEnv* env, jbyteArray buffer, jint capacity) {
        int16_t temp[capacity / 2];
        size_t read = recordBuffer_.read(temp, capacity / 2);
        if (read > 0) {
            env->SetByteArrayRegion(buffer, 0, read * 2, 
                reinterpret_cast<jbyte*>(temp));
        }
        return read * 2;
    }
    
private:
    oboe::ManagedStream mixStream_;
    int sampleRate_, channels_;
    
    // Lock-free ring buffers for thread-safe audio passing
    SPSC_RingBuffer<int16_t> aiAudioBuffer_{16384};
    SPSC_RingBuffer<int16_t> micAudioBuffer_{16384};
    SPSC_RingBuffer<int16_t> recordBuffer_{32768};
    
    NoiseSuppressor noiseSuppressor_;
};
```

---

## 3. Native iOS Layer (AVAssetWriter)

### 3.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                      iOS LAYER                           │
│                                                          │
│  ┌─────────────────────────────────────────────────┐  │
│  │              FlutterViewController                 │  │
│  │  ┌─────────────┐      ┌─────────────────────┐   │  │
│  │  │  Flutter    │◄────►│  FlutterTextureRegistry│  │  │
│  │  │  Engine     │      │  (Camera Texture)      │  │  │
│  │  │  (Metal)    │      └─────────────────────┘   │  │
│  │  └──────┬──────┘                                  │  │
│  │         │                                           │  │
│  │  ┌──────▼─────────────────────────────────────┐   │  │
│  │  │   CustomMetalLayer (CAMetalLayer)            │   │  │
│  │  │   - Offscreen texture for Flutter rendering  │   │  │
│  │  │   - Shared with AVAssetWriterInputPixelBuffer│   │  │
│  │  └──────┬─────────────────────────────────────┘   │  │
│  │         │                                           │  │
│  │  ┌──────▼─────────────────────────────────────┐   │  │
│  │  │   AVAssetWriter (H.264/AAC)                  │   │  │
│  │  │   ┌─────────────────────────────────────┐   │   │  │
│  │  │   │  AVAssetWriterInput (Video)         │   │   │  │
│  │  │   │  - PixelBufferAdaptor                 │   │   │  │
│  │  │   │  - Appends CVPixelBuffer from Metal   │   │   │  │
│  │  │   └─────────────────────────────────────┘   │   │  │
│  │  │   ┌─────────────────────────────────────┐   │   │  │
│  │  │   │  AVAssetWriterInput (Audio)         │   │   │  │
│  │  │   │  - AVAudioEngine + Manual Rendering │   │   │  │
│  │  │   │  - Mixes AI TTS + User Mic          │   │   │  │
│  │  │   └─────────────────────────────────────┘   │   │  │
│  │  │   └─────────────────────────────────────┘   │   │  │
│  │  │            ┌─────────────┐                  │   │  │
│  │  │            │  MP4 Output │                  │   │  │
│  │  │            │  (File URL) │                  │   │  │
│  │  │            └─────────────┘                  │   │  │
│  │  └─────────────────────────────────────────────┘   │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  AudioMixer (Objective-C++ / AVAudioEngine) │  │  │
│  │  │  - AVAudioSourceNode (AI TTS)                 │  │  │
│  │  │  - AVAudioSourceNode (User Mic)               │  │  │
│  │  │  - AVAudioSinkNode (Recording tap)            │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Implementation: AVAssetWriter + Metal

```swift
// ios/Runner/InterviewRecorder.swift
import Flutter
import AVFoundation
import Metal
import MetalKit

class InterviewRecorder: NSObject, FlutterTexture {
    
    // MARK: - Configuration
    struct Config {
        static let videoWidth = 1280
        static let videoHeight = 720
        static let videoBitrate = 4_000_000
        static let frameRate = 30
        static let audioSampleRate = 48000
        static let audioChannels = 2
    }
    
    // MARK: - Properties
    private var assetWriter: AVAssetWriter?
    private var videoInput: AVAssetWriterInput?
    private var audioInput: AVAssetWriterInput?
    private var pixelBufferAdaptor: AVAssetWriterInputPixelBufferAdaptor?
    
    // Metal for offscreen Flutter rendering
    private var metalDevice: MTLDevice!
    private var metalCommandQueue: MTLCommandQueue!
    private var offscreenTexture: MTLTexture!
    private var renderPipeline: MTLRenderPipelineState!
    
    // Audio
    private var audioEngine: AVAudioEngine!
    private var audioMixer: AVAudioMixerNode!
    private var ttsPlayer: AVAudioPlayerNode!
    private var micInput: AVAudioInputNode!
    
    // Flutter texture registry
    private var textureRegistry: FlutterTextureRegistry?
    private var flutterTextureId: Int64 = 0
    
    // Recording state
    private var isRecording = false
    private var videoFrameCount = 0
    private var startTime: CMTime?
    
    // MARK: - Initialization
    override init() {
        super.init()
        setupMetal()
        setupAudioEngine()
    }
    
    // MARK: - Metal Setup (Offscreen Rendering)
    private func setupMetal() {
        metalDevice = MTLCreateSystemDefaultDevice()
        metalCommandQueue = metalDevice.makeCommandQueue()
        
        // Create offscreen texture that Flutter will render into
        let textureDescriptor = MTLTextureDescriptor.texture2DDescriptor(
            pixelFormat: .bgra8Unorm,
            width: Config.videoWidth,
            height: Config.videoHeight,
            mipmapped: false
        )
        textureDescriptor.usage = [.renderTarget, .shaderRead]
        textureDescriptor.storageMode = .shared // Shared with CPU for AVAssetWriter
        
        offscreenTexture = metalDevice.makeTexture(descriptor: textureDescriptor)
    }
    
    // MARK: - AVAssetWriter Setup
    func setupWriter(outputURL: URL) throws {
        assetWriter = try AVAssetWriter(url: outputURL, fileType: .mp4)
        
        // Video Input
        let videoSettings: [String: Any] = [
            AVVideoCodecKey: AVVideoCodecType.h264,
            AVVideoWidthKey: Config.videoWidth,
            AVVideoHeightKey: Config.videoHeight,
            AVVideoCompressionPropertiesKey: [
                AVVideoAverageBitRateKey: Config.videoBitrate,
                AVVideoProfileLevelKey: AVVideoProfileLevelH264HighAutoLevel,
                AVVideoMaxKeyFrameIntervalKey: Config.frameRate
            ]
        ]
        
        videoInput = AVAssetWriterInput(mediaType: .video, outputSettings: videoSettings)
        videoInput?.expectsMediaDataInRealTime = true
        
        // Pixel buffer adaptor for efficient buffer passing
        let sourcePixelBufferAttributes: [String: Any] = [
            kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA,
            kCVPixelBufferWidthKey as String: Config.videoWidth,
            kCVPixelBufferHeightKey as String: Config.videoHeight
        ]
        
        pixelBufferAdaptor = AVAssetWriterInputPixelBufferAdaptor(
            assetWriterInput: videoInput!,
            sourcePixelBufferAttributes: sourcePixelBufferAttributes
        )
        
        // Audio Input
        let audioSettings: [String: Any] = [
            AVFormatIDKey: Int(kAudioFormatMPEG4AAC),
            AVSampleRateKey: Config.audioSampleRate,
            AVNumberOfChannelsKey: Config.audioChannels,
            AVEncoderBitRateKey: 128_000
        ]
        
        audioInput = AVAssetWriterInput(mediaType: .audio, outputSettings: audioSettings)
        audioInput?.expectsMediaDataInRealTime = true
        
        // Add inputs
        guard assetWriter?.canAdd(videoInput!) == true,
              assetWriter?.canAdd(audioInput!) == true else {
            throw RecorderError.setupFailed
        }
        
        assetWriter?.add(videoInput!)
        assetWriter?.add(audioInput!)
    }
    
    // MARK: - Audio Engine Setup (Mixing)
    private func setupAudioEngine() {
        audioEngine = AVAudioEngine()
        
        // TTS Player Node (AI Voice)
        ttsPlayer = AVAudioPlayerNode()
        audioEngine.attach(ttsPlayer)
        
        // Microphone Input
        micInput = audioEngine.inputNode
        
        // Mixer Node
        audioMixer = AVAudioMixerNode()
        audioEngine.attach(audioMixer)
        
        // Connect TTS to mixer
        let ttsFormat = AVAudioFormat(
            standardFormatWithSampleRate: Double(Config.audioSampleRate),
            channels: AVAudioChannelCount(Config.audioChannels)
        )!
        audioEngine.connect(ttsPlayer, to: audioMixer, format: ttsFormat)
        
        // Connect Mic to mixer (with noise suppression)
        let micFormat = micInput.outputFormat(forBus: 0)
        audioEngine.connect(micInput, to: audioMixer, format: micFormat)
        
        // Install tap on mixer to capture mixed audio for recording
        let recordingFormat = AVAudioFormat(
            commonFormat: .pcmFormatInt16,
            sampleRate: Double(Config.audioSampleRate),
            channels: AVAudioChannelCount(Config.audioChannels),
            interleaved: true
        )!
        
        audioMixer.installTap(onBus: 0, bufferSize: 1024, format: recordingFormat) { [weak self] buffer, time in
            self?.processAudioBuffer(buffer, presentationTime: time)
        }
        
        // Main mixer output (for playback monitoring - optional)
        audioEngine.connect(audioMixer, to: audioEngine.mainMixerNode, format: ttsFormat)
    }
    
    // MARK: - Flutter Frame Capture
    /**
     * Called by Flutter for every frame via MethodChannel
     * Instead of reading pixels back, we render Flutter's output 
     * directly to our Metal texture, then convert to CVPixelBuffer
     */
    func onFlutterFrame(textureId: Int64, transform: FlutterStandardTypedData) {
        guard isRecording, let pixelBufferAdaptor = pixelBufferAdaptor else { return }
        
        // Create command buffer
        guard let commandBuffer = metalCommandQueue.makeCommandBuffer() else { return }
        
        // Render Flutter's texture to our offscreen texture
        // (This uses a custom Metal shader that applies the transform matrix)
        renderFlutterTexture(
            commandBuffer: commandBuffer,
            sourceTextureId: textureId,
            destinationTexture: offscreenTexture,
            transform: transform
        )
        
        commandBuffer.commit()
        commandBuffer.waitUntilCompleted()
        
        // Convert Metal texture to CVPixelBuffer for AVAssetWriter
        var pixelBuffer: CVPixelBuffer?
        let pixelBufferAttributes: [String: Any] = [
            kCVPixelBufferMetalCompatibilityKey as String: true
        ]
        
        CVPixelBufferCreate(
            kCFAllocatorDefault,
            Config.videoWidth,
            Config.videoHeight,
            kCVPixelFormatType_32BGRA,
            pixelBufferAttributes as CFDictionary,
            &pixelBuffer
        )
        
        guard let pb = pixelBuffer else { return }
        
        // Copy Metal texture to CVPixelBuffer
        // (Using blit command or shared memory)
        copyMetalTextureToPixelBuffer(offscreenTexture, pixelBuffer: pb)
        
        // Calculate presentation time
        let frameTime = CMTime(
            value: CMTimeValue(videoFrameCount),
            timescale: CMTimeScale(Config.frameRate)
        )
        if startTime == nil { startTime = frameTime }
        let presentationTime = CMTimeAdd(startTime!, frameTime)
        
        // Append to video input
        if pixelBufferAdaptor.assetWriterInput.isReadyForMoreMediaData {
            pixelBufferAdaptor.append(pb, withPresentationTime: presentationTime)
        }
        
        videoFrameCount += 1
    }
    
    // MARK: - Audio Processing
    private func processAudioBuffer(_ buffer: AVAudioPCMBuffer, presentationTime: AVAudioTime) {
        guard isRecording, let audioInput = audioInput, audioInput.isReadyForMoreMediaData else { return }
        
        // Convert AVAudioPCMBuffer to CMSampleBuffer
        guard let sampleBuffer = buffer.toCMSampleBuffer(presentationTime: presentationTime) else { return }
        
        audioInput.append(sampleBuffer)
    }
    
    // MARK: - Recording Control
    func startRecording(outputURL: URL) throws {
        try setupWriter(outputURL: outputURL)
        
        guard assetWriter?.startWriting() == true else {
            throw RecorderError.startFailed
        }
        
        assetWriter?.startSession(atSourceTime: .zero)
        isRecording = true
        
        // Start audio engine
        try audioEngine.start()
    }
    
    func stopRecording(completion: @escaping (URL?) -> Void) {
        isRecording = false
        audioEngine.stop()
        
        videoInput?.markAsFinished()
        audioInput?.markAsFinished()
        
        assetWriter?.finishWriting { [weak self] in
            completion(self?.assetWriter?.outputURL)
        }
    }
    
    // MARK: - FlutterTexture Protocol
    func copyPixelBuffer() -> Unmanaged<CVPixelBuffer>? {
        // Return our offscreen texture as a CVPixelBuffer for Flutter
        // This allows Flutter to render into our texture
        return nil // Implementation depends on Metal-CVPixelBuffer bridging
    }
}

// MARK: - Audio Buffer Extension
extension AVAudioPCMBuffer {
    func toCMSampleBuffer(presentationTime: AVAudioTime) -> CMSampleBuffer? {
        var sampleBuffer: CMSampleBuffer?
        
        var formatDescription: CMAudioFormatDescription?
        CMAudioFormatDescriptionCreate(
            allocator: kCFAllocatorDefault,
            asbd: format.streamDescription,
            layoutSize: 0,
            layout: nil,
            magicCookieSize: 0,
            magicCookie: nil,
            formatDescriptionOut: &formatDescription
        )
        
        var timing = CMSampleTimingInfo(
            duration: CMTime(value: 1, timescale: CMTimeScale(format.sampleRate)),
            presentationTimeStamp: presentationTime.audioTime,
            decodeTimeStamp: .invalid
        )
        
        CMSampleBufferCreate(
            allocator: kCFAllocatorDefault,
            dataBuffer: nil,
            dataReady: false,
            makeDataReadyCallback: nil,
            refcon: nil,
            formatDescription: formatDescription,
            sampleCount: CMItemCount(frameLength),
            sampleTimingEntryCount: 1,
            sampleTimingArray: &timing,
            sampleSizeEntryCount: 0,
            sampleSizeArray: nil,
            sampleBufferOut: &sampleBuffer
        )
        
        // Copy audio data into sample buffer
        if let sb = sampleBuffer {
            CMSampleBufferSetDataBufferFromAudioBufferList(
                sb,
                blockBufferAllocator: kCFAllocatorDefault,
                blockBufferMemoryAllocator: kCFAllocatorDefault,
                flags: 0,
                bufferList: mutableAudioBufferList
            )
        }
        
        return sampleBuffer
    }
}
```

---

## 4. Audio Mixing Architecture (Cross-Platform)

```
┌────────────────────────────────────────────────────────────┐
│                     AUDIO PIPELINE                          │
│                                                             │
│   AI TTS Output          User Microphone                     │
│        │                       │                             │
│        ▼                       ▼                             │
│   ┌─────────┐            ┌─────────┐                      │
│   │ Text-to │            │ Audio   │                      │
│   │ Speech  │            │ Record  │                      │
│   │ (AVSpeech│           │ (AudioRecord/│                  │
│   │ Synthesizer│         │  AVAudioEngine)│                │
│   │ / ExoPlayer)│         │               │                  │
│   └────┬────┘            └────┬────┘                      │
│        │                       │                             │
│        │    ┌─────────────┐   │                             │
│        └───►│  Audio      │◄──┘                             │
│             │  Mixer      │                                 │
│             │  (C++ /     │                                 │
│             │   Oboe /     │                                 │
│             │   AVAudioEngine)│                             │
│             └──────┬──────┘                                 │
│                    │                                         │
│         ┌────────┴────────┐                                │
│         │                 │                                │
│         ▼                 ▼                                │
│    ┌─────────┐      ┌─────────┐                           │
│    │ Playback│      │ Recording│                           │
│    │ (Speaker)│      │ Encoder │                           │
│    │         │      │ (AAC)   │                           │
│    └─────────┘      └────┬────┘                           │
│                          │                                  │
│                          ▼                                  │
│                    ┌─────────────┐                          │
│                    │  MediaMuxer │                          │
│                    │  / AVAssetWriter│                       │
│                    │  (MP4)       │                          │
│                    └─────────────┘                          │
└────────────────────────────────────────────────────────────┘
```

### 4.1 Shared C++ Audio Mixer (Used by both Android & iOS via JNI/FFI)

```cpp
// common/cpp/audio_mixer.h
#pragma once
#include <vector>
#include <cstdint>
#include <atomic>
#include <mutex>

// Lock-free SPSC ring buffer for real-time audio
template<typename T, size_t Size>
class LockFreeRingBuffer {
    std::atomic<size_t> writeIndex{0};
    std::atomic<size_t> readIndex{0};
    T buffer[Size];
    
public:
    bool write(const T* data, size_t count) {
        size_t currentWrite = writeIndex.load(std::memory_order_relaxed);
        size_t currentRead = readIndex.load(std::memory_order_acquire);
        
        size_t available = Size - (currentWrite - currentRead);
        if (count > available) return false;
        
        for (size_t i = 0; i < count; i++) {
            buffer[(currentWrite + i) % Size] = data[i];
        }
        
        writeIndex.store(currentWrite + count, std::memory_order_release);
        return true;
    }
    
    size_t read(T* data, size_t maxCount) {
        size_t currentRead = readIndex.load(std::memory_order_relaxed);
        size_t currentWrite = writeIndex.load(std::memory_order_acquire);
        
        size_t available = currentWrite - currentRead;
        size_t toRead = std::min(available, maxCount);
        
        for (size_t i = 0; i < toRead; i++) {
            data[i] = buffer[(currentRead + i) % Size];
        }
        
        readIndex.store(currentRead + toRead, std::memory_order_release);
        return toRead;
    }
    
    size_t available() const {
        return writeIndex.load(std::memory_order_relaxed) - 
               readIndex.load(std::memory_order_relaxed);
    }
};

class AudioMixer {
public:
    static constexpr int SAMPLE_RATE = 48000;
    static constexpr int CHANNELS = 2;
    static constexpr int BUFFER_SIZE = 4096;
    
    AudioMixer();
    
    // Input streams
    void pushAIAudio(const int16_t* data, size_t samples);
    void pushMicrophoneAudio(const int16_t* data, size_t samples);
    
    // Output for recording
    size_t readMixedAudio(int16_t* output, size_t maxSamples);
    
    // Apply noise suppression to microphone
    void setNoiseSuppression(bool enabled);
    
    // Volume controls
    void setAIVolume(float volume); // 0.0 - 1.0
    void setMicVolume(float volume); // 0.0 - 1.0
    
private:
    LockFreeRingBuffer<int16_t, 16384> aiBuffer_;
    LockFreeRingBuffer<int16_t, 16384> micBuffer_;
    LockFreeRingBuffer<int16_t, 32768> outputBuffer_;
    
    float aiVolume_ = 0.7f;
    float micVolume_ = 0.7f;
    bool noiseSuppression_ = true;
    
    // WebRTC noise suppression
    std::unique_ptr<NsHandle> noiseSuppressor_;
    
    void mixAndProcess();
    int16_t applyNoiseSuppression(int16_t sample);
    void preventClipping(int16_t* buffer, size_t samples);
};
```

---

## 5. FFmpeg Integration (Post-Processing & Fallback)

### 5.1 When to Use FFmpeg

| Scenario | Solution |
|----------|----------|
| **Real-time encoding** | Native MediaCodec/AVAssetWriter (preferred) |
| **Post-processing** | FFmpeg (merge, transcode, add metadata) |
| **Cross-platform consistency** | FFmpeg for final muxing |
| **Complex filters** | FFmpeg audio normalization, video stabilization |

### 5.2 FFmpeg Integration (Mobile)

```dart
// lib/core/ffmpeg_processor.dart
import 'package:ffmpeg_kit_flutter/ffmpeg_kit.dart';
import 'package:ffmpeg_kit_flutter/return_code.dart';

class FFmpegProcessor {
  /// Post-process recorded MP4 for optimal upload
  static Future<String> postProcess(String inputPath) async {
    final outputPath = inputPath.replaceAll('.mp4', '_processed.mp4');
    
    // Command: Re-encode with optimal settings for interview content
    // - H.264 baseline for compatibility
    // - AAC audio with normalization
    // - Faststart for streaming playback
    final command = '''
      -i $inputPath
      -c:v libx264
      -preset fast
      -crf 23
      -profile:v baseline
      -level 3.0
      -movflags +faststart
      -c:a aac
      -b:a 128k
      -af "loudnorm=I=-16:TP=-1.5:LRA=11"
      -pix_fmt yuv420p
      -r 30
      -g 30
      $outputPath
    ''';
    
    final session = await FFmpegKit.execute(command);
    final returnCode = await session.getReturnCode();
    
    if (ReturnCode.isSuccess(returnCode)) {
      return outputPath;
    } else {
      final logs = await session.getLogs();
      throw Exception('FFmpeg failed: ${logs.last.getMessage()}');
    }
  }
  
  /// Extract audio for speech-to-text analysis
  static Future<String> extractAudio(String videoPath) async {
    final outputPath = videoPath.replaceAll('.mp4', '_audio.wav');
    
    final command = '''
      -i $videoPath
      -vn
      -acodec pcm_s16le
      -ar 16000
      -ac 1
      $outputPath
    ''';
    
    await FFmpegKit.execute(command);
    return outputPath;
  }
  
  /// Generate thumbnail for upload preview
  static Future<String> generateThumbnail(String videoPath) async {
    final outputPath = videoPath.replaceAll('.mp4', '_thumb.jpg');
    
    final command = '''
      -i $videoPath
      -ss 00:00:01
      -vframes 1
      -q:v 2
      -vf "scale=480:-1"
      $outputPath
    ''';
    
    await FFmpegKit.execute(command);
    return outputPath;
  }
}
```

---

## 6. Platform Channel Communication

```dart
// lib/core/platform_recorder.dart
class PlatformRecorder {
  static const MethodChannel _channel = 
      MethodChannel('com.interview.recorder');
  static const EventChannel _eventChannel = 
      EventChannel('com.interview.recorder.events');
  
  // Stream of recording progress
  Stream<RecordingProgress>? _progressStream;
  
  /// Initialize native recorders
  Future<void> initialize() async {
    await _channel.invokeMethod('initialize', {
      'videoWidth': 1280,
      'videoHeight': 720,
      'frameRate': 30,
      'audioSampleRate': 48000,
      'audioChannels': 2,
    });
  }
  
  /// Start recording interview
  Future<void> startRecording({
    required String outputPath,
    required int flutterTextureId,
  }) async {
    await _channel.invokeMethod('startRecording', {
      'outputPath': outputPath,
      'flutterTextureId': flutterTextureId,
    });
  }
  
  /// Notify native side of new Flutter frame
  Future<void> onFrameRendered(FrameData frame) async {
    await _channel.invokeMethod('onFrame', {
      'textureId': frame.textureId,
      'transform': frame.transformMatrix,
      'timestamp': frame.timestamp.microsecondsSinceEpoch,
    });
  }
  
  /// Push AI TTS audio to native mixer
  Future<void> pushAIAudio(Uint8List pcmData) async {
    await _channel.invokeMethod('pushAIAudio', {
      'pcmData': pcmData,
      'sampleRate': 48000,
      'channels': 2,
    });
  }
  
  /// Stop recording and get final MP4 path
  Future<RecordingResult> stopRecording() async {
    final result = await _channel.invokeMethod('stopRecording');
    return RecordingResult.fromMap(result);
  }
  
  Stream<RecordingProgress> get progressStream {
    _progressStream ??= _eventChannel
        .receiveBroadcastStream()
        .map((data) => RecordingProgress.fromMap(data));
    return _progressStream!;
  }
}
```

---

## 7. Performance Optimization Strategies

### 7.1 Video Pipeline

| Optimization | Implementation | Impact |
|------------|---------------|--------|
| **Zero-copy texture sharing** | Metal/Vulkan shared textures between Flutter and encoder | Eliminates GPU→CPU→GPU roundtrip |
| **Hardware encoding** | MediaCodec/VideoToolbox | 10x faster than software |
| **Variable frame rate** | Drop frames during static UI | 30% bitrate reduction |
| **Resolution scaling** | Record at 720p, display at 1080p | 50% less encoding load |
| **OpenGL FBO reuse** | Single FBO for entire session | Reduces memory allocation |

### 7.2 Audio Pipeline

| Optimization | Implementation | Impact |
|-------------|---------------|--------|
| **Lock-free ring buffers** | SPSC queues between threads | No mutex contention |
| **Oboe/AAudio** | Low-latency audio path | <10ms latency |
| **Hardware noise suppression** | Qualcomm AANC / Apple Voice Processing | 0% CPU overhead |
| **Float32 internal mixing** | Prevent quantization distortion | Better audio quality |
| **Resampling once** | 48kHz throughout pipeline | Avoid multiple SRC |

### 7.3 Memory Management

```dart
// lib/core/memory_pool.dart
/// Object pool for frame buffers to reduce GC pressure
class FrameBufferPool {
  static const int POOL_SIZE = 3;
  final List<Uint8List> _available = [];
  final Set<Uint8List> _inUse = {};
  
  Uint8List acquire(int size) {
    // Find buffer of sufficient size
    final buffer = _available.firstWhere(
      (b) => b.length >= size,
      orElse: () => Uint8List(size),
    );
    _available.remove(buffer);
    _inUse.add(buffer);
    return buffer;
  }
  
  void release(Uint8List buffer) {
    if (_inUse.remove(buffer)) {
      _available.add(buffer);
      // Trim pool if too large
      if (_available.length > POOL_SIZE) {
        _available.removeAt(0);
      }
    }
  }
}
```

---

## 8. Upload Flow Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      UPLOAD PIPELINE                         │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Recording  │───►│  Post-      │───►│  Upload     │     │
│  │  Complete   │    │  Processing │    │  Manager    │     │
│  └─────────────┘    └──────┬──────┘    └──────┬──────┘     │
│                            │                    │            │
│                     ┌──────┴──────┐      ┌──────▼──────┐    │
│                     │  FFmpeg     │      │  Chunked    │    │
│                     │  - Normalize│      │  Upload     │    │
│                     │  - Thumbnail│      │  - Resume   │    │
│                     │  - Metadata │      │  - Retry    │    │
│                     └─────────────┘      │  - Progress │    │
│                                          └──────┬──────┘    │
│                                                 │            │
│                                          ┌──────▼──────┐    │
│                                          │   Cloud     │    │
│                                          │  Storage    │    │
│                                          │ (S3/GCS)    │    │
│                                          └─────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Background Upload (WorkManager / BGTaskScheduler)  │    │
│  │  - Continue upload when app backgrounded            │    │
│  │  - Retry on network failure                         │    │
│  │  - Battery-aware (defer on low battery)             │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

```dart
// lib/core/upload_manager.dart
class UploadManager {
  final Dio _dio;
  final BackgroundUpload _backgroundUpload;
  
  /// Upload with resumable chunks
  Future<void> uploadRecording(Recording recording) async {
    final file = File(recording.processedPath);
    final fileSize = await file.length();
    const chunkSize = 5 * 1024 * 1024; // 5MB chunks
    
    // Get presigned URL for multipart upload
    final uploadSession = await _createUploadSession(
      fileName: recording.fileName,
      fileSize: fileSize,
      metadata: {
        'interviewId': recording.interviewId,
        'duration': recording.duration.inSeconds,
        'thumbnail': recording.thumbnailPath,
      },
    );
    
    // Upload chunks in parallel
    final chunkFutures = <Future<void>>[];
    for (var start = 0; start < fileSize; start += chunkSize) {
      final end = (start + chunkSize < fileSize) ? start + chunkSize : fileSize;
      chunkFutures.add(_uploadChunk(
        session: uploadSession,
        chunkIndex: start ~/ chunkSize,
        start: start,
        end: end,
        file: file,
      ));
    }
    
    await Future.wait(chunkFutures);
    
    // Complete multipart upload
    await _completeUpload(uploadSession);
  }
  
  /// Background upload using WorkManager
  Future<void> scheduleBackgroundUpload(Recording recording) async {
    await Workmanager().registerOneOffTask(
      'upload-${recording.id}',
      'uploadRecording',
      inputData: {
        'filePath': recording.processedPath,
        'interviewId': recording.interviewId,
      },
      constraints: Constraints(
        networkType: NetworkType.connected,
        requiresBatteryNotLow: true,
      ),
      backoffPolicy: BackoffPolicy.exponential,
      existingWorkPolicy: ExistingWorkPolicy.replace,
    );
  }
}
```

---

## 9. Complete Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           INTERVIEW SESSION                              │
│                                                                          │
│  1. START                                                                │
│     │                                                                    │
│     ▼                                                                    │
│  ┌─────────────────┐                                                      │
│  │ Initialize      │  • Setup OpenGL/Metal context                       │
│  │ Native Recorder │  • Create shared surface (Flutter ↔ Encoder)        │
│  │                 │  • Initialize audio mixer                            │
│  │                 │  • Configure MediaCodec / AVAssetWriter             │
│  └────────┬────────┘                                                      │
│           │                                                              │
│           ▼                                                              │
│  ┌─────────────────┐                                                      │
│  │ Start Recording │  • Begin audio capture (mic + TTS)                 │
│  │                 │  • Start encoding threads                             │
│  │                 │  • Start flutter frame capture timer (30fps)        │
│  └────────┬────────┘                                                      │
│           │                                                              │
│           ▼                                                              │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐    │
│  │ AI Speaks (TTS) │────►│ Audio Mixer     │────►│ AAC Encoder     │    │
│  │                 │     │ (AI + Mic mix)  │     │ (Real-time)     │    │
│  └─────────────────┘     └─────────────────┘     └────────┬────────┘    │
│                                                             │            │
│  ┌─────────────────┐     ┌─────────────────┐                │            │
│  │ User Speaks     │────►│ Audio Mixer     │────────────────┘            │
│  │ (Microphone)    │     │                 │                              │
│  └─────────────────┘     └─────────────────┘                              │
│                                                                             │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐       │
│  │ Flutter Renders │────►│ Shared Surface  │────►│ H.264 Encoder   │    │
│  │ UI (60fps)      │     │ (OpenGL/Metal)  │     │ (Hardware)      │    │
│  │                 │     │ Every 2nd frame │     │ (30fps output)  │    │
│  └─────────────────┘     └─────────────────┘     └────────┬────────┘    │
│                                                            │             │
│                                                            ▼             │
│                                                     ┌─────────────────┐   │
│                                                     │ MediaMuxer      │   │
│                                                     │ / AVAssetWriter │   │
│                                                     │ (MP4 muxing)    │   │
│                                                     └────────┬────────┘   │
│                                                              │            │
│                                                              ▼            │
│  2. STOP                                                     │            │
│     │                                                        │            │
│     ▼                                                        ▼            │
│  ┌─────────────────┐                                  ┌─────────────────┐   │
│  │ Finalize        │  • Drain encoder buffers         │ Final MP4 File  │   │
│  │ Recording       │  • Write trailer                │ (Local Cache)   │   │
│  │                 │  • Close file                     │                 │   │
│  └────────┬────────┘                                  └────────┬────────┘   │
│           │                                                    │            │
│           ▼                                                    ▼            │
│  ┌─────────────────┐                                  ┌─────────────────┐   │
│  │ FFmpeg Post-    │  • Normalize audio               │ Upload to Cloud │   │
│  │ Processing      │  • Generate thumbnail            │ (Chunked + BG)  │   │
│  │                 │  • Add metadata                    │                 │   │
│  └─────────────────┘                                  └─────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Why This Avoids MediaProjection

| MediaProjection Approach | Our Internal Recording Approach |
|-------------------------|--------------------------------|
| Captures **entire screen** | Captures **only Flutter UI** |
| Includes status bar, notifications | Excludes system UI |
| Requires user permission dialog | No permission needed (internal) |
| Can capture other apps | Only records our app content |
| Hardware compositor capture | GPU texture-to-encoder pipeline |
| Higher latency (2-3 frames) | Zero-copy, sub-frame latency |
| Android only | Cross-platform (Android + iOS) |

---

## 11. Key Files Structure

```
lib/
├── screens/
│   └── interview_screen.dart          # Main UI
├── widgets/
│   ├── ai_avatar.dart                 # AI video/image
│   ├── camera_preview.dart            # User camera
│   ├── transcript.dart                # Live transcript
│   └── interview_controls.dart      # Start/stop/pause
├── core/
│   ├── platform_recorder.dart         # MethodChannel API
│   ├── render_capture.dart            # Flutter frame capture
│   ├── audio_manager.dart             # TTS + Mic coordination
│   ├── ffmpeg_processor.dart          # Post-processing
│   └── upload_manager.dart            # Cloud upload
└── models/
    └── recording.dart                 # Data models

android/app/src/main/
├── kotlin/com/interview/
│   ├── recorder/
│   │   ├── FlutterRecorder.kt         # Main recorder
│   │   ├── AudioMixer.kt              # Kotlin audio wrapper
│   │   └── TextureRenderer.kt         # OpenGL rendering
│   └── MainActivity.kt
└── cpp/
    ├── audio_mixer.cpp                # Shared C++ mixer
    ├── noise_suppressor.cpp           # WebRTC NS
    └── CMakeLists.txt

ios/Runner/
├── InterviewRecorder.swift            # AVAssetWriter setup
├── AudioMixer.swift                   # AVAudioEngine mixer
├── MetalRenderer.swift                # Metal texture bridge
└── AppDelegate.swift
```

---

## 12. Critical Implementation Notes

1. **Flutter Texture Registry**: Register your camera preview as a Flutter texture so it renders within the Flutter tree, not as a platform view overlay.

2. **OpenGL Context Sharing**: On Android, the EGL context used by Flutter must be shared with your recorder's context for texture sharing to work.

3. **Metal Shared Textures**: On iOS, use `MTLStorageMode.shared` for textures that need CPU access (AVAssetWriter pixel buffers).

4. **Audio Clock Synchronization**: Use `AudioTimestamp` on Android and `CMAudioClock` on iOS to ensure audio/video sync within ±40ms.

5. **Memory Pressure Handling**: Monitor `onTrimMemory` (Android) and `didReceiveMemoryWarning` (iOS) to pause recording and flush buffers.

6. **Background Recording**: Use `foregroundService` (Android) and `beginBackgroundTask` (iOS) to keep recording alive when user switches apps.

This architecture gives you **complete control** over what gets recorded, **excludes all system UI**, and achieves **broadcast-quality encoding** using hardware-accelerated pipelines on both platforms.
