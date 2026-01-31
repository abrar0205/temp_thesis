# 4 Implementation

This chapter describes how we turned the methodology from Chapter 3 into working code and systems. The implementation happened in three phases, each building on the previous one. We explain the tools we chose, the specific steps we followed, and the problems we solved along the way.

## 4.1 Toolchain Selection and Rationale

Before writing any code, we needed to select the right tools. This decision was not straightforward because we had constraints from multiple directions. The university wanted reproducible research. CARIAD needed enterprise compatibility. And we wanted tools that would work on the hardware we had available.

### 4.1.1 Fuzzing Infrastructure

For the fuzzing engine itself, we chose libFuzzer combined with Clang sanitizers. This was not a controversial choice. LibFuzzer is the industry standard for coverage-guided fuzzing in C and C++ projects. It integrates directly with the LLVM toolchain, which means compilation and fuzzing use the same infrastructure. Google uses libFuzzer for OSS-Fuzz, so the documentation and community support are excellent.

We compiled all targets with AddressSanitizer (ASan) and UndefinedBehaviorSanitizer (UBSan) enabled. ASan catches memory errors like buffer overflows, use-after-free, and memory leaks. UBSan catches undefined behavior like signed integer overflow and null pointer dereferences. These sanitizers add runtime overhead, but for security testing the overhead is acceptable. Finding bugs is more important than running fast.

For integration with our build systems, we used cifuzz from Code Intelligence. Cifuzz wraps libFuzzer and provides additional functionality like corpus management, coverage reporting, and build system integration. The critical feature for our work was cifuzz spark, which handles the LLM integration. Spark takes a CMake target as input, extracts API information automatically, sends it to the configured LLM, and produces a fuzz driver ready for execution.

Our choice of cifuzz was deliberate. We evaluated several alternatives including Google's ClusterFuzz and standalone libFuzzer setups. Cifuzz offered the best balance between ease of use and enterprise features. Its spark module specifically addressed our core research question by providing a clean interface between the fuzzing infrastructure and LLM backends.

### 4.1.2 LLM Infrastructure

We needed to run models locally for several reasons. First, we wanted to evaluate many different models, and running them through paid APIs would have been expensive. Second, some models are open-source and only available for local deployment. Third, for the enterprise integration phase, we needed to understand how local deployment might work as an alternative to cloud APIs.

We selected Ollama as our primary local inference server. Ollama simplifies model management. Installing a new model is one command: `ollama pull qwen2.5-coder:32b`. The server exposes an OpenAI-compatible API, which meant cifuzz spark worked without modification. Ollama handled quantized models well, which let us run larger models on the available hardware.

For models not available through Ollama, we used llama.cpp directly. This required more manual configuration but gave us flexibility. Some models from Hugging Face came as split GGUF files that needed merging before use. We handled this with the following command:

```bash
llama-gguf-split --merge qwen2.5-coder-14b-instruct-q5_k_m-00001-of-00002.gguf \
                 qwen2.5-coder-14b-instruct-q5_k_m.gguf
```

This merging step added complexity to our workflow but was necessary for the larger models we wanted to evaluate.

For cloud deployment in the enterprise phase, we configured Azure OpenAI. Azure provided GPT-4o through a managed endpoint. The authentication used API keys stored as GitHub secrets. We configured the following environment variables:

```bash
export CIFUZZ_LLM_API_URL="https://ny4mdomain.openai.azure.com"
export CIFUZZ_LLM_AZURE_DEPLOYMENT_NAME="gpt-4o"
export CIFUZZ_LLM_API_TOKEN="<api-key>"
export CIFUZZ_LLM_MAX_TOKENS=40000
```

A token limit of 40000 was necessary because fuzz driver generation requires substantial context. Prompts include header files, example code, and detailed instructions. Smaller token limits caused truncation and worse results.

### 4.1.3 Development Environment

Local development happened on a MacBook Pro with an M1 Pro chip. This created some challenges. Docker Desktop had performance issues on Apple Silicon, so we switched to Podman. Podman provided better Linux container compatibility, which mattered because the enterprise CI/CD runners used Linux.

```bash
brew install podman
podman machine init --cpus 4 --memory 8192
podman machine start
```

I should note that transitioning from Docker to Podman was not entirely smooth. Some scripts assumed Docker-specific behavior. We spent about a week debugging container networking issues before everything worked reliably.

For the enterprise environment, we used CARIAD's internal GitHub Enterprise instance with self-hosted runners. These runners used Buildah instead of Docker for container operations. Buildah is daemonless, which fits better with security policies that restrict long-running processes.

Our build toolchain used CMake for project configuration and Clang for compilation. We required CMake version 3.5 or higher because cifuzz integration depends on features from that version. CMake configuration for enabling fuzzing was straightforward:

```cmake
cmake_minimum_required(VERSION 3.5)
cmake_policy(VERSION 3.5)

project(FuzzTarget)

find_package(cifuzz NO_SYSTEM_ENVIRONMENT_PATH)
enable_fuzz_testing()
add_subdirectory(cifuzz-spark)
```

We also needed to update CMakeLists.txt files in subdirectories. Several target libraries had older CMake configurations that conflicted with our setup. The fix was consistent across all subdirectories:

```cmake
cmake_minimum_required(VERSION 3.10...3.22)
cmake_policy(VERSION 3.10...3.22)
```

**[Figure 4.1: Toolchain architecture diagram showing the relationships between components. Local Development (Mac with Podman, Ollama) connects to Cifuzz Spark, which connects to either Local LLM or Azure OpenAI. Output flows to LibFuzzer with ASan/UBSan, producing Coverage Reports and Crash Reports.]**

## 4.2 Phase 1: Local LLM Evaluation Setup

Phase 1 focused on understanding which models could actually generate useful fuzz drivers. This phase directly addresses **RQ1** (Can LLMs generate compilable, executable fuzz drivers?) and **RQ2** (Does model size determine quality?). We did not assume any model would work well. Literature suggested mixed results for LLM code generation, and fuzz drivers are a specialized task.

### 4.2.1 Benchmarked Models

We evaluated 14 models spanning a wide range of sizes and architectures. The selection included both general-purpose models and code-specialized variants. Our goal was to understand whether model size, training data, or architecture had the biggest impact on fuzz driver quality.

**Large Models (25-35 billion parameters):**
- Qwen 2.5-Coder 32B
- DeepSeek Coder 33B
- CodeLlama 34B Instruct
- WizardCoder 34B
- Yi 34B

**Extra Large Models (above 40 billion parameters):**
- Mixtral 46.7B

**Medium Models (7-14 billion parameters):**
- Qwen 2.5-Coder 7B and 14B
- DeepSeek Coder V2 Lite 16B
- Gemma 3 9B

**Small Models (below 7 billion parameters):**
- Qwen 2.5-Coder 1.5B
- Gemma 3 4B
- Phi 4 3.8B

Code-specialized models like Qwen Coder and DeepSeek Coder were trained specifically on programming tasks. General-purpose models like Mixtral were included to test whether raw scale could compensate for lack of specialization. We wanted to know: does a 46B general model outperform a 7B code-specialized model? Results, discussed in Chapter 5, were surprising: general-purpose models like Mixtral (46.7B) and CodeLlama (34B) achieved 0% code coverage, while the smaller specialized Qwen 2.5-Coder (32B) consistently achieved 45% line coverage.

Installing these models required significant disk space. The larger models exceeded 20GB each in their quantized forms. We used 4-bit and 5-bit quantization to fit models into available VRAM while maintaining reasonable output quality.

### 4.2.2 Target Repositories

We needed consistent test targets to compare models fairly. Our selection included open-source C++ libraries with varying complexity:

**Primary Target: yaml-cpp**

yaml-cpp is a YAML parsing library with approximately 35 source files and over 1000 API candidates that could potentially be fuzzed. We chose it as our primary benchmark because it has enough complexity to challenge the models while being well documented and actively maintained. The library also has existing fuzz targets in OSS-Fuzz, giving us a baseline for comparison.

**Secondary Targets:**

- **RapidJSON**: A JSON parsing library widely used in performance-critical applications. We had some initial difficulties with RapidJSON's CMake configuration, but resolved them by re-downloading a clean version of the source.
- **pugixml**: XML processing library with similar parsing challenges to yaml-cpp.
- **fmt**: Text formatting library with different API patterns than the parsers.
- **spdlog**: Logging library with many configuration options.
- **jsoncons**: Another JSON library for comparison with RapidJSON.

These libraries cover different domains but share the characteristic of being C++ with clean public APIs. They all have existing fuzz targets in OSS-Fuzz, which gave us baselines for comparison.

We also tested on several CARIAD internal libraries, though we cannot report those results in detail due to confidentiality constraints. The patterns we observed on open-source libraries generally held for the internal ones as well.

### 4.2.3 Evaluation Environment

Hardware for local evaluation was a workstation with sufficient GPU memory for running 32B parameter models with 4-bit quantization. Larger models like Mixtral 46.7B required offloading to system RAM, which slowed inference.

Each model's evaluation followed these steps:

1. Start the model server (Ollama or llama.cpp)
2. Configure cifuzz spark environment variables
3. Run `cifuzz spark` on the target library
4. Attempt to compile the generated driver
5. If compilation succeeded, run the fuzzer for a fixed duration (60 seconds)
6. Record coverage metrics using llvm-cov

We ran each model three times on yaml-cpp to check consistency. Some models produced different outputs on identical prompts, so multiple runs were necessary for reliable results. This variability was itself an interesting finding that we discuss further in Chapter 5.

A full evaluation cycle for one model on one target took approximately 10-15 minutes. With 14 models and 6 primary targets, plus multiple runs for consistency checking, the complete Phase 1 evaluation required about two weeks of continuous testing.

**[Figure 4.2: Evaluation pipeline flowchart. Start with "Model Server Running", then "cifuzz spark generates driver", then decision diamond "Compiles?", if No record failure and stop, if Yes continue to "Run Fuzzer (60s)", then "Record Coverage with llvm-cov", finally "Store Results".]**

## 4.3 Phase 2: Model Optimization with LoRA Fine-Tuning

After the initial evaluation showed promising results from Qwen 2.5-Coder models, we investigated whether fine-tuning could improve performance further. This phase addresses **RQ2** more directly: can smaller, fine-tuned models outperform larger general-purpose models? The goal was to see if a small model, specifically trained for fuzz driver generation, could match or exceed larger ones.

### 4.3.1 Training Data Preparation

Most important for fine-tuning is the training data. High-quality examples of fuzz drivers paired with source code were essential.

To build our dataset, we extracted training examples from OSS-Fuzz. Google has fuzz-tested hundreds of open-source projects through OSS-Fuzz, and the fuzz drivers are publicly available. We focused on C and C++ drivers because those matched our target domain.

Our extraction process involved four steps:

1. Clone the OSS-Fuzz repository
2. Identify projects with C/C++ fuzz drivers
3. For each driver, extract the header files it includes
4. Create input/output pairs: headers as input, driver as expected output

We created two datasets:
- **Small dataset**: 172 examples, focusing on quality over quantity
- **Extended dataset**: 709 examples, broader coverage but more noise

These examples included drivers for libraries like FreeType, LibPNG, SQLite, and zlib, covering a range of API styles from simple functions to complex object-oriented interfaces. We deliberately included diverse examples to help the model generalize rather than memorize specific patterns.

Quality control was important. We manually reviewed a sample of the extracted drivers to ensure they were actually functional and not stub implementations. About 15% of initially extracted drivers were discarded because they were incomplete or contained obvious errors.

### 4.3.2 LoRA Configuration and Training

We used Low-Rank Adaptation (LoRA) rather than full fine-tuning. Full fine-tuning updates all model weights, which requires enormous compute resources. LoRA adds small trainable matrices alongside the frozen base model. This reduces memory requirements by a large margin while achieving similar results on specialized tasks.

Our LoRA configuration:

```python
# LoRA Configuration
LORA_RANK = 16          # Rank controls adaptation complexity
LORA_ALPHA = 32         # Alpha is the scaling factor
DROPOUT = 0.1           # Prevents overfitting
TARGET_MODULES = ["q_proj", "v_proj", "k_proj", "o_proj"]
DEVICE = "auto"         # Uses optimal hardware
DTYPE = "float16"       # Memory-efficient precision
```

A rank of 16 was a balance between adaptation capacity and training speed. Higher ranks can capture more complex patterns but require more memory and training time. We tested ranks of 8, 16, and 32. Rank 16 gave the best results for our dataset size.

We trained on the Qwen 2.5-Coder 1.5B base model. This is a small model that runs quickly on modest hardware. We chose this size specifically because we wanted to test whether fine-tuning could make small models competitive with larger ones.

Training used standard supervised fine-tuning:
- Input: Header files and fuzzing instructions
- Target: Complete fuzz driver
- Learning rate: 2e-5 with cosine decay
- Epochs: 3
- Batch size: 4 (with gradient accumulation)

### 4.3.3 Fine-Tuned Model Evaluation

After training, we evaluated the fine-tuned models on the same yaml-cpp benchmark used in Phase 1. The results showed that fine-tuning provided significant efficiency improvements:

- **Generation time reduced by 33%** compared to the base model
- **Token usage reduced by 55%** compared to the base model
- Coverage remained comparable to the larger Qwen 2.5-Coder 32B model, which achieved 45% line coverage

Fine-tuned models generated more concise code. They learned the structure of fuzz drivers and did not waste tokens on unnecessary explanations or boilerplate. These efficiency gains meant that generating each driver took substantially less time and fewer API tokens, which directly impacts operational costs.

Interestingly, more training data did not always help. The 709-example dataset included some lower-quality drivers, and this may have hurt the model's output. Quality matters more than quantity for fine-tuning specialized tasks. This finding aligns with recent research on data quality in LLM training.

We also tested the fine-tuned models on RapidJSON and pugixml to check generalization. Performance was similar to yaml-cpp, suggesting the fine-tuning improved general fuzz driver generation rather than overfitting to specific libraries.

**[Figure 4.3: Bar chart comparing fine-tuning efficiency gains: 33% reduction in generation time, 55% reduction in token usage, while maintaining comparable coverage to larger models.]**

## 4.4 Phase 3: Enterprise CI/CD Integration

This final phase moved from controlled experiments to real-world deployment, addressing **RQ3**: Can LLM-assisted fuzz driver generation be integrated into secure enterprise CI/CD pipelines? I should note that this is where things got complicated. Academic research happens in ideal conditions. Enterprise deployment happens in conditions dictated by security policies, legacy infrastructure, and organizational constraints.

### 4.4.1 Architectural Challenges

Our first attempt at integration seemed straightforward. We had a working pipeline locally. Enterprise runners used similar Linux environments. How hard could it be?

Very hard, it turned out.

**Challenge 1: Network Isolation**

CARIAD operates with a zero-trust security model. CI/CD runners cannot access the public internet by default. Every external connection requires explicit approval. This makes sense for security, but it creates problems for AI integration.

Our initial pipeline tried to call Azure OpenAI during the build. The request never reached the Azure endpoint. It was blocked by the firewall before leaving the network.

The error message was cryptic:
```
Error: fatal: unable to access 'https://git.hub.vwgroup.com/...': 
Failed to connect to 127.0.0.1 port 9000 after 0 ms: Couldn't connect to server
```

We spent considerable time debugging this. The proxy configuration we set for Azure was interfering with internal Git operations. The runner was trying to send Git traffic through a proxy that did not exist.

**Challenge 2: Container Networking**

Runners use Buildah containers for isolation. Each build step runs inside a container with its own network namespace. Even if the host could reach external services, the container might not be able to.

We tried several approaches:

Host network mode:
```bash
buildah run --network host -v $(pwd):/project $containerid /bin/bash -c "cifuzz spark target"
```
This gave the container the host's network access. Still blocked by the firewall.

Custom DNS entries:
```bash
buildah run --add-host ny4mdomain.openai.azure.com:${OPENAI_IP} ...
```
Did not help because the issue was firewall policy, not DNS resolution.

**Challenge 3: Credential Management**

Even if network connectivity worked, we needed to securely pass API keys to the container. GitHub Actions secrets provided the storage, but injecting them into Buildah containers required explicit environment variable passing:

```bash
buildah run \
  --env CIFUZZ_LLM_API_URL \
  --env CIFUZZ_LLM_AZURE_DEPLOYMENT_NAME \
  --env CIFUZZ_LLM_MAX_TOKENS \
  --env CIFUZZ_LLM_API_TOKEN \
  -v $(pwd):/$PROJECT_FOLDER ${{ env.containerid }} \
  /usr/bin/bash -c "cd /project/build && cifuzz spark target_name"
```

Each environment variable had to be passed explicitly. Missing one caused silent failures where the LLM call would not work but the pipeline continued without clear error messages.

### 4.4.2 Self-Hosted Runner Solution

After weeks of debugging network issues, we concluded that the standard enterprise firewall would not allow direct Azure OpenAI access. The solution required infrastructure changes beyond what we could do ourselves.

**Azure Private Link Architecture**

Our approved solution used Azure Private Link. This creates a private endpoint for Azure services that appears inside the corporate network. Traffic to the LLM endpoint never crosses the public internet.

Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│  CARIAD Internal Network                                        │
├─────────────────────────────────────────────────────────────────┤
│  CI/CD Runners (auto-buildah-base)                             │
│       │                                                         │
│       ▼                                                         │
│  Azure Private Link Endpoint                                   │
│       │ (appears as internal IP)                               │
│       ▼                                                         │
├─────────────────────────────────────────────────────────────────┤
│  Azure OpenAI Service (Microsoft infrastructure)               │
└─────────────────────────────────────────────────────────────────┘
```

From the runner's perspective, the Azure OpenAI endpoint has an internal IP address. The firewall allows internal traffic, so the LLM calls succeed.

Setting up Private Link required coordination with the infrastructure team. The process took approximately one month:

1. Request submission with business justification
2. Security review of the use case
3. Azure subscription configuration
4. DNS updates for private endpoint resolution
5. Network security group rules
6. Testing and validation

We prepared a detailed business case including cost analysis (approximately €73 to €1,452 annually depending on usage) and security risk assessment. This documentation was essential for getting approval.

**Working Pipeline Configuration**

Once Private Link was operational, the pipeline worked reliably. The final GitHub Actions workflow:

```yaml
name: fuzz
env:
  PROJ_NAME: "INIFUZZ"
  VERSION: "0.1"
  ARTIFACTORY_CORPUS_URL: "https://jfrog.hub.vwgroup.com/artifactory/cifuzz-generic-internal/"

jobs:
  fuzz:
    runs-on: [auto-buildah-base]
    env:
      JFROG_API_KEY: ${{ secrets.JFROG_API_KEY }}
      ARTIFACTORY_USER: ${{ secrets.ARTIFACTORY_USER }}
      PROJECT_FOLDER: "/project/"
      CIFUZZ_LLM_API_URL: "https://ny4mdomain.openai.azure.com"
      CIFUZZ_LLM_AZURE_DEPLOYMENT_NAME: "gpt-4o"
      CIFUZZ_LLM_MAX_TOKENS: "40000"
      CIFUZZ_LLM_API_TOKEN: ${{ secrets.CIFUZZ_LLM_API_TOKEN }}
      
    steps:
      - uses: actions/checkout@v3
      
      - name: Login to JFrog
        run: buildah login -u $ARTIFACTORY_USER -p $JFROG_API_KEY jfrog.hub.vwgroup.com
          
      - name: Download corpus if exists
        run: |
          HTTP_STATUS=$(curl -u "$ARTIFACTORY_USER:$JFROG_API_KEY" \
            -o /dev/null -w "%{http_code}" -s -I \
            "${ARTIFACTORY_CORPUS_URL}${PROJ_NAME}_corpus.gz")
          if [ "$HTTP_STATUS" -eq 200 ]; then
            curl -u $ARTIFACTORY_USER:$JFROG_API_KEY \
              "${ARTIFACTORY_CORPUS_URL}${PROJ_NAME}_corpus.gz" | tar -xz
          fi
          
      - name: Get container
        run: |
          containerid=$(buildah from jfrog.hub.vwgroup.com/cifuzz-docker-internal/cifuzz:${VERSION})
          echo "containerid=$containerid" >> $GITHUB_ENV
          
      - name: Build project
        run: |
          buildah run -v $(pwd):/$PROJECT_FOLDER ${{ env.containerid }} \
            /usr/bin/bash -c "cd /project && mkdir build && cd build && cmake .. && make"
            
      - name: Generate fuzz tests with LLM
        run: |
          buildah run \
            --env CIFUZZ_LLM_API_URL --env CIFUZZ_LLM_AZURE_DEPLOYMENT_NAME \
            --env CIFUZZ_LLM_MAX_TOKENS --env CIFUZZ_LLM_API_TOKEN \
            -v $(pwd):/$PROJECT_FOLDER ${{ env.containerid }} \
            /usr/bin/bash -c "cd /project/build && cifuzz spark parser_target"
            
      - name: Rebuild with generated tests
        run: |
          buildah run -v $(pwd):/$PROJECT_FOLDER ${{ env.containerid }} \
            /usr/bin/bash -c "cd /project/build && make"
            
      - name: Run fuzzing
        run: |
          buildah run -v $(pwd):/$PROJECT_FOLDER ${{ env.containerid }} \
            /usr/bin/bash -c "cd /project/build && \
              cifuzz run parser_target --libfuzzer-arg=-max_total_time=30"
              
      - name: Upload corpus
        run: |
          tar -czf ${PROJ_NAME}_corpus.gz .cifuzz-corpus
          curl -u $ARTIFACTORY_USER:$JFROG_API_KEY \
            -T ${PROJ_NAME}_corpus.gz "${ARTIFACTORY_CORPUS_URL}${PROJ_NAME}_corpus.gz"
```

Our pipeline includes corpus persistence through JFrog Artifactory. Each run downloads the previous corpus, runs fuzzing to extend it, and uploads the updated corpus. This ensures coverage improvements accumulate over time rather than starting fresh each build.

**[Figure 4.4: Network architecture diagram showing the Azure Private Link setup. The CI/CD Runner in CARIAD internal network connects through Azure Private Link Endpoint to Azure OpenAI. Firewall boundaries are illustrated to show why direct connections fail but Private Link succeeds.]**

### 4.4.3 Operational Considerations

With the technical implementation complete, several operational concerns remained.

**Cost Management**

Azure OpenAI charges per token. Each fuzz driver generation uses approximately 3000 to 8000 tokens depending on the target complexity. We calculated costs for different usage scenarios:

| Scenario | Targets | Builds/Day | Annual Tokens | Annual Cost |
|----------|---------|------------|---------------|-------------|
| Light | 5 | 1 | 3.1M | ~€73 |
| Medium | 10 | 2 | 12.5M | ~€400 |
| Heavy | 25 | 5 | 78M | ~€1,452 |

These costs are minimal compared to hiring dedicated security engineers. Even the heavy usage scenario costs less than two days of consultant time annually.

**Failure Handling**

LLM outputs are not deterministic. Sometimes the generated driver does not compile. Sometimes it compiles but crashes immediately. The pipeline needed to handle these cases gracefully.

We added fallback logic: if the LLM-generated driver fails, the pipeline falls back to any existing manually-written drivers. The build does not fail just because AI generation had a bad day. Generated drivers that work are kept. Those that fail are logged for review but do not block the pipeline.

**Security Review**

Generated code running in CI/CD pipelines poses security considerations. What if the LLM generates malicious code? We addressed this concern in our security review:

1. The code runs inside containers with limited privileges
2. ASan and UBSan catch most dangerous behaviors during execution
3. Generated code is compiled and executed, not given shell access
4. The LLM only sees public header files, not sensitive source code
5. All generated code is logged for audit purposes

After reviewing these controls, the security team approved the architecture. They requested additional monitoring of token usage and periodic audits of generated code quality.

The next chapter presents experimental results in detail, with quantitative analysis of model performance and coverage metrics across all evaluated configurations.
