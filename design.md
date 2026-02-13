# Design Document: Virtual Bookset VR Reading Experience

## Overview

The Virtual Bookset VR Reading Experience is a distributed system that combines client-side VR rendering with cloud-based AI image generation to create an immersive reading experience. The architecture prioritizes low-latency synchronization between audio narration and visual content while maintaining narrative consistency across the entire book.

### Key Design Principles

1. **Latency Minimization**: Aggressive pre-generation and caching strategies to achieve <100ms audio-visual sync
2. **Narrative Coherence**: Persistent tracking of visual attributes for characters, objects, and settings
3. **Graceful Degradation**: System adapts to network and hardware constraints without breaking the experience
4. **Resource Efficiency**: Optimized for mobile VR hardware (Meta Quest) with limited GPU/memory
5. **Scalability**: Cloud-based generation allows high-quality visuals without client-side computational burden

## Architecture

The system follows a client-server architecture with three primary layers:

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        VR Client                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ TTS Engine   │  │ VR Renderer  │  │ Session      │      │
│  │              │  │              │  │ Monitor      │      │
│  └──────┬───────┘  └──────▲───────┘  └──────────────┘      │
│         │                 │                                  │
│         │                 │                                  │
│  ┌──────▼─────────────────┴───────────────────────────┐    │
│  │         Sync Controller & Cache Manager            │    │
│  └──────────────────────┬─────────────────────────────┘    │
└─────────────────────────┼───────────────────────────────────┘
                          │ HTTPS/WebSocket
┌─────────────────────────▼───────────────────────────────────┐
│                     Cloud Services                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Visual       │  │ Narrative    │  │ Content      │      │
│  │ Generator    │  │ Tracker      │  │ Parser       │      │
│  │ (AI Service) │  │ (Database)   │  │ Service      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**Client-Side (VR Headset):**
- TTS Engine: Audio narration generation and playback
- VR Renderer: Display of 3D scenes and UI elements
- Sync Controller: Coordination of audio-visual timing
- Cache Manager: Local storage of generated scenes
- Session Monitor: User comfort and engagement tracking

**Server-Side (Cloud):**
- Visual Generator: AI-powered image/scene generation
- Narrative Tracker: Persistent storage of narrative element attributes
- Content Parser: Book text analysis and structure extraction
- API Gateway: Request routing and rate limiting

## Components and Interfaces

### 1. Content Parser Service

**Purpose:** Analyzes book text to extract structure, narrative elements, and visual descriptions.

**Interface:**
```typescript
interface ContentParser {
  // Parse entire book and extract structure
  parseBook(bookContent: string): ParsedBook
  
  // Extract visual descriptions from a sentence
  extractVisualDescriptions(sentence: string): VisualDescription[]
  
  // Identify narrative elements (characters, objects, settings)
  identifyNarrativeElements(sentence: string): NarrativeElement[]
}

interface ParsedBook {
  bookId: string
  chapters: Chapter[]
  metadata: BookMetadata
}

interface Chapter {
  chapterId: string
  title: string
  sentences: Sentence[]
}

interface Sentence {
  sentenceId: string
  text: string
  type: 'dialogue' | 'description' | 'action'
  narrativeElements: NarrativeElement[]
  visualDescriptions: VisualDescription[]
}

interface VisualDescription {
  subject: string
  action?: string
  setting?: string
  attributes: Map<string, string>
}

interface NarrativeElement {
  elementId: string
  type: 'character' | 'object' | 'setting'
  name: string
  attributes: Map<string, string>
}
```

**Implementation Notes:**
- Uses NLP techniques (named entity recognition, dependency parsing) to extract elements
- Maintains sentence-level granularity for precise synchronization
- Categorizes sentences to prioritize visual generation (descriptions > actions > dialogue)

### 2. TTS Engine

**Purpose:** Converts text to natural-sounding speech with appropriate pacing and intonation.

**Interface:**
```typescript
interface TTSEngine {
  // Generate audio for a sentence
  generateAudio(sentence: Sentence, voice: VoiceConfig): AudioBuffer
  
  // Control playback
  play(): void
  pause(): void
  resume(): void
  skipTo(sentenceId: string): void
  
  // Get current playback state
  getCurrentPosition(): PlaybackPosition
  
  // Event notifications
  onSentenceStart(callback: (sentenceId: string) => void): void
  onSentenceEnd(callback: (sentenceId: string) => void): void
}

interface VoiceConfig {
  voiceId: string
  speed: number // 0.5 to 2.0
  pitch: number
  volume: number
}

interface PlaybackPosition {
  sentenceId: string
  timeOffset: number // milliseconds into current sentence
}
```

**Implementation Notes:**
- Uses Web Speech API or cloud TTS services (Google Cloud TTS, Amazon Polly)
- Pre-generates audio for upcoming sentences during playback
- Maintains audio buffer of 5-10 sentences ahead
- Supports multiple voice profiles for different age groups

### 3. Visual Generator Service

**Purpose:** Generates visual scenes from text descriptions using AI image generation.

**Interface:**
```typescript
interface VisualGenerator {
  // Generate scene for a sentence
  generateScene(request: SceneRequest): Promise<GeneratedScene>
  
  // Batch generation for pre-loading
  generateSceneBatch(requests: SceneRequest[]): Promise<GeneratedScene[]>
  
  // Get generation status
  getGenerationStatus(requestId: string): GenerationStatus
}

interface SceneRequest {
  requestId: string
  sentenceId: string
  visualDescriptions: VisualDescription[]
  narrativeContext: NarrativeContext
  qualityLevel: 'high' | 'medium' | 'low'
}

interface NarrativeContext {
  activeCharacters: NarrativeElement[]
  currentSetting: NarrativeElement
  recentObjects: NarrativeElement[]
}

interface GeneratedScene {
  sceneId: string
  sentenceId: string
  imageData: Blob // Compressed image
  metadata: SceneMetadata
  generationTime: number // milliseconds
}

interface SceneMetadata {
  resolution: { width: number, height: number }
  format: string
  compressionRatio: number
}

interface GenerationStatus {
  requestId: string
  status: 'queued' | 'generating' | 'completed' | 'failed'
  progress: number // 0-100
  estimatedCompletion: number // milliseconds
}
```

**Implementation Notes:**
- Uses Stable Diffusion or DALL-E API for image generation
- Implements prompt engineering to incorporate narrative consistency
- Generates at multiple quality levels based on network/latency conditions
- Compresses images using WebP or JPEG-XL for fast transmission
- Maintains generation queue with priority for imminent sentences

### 4. Narrative Tracker Service

**Purpose:** Maintains consistency of visual attributes for narrative elements across the book.

**Interface:**
```typescript
interface NarrativeTracker {
  // Register a new narrative element
  registerElement(bookId: string, element: NarrativeElement): void
  
  // Update element attributes
  updateElement(bookId: string, elementId: string, attributes: Map<string, string>): void
  
  // Retrieve element for consistency
  getElement(bookId: string, elementId: string): NarrativeElement | null
  
  // Get all active elements for current context
  getActiveElements(bookId: string, sentenceId: string): NarrativeElement[]
  
  // Clear book data
  clearBook(bookId: string): void
}
```

**Implementation Notes:**
- Uses persistent database (PostgreSQL or MongoDB) for storage
- Implements fuzzy matching for element identification (e.g., "the dog" = "blue dog")
- Tracks element lifecycle (introduction, modifications, last appearance)
- Generates visual embeddings for elements to ensure AI consistency
- Supports element relationships (e.g., "John's hat" links character to object)

### 5. Sync Controller

**Purpose:** Coordinates timing between audio narration and visual scene display.

**Interface:**
```typescript
interface SyncController {
  // Initialize reading session
  startSession(bookId: string, startPosition: string): void
  
  // Coordinate playback
  synchronize(): void
  
  // Handle user controls
  handlePause(): void
  handleResume(): void
  handleSkip(targetSentenceId: string): void
  
  // Monitor synchronization quality
  getSyncMetrics(): SyncMetrics
  
  // Event notifications
  onSyncDrift(callback: (drift: number) => void): void
}

interface SyncMetrics {
  averageLatency: number // milliseconds
  maxLatency: number
  syncDriftEvents: number
  cacheHitRate: number // percentage
}
```

**Implementation Notes:**
- Implements predictive pre-loading based on reading speed
- Maintains strict timing using high-resolution timers
- Falls back to transition effects when latency exceeds threshold
- Coordinates with Cache Manager for scene retrieval
- Logs detailed timing metrics for performance analysis

### 6. Cache Manager

**Purpose:** Manages local storage of generated scenes for quick retrieval.

**Interface:**
```typescript
interface CacheManager {
  // Store generated scene
  cacheScene(scene: GeneratedScene): void
  
  // Retrieve cached scene
  getScene(sentenceId: string): GeneratedScene | null
  
  // Pre-load upcoming scenes
  preloadScenes(sentenceIds: string[]): void
  
  // Manage cache size
  evictOldScenes(): void
  
  // Get cache statistics
  getCacheStats(): CacheStats
}

interface CacheStats {
  totalScenes: number
  cacheSize: number // bytes
  hitRate: number // percentage
  evictionCount: number
}
```

**Implementation Notes:**
- Uses IndexedDB for client-side storage (up to 1GB)
- Implements LRU eviction policy when cache is full
- Prioritizes caching of scenes with complex narrative elements
- Pre-loads 10-20 scenes ahead based on available bandwidth
- Compresses cached data to maximize storage

### 7. VR Renderer

**Purpose:** Displays generated scenes in the VR environment with smooth transitions.

**Interface:**
```typescript
interface VRRenderer {
  // Display a scene
  renderScene(scene: GeneratedScene, transition: TransitionEffect): void
  
  // Update display settings
  updateSettings(settings: DisplaySettings): void
  
  // Get rendering performance
  getPerformanceMetrics(): RenderMetrics
  
  // Handle VR-specific features
  enableSpatialAudio(enabled: boolean): void
  adjustIPD(ipd: number): void
}

interface TransitionEffect {
  type: 'fade' | 'dissolve' | 'none'
  duration: number // milliseconds
}

interface DisplaySettings {
  brightness: number
  contrast: number
  fieldOfView: number
  stereoscopicDepth: number
}

interface RenderMetrics {
  currentFPS: number
  averageFPS: number
  droppedFrames: number
  gpuUtilization: number // percentage
}
```

**Implementation Notes:**
- Uses WebXR API for VR rendering
- Implements 360-degree panoramic scene display
- Maintains 72+ FPS for comfort (90 FPS target for Quest 2/3)
- Uses foveated rendering to reduce GPU load
- Supports stereoscopic 3D for depth perception

### 8. Session Monitor

**Purpose:** Tracks user comfort, engagement, and comprehension during reading sessions.

**Interface:**
```typescript
interface SessionMonitor {
  // Start monitoring
  startMonitoring(userId: string, bookId: string): void
  
  // Record user comfort feedback
  recordComfortRating(rating: number): void
  
  // Track engagement metrics
  trackEngagement(event: EngagementEvent): void
  
  // Present comprehension quiz
  presentQuiz(questions: QuizQuestion[]): void
  
  // Get session summary
  getSessionSummary(): SessionSummary
}

interface EngagementEvent {
  type: 'pause' | 'resume' | 'skip' | 'reread' | 'bookmark'
  timestamp: number
  sentenceId: string
}

interface QuizQuestion {
  questionId: string
  question: string
  options: string[]
  correctAnswer: number
}

interface SessionSummary {
  sessionId: string
  duration: number // milliseconds
  sentencesRead: number
  averageComfortRating: number
  engagementEvents: EngagementEvent[]
  comprehensionScore: number // percentage
  motionSicknessIndicators: number
}
```

**Implementation Notes:**
- Tracks head movement patterns using VR sensors
- Prompts for comfort ratings every 10 minutes
- Detects rapid head movements indicating discomfort
- Records all user interactions for engagement analysis
- Generates session reports for effectiveness evaluation

## Data Models

### Book Data Model

```typescript
interface Book {
  bookId: string
  title: string
  author: string
  isbn?: string
  language: string
  ageRating: 'children' | 'teen' | 'adult' | 'all'
  totalSentences: number
  averageReadingTime: number // minutes
  coverImage: string
  parsedContent: ParsedBook
  createdAt: Date
  updatedAt: Date
}
```

### User Session Data Model

```typescript
interface UserSession {
  sessionId: string
  userId: string
  bookId: string
  startTime: Date
  endTime?: Date
  currentPosition: string // sentenceId
  bookmarks: Bookmark[]
  settings: UserSettings
  metrics: SessionMetrics
}

interface Bookmark {
  bookmarkId: string
  sentenceId: string
  note?: string
  createdAt: Date
}

interface UserSettings {
  voiceConfig: VoiceConfig
  displaySettings: DisplaySettings
  comprehensionQuizEnabled: boolean
  comfortRemindersEnabled: boolean
}

interface SessionMetrics {
  totalReadingTime: number // milliseconds
  sentencesCompleted: number
  pauseCount: number
  skipCount: number
  rereadCount: number
  averageLatency: number
  cacheHitRate: number
  comfortRatings: number[]
  comprehensionScore?: number
}
```

### Narrative Element Data Model

```typescript
interface StoredNarrativeElement {
  elementId: string
  bookId: string
  type: 'character' | 'object' | 'setting'
  name: string
  attributes: Map<string, string>
  visualEmbedding: number[] // Vector for AI consistency
  firstAppearance: string // sentenceId
  lastAppearance: string // sentenceId
  appearanceCount: number
  relatedElements: string[] // elementIds
  createdAt: Date
  updatedAt: Date
}
```

### Scene Cache Data Model

```typescript
interface CachedScene {
  sceneId: string
  sentenceId: string
  bookId: string
  imageData: Blob
  metadata: SceneMetadata
  narrativeElementIds: string[]
  generationTime: number
  cacheTime: Date
  lastAccessed: Date
  accessCount: number
  priority: number // For eviction policy
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property Reflection

After analyzing all acceptance criteria, I identified several areas where properties can be consolidated:

**Consolidations:**
- Properties 1.1 and 5.1 both test parsing behavior - can be combined into comprehensive parsing property
- Properties 4.2 and 9.2 both test pause coordination - redundant, keep one comprehensive version
- Properties 7.1 and 10.1 both test session time tracking - combine into single time tracking property
- Properties 6.5 and 6.2 (caching and compression) can be tested together as part of scene generation pipeline
- Properties related to pre-generation (6.4, 8.5, 11.3) can be combined into opportunistic pre-loading property

**Unique Value Properties:**
Each remaining property provides distinct validation:
- Round-trip properties (3.2, 9.4) validate data persistence
- Threshold properties (4.1, 7.2) validate performance and timing requirements
- Fallback properties (2.5, 4.4, 6.3) validate graceful degradation
- Synchronization properties (4.3, 4.5) validate coordination between components

### Correctness Properties

Property 1: Content parsing completeness
*For any* book content with chapters and sentences, parsing should extract all structural elements (chapters, sentences, narrative elements) and each element should be retrievable from the parsed result.
**Validates: Requirements 1.1, 5.1, 5.3, 5.4**

Property 2: TTS audio generation
*For any* valid sentence, the TTS engine should generate an audio buffer with non-zero duration.
**Validates: Requirements 1.2**

Property 3: Sequential playback progression
*For any* sequence of sentences, when playback completes a sentence without user intervention, the next sentence should automatically begin playing.
**Validates: Requirements 1.4**

Property 4: Playback control operations
*For any* playback state and any control operation (pause, resume, skip), applying the control should result in the expected state transition without errors.
**Validates: Requirements 1.5**

Property 5: Visual description extraction
*For any* sentence containing visual descriptions, the Visual_Generator should extract at least one VisualDescription object with non-empty subject field.
**Validates: Requirements 2.1**

Property 6: Scene generation from descriptions
*For any* valid VisualDescription, the Visual_Generator should produce a GeneratedScene with valid image data.
**Validates: Requirements 2.2**

Property 7: Scene rendering pipeline
*For any* GeneratedScene, the VR_Renderer should successfully display it without throwing errors.
**Validates: Requirements 2.3**

Property 8: Narrative element prioritization
*For any* batch of scene requests, requests containing narrative elements should be processed before requests with only background descriptions.
**Validates: Requirements 2.4**

Property 9: Scene persistence for non-visual content
*For any* sequence where a visual sentence is followed by non-visual sentences, the visual scene should remain displayed until the next visual sentence appears.
**Validates: Requirements 2.5**

Property 10: Narrative element registration
*For any* narrative element introduced in book content, the Narrative_Tracker should record it with all specified attributes.
**Validates: Requirements 3.1**

Property 11: Narrative consistency round-trip
*For any* narrative element, after registering it with the Narrative_Tracker and then retrieving it by ID, the retrieved element should have identical attributes to the original.
**Validates: Requirements 3.2, 3.3**

Property 12: Narrative element updates
*For any* registered narrative element, updating its attributes should result in subsequent retrievals returning the updated attributes.
**Validates: Requirements 3.4**

Property 13: Narrative element disambiguation
*For any* two narrative elements with different attributes (e.g., "blue dog" vs "brown dog"), the Narrative_Tracker should assign them distinct element IDs.
**Validates: Requirements 3.5**

Property 14: Audio-visual synchronization latency
*For any* sentence being narrated, the time difference between TTS audio output and corresponding visual scene display should be less than 100 milliseconds.
**Validates: Requirements 4.1**

Property 15: Coordinated pause behavior
*For any* playback position, when pause is activated, both audio narration and visual scene transitions should stop within the same frame.
**Validates: Requirements 4.2, 9.2**

Property 16: Synchronized seeking
*For any* target sentence position, when skip is executed, both audio and visual content should synchronize to that position within 100 milliseconds of each other.
**Validates: Requirements 4.3**

Property 17: Graceful latency degradation
*For any* scene generation that exceeds 100ms, the Sync_Controller should display a transition effect while audio continues playing without interruption.
**Validates: Requirements 4.4**

Property 18: Synchronization metrics logging
*For any* sentence processed, the Sync_Controller should log timing metrics including latency and cache hit status.
**Validates: Requirements 4.5**

Property 19: Chapter boundary detection
*For any* book content with chapter markers, the Content_Parser should detect all chapter boundaries and signal appropriate pauses.
**Validates: Requirements 5.2**

Property 20: Cloud generation routing
*For any* scene request marked as complex, the Visual_Generator should route it to cloud-based AI services rather than local generation.
**Validates: Requirements 6.1**

Property 21: Image compression
*For any* generated scene, the image data should be compressed to reduce size while maintaining visual quality above a minimum threshold.
**Validates: Requirements 6.2**

Property 22: Network fallback behavior
*For any* scene generation request, when network connectivity is unavailable, the Visual_Generator should fall back to cached or simplified local generation without failing.
**Validates: Requirements 6.3**

Property 23: Opportunistic pre-loading
*For any* reading session with available bandwidth and resources, the system should pre-generate scenes for upcoming sentences before they are needed.
**Validates: Requirements 6.4, 8.5, 11.3**

Property 24: Scene cache round-trip
*For any* generated scene, after caching it and then retrieving it by sentence ID, the retrieved scene should have identical image data to the original.
**Validates: Requirements 6.5**

Property 25: Session duration tracking
*For any* reading session, the Session_Monitor should accurately track the total duration from start to end.
**Validates: Requirements 7.1, 10.1**

Property 26: Break prompts at threshold
*For any* reading session, when duration exceeds 30 minutes, the Session_Monitor should prompt the user to take a break.
**Validates: Requirements 7.2**

Property 27: Periodic comfort rating collection
*For any* reading session, comfort ratings should be collected at regular intervals (every 10 minutes).
**Validates: Requirements 7.3**

Property 28: Head movement tracking
*For any* reading session, all head movement data from VR sensors should be recorded with timestamps.
**Validates: Requirements 7.4**

Property 29: Comfort-based adjustment suggestions
*For any* session where comfort metrics fall below threshold, the Session_Monitor should suggest visual setting adjustments.
**Validates: Requirements 7.5**

Property 30: API request rate limiting
*For any* sequence of cloud generation requests, the Visual_Generator should enforce a maximum concurrent request limit to prevent rate limiting.
**Validates: Requirements 8.2**

Property 31: Resource-constrained degradation priority
*For any* system state with constrained resources, visual complexity should be reduced before frame rate is reduced.
**Validates: Requirements 8.4**

Property 32: Progress indicator accuracy
*For any* reading position in a book, the displayed progress indicator should accurately reflect the percentage of book completed.
**Validates: Requirements 9.3**

Property 33: Bookmark round-trip
*For any* sentence position, after creating a bookmark and then retrieving bookmarks for that session, the bookmark should be present with the correct sentence ID.
**Validates: Requirements 9.4**

Property 34: Reading event tracking
*For any* user action (pause, skip, reread), the system should record an engagement event with correct type and timestamp.
**Validates: Requirements 10.2**

Property 35: Quiz presentation at boundaries
*For any* book with comprehension assessment enabled, quiz questions should be presented at chapter boundaries.
**Validates: Requirements 10.3**

Property 36: Comprehension score calculation
*For any* set of quiz answers, the system should calculate a comprehension score as the percentage of correct answers.
**Validates: Requirements 10.4**

Property 37: Engagement report generation
*For any* completed reading session, the system should generate an engagement report containing all tracked metrics.
**Validates: Requirements 10.5**

Property 38: Adaptive quality reduction
*For any* network condition where speed drops below optimal, the system should reduce visual quality while maintaining synchronization.
**Validates: Requirements 11.2**

Property 39: Network recovery behavior
*For any* network interruption followed by restoration, the system should resume cloud-based generation within 5 seconds.
**Validates: Requirements 11.4**

Property 40: Network status display
*For any* network condition (good, degraded, offline), the system should display an appropriate status indicator to the user.
**Validates: Requirements 11.5**

Property 41: Narration speed adjustment range
*For any* speed value between 0.5x and 2.0x, the TTS engine should successfully adjust narration speed to that value.
**Validates: Requirements 12.1**

Property 42: Display settings application
*For any* valid text size and contrast settings, the system should apply them to the display without errors.
**Validates: Requirements 12.2**

Property 43: Age-appropriate content filtering
*For any* book with an age rating and any user age setting, the system should only show books appropriate for that age.
**Validates: Requirements 12.3**

Property 44: Voice selection
*For any* available voice option, the TTS engine should successfully switch to that voice for narration.
**Validates: Requirements 12.4**

## Error Handling

### Error Categories

**1. Network Errors**
- Connection timeout during cloud generation
- Rate limiting from AI service
- Network interruption during scene download

**Handling Strategy:**
- Implement exponential backoff for retries
- Fall back to cached scenes or simplified local generation
- Display clear network status to user
- Queue failed requests for retry when connectivity restored

**2. Content Parsing Errors**
- Malformed book content
- Unsupported file formats
- Missing metadata

**Handling Strategy:**
- Validate book content before processing
- Provide detailed error messages for unsupported formats
- Attempt graceful degradation (skip problematic sections)
- Log parsing errors for debugging

**3. Resource Exhaustion**
- GPU memory exceeded
- Cache storage full
- API quota exceeded

**Handling Strategy:**
- Implement aggressive cache eviction when storage full
- Reduce visual quality before failing
- Display quota warnings to user
- Throttle generation requests proactively

**4. Synchronization Errors**
- Latency exceeds threshold
- Audio-visual drift detected
- Scene generation timeout

**Handling Strategy:**
- Display transition effects during delays
- Re-synchronize on user pause/resume
- Skip problematic scenes if necessary
- Log sync errors for analysis

**5. User Comfort Issues**
- Motion sickness indicators detected
- Extended session without breaks
- Rapid comfort rating decline

**Handling Strategy:**
- Prompt immediate break suggestions
- Reduce visual motion and effects
- Offer to pause session automatically
- Provide comfort recovery tips

### Error Recovery Patterns

**Retry with Backoff:**
```typescript
async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;
      const delay = baseDelay * Math.pow(2, attempt);
      await sleep(delay);
    }
  }
  throw new Error('Max retries exceeded');
}
```

**Circuit Breaker:**
```typescript
class CircuitBreaker {
  private failureCount: number = 0;
  private lastFailureTime: number = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > 60000) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is open');
      }
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  private onSuccess(): void {
    this.failureCount = 0;
    this.state = 'closed';
  }
  
  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= 5) {
      this.state = 'open';
    }
  }
}
```

## Testing Strategy

### Dual Testing Approach

The Virtual Bookset VR system requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests** focus on:
- Specific examples of content parsing (dialogue, descriptions, actions)
- Edge cases (empty content, special characters, malformed input)
- Error conditions (network failures, invalid data, resource exhaustion)
- Integration points between components (TTS → Sync Controller, Visual Generator → VR Renderer)
- VR-specific functionality (stereoscopic rendering, spatial audio)

**Property-Based Tests** focus on:
- Universal properties that hold for all inputs (round-trip consistency, synchronization latency)
- Comprehensive input coverage through randomization (random books, sentences, narrative elements)
- Invariants that must be maintained (narrative consistency, cache coherence)
- Performance characteristics (latency bounds, resource limits)

### Property-Based Testing Configuration

**Framework Selection:**
- **TypeScript/JavaScript**: fast-check library
- **Python**: Hypothesis library
- **Rust**: proptest or quickcheck

**Test Configuration:**
- Minimum 100 iterations per property test (due to randomization)
- Each property test references its design document property
- Tag format: **Feature: virtual-bookset-vr, Property {number}: {property_text}**

**Example Property Test Structure:**
```typescript
import fc from 'fast-check';

// Feature: virtual-bookset-vr, Property 11: Narrative consistency round-trip
test('narrative element round-trip preserves attributes', () => {
  fc.assert(
    fc.property(
      fc.record({
        elementId: fc.uuid(),
        type: fc.constantFrom('character', 'object', 'setting'),
        name: fc.string({ minLength: 1 }),
        attributes: fc.dictionary(fc.string(), fc.string())
      }),
      (element) => {
        const tracker = new NarrativeTracker();
        tracker.registerElement('test-book', element);
        const retrieved = tracker.getElement('test-book', element.elementId);
        
        expect(retrieved).not.toBeNull();
        expect(retrieved.name).toBe(element.name);
        expect(retrieved.type).toBe(element.type);
        expect(retrieved.attributes).toEqual(element.attributes);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing Strategy

**Component-Level Tests:**
- Content Parser: Test parsing of various book formats and structures
- TTS Engine: Test audio generation, playback controls, speed adjustment
- Visual Generator: Test scene generation, compression, caching
- Narrative Tracker: Test element registration, retrieval, updates
- Sync Controller: Test timing coordination, latency measurement
- VR Renderer: Test scene display, transitions, performance

**Integration Tests:**
- End-to-end reading flow (load book → parse → narrate → generate → display)
- Synchronization across components (audio-visual timing)
- Error recovery scenarios (network loss, resource exhaustion)
- User interaction flows (pause, skip, bookmark)

**Performance Tests:**
- Latency measurement under various network conditions
- Frame rate stability during extended sessions
- Memory usage over time (cache growth, leak detection)
- Cloud API response times and rate limiting

### Test Data Generation

**Synthetic Book Content:**
- Generate books with varying complexity (simple children's books to complex novels)
- Include diverse narrative elements (characters, objects, settings)
- Mix content types (dialogue, descriptions, actions)
- Include edge cases (special characters, formatting, very long sentences)

**Narrative Element Generators:**
- Random character descriptions with consistent attributes
- Object descriptions with varying complexity
- Setting descriptions with spatial relationships
- Attribute modifications over time

**Performance Test Scenarios:**
- Optimal network conditions (high bandwidth, low latency)
- Degraded network (low bandwidth, high latency, packet loss)
- Resource constraints (limited GPU, memory pressure)
- Extended sessions (30+ minutes, 100+ pages)

### Success Criteria

**Functional Correctness:**
- All property tests pass with 100+ iterations
- All unit tests pass
- No critical bugs in integration tests

**Performance Targets:**
- Average latency < 100ms (95th percentile < 150ms)
- Frame rate ≥ 72 FPS (target 90 FPS)
- Cache hit rate > 80% during normal reading
- Scene generation time < 2 seconds per sentence

**User Comfort:**
- Motion sickness indicators < 5% of sessions
- Average comfort rating ≥ 4.0/5.0
- Session completion rate > 80% (users finish intended reading)

**Engagement Metrics:**
- Average VR reading time ≥ 1.5x physical book reading time
- Comprehension scores ≥ 90% of physical book baseline
- User retention rate > 70% after first week
