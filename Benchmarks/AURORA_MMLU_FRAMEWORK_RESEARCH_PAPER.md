# MMLU Benchmark Framework: Evaluation System for Adaptive Cognitive Layers

**Thynaptic TR-2025-07**

**Authors:** Evaluation & Benchmarking Team  
**Division:** Thynaptic Research  
**Date:** 2025-11-24  
**Version:** v1.0.0

---

## Abstract

We analyze the Aurora MMLU Benchmark Framework, an evaluation system that assesses adaptive cognitive layer performance on the Massive Multitask Language Understanding benchmark. The framework integrates with Aurora's complete cognitive pipeline, routing test requests through intent classification, context engineering, memory systems, reasoning layers, safety mechanisms, and model routing. The system supports single-model, multi-model, and all-model testing modes across local, cloud, and hybrid routing configurations. The framework collects accuracy, latency, routing distribution, confidence scores, and domain-specific metrics. All components are production-grade with no placeholders or simulated behavior.

---

## Overview

The framework structures evaluation of adaptive cognitive layers by routing all test requests through the complete ACL pipeline rather than testing models in isolation. This mechanism ensures benchmark results reflect production system behavior, where requests pass through ten cognitive components before model inference.

The framework maps MMLU questions through the ACL pipeline, collects performance metrics at each stage, and generates comparative analysis across models and routing strategies. The system integrates with Aurora's ACL via HTTP-based server-client architecture, enabling evaluation of local models, cloud models, and hybrid routing configurations.

The framework addresses the limitation of traditional benchmarking approaches that bypass cognitive layers, routing systems, and memory mechanisms. By testing the complete pipeline, we assess how cognitive architecture affects performance rather than measuring isolated model capabilities.

---

## Architectural Context

### Cognitive Pipeline Integration

The framework routes all test requests through Aurora's ten-component ACL pipeline:

1. Intent Classifier: Categorizes user requests by intent type
2. Context Engine: Builds contextual information from workspace and memory
3. Workspace Memory: Retrieves relevant workspace items
4. Long-term Memory: Accesses conversation history and semantic memory
5. Reasoning Layer: Routes complex questions to specialized reasoning modules (conditional)
6. Safety Layer: Validates responses for hallucinations and factual accuracy
7. Personality + Style Layer: Adapts response tone and style
8. Response Construction: Builds full prompt with all context layers
9. Confidence Scoring: Calculates self-aware confidence metrics
10. HybridBridge Model Routing: Selects optimal model (local/cloud/hybrid)

### System Architecture

```
MMLU Dataset Loader
    ↓
Test Runner (Model Rotation)
    ↓
Aurora ACL Client (HTTP)
    ↓
Aurora ACL Server
    ↓
ACL Cognitive Pipeline (10 Components)
    ↓
Model Inference (Local/Cloud/Hybrid)
    ↓
Result Aggregation
    ↓
Report Generation
```

### ACL Integration Mechanism

**ACL Server (`aurora_acl_server.py`):**
- Exposes ACL pipeline via HTTP endpoint (`/generate`)
- Orchestrates all 10 ACL components
- Handles request/response serialization
- Manages error handling and timeouts

**ACL Client (`aurora_acl_client.py`):**
- Sends HTTP requests to ACL server
- Formats MMLU questions as ACL requests
- Parses ACL responses to extract answers
- Handles connection errors and retries

**Request Flow:**
1. MMLU question formatted as ACL request
2. Request sent to ACL server via HTTP POST
3. ACL server processes through full pipeline
4. Response includes answer, confidence, routing path, metadata
5. Framework extracts answer and records metrics

### Model Rotation System

The framework structures three testing modes:

**Single Model Mode:**
- `--model MODEL` parameter
- Generates single result file
- Example: `--model qwen3:1.7b`

**Multi-Model Mode:**
- `--models MODEL1 MODEL2 ...` parameter
- Generates individual result files per model
- Optional combined report with `--combined-report`
- Example: `--models qwen3:1.7b gemma3:4b deepseek-r1:1.5b`

**All-Model Mode:**
- `--all-models` parameter
- Tests all available local models: gemma3:1b, qwen3:1.7b, gemma3:4b, qwen2.5-coder:1.5b, deepseek-r1:1.5b, granite3.2:2b
- Generates individual files plus optional combined report

### Routing Strategy Configuration

**Local-Only (`--local-only`):**
- All requests routed to local Ollama models
- No cloud fallback
- Privacy-preserving evaluation

**Cloud-Only (`--cloud-only`):**
- All requests routed to cloud Ollama API
- No local fallback
- Enables evaluation of large cloud models

**Hybrid (`--use-hybrid-bridge`, default):**
- Intelligent routing between local and cloud
- Automatic failover on errors
- Health monitoring and performance optimization

---

## Methodology

### Dataset Handling

The framework loads MMLU datasets from HuggingFace's `datasets` library. Questions are organized by subject/domain. Each question includes: question text, four choices (A-D), correct answer, subject, task. The framework parses into `MMLUQuestion` dataclass.

**Domain Filtering:**
- `--domains DOMAIN1 DOMAIN2 ...` parameter
- Default: all domains in dataset
- Enables focused evaluation on specific knowledge areas

**Question Limiting:**
- `--max-questions N` parameter
- Default: all questions in domain
- Supports iterative development workflows

### Test Execution Protocol

**Test Flow:**
1. Load MMLU dataset from specified path
2. Filter domains (if specified)
3. For each model (in rotation):
   a. Create `MMLUTestRunner` instance
   b. Initialize ACL client
   c. For each domain:
      - Format questions with few-shot examples
      - Send to ACL pipeline
      - Extract answer and record metrics
   d. Generate summary statistics
   e. Save results to JSON file
4. Generate combined report (if multiple models)

**Few-Shot Learning:**
- `--few-shot N` parameter (default: 5)
- Examples selected from same domain
- Improves model performance on domain-specific questions

### Metric Collection

**Accuracy Metrics:**
- Overall accuracy: (correct / total) × 100
- Domain-specific accuracy: (domain correct / domain total) × 100
- Category accuracy: (category correct / category total) × 100
- Per-model accuracy (multi-model tests)

**Latency Metrics:**
- Average latency per question
- Domain-specific latency
- Routing path latency (local vs. cloud vs. hybrid)
- ACL processing overhead

**Routing Analytics:**
- Routing path distribution: acl_local, acl_cloud, acl_hybrid, acl_local_reasoning, acl_cloud_reasoning, acl_hybrid_reasoning
- Reasoning layer trigger rate
- Model selection distribution
- Failover frequency (hybrid mode)

**Confidence Metrics:**
- Average confidence scores
- Confidence distribution
- Confidence-accuracy correlation

**Error Tracking:**
- Error rate per domain
- Error types: timeout, network, model error
- Error recovery (hybrid mode)

### Result Output Structure

**Individual Model Results:**
- JSON file per model (if multi-model test)
- Filename format: `{output_prefix}_{model_name}.json`
- Includes: summary statistics, detailed results, domain breakdowns

**Combined Report (`--combined-report`):**
- Aggregated statistics across all models
- Best model identification (highest accuracy)
- Fastest model identification (lowest latency)
- Domain aggregates across models
- Model-by-model comparison

**Report JSON Structure:**
```json
{
  "overall_accuracy": 45.23,
  "total_questions": 180,
  "total_correct": 81,
  "avg_latency_ms": 5234.5,
  "routing_distribution": {
    "acl_local": 120,
    "acl_cloud": 45,
    "acl_hybrid": 15
  },
  "domain_results": {
    "college_computer_science": {
      "accuracy": 48.33,
      "correct": 29,
      "total_questions": 60
    }
  },
  "detailed_results": [...],
  "timestamp": "2025-11-24T..."
}
```

---

## Evaluation Results

### Framework Capabilities

The framework ensures every test request flows through the complete ACL pipeline. This mechanism produces benchmark results that reflect production system performance rather than isolated model capabilities.

### Comparative Model Analysis

**Single-Model Evaluation:**
- Detailed performance metrics for one model
- Domain-specific strengths and weaknesses
- Routing behavior analysis
- Confidence calibration

**Multi-Model Comparison:**
- Side-by-side accuracy comparison
- Latency trade-offs
- Best model identification per domain
- Routing strategy comparison

**All-Model Comprehensive Evaluation:**
- Complete model portfolio assessment
- Model selection recommendations
- Performance baseline establishment
- Architecture optimization insights

### Routing Strategy Evaluation

**Local-Only Analysis:**
- Privacy-preserving performance
- Resource utilization
- Model capability assessment
- Latency characteristics

**Cloud-Only Analysis:**
- Large model performance
- Network latency impact
- Cost-performance trade-offs
- Reliability assessment

**Hybrid Strategy Analysis:**
- Intelligent routing effectiveness
- Failover behavior
- Performance optimization
- Cost efficiency

### Domain-Specific Performance

**STEM Domains:**
- Mathematics, physics, chemistry, computer science
- Reasoning layer effectiveness
- Model selection patterns

**Humanities Domains:**
- History, philosophy, literature
- Context retrieval effectiveness
- Memory system utilization

**Social Sciences:**
- Psychology, economics, geography
- Cross-domain knowledge application
- Confidence calibration

### Production-Grade Implementation

**Implementation Characteristics:**
- All components fully implemented (no placeholders)
- Real ACL integration (not simulated)
- Actual model inference (not mocked)
- Robust error handling and recovery
- Graceful degradation
- Detailed error logging

**Extensibility:**
- Modular architecture
- Easy addition of new benchmarks
- Custom metric support

---

## System Behavior Analysis

### HTTP-Based ACL Integration

We structure HTTP-based integration over direct Python imports to enable remote ACL server deployment, support distributed testing architectures, maintain clear separation of concerns, and facilitate future Swift-native ACL integration.

### Model Rotation System Behavior

The flexible model rotation system enables efficient comparative analysis, comprehensive portfolio evaluation, focused single-model deep dives, and iterative development workflows.

### Metric Collection Behavior

Collecting detailed metrics enables performance optimization insights, routing strategy evaluation, confidence calibration analysis, and domain-specific optimization.

### Routing Path Distribution

The framework tracks routing path distribution across acl_local, acl_cloud, acl_hybrid, and reasoning-triggered variants. This data maps how routing decisions correlate with question complexity and domain characteristics.

### Confidence-Accuracy Correlation

The framework measures confidence scores alongside accuracy to assess calibration. Higher confidence should correlate with higher accuracy if the confidence scoring mechanism functions correctly.

### Error Handling Behavior

The framework handles errors through robust error logging, graceful degradation, and automatic failover in hybrid mode. Error types are categorized as timeout, network, or model errors.

---

## Limitations

### ACL Server Dependency

The framework requires the ACL server to be running, adding setup complexity. Future versions may support direct Swift integration to remove this dependency.

### Dataset Size Constraints

Full MMLU evaluation (57 domains, 15,908 questions) requires significant time and resources. The framework supports domain filtering and question limiting for iterative development, but complete evaluation remains resource-intensive.

### Cloud Model Availability

Cloud model testing depends on Ollama Cloud API availability and API key configuration. The framework gracefully handles cloud unavailability but cannot test cloud models when the API is unavailable.

### Single Benchmark Focus

Currently focused on MMLU. Future versions may support additional benchmarks (HellaSwag, TruthfulQA, HaluEval), but current implementation is MMLU-specific.

### HTTP Overhead

HTTP-based ACL integration adds latency overhead compared to direct integration. This overhead is measurable but necessary for current architecture.

### Limited Reasoning Depth Analysis

The framework tracks reasoning layer triggers but does not analyze reasoning depth or step-by-step reasoning chains. This limits understanding of how reasoning affects performance.

---

## Forward Research Trajectories

### Additional Benchmarks

- HellaSwag (commonsense reasoning)
- TruthfulQA (truthfulness evaluation)
- HaluEval (hallucination detection)
- Custom domain-specific benchmarks

### Direct Swift Integration

- Native Swift ACL integration (no HTTP server)
- Improved performance and reduced latency
- Better error handling and type safety

### Advanced Analytics

- Confidence-accuracy correlation analysis
- Routing decision explainability
- Performance prediction models
- Automated optimization recommendations

### Distributed Testing

- Parallel model testing
- Distributed dataset processing
- Cloud-based test execution
- Real-time result aggregation

### Visualization

- Interactive result dashboards
- Performance comparison charts
- Domain-specific heatmaps
- Routing decision trees

### Reasoning Depth Analysis

- Step-by-step reasoning chain extraction
- Reasoning depth measurement
- Reasoning quality assessment
- Reasoning-performance correlation

---

## References

1. Hendrycks, D., et al. (2020). "Measuring Massive Multitask Language Understanding." *arXiv preprint arXiv:2009.03300*.

2. HELM: Holistic Evaluation of Language Models. (2022). *Stanford Center for Research on Foundation Models*.

3. Thynaptic Development Team. (2025). "Aurora: An Adaptive Cognitive Layer for Human-AI Collaboration." *Thynaptic TR-2025-04*.

4. Thynaptic Development Team. (2025). "Aurora Routing Layer: Intelligent Model Selection for Adaptive Cognitive Systems." *Thynaptic TR-2025-01*.

5. Thynaptic Development Team. (2025). "Aurora Hybrid Bridge: Local Routing with Failover Architecture for Adaptive Cognitive Systems." *Thynaptic TR-2025-02*.

6. Thynaptic Development Team. (2025). "Aurora Cloud Routing: Cloud-Only Inference Performance for Adaptive Cognitive Systems." *Thynaptic TR-2025-03*.

---

## Appendices

### Appendix A: Framework Usage Examples

**Single Model Test:**
```bash
python3 scripts/mmlu_test_framework.py \
  --dataset mmlu_dataset \
  --model qwen3:1.7b \
  --domains college_computer_science college_mathematics \
  --max-questions 15 \
  --local-only \
  --output mmlu_qwen3_results.json
```

**All Models Test:**
```bash
python3 scripts/mmlu_test_framework.py \
  --dataset mmlu_dataset \
  --all-models \
  --domains college_computer_science college_mathematics high_school_biology \
  --max-questions 15 \
  --local-only \
  --combined-report \
  --output mmlu_all_models_results.json
```

**Cloud-Only Test:**
```bash
python3 scripts/mmlu_test_framework.py \
  --dataset mmlu_dataset \
  --model gpt-oss:20b \
  --domains college_computer_science \
  --max-questions 15 \
  --cloud-only \
  --output mmlu_cloud_results.json
```

### Appendix B: Available Models

**Local Models (from ModelTierMap.swift):**
- `gemma3:1b` - Casual, foreground, chat (Tier 1)
- `qwen3:1.7b` - Casual fallback, tasks, thinking (Tier 1)
- `gemma3:4b` - Documents, multimodal, images (Tier 2)
- `qwen2.5-coder:1.5b` - Coding, reasoning, structured-reasoning (Tier 2)
- `deepseek-r1:1.5b` - Research, document fallback, deep-reasoning (Tier 3)
- `granite3.2:2b` - Background, summaries, memory (Tier 4)

**Cloud Models (Ollama Cloud API):**
- `gpt-oss:20b` - 20B parameters, general purpose
- `glm-4.6` - Large model, general purpose
- `minimax-m2` - Large model, general purpose
- `qwen3-vl:235b-instruct` - 235B parameters, vision-language
- `cogito-2.1:671b` - 671B parameters, research-focused

### Appendix C: Framework Components

**Core Files:**
- `scripts/mmlu_test_framework.py` - Main test framework
- `scripts/aurora_acl_client.py` - ACL HTTP client
- `scripts/aurora_acl_server.py` - ACL HTTP server
- `scripts/mmlu_report_generator.py` - Markdown report generator
- `scripts/MMLU_MODEL_ROTATION_GUIDE.md` - Usage documentation

**Dependencies:**
- `requests` - HTTP client library
- `datasets` - HuggingFace datasets library
- Python 3.8+

### Appendix D: Metric Definitions

**Accuracy:**
- Overall: (total correct / total questions) × 100
- Domain: (domain correct / domain total) × 100
- Category: (category correct / category total) × 100

**Latency:**
- Per-question: Time from request to response (includes ACL processing)
- Average: Mean latency across all questions
- Domain average: Mean latency per domain

**Routing Paths:**
- `acl_local` - Local-only routing with ACL
- `acl_cloud` - Cloud-only routing with ACL
- `acl_hybrid` - Hybrid routing with ACL
- `acl_local_reasoning` - Local with reasoning layer
- `acl_cloud_reasoning` - Cloud with reasoning layer
- `acl_hybrid_reasoning` - Hybrid with reasoning layer

**Confidence Score:**
- Range: 0.0 to 1.0
- Based on: Recall quality, context freshness, intent signals
- Higher confidence indicates stronger system certainty

---
