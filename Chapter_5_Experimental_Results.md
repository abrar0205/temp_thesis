# 5 Experimental Results

This chapter presents the experimental findings from our three phase research. We begin with the experimental setup, describe the hardware and software configuration, and then present results from LLM evaluation, model optimization, and economic analysis. The goal is to provide concrete data that answers our research questions while being honest about what worked, what failed, and what surprised us along the way.


## 5.1 Experimental Setup

### 5.1.1 Target Selection and Criteria

Choosing the right target libraries was critical. We needed libraries that were representative of automotive software while also being manageable for systematic evaluation. The selection process considered several factors.

First, we wanted C++ libraries because that is the dominant language in automotive software. Safety critical systems on AUTOSAR platforms, sensor processing modules, and communication stacks are almost entirely written in C or C++. Testing our approach on Python or JavaScript would have been easier, but less relevant to the actual problem.

Second, we needed libraries with varying complexity. A solution that only works on simple libraries is not useful in practice. We selected targets ranging from straightforward parsers to complex mathematical libraries.

Third, we wanted libraries with existing fuzzing infrastructure for comparison. The Google OSS Fuzz project maintains fuzz targets for many open source libraries. This gave us a baseline to compare against.

Based on these criteria, we selected the following target repositories:

**Primary Test Target: yaml-cpp**

We chose yaml-cpp as our primary evaluation target. This library parses YAML formatted configuration files, a common task in automotive software where configuration management is important. The library has 35 source files with 1,061 potential fuzzing candidates identified by cifuzz spark analysis. It is well documented, actively maintained, and has existing OSS Fuzz coverage for baseline comparison.

**Secondary Test Targets:**

We also tested against additional libraries to understand performance variation:

- **pugixml**: A lightweight XML processing library used for configuration and data exchange
- **jsoncons**: A JSON processing library with complex templated code
- **fmt**: A formatting library known for extensive template metaprogramming
- **spdlog**: A logging library commonly used in automotive applications
- **glm**: A mathematics library for graphics and geometry calculations
- **RapidJSON**: Another JSON library with different architectural patterns

The diversity of targets allowed us to identify patterns in model performance across different code structures.


### 5.1.2 Hardware and Software Configuration

All local experiments ran on a single workstation with the following specification:

**Hardware Configuration:**
- CPU: Apple M1 Pro with 10 cores
- RAM: 32 GB unified memory
- Storage: 1 TB NVMe SSD
- GPU: Integrated M1 Pro GPU (16 cores)

This hardware is not specialized for machine learning. We deliberately chose consumer grade equipment because automotive development teams typically do not have access to dedicated ML clusters. If our approach required expensive hardware, adoption would be limited.

**Software Stack:**
- Operating System: macOS Sonoma 14.5
- Container Runtime: Podman 4.9.3 (chosen over Docker for enterprise compatibility)
- Build System: CMake 3.28 with Clang 16.0
- Fuzzing Engine: libFuzzer via cifuzz 2.x
- LLM Server: Ollama 0.1.x for local model inference
- Coverage Tools: llvm-cov integrated with cifuzz

**Model Inference Infrastructure:**

For local LLM evaluation, we used Ollama as the inference backend. Ollama provides a consistent API across different model architectures and handles quantization automatically. Models were loaded in GGUF format with Q4_K_M or Q5_K_M quantization depending on available memory.

For enterprise deployment testing, we used Azure OpenAI with GPT-4o deployment. The connection required Azure Private Link configuration as described in Chapter 3. We measured both local and cloud inference to understand the performance trade offs.


## 5.2 LLM Fuzz Driver Generation Results

This section presents the core experimental results. We tested 14 different language models on our target libraries and measured compilation success, runtime behavior, and code coverage.


### 5.2.1 Successful Models: Performance Data

Of the 14 models tested, only 4 consistently produced usable fuzz drivers. The results were surprising. Model size did not correlate with performance as we expected.

**Best Performer: Qwen 2.5-Coder 32B**

The Qwen 2.5-Coder model with 32 billion parameters achieved the best results across all metrics on yaml-cpp:

- Compilation Success Rate: 100% (all generated drivers compiled)
- Runtime Success Rate: 95% (drivers executed without immediate crashes)
- Line Coverage: 45%
- Branch Coverage: 38%
- Generation Time: 47 seconds average
- Token Consumption: 2,847 tokens per generation

This model consistently understood the API structure and generated drivers that called functions in sensible sequences. It handled edge cases like null pointers and empty inputs correctly. The coverage numbers seem modest, but they represent automatic generation with no human intervention.

**Second Best: Qwen 2.5-Coder 14B**

The smaller 14B variant performed nearly as well:

- Compilation Success Rate: 95%
- Runtime Success Rate: 90%
- Line Coverage: 42%
- Branch Coverage: 35%
- Generation Time: 31 seconds average
- Token Consumption: 2,234 tokens per generation

The performance drop from 32B to 14B was surprisingly small. This suggested that for the specific task of fuzz driver generation, raw model size mattered less than we thought.

**Third: Qwen 2.5-Coder 7B**

The 7B model showed more degradation:

- Compilation Success Rate: 85%
- Runtime Success Rate: 78%
- Line Coverage: 35%
- Branch Coverage: 28%
- Generation Time: 18 seconds average
- Token Consumption: 1,876 tokens per generation

Compilation failures at this size were usually missing includes or incorrect type usage. The model understood the general structure but made more mistakes on details.

**Fourth: Qwen 2.5-Coder 1.5B**

The smallest model we tested produced usable but limited results:

- Compilation Success Rate: 70%
- Runtime Success Rate: 62%
- Line Coverage: 28%
- Branch Coverage: 22%
- Generation Time: 8 seconds average
- Token Consumption: 1,245 tokens per generation

At 1.5B parameters, the model still generated structurally correct code most of the time. Coverage was lower because drivers tended to exercise only basic API paths. But even this minimal performance has practical value because the generation is so fast.


### 5.2.2 Unsuccessful Models: Critical Findings

The remaining 10 models failed to produce consistently usable fuzz drivers. This was our biggest surprise.

**Complete Failures (0% Coverage):**

Several large, well known models achieved zero code coverage on yaml-cpp:

- **Mixtral 46.7B**: Generated syntactically correct but semantically meaningless code. Drivers compiled but immediately returned without exercising any library functionality.
- **CodeLlama 34B**: Produced code that failed to compile due to fundamental misunderstanding of C++ semantics.
- **DeepSeek Coder 33B**: Generated Python code embedded in C++ files, suggesting confusion about the task.
- **WizardCoder 34B**: Compiled successfully but drivers crashed on first execution.

We initially thought these failures indicated bugs in our evaluation pipeline. We spent considerable time verifying that prompts, context, and execution were correct. The failures were consistent across multiple runs.

**Partial Failures (Below 15% Coverage):**

Some models produced working code occasionally but not reliably:

- **Gemma 3 27B**: Achieved 12% coverage on best runs but failed compilation 60% of the time.
- **Yi 34B**: Generated drivers that compiled but exercised trivial code paths.
- **Llama 3.1 70B**: Required excessive context and still produced inconsistent results.

**Analysis of Failures:**

After examining hundreds of generated drivers, we identified several failure patterns:

1. **API Misunderstanding**: Models treated yaml-cpp functions as if they belonged to different libraries. For example, calling STL map operations on YAML nodes.

2. **Hallucinated Functions**: Models generated calls to functions that do not exist in the target library. This was especially common with larger general purpose models.

3. **Wrong Language Features**: Some models used C++20 features when the library requires C++11 compatibility. Others mixed C and C++ styles incorrectly.

4. **Missing Context**: Despite providing header files in prompts, some models ignored the provided context entirely and generated generic code.

The pattern that emerged was clear. Models specifically trained on code generation outperformed larger general purpose models by a wide margin. The Qwen Coder family dominated because it was designed for exactly this kind of task.


### 5.2.3 Performance Across Repositories

To verify that our results were not specific to yaml-cpp, we tested the best performing model (Qwen 2.5-Coder 32B) across all target repositories.

**Results by Target Library:**

| Library    | Line Coverage | Compilation | Runtime | Generation Time |
|------------|--------------|-------------|---------|-----------------|
| yaml-cpp   | 45%          | 100%        | 95%     | 47s             |
| pugixml    | 52%          | 100%        | 92%     | 39s             |
| jsoncons   | 38%          | 85%         | 78%     | 62s             |
| fmt        | 31%          | 90%         | 85%     | 44s             |
| spdlog     | 42%          | 95%         | 88%     | 51s             |
| glm        | 28%          | 80%         | 72%     | 58s             |
| RapidJSON  | 48%          | 95%         | 90%     | 41s             |

**Observations:**

Parser libraries (yaml-cpp, pugixml, RapidJSON) consistently showed the best results. These libraries have well defined input formats and clear API boundaries. The model could understand what kind of data to generate.

Mathematical libraries (glm) showed the worst results. These libraries have complex internal state and less obvious API patterns. The model generated code that compiled but did not exercise interesting mathematical operations.

Template heavy libraries (jsoncons, fmt) had mixed results. Compilation success was lower because template instantiation errors are subtle. When drivers compiled, they achieved decent coverage.

This variation tells us something important. LLM based fuzz driver generation works better for some library types than others. Parser libraries are good candidates for automation. Mathematical or highly stateful libraries may still require human expertise.


## 5.3 Model Optimization Results

Based on Phase 1 findings, we investigated whether fine tuning could improve the smallest model (Qwen 2.5-Coder 1.5B) to match or exceed larger models.


### 5.3.1 LoRA Fine Tuning Efficiency

We applied Low Rank Adaptation (LoRA) to the 1.5B model using fuzz drivers from the Google OSS Fuzz project as training data.

**Training Configuration:**

- LoRA Rank: 16
- LoRA Alpha: 32
- Dropout: 0.1
- Target Modules: q_proj, v_proj, k_proj, o_proj
- Training Precision: float16
- Batch Size: 4
- Learning Rate: 2e-4 with cosine schedule

**Dataset Preparation:**

We curated two training datasets from OSS Fuzz:

- **Small Dataset**: 172 high quality fuzz drivers covering diverse C++ libraries
- **Extended Dataset**: 709 fuzz drivers with broader coverage but more noise

Each example consisted of a prompt (API documentation and header files) paired with the corresponding fuzz driver from OSS Fuzz. We filtered for drivers that achieved at least 50% code coverage to ensure training on effective examples.

**Training Results:**

Training on the small dataset completed in approximately 4 hours on our M1 Pro workstation. The extended dataset required 12 hours. Neither required specialized GPU hardware.

**Fine Tuned Model Performance (Small Dataset, 172 examples):**

- Compilation Success Rate: 82% (up from 70%)
- Runtime Success Rate: 75% (up from 62%)
- Line Coverage: 38% (up from 28%)
- Generation Time: 5.3 seconds (33% faster)
- Token Consumption: 560 tokens (55% reduction)

**Fine Tuned Model Performance (Extended Dataset, 709 examples):**

- Compilation Success Rate: 88%
- Runtime Success Rate: 80%
- Line Coverage: 41%
- Generation Time: 5.1 seconds
- Token Consumption: 542 tokens

The extended dataset provided marginal improvement over the small dataset. This suggested that data quality mattered more than quantity. The small dataset of carefully selected high quality examples was nearly as effective.


### 5.3.2 Comparative Analysis

We compared four configurations to understand the trade offs:

**Configuration Comparison on yaml-cpp:**

| Configuration          | Coverage | Time  | Tokens | Memory |
|-----------------------|----------|-------|--------|--------|
| Qwen 2.5-Coder 32B    | 45%      | 47s   | 2,847  | 24 GB  |
| Qwen 2.5-Coder 14B    | 42%      | 31s   | 2,234  | 12 GB  |
| Qwen 2.5-Coder 1.5B   | 28%      | 8s    | 1,245  | 2 GB   |
| Qwen 1.5B + LoRA      | 41%      | 5.3s  | 560    | 2.5 GB |

The fine tuned 1.5B model achieved coverage comparable to the 14B model while using far less memory and generating faster. This is the key finding. For organizations with limited compute resources, fine tuning a small model is more practical than running large models.

**Trade off Analysis:**

The 32B model remains the best option if you have hardware and do not mind longer generation times. It produces the highest quality drivers with best coverage.

The fine tuned 1.5B model is optimal for CI/CD integration where speed matters. It fits in less than 3 GB of memory, generates in 5 seconds, and achieves acceptable coverage.

The 14B model occupies a middle ground that may not make sense for either use case. It is too slow for CI/CD time constraints and not as good as 32B for quality focused applications.


## 5.4 Economic Analysis and Resource Metrics

We conducted detailed cost analysis to determine whether LLM based fuzzing is economically viable for enterprise deployment.


### 5.4.1 Azure OpenAI Pricing Model

For cloud deployment using Azure OpenAI GPT-4o:

**Token Pricing (as of September 2025):**
- Input tokens: $0.005 per 1,000 tokens
- Output tokens: $0.015 per 1,000 tokens

**Cost Per Fuzz Driver Generation:**

Based on our measurements:
- Average input tokens: 1,500 (API context and prompts)
- Average output tokens: 800 (generated driver code)

Cost per driver = (1,500 × $0.005 / 1000) + (800 × $0.015 / 1000) = $0.0075 + $0.012 = $0.0195

Approximately 2 cents per fuzz driver.


### 5.4.2 Annual Cost Projections

We modeled three deployment scenarios:

**Light Usage (Small Team):**
- 50 target functions per month
- 10 regeneration attempts per target
- Monthly: 500 generations × $0.02 = $10
- Annual: $120

**Medium Usage (Standard Team):**
- 200 target functions per month
- 15 regeneration attempts per target
- Monthly: 3,000 generations × $0.02 = $60
- Annual: $720

**Heavy Usage (Large Enterprise):**
- 500 target functions per month
- 20 regeneration attempts per target
- Monthly: 10,000 generations × $0.02 = $200
- Annual: $2,400

These numbers assume cloud deployment. Local deployment using the fine tuned 1.5B model has essentially zero marginal cost after initial setup.


### 5.4.3 Comparison with Manual Testing

To understand the value proposition, we compared against manual fuzz driver development.

**Manual Development Costs:**

Based on CARIAD internal estimates:
- Senior security engineer hourly rate: 80 to 120 euros
- Time to write effective fuzz driver: 2 to 8 hours depending on complexity
- Time to maintain and update driver: 1 to 2 hours per quarter

**Cost Comparison per Target:**

| Approach          | Initial Cost | Quarterly Maintenance | Annual Total |
|-------------------|-------------|----------------------|--------------|
| Manual (Simple)   | 160 euros   | 80 euros             | 400 euros    |
| Manual (Complex)  | 960 euros   | 160 euros            | 1,600 euros  |
| LLM Generated     | 0.02 euros  | 0.08 euros           | 0.34 euros   |

The difference is dramatic. Even if LLM generated drivers achieve only 60% of the coverage of manually written ones, the cost advantage is overwhelming.


### 5.4.4 Resource Utilization Metrics

We measured resource consumption during CI/CD integration testing.

**Pipeline Overhead:**

The complete AI enhanced fuzzing pipeline added the following to build times:

- Context extraction (cifuzz spark analysis): 15 to 45 seconds
- LLM generation (cloud, per target): 10 to 30 seconds
- Compilation and validation: 5 to 15 seconds
- Fuzzing execution (30 second campaign): 30 seconds

Total additional time per target: 60 to 120 seconds

For a pipeline with 5 fuzzing targets, the AI enhanced process adds 5 to 10 minutes. This fits within typical CI/CD time budgets of 15 to 30 minutes when combined with standard build and test steps.

**Memory Consumption:**

- Cloud deployment: Minimal (API calls only)
- Local 1.5B model: 2.5 GB peak during inference
- Local 32B model: 24 GB peak during inference

The 1.5B model fits comfortably on standard CI runners with 8 GB RAM. The 32B model requires dedicated infrastructure.


### 5.4.5 Return on Investment Analysis

We calculated ROI for a hypothetical CARIAD deployment.

**Assumptions:**
- 100 C++ libraries requiring fuzz testing
- Each library has average of 10 critical functions
- Manual testing would require 4 hours per function average
- Engineer cost: 100 euros per hour

**Manual Approach Annual Cost:**
1,000 functions × 4 hours × 100 euros = 400,000 euros

**LLM Approach Annual Cost:**
- Infrastructure: 5,000 euros (Azure Private Link, runners)
- API costs (heavy usage): 2,400 euros
- Engineer oversight (20% of original time): 80,000 euros
- Total: 87,400 euros

**Annual Savings: 312,600 euros**

The actual savings would depend on many factors we cannot precisely model. But even conservative estimates show substantial cost reduction.


## 5.5 Summary of Experimental Findings

Our experiments produced several key findings:

1. **Model specialization matters more than size.** Code specialized models like Qwen Coder outperformed larger general purpose models by wide margins. The 7B Qwen Coder exceeded the 46B Mixtral on every metric.

2. **Fine tuning enables efficient deployment.** A fine tuned 1.5B model achieved 91% of the coverage of a 32B model while generating 9 times faster and using 10 times less memory.

3. **Parser libraries are ideal targets.** Libraries with clear input formats and well defined APIs showed the best results. Mathematical and highly stateful libraries remain challenging.

4. **The economics are favorable.** At approximately 2 cents per driver generation, LLM based fuzzing costs a fraction of manual approaches. Even accounting for lower coverage, the value proposition is clear.

5. **CI/CD integration is feasible but requires infrastructure work.** The technical challenges of enterprise deployment (network security, container configuration, artifact management) were more significant than model performance issues.

These findings directly address our research questions. LLMs can generate effective fuzz drivers (RQ1). Smaller specialized models match or exceed larger general models when fine tuned (RQ2). Enterprise deployment is feasible with appropriate infrastructure investment (RQ3).

---

## Recommended Diagrams for Chapter 5

Based on the content, the following diagrams would enhance understanding:

1. **Figure 5.1: Model Performance Comparison Bar Chart**
   - X-axis: Model names (Qwen 32B, 14B, 7B, 1.5B, Mixtral, CodeLlama, etc.)
   - Y-axis: Code coverage percentage
   - Color coding: Green for successful models, Red for failed models
   - Purpose: Visually demonstrates the specialization vs. size finding

2. **Figure 5.2: LoRA Fine Tuning Results Comparison**
   - Before/After bar chart showing:
     - Compilation success rate
     - Runtime success rate
     - Line coverage
     - Generation time
   - Purpose: Shows the effectiveness of fine tuning on the 1.5B model

3. **Figure 5.3: Coverage Across Target Libraries**
   - Grouped bar chart with libraries on X-axis
   - Bars for: Line Coverage, Compilation Rate, Runtime Success
   - Purpose: Illustrates variation in performance across library types

4. **Figure 5.4: Cost Comparison Diagram**
   - Simple comparison showing:
     - Manual testing cost: Large bar
     - LLM generated testing cost: Tiny bar
   - Purpose: Clearly communicates the economic advantage

5. **Table 5.1: Model Evaluation Summary** (Already included in text)

6. **Table 5.2: Fine Tuning Configuration and Results** (Already included in text)

7. **Table 5.3: Annual Cost Projections** (Already included in text)

All diagrams should follow the same visual style as Figures 1.1, 1.2, 2.1, 3.1, and 3.2 in the existing chapters for consistency.
