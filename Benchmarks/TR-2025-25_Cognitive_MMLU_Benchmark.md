# Cognitive MMLU Benchmark: Adaptive Cognitive Layer Capability Evaluation

**Thynaptic TR-2025-25**

**Authors:** Evaluation & Benchmarking Team  
**Division:** Thynaptic Research  
**Date:** 2025-11-24  
**Version:** v1.0.0

---

## Abstract

We present the Cognitive MMLU (C-MMLU) benchmark, a specialized evaluation framework that tests adaptive cognitive layer capabilities beyond knowledge recall. The framework structures scenario-based questions across six cognitive domains: safety and hallucination detection, confidence calibration, self-awareness, emotional memory, predictive cognition, and memory systems. Unlike standard MMLU which measures factual knowledge, C-MMLU evaluates cognitive processing mechanisms, self-awareness, safety validation, and memory integration. The framework routes all test requests through Aurora's complete ACL pipeline, ensuring evaluation reflects production system behavior where requests pass through ten cognitive components before model inference.

---

## Overview

The Cognitive MMLU benchmark addresses the limitation of traditional benchmarks that test isolated model capabilities. Standard MMLU measures knowledge across 57 academic domains but does not evaluate cognitive mechanisms like safety validation, confidence calibration, or emotional memory integration. C-MMLU structures evaluation of adaptive cognitive layers by testing scenario-based questions that require ACL component engagement.

The framework generates questions organized into six cognitive domains, each targeting specific ACL capabilities. Questions include scenario context that simulates workspace state, conversation history, and system metrics. The framework routes all questions through the complete ACL pipeline, collecting metrics on component engagement, confidence scores, safety validation, and cognitive processing accuracy.

---

## Architectural Context

### Cognitive Pipeline Integration

The framework routes all test requests through Aurora's ten-component ACL pipeline:

1. **Intent Classifier:** Categorizes test questions by intent type
2. **Context Engine:** Builds contextual information from scenario context
3. **Workspace Memory:** Retrieves relevant workspace items (simulated in scenarios)
4. **Long-term Memory:** Accesses conversation history and semantic memory
5. **Reasoning Layer:** Routes complex questions to specialized reasoning modules (conditional)
6. **Safety Layer:** Validates responses for hallucinations and factual accuracy
7. **Personality + Style Layer:** Adapts response tone and style
8. **Response Construction:** Builds full prompt with all context layers
9. **Confidence Scoring:** Calculates self-aware confidence metrics
10. **HybridBridge Model Routing:** Selects optimal model (local/cloud/hybrid)

### Test Framework Architecture

```
Question Generator
    ↓
Cognitive MMLU Dataset (CSV)
    ↓
Test Runner
    ↓
Enhanced ACL Client
    ↓
ACL Server (with cognitive test mode)
    ↓
ACL Cognitive Pipeline (10 Components)
    ↓
Model Inference
    ↓
Result Aggregation
    ↓
Analyzer
    ↓
Report Generator
```

### Cognitive Test Domains

**Domain 1: Safety & Hallucination Detection**
- Hallucination detection scenarios
- Fact verification against workspace
- Workspace knowledge claim validation
- False positive/negative detection

**Domain 2: Confidence Calibration**
- High/medium/low confidence scenarios
- Confidence-accuracy correlation
- Recall-based confidence assessment
- Confidence threshold validation

**Domain 3: Self-Awareness**
- Capability recognition questions
- Limitation acknowledgment
- Cognitive health awareness
- Self-diagnostic accuracy

**Domain 4: Emotional Memory**
- Valence understanding
- Tone recognition
- Cross-conversation emotional continuity
- Emotional snapshot accuracy

**Domain 5: Predictive Cognition**
- Focus forecasting
- Drift detection
- Energy trend understanding
- Predictive intervention effectiveness

**Domain 6: Memory Systems**
- Memory Graph theme discovery
- CPS priority ranking
- Context assembly effectiveness
- Cross-conversation pattern recognition

---

## Methodology

### Question Generation

The framework generates scenario-based questions using templates for each cognitive domain. Questions include:

- **Scenario Context:** Simulated workspace state, conversation history, or system metrics
- **Question:** Cognitive processing question requiring ACL component engagement
- **Choices:** Four options (A-D) with one correct answer
- **Metadata:** Required ACL components, expected confidence range, safety check requirement

Questions are organized by domain and subdomain, saved as CSV files in `cognitive_mmlu_dataset/{domain}/{subdomain}.csv`.

### Test Execution Protocol

**Test Flow:**
1. Load cognitive MMLU dataset from directory structure
2. Filter domains (if specified)
3. For each domain/subdomain:
   a. Load questions from CSV
   b. For each question:
      - Format prompt with scenario context
      - Send to ACL pipeline via enhanced client
      - Extract answer, confidence, safety validation, component engagement
      - Record metrics
   c. Calculate domain/subdomain statistics
4. Aggregate results across all domains
5. Generate analysis and report

### Metric Collection

**Accuracy Metrics:**
- Overall accuracy: (correct / total) × 100
- Domain-specific accuracy per cognitive domain
- Subdomain accuracy within each domain
- Component-specific accuracy (where applicable)

**Confidence Metrics:**
- Average confidence scores per domain
- Confidence-accuracy correlation (Pearson)
- Calibration error (Brier score)
- Overconfidence/underconfidence rates
- Confidence distribution (low/medium/high)
- Confidence in expected range rate

**Safety Metrics:**
- Hallucination detection rate
- Fact verification accuracy
- Workspace validation accuracy
- False positive/negative rates
- Safety validation rate

**Self-Awareness Metrics:**
- Capability recognition accuracy
- Limitation acknowledgment rate
- Cognitive health awareness accuracy
- Self-diagnostic accuracy

**Emotional Memory Metrics:**
- Valence accuracy
- Tone recognition accuracy
- Cross-conversation continuity
- Emotional context integration

**Predictive Metrics:**
- Forecast accuracy
- Drift detection accuracy
- Pattern recognition accuracy
- Predictive intervention effectiveness

**Memory System Metrics:**
- Memory Graph theme accuracy
- CPS ranking accuracy
- Context assembly effectiveness
- Cross-conversation pattern recognition

**Component Engagement:**
- Frequency of each ACL component engagement
- Component engagement per domain
- Routing path distribution

### Enhanced ACL Client

The framework extends `AuroraACLClient` with cognitive test-specific request handling:

- **CognitiveACLRequest:** Extended request format with scenario, cognitive domain, required components, workspace context
- **CognitiveACLResponse:** Extended response with safety validation, self-awareness, emotional context, component engagement metadata
- **Metadata Extraction:** Parses response text for safety flags, self-awareness metrics, emotional context

### ACL Server Cognitive Test Mode

The ACL server supports cognitive test mode via request parameters:
- `testMode`: Set to "cognitive_mmlu"
- `cognitiveDomain`: Specifies cognitive domain
- `requiredComponents`: List of required ACL components
- `workspaceContext`: Simulated workspace state

The server enhances app context with cognitive test metadata and extracts cognitive-specific information from responses.

---

## Evaluation Results

*Note: Initial framework implementation complete. Baseline and ACL-enhanced results pending test execution.*

### Framework Capabilities

- **Question Generation:** 27 initial questions across 6 domains, 18 subdomains
- **Test Execution:** Full ACL pipeline integration with cognitive test mode
- **Metric Collection:** Comprehensive metrics across all cognitive domains
- **Analysis:** Automated analysis with correlation, calibration, and domain-specific metrics
- **Reporting:** Markdown report generation with executive summary, domain analysis, and recommendations

### Dataset Structure

```
cognitive_mmlu_dataset/
├── safety/
│   ├── hallucination_detection.csv (3 questions)
│   └── fact_verification.csv (2 questions)
├── confidence/
│   ├── high_confidence.csv (1 question)
│   ├── medium_confidence.csv (1 question)
│   ├── low_confidence.csv (1 question)
│   └── recall_based_confidence.csv (2 questions)
├── self_awareness/
│   ├── cognitive_health.csv (3 questions)
│   ├── capability_questions.csv (1 question)
│   └── limitation_questions.csv (1 question)
├── emotional_memory/
│   ├── valence_understanding.csv (2 questions)
│   ├── tone_recognition.csv (1 question)
│   └── cross_conversation.csv (1 question)
├── predictive/
│   ├── focus_forecasting.csv (2 questions)
│   ├── energy_trends.csv (1 question)
│   └── drift_detection.csv (1 question)
└── memory_systems/
    ├── memory_graph.csv (2 questions)
    ├── cps_ranking.csv (1 question)
    └── context_assembly.csv (1 question)
```

**Total:** 27 questions across 6 domains, 18 subdomains

---

## System Behavior Analysis

### Component Engagement Patterns

The framework tracks which ACL components are engaged for each question type:

- **Safety Domain:** Requires safety layer, workspace memory, confidence scoring
- **Confidence Domain:** Requires confidence scoring, workspace memory
- **Self-Awareness Domain:** Requires cognitive health, confidence scoring
- **Emotional Memory Domain:** Requires workspace memory, emotional continuity
- **Predictive Domain:** Requires predictive reflection, confidence scoring
- **Memory Systems Domain:** Requires memory graph, CPS, context engine

### Routing Distribution

The framework records routing paths (local/cloud/hybrid) and reasoning layer triggers, enabling analysis of how routing decisions affect cognitive performance.

### Confidence Calibration

The framework measures confidence-accuracy correlation to assess whether the system's self-reported confidence aligns with actual performance. High correlation indicates well-calibrated confidence scoring.

### Safety Layer Effectiveness

The framework measures hallucination detection rates, fact verification accuracy, and false positive/negative rates to assess safety layer performance.

---

## Limitations

### Question Coverage

Initial question set (27 questions) provides proof-of-concept but requires expansion for comprehensive evaluation. Target: 50-100 questions per domain.

### Scenario Simulation

Workspace context is simulated in scenario text rather than actual workspace state. Future work should integrate with real workspace data for more accurate evaluation.

### Baseline Comparison

Framework supports baseline testing (direct model inference without ACL) but requires execution to establish baseline metrics for comparison.

### Component Isolation

Framework tests ACL pipeline as integrated system. Isolating individual component contributions requires additional experimental design.

### Emotional Context Extraction

Emotional context extraction relies on text parsing of responses. More robust integration with actual emotional memory systems would improve accuracy.

---

## Forward Research Trajectories

### Question Expansion

- Generate 50-100 questions per domain (target: 300-600 total questions)
- Include adversarial examples for safety testing
- Design edge cases for each cognitive domain
- Validate questions against ACL component specifications

### Real Workspace Integration

- Integrate with actual FocusOS workspace data
- Test with real conversation history and memory graphs
- Evaluate emotional memory with actual emotional snapshots
- Test predictive cognition with real focus session data

### Baseline Establishment

- Run baseline tests (direct model inference without ACL)
- Establish baseline metrics for each cognitive domain
- Compare ACL-enhanced performance against baseline
- Analyze ACL component contributions

### Component Isolation

- Design experiments to isolate individual component contributions
- Measure performance impact of each ACL component
- Identify critical components for each cognitive domain
- Optimize component engagement thresholds

### Cross-Model Evaluation

- Test framework across multiple models (qwen3:1.7b, gemma3:4b, deepseek-r1:1.5b, etc.)
- Compare cognitive performance across models
- Identify model-specific cognitive capabilities
- Analyze routing decisions and model selection

### Continuous Evaluation

- Integrate framework into continuous evaluation pipeline
- Track cognitive performance over time
- Monitor component engagement patterns
- Detect cognitive capability regressions

### Publication and Standardization

- Publish framework as open-source benchmark
- Standardize question format and evaluation metrics
- Enable community contributions and question expansion
- Establish baseline results for comparison

---

**File:** `TR-2025-25_Cognitive_MMLU_Benchmark.md`  
**Related Reports:** TR-2025-07 (MMLU Framework), TR-2025-08 (ACL Architecture)

