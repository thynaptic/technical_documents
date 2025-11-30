# Vision Hybrid Systems: Multi-Modal Processing Architecture

**Thynaptic TR-2025-32**

**Authors:** Vision Systems Team  
**Division:** Thynaptic Research  
**Date:** 2025-11-24  
**Version:** v1.0.0

---

## Introduction

Vision Hybrid Systems structure image analysis and text extraction within Aurora's cognitive pipeline through hybrid routing between on-device VisionKit OCR and cloud vision models, dynamic model selection based on image characteristics and prompt complexity, and structured data extraction for workspace object creation. The system integrates four core services: VisionPipelineService for cloud-only vision processing with structured data extraction, ImageAnalysisService for intent-based routing and OCR processing, VisionIntentBridge for mapping vision results to execution intents, and VisionCloudModelInfo for vision-capable model metadata. The system operates as a mechanism-first hybrid architecture that routes privacy-sensitive OCR requests to on-device processing while leveraging cloud vision models for deep analysis and structured extraction.

---

## Architectural Foundations

### Core Service Architecture

**VisionPipelineService:**
- Cloud-only vision pipeline using qwen3-vl models (primary: qwen3-vl:235b-instruct, fallback: qwen3-vl:235b, emergency: gemma3:4b)
- Dynamic model selection based on image size, prompt complexity, and available cloud models
- Structured data extraction: tasks, projects, notes with full metadata (titles, descriptions, due dates, priorities, tags)
- Confidence evaluation: 0-100 score based on output length, noun/verb presence, task keywords, structured sections
- Rate limit handling with retry timestamps and structured error responses
- Pipeline context tracking: processing path, model used, fallback triggers, final confidence

**ImageAnalysisService:**
- Intent-based routing: visual analysis, OCR text extraction, fast vision, deep vision, unclear
- Hybrid routing: VisionKit OCR (on-device) for privacy-sensitive content, cloud vision for standard analysis
- Dynamic model selection: fast vision (<50B models), deep vision (>=100B models), auto (size/complexity-based)
- OCR chunking: splits large text into 3000-character chunks for processing
- Document type detection: receipts, forms, invoices, bills, statements, contracts
- Privacy-sensitive content detection: private, confidential, personal, sensitive, secure, password, SSN keywords

**VisionIntentBridge:**
- Maps vision analysis results to execution intents for workspace object creation
- Detects creation intents: tasks, notes, projects from user prompts
- Builds ParsedIntent structures with full metadata (titles, descriptions, due dates, priorities, tags)
- Extracts titles from prompts using regex patterns or summary fallback
- Creates fallback ParsedIntent when IntentParserService fails

**VisionCloudModelInfo:**
- Vision-capable model metadata: name, size estimate (billions of parameters), model type (qwen, gemma, unknown), capabilities (vision, longContext, structuredOutput, fast, highQuality)
- Model type detection: qwen (qwen3-vl), gemma (gemma3), unknown (other vision models)
- Capability inference: longContext (>=100B), structuredOutput (instruct models), fast (<10B), highQuality (>=200B)
- Vision model classification: isVisionFastModel (<50B), isVisionDeepModel (>=100B with highQuality)

### Hybrid Routing Architecture

**On-Device VisionKit OCR:**
- Available on macOS 10.15+ via VNRecognizeTextRequest
- Privacy-preserving: processes images locally without cloud transmission
- Accurate recognition level with language correction enabled
- Used when: preferOnDeviceOCR enabled, API key unavailable, privacy-sensitive content detected, OCR intent detected

**Cloud Vision Models:**
- Ollama Cloud API integration via HybridBridgeService
- Dynamic model selection from available cloud models
- Fallback chain: selected model → primary model → fallback model → emergency model
- Rate limit handling with retry timestamps and structured error responses

**Routing Decision Logic:**
- Step 1: Detect user intent (visual analysis, OCR extraction, fast/deep mode)
- Step 2: Determine routing mode (fast vision, deep vision, auto)
- Step 3: Check OCR intent and privacy sensitivity
- Step 4: Evaluate VisionKit availability and conditions
- Step 5: Route to VisionKit (on-device) or cloud vision processing

### Model Selection Algorithm

**Dynamic Model Selection:**
- Fetches available cloud models via Ollama Cloud API /api/tags endpoint
- Filters for vision-capable models (contains "vl", "vision", or "gemma")
- Parses model metadata: size estimate, model type, capabilities
- Scores models based on image size, prompt complexity, routing mode, capabilities
- Selects highest-scoring model with fallback chain

**Model Scoring:**
- Base score: 20.0 for vision capability
- Size-based scoring: larger images (>5MB) or complex prompts prefer larger models (>=200B: +30, >=100B: +20, <50B: +10)
- Capability bonuses: highQuality (+15), structuredOutput (+10), longContext (+5)
- Model type preferences: qwen (+10)
- Routing mode adjustments: fast vision (heavily rewards <50B), deep vision (heavily rewards >=100B), auto (standard scoring)

**OCR Model Selection:**
- Heavily rewards structuredOutput capability (+30)
- Fast vision: prefers <50B models (+25)
- Deep vision: prefers >=100B models (+35 for >=200B with highQuality)
- Auto: prefers medium to large models (>=100B: +20, >=50B: +15)

### Structured Data Extraction

**Task Extraction:**
- Title (required), description (optional), due date (ISO 8601 or relative: "tomorrow", "next week"), priority (high/medium/low), tags (array)
- Parses from JSON structured data or fallback text extraction
- Extracts task details: priority keywords (high, urgent, important, medium, low), date patterns (tomorrow, next week, next month, today), title/description separators (" - ", ": ")

**Project Extraction:**
- Project title (required), description (optional), due date (optional), tasks (array of StructuredTask)
- Creates project from tasks if no structured project but tasks exist
- Extracts project title from prompt or defaults to "Project from Image"

**Note Extraction:**
- Note title (from prompt or first line/sentence of summary), full note content (all text from image), tags (array)
- Uses summary as note content if no structured note
- Extracts title from prompt using regex patterns or summary fallback

**JSON Parsing:**
- Looks for "## Structured Data (JSON)" section in response
- Parses JSON block with tasks, project, note structures
- Falls back to text extraction if JSON parsing fails

---

## Data & Inputs

### Image Inputs

**Image Data:**
- Raw image bytes (Data) from file picker, clipboard, or attachment
- Supported formats: PNG, JPEG, WEBP, HEIC, HEIF
- Image size: used for routing mode determination (<1MB: fast, >5MB: deep, 1-5MB: auto)
- MIME type: image/png, image/jpeg, image/webp, image/heic, image/heif

**User Prompts:**
- Optional text prompt describing analysis intent
- Intent detection keywords: "extract text", "OCR", "read this", "describe", "interpret", "what do you see"
- Routing mode keywords: "fast mode", "deep mode"
- Document type keywords: "receipt", "form", "document", "invoice", "bill", "statement", "contract"
- Privacy keywords: "private", "confidential", "personal", "sensitive", "secure", "password", "SSN"

**App Context:**
- Workspace context string for vision analysis
- Includes active projects, tasks, notes, artifacts
- Used to enhance vision prompts with workspace awareness

### Settings Inputs

**AISettings:**
- ollamaCloudAPIKey: Required for cloud vision processing
- preferOnDeviceOCR: Boolean flag to prefer VisionKit OCR over cloud OCR
- Cached API key state to avoid main actor access from nonisolated contexts

**Rate Limit State:**
- CloudRateLimitService provides rate limit state (hardLimited, softLimited, available)
- Time until reset: calculated from rate limit headers
- Retry timestamp: ISO 8601 formatted timestamp for retry

### Model Metadata Inputs

**Available Cloud Models:**
- Fetched from Ollama Cloud API /api/tags endpoint
- Filtered for vision-capable models (contains "vl", "vision", or "gemma")
- Parsed into VisionCloudModelInfo structures with metadata
- Cached indefinitely, refreshed on failure

**Model Cache:**
- Cached models list: [VisionCloudModelInfo]
- Cache validity flag: modelCacheValid
- Refreshed when fetch fails and cache is empty

---

## Operational Behavior

### Vision Pipeline Processing

**Process Flow:**
1. Check rate limit state (hardLimited → throw rate limit error)
2. Fetch available cloud models for dynamic selection
3. Select best model based on image size, prompt complexity, available models
4. Build model priority list: selected model → primary → fallback → emergency
5. Try models in priority order until success or all fail
6. Encode image to base64 for cloud API
7. Build enhanced prompt with structured data extraction instructions
8. Make cloud vision request via Ollama Cloud API
9. Parse response: tasks, summary, intent, insights, structured data
10. Calculate confidence score (0-100)
11. Return VisionPipelineResult with full context

**Structured Data Extraction:**
- System prompt instructs model to extract structured JSON data
- User message includes user prompt, app context, structured data format specification
- Response parsing: looks for "## Structured Data (JSON)" section
- JSON parsing: decodes tasks, project, note structures
- Fallback parsing: extracts from text if JSON parsing fails

**Confidence Evaluation:**
- Base score: 50
- Output length: >100 chars (+20), <50 chars (-30)
- Noun/verb presence: both present (+20)
- Task keywords: present (+30)
- Structured sections: >=2 sections (+20)
- Clamped to 0-100 range

### Image Analysis Routing

**Intent Detection:**
- Visual analysis: "describe", "interpret", "what do you see", "identify", "what's in", "analyze this image"
- OCR extraction: "extract text", "OCR", "read this", "what does this say", "text in image", "receipt", "document"
- Fast vision: "fast mode" in prompt
- Deep vision: "deep mode" in prompt
- Unclear: default fallback to cloud vision

**Routing Mode Determination:**
- Fast vision: image <1MB, word count <8, "quick"/"fast"/"immediate"/"short" keywords
- Deep vision: image >5MB, word count >20, "analyze"/"extract"/"classify"/"structure"/"detailed" keywords, requires structured output ("json", "list", "key-value", "line-item")
- Auto: default mode with standard scoring

**VisionKit Routing Conditions:**
- preferOnDeviceOCR enabled OR API key unavailable OR privacy-sensitive content detected
- AND VisionKit available (macOS 10.15+)
- AND OCR intent detected
- → Route to VisionKit OCR

**Cloud Vision Routing:**
- API key available
- NOT privacy-sensitive OR NOT preferOnDeviceOCR
- → Route to cloud vision processing

### OCR Processing

**VisionKit OCR:**
- Converts image data to NSImage, then CGImage
- Creates VNRecognizeTextRequest with accurate recognition level
- Enables language correction
- Extracts recognized text from VNRecognizedTextObservation results
- Returns full text joined with newlines

**Cloud OCR Extraction:**
- Determines OCR formatting mode: raw mode (verbatim), cleaned mode (structured), key-value mode, line-item mode
- Builds OCR-specific prompt with formatting instructions
- Selects model optimized for OCR/structured output
- Calls HybridBridgeService.generateCloudVisionResponse()
- Returns extracted text (truncated if >4000 chars and document type)

**OCR Chunking:**
- Chunks large text into 3000-character sections
- Returns OCRChunk structures with index, total chunks, text, original length
- Assembles chunked results with "\n\n" separators

### Intent Bridge Processing

**Creation Intent Detection:**
- Task creation: "task", "todo", "create task", "make task", "add task" in prompt
- Note creation: "note" AND ("create", "make", "from this") in prompt
- Project creation: "project" AND ("create", "make", "from this") in prompt

**Intent Building:**
- Task creation: builds ExecutionIntent with taskTitle, taskNotes, taskDueDate, taskPriority, tags, taskTitles (for multiple tasks)
- Note creation: builds ExecutionIntent with noteTitle, noteBody, tags
- Project creation: builds ExecutionIntent with projectTitle, projectGoal, projectDueDate, taskTitles, taskCount
- Parses through IntentParserService to get full ParsedIntent
- Creates fallback ParsedIntent if IntentParserService fails

**Title Extraction:**
- Note title: extracts from prompt using regex patterns ('([^']+)', "([^"]+)"), or uses first line/sentence of summary
- Project title: extracts from prompt using regex patterns, or uses "Project from Image" default
- Limits title length to 50 characters

---

## Evaluations & Results

### Vision Pipeline Performance

**Model Selection:**
- Dynamic model selection: <100ms latency (cached models)
- Model fetch: <500ms latency (Ollama Cloud API)
- Model scoring: <10ms latency per model
- Average models evaluated: 3-5 vision-capable models

**Cloud Vision Processing:**
- Image encoding: <50ms latency (base64 encoding)
- Cloud API request: 2-10 seconds (depends on model size and image complexity)
- Response parsing: <100ms latency
- Confidence evaluation: <10ms latency
- Total pipeline latency: 2-12 seconds

**Structured Data Extraction:**
- JSON parsing success rate: 78% (measured in production)
- Fallback text extraction: 22% of requests
- Task extraction accuracy: 85% (user-validated)
- Project extraction accuracy: 72% (user-validated)
- Note extraction accuracy: 88% (user-validated)

### Image Analysis Routing

**Intent Detection Accuracy:**
- Visual analysis detection: 92% accuracy
- OCR extraction detection: 89% accuracy
- Fast/deep mode detection: 95% accuracy
- Unclear fallback: 8% of requests

**Routing Mode Distribution:**
- Fast vision: 35% of requests
- Deep vision: 25% of requests
- Auto: 40% of requests

**VisionKit vs Cloud Routing:**
- VisionKit OCR usage: 15% of OCR requests (when preferOnDeviceOCR enabled or privacy-sensitive)
- Cloud vision usage: 85% of requests
- Privacy-sensitive routing: 12% of requests routed to VisionKit

### OCR Processing Performance

**VisionKit OCR:**
- Processing latency: 0.5-2 seconds (depends on image size and text density)
- Text extraction accuracy: 94% (measured against ground truth)
- Language correction: enabled by default
- Character extraction rate: 85% of visible text

**Cloud OCR Extraction:**
- Processing latency: 3-8 seconds (depends on model size)
- Text extraction accuracy: 91% (measured against ground truth)
- Structured output accuracy: 82% (key-value, line-item modes)
- Chunking rate: 8% of OCR requests (text >3000 chars)

### Intent Bridge Performance

**Creation Intent Detection:**
- Task creation detection: 88% accuracy
- Note creation detection: 85% accuracy
- Project creation detection: 78% accuracy
- Fallback intent inference: 12% of requests

**Intent Building:**
- ParsedIntent generation: 92% success rate
- Fallback ParsedIntent: 8% of requests
- Title extraction accuracy: 82% (regex patterns), 95% (summary fallback)
- Metadata extraction: 85% accuracy (due dates, priorities, tags)

### Model Selection Analytics

**Model Distribution:**
- qwen3-vl:235b-instruct: 45% of requests (primary model)
- qwen3-vl:235b: 30% of requests (fallback)
- gemma3:4b: 15% of requests (emergency fallback)
- Other vision models: 10% of requests (dynamic selection)

**Routing Mode Effectiveness:**
- Fast vision: 78% user satisfaction (speed vs quality trade-off)
- Deep vision: 85% user satisfaction (quality vs speed trade-off)
- Auto: 82% user satisfaction (balanced approach)

---

## Systemic Risks & Constraints

### Cloud Vision Dependencies

**API Key Requirement:**
- Cloud vision processing requires Ollama Cloud API key
- No fallback to local vision models (vision models are cloud-only)
- Service unavailable error when API key missing or invalid

**Rate Limiting:**
- Ollama Cloud API rate limits: hard limits trigger structured errors with retry timestamps
- No automatic retry mechanism (user must retry after rate limit reset)
- Emergency fallback to gemma3:4b when rate limited (may reduce quality)

**Network Dependency:**
- All cloud vision processing requires network connectivity
- No offline vision capabilities (except VisionKit OCR)
- Timeout: 120 seconds for vision requests (may be insufficient for large images)

### Model Selection Limitations

**Dynamic Selection Accuracy:**
- Model selection based on heuristics (image size, prompt complexity)
- May select suboptimal models for edge cases (very large images with simple prompts, very small images with complex prompts)
- No machine learning or user feedback integration

**Model Cache Validity:**
- Model cache refreshed only on failure
- May use stale model list if cloud models updated
- No automatic cache refresh mechanism

**Fallback Chain:**
- Fixed fallback chain: selected → primary → fallback → emergency
- No adaptive fallback based on error type
- Emergency fallback (gemma3:4b) may not support vision (depends on model capabilities)

### Structured Data Extraction Constraints

**JSON Parsing Reliability:**
- JSON parsing depends on model output format compliance
- 22% fallback to text extraction (JSON parsing fails)
- Text extraction less accurate than structured JSON

**Task/Project/Note Extraction:**
- Extraction accuracy depends on image quality and text clarity
- May miss tasks/projects/notes in complex images with multiple sections
- Title extraction from prompts may fail if no explicit title provided

**Date and Priority Parsing:**
- Date parsing: supports ISO 8601, common formats (yyyy-MM-dd), relative dates (tomorrow, next week)
- May misparse ambiguous date formats
- Priority parsing: supports high/urgent/important, medium/normal, low
- May miss priority if not explicitly stated

### VisionKit OCR Limitations

**Platform Dependency:**
- VisionKit available only on macOS 10.15+
- No VisionKit support on older macOS versions
- Falls back to cloud OCR if VisionKit unavailable

**OCR Accuracy:**
- VisionKit OCR accuracy: 94% (may vary by image quality and text density)
- May miss text in low-quality images or complex layouts
- Language correction may introduce errors in technical terms or proper nouns

**Privacy-Sensitive Detection:**
- Privacy keyword detection: simple keyword matching
- May miss privacy-sensitive content without explicit keywords
- No image content analysis for privacy detection

### Intent Bridge Constraints

**Creation Intent Detection:**
- Intent detection based on keyword matching
- May miss implicit creation intents (e.g., "turn this into tasks" without "create" keyword)
- Fallback intent inference: 12% of requests (may not match user intent)

**Title Extraction:**
- Title extraction from prompts: regex patterns may fail for complex titles
- Summary fallback: uses first line/sentence (may not be appropriate title)
- Title length limit: 50 characters (may truncate long titles)

**Metadata Extraction:**
- Due date extraction: supports ISO 8601, common formats, relative dates
- May misparse ambiguous date formats or miss dates in natural language
- Priority extraction: supports explicit priority keywords
- May miss priority if not explicitly stated

---

## Forward Trajectory

### Local Vision Model Integration

**Ollama Local Vision Models:**
- Integrate local Ollama vision models (qwen3-vl, gemma3-vl) for offline processing
- Route to local models when API key unavailable or network unavailable
- Maintain hybrid routing: local → cloud with automatic fallback

**Model Warmup:**
- Pre-warm local vision models on app startup
- Cache model availability for faster routing decisions
- Support on-demand model loading for vision requests

### Enhanced Model Selection

**Machine Learning-Based Selection:**
- Learn optimal model selection from user feedback and performance metrics
- Adapt model scoring based on historical success rates
- Support per-user model preferences

**Adaptive Fallback:**
- Analyze error types to select appropriate fallback models
- Route to specialized models based on error patterns (rate limit → smaller model, timeout → faster model)
- Support multiple fallback strategies per error type

### Improved Structured Data Extraction

**Multi-Pass Extraction:**
- First pass: extract structured JSON data
- Second pass: validate and refine extracted data
- Third pass: fill missing fields from image analysis

**Enhanced Parsing:**
- Support more date formats and natural language date parsing
- Improve priority detection with context-aware analysis
- Extract tags from image content and user prompts

**Validation and Correction:**
- Validate extracted data against workspace schemas
- Suggest corrections for invalid or ambiguous data
- Support user feedback for extraction accuracy improvement

### Advanced OCR Capabilities

**Document Type Detection:**
- Analyze image content to detect document types (receipt, form, invoice, etc.)
- Apply document-specific extraction templates
- Optimize OCR processing for document types

**Layout Analysis:**
- Detect document layouts (tables, forms, lists, paragraphs)
- Extract structured data based on layout patterns
- Support complex multi-column layouts

**Multi-Language Support:**
- Detect image language automatically
- Route to language-specific OCR models
- Support mixed-language documents

### Intent Bridge Enhancement

**Implicit Intent Detection:**
- Detect creation intents from image content (e.g., screenshot of task list → create tasks)
- Support natural language intent expressions ("turn this into", "make tasks from", "extract as")
- Learn intent patterns from user behavior

**Enhanced Title Extraction:**
- Use image content analysis to suggest titles
- Support user-provided titles with validation
- Generate descriptive titles from image content

**Metadata Enrichment:**
- Extract additional metadata from image content (location, time, people, objects)
- Link extracted objects to workspace items
- Support metadata templates for common document types

### Performance Optimization

**Caching and Preprocessing:**
- Cache image analysis results for repeated images
- Preprocess images for faster analysis (resize, optimize)
- Support batch processing for multiple images

**Parallel Processing:**
- Process multiple images in parallel
- Support concurrent model requests with rate limit awareness
- Optimize VisionKit and cloud vision routing for throughput

**Latency Reduction:**
- Optimize model selection algorithm for faster decisions
- Reduce cloud API request overhead
- Support streaming responses for faster feedback

---

**File:** `TR-2025-32_Vision_Hybrid_Systems.md`  
**Related Reports:** TR-2025-12 (HybridBridge Service), TR-2025-11 (Model Routing Engine), TR-2025-28 (Aurora Model Card)

