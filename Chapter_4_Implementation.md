# 4 Implementation

This chapter describes how we turned the methodology from Chapter 3 into working code and systems. The implementation happened in three phases, each building on the previous one. I will explain the tools we chose, the specific steps we followed, and the problems we solved along the way.

## 4.1 Toolchain Selection and Rationale

Before writing any code, we needed to select the right tools. This decision was not straightforward because we had constraints from multiple directions. The university wanted reproducible research. CARIAD needed enterprise compatibility. And we wanted tools that would actually work on the hardware we had available.

### 4.1.1 Fuzzing Infrastructure

For the fuzzing engine itself, we chose libFuzzer combined with Clang sanitizers. This was not a controversial choice. LibFuzzer is the industry standard for coverage guided fuzzing in C and C++ projects. It integrates directly with the LLVM toolchain, which means compilation and fuzzing use the same infrastructure. Google uses libFuzzer for OSS Fuzz, so the documentation and community support are excellent.

We compiled all targets with AddressSanitizer (ASan) and UndefinedBehaviorSanitizer (UBSan) enabled. ASan catches memory errors like buffer overflows, use after free, and memory leaks. UBSan catches undefined behavior like signed integer overflow and null pointer dereferences. These sanitizers add runtime overhead, but for security testing the overhead is acceptable. Finding bugs is more important than running fast.

For integration with our build systems, we used cifuzz from Code Intelligence. Cifuzz wraps libFuzzer and provides additional functionality like corpus management, coverage reporting, and build system integration. The critical feature for our work was cifuzz spark, which handles the LLM integration. Spark takes a CMake target as input, extracts API information automatically, sends it to the configured LLM, and produces a fuzz driver ready for execution.

### 4.1.2 LLM Infrastructure

We needed to run models locally for several reasons. First, we wanted to evaluate many different models, and running them through paid APIs would have been expensive. Second, some models are open source and only available for local deployment. Third, for the enterprise integration phase, we needed to understand how local deployment might work as an alternative to cloud APIs.

We selected Ollama as our primary local inference server. Ollama simplifies model management significantly. Installing a new model is one command: `ollama pull qwen2.5-coder:32b`. The server exposes an OpenAI compatible API, which meant cifuzz spark worked without modification. Ollama handled quantized models well, which let us run larger models on the available hardware.

For models not available through Ollama, we used llama.cpp directly. This required more manual configuration but gave us flexibility. Some models from Hugging Face came as split GGUF files that needed merging before use. The command `llama-gguf-split --merge` handled this, but it added steps to our workflow.

For cloud deployment in the enterprise phase, we configured Azure OpenAI. Azure provided GPT 4o through a managed endpoint. The authentication used API keys stored as GitHub secrets. Azure OpenAI required specific environment variables:

```
CIFUZZ_LLM_API_URL="https://ny4mdomain.openai.azure.com"
CIFUZZ_LLM_AZURE_DEPLOYMENT_NAME="gpt-4o"
CIFUZZ_LLM_API_TOKEN="<api-key>"
CIFUZZ_LLM_MAX_TOKENS=40000
```

The token limit of 40000 was necessary because fuzz driver generation requires substantial context. The prompt includes header files, example code, and detailed instructions. Smaller token limits caused truncation and worse results.

### 4.1.3 Development Environment

The local development happened on a MacBook Pro with an M1 Pro chip. This created some challenges. Docker Desktop had performance issues on Apple Silicon, so we switched to Podman. Podman provided better Linux container compatibility, which mattered because the enterprise CI/CD runners used Linux.

```bash
brew install podman
podman machine init --cpus 4 --memory 8192
podman machine start
```

For the enterprise environment, we used CARIAD's internal GitHub Enterprise instance with self hosted runners. These runners used Buildah instead of Docker for container operations. Buildah is daemonless, which fits better with security policies that restrict long running processes.

The build toolchain used CMake for project configuration and Clang for compilation. We required CMake version 3.5 or higher because cifuzz integration depends on features from that version. The cmake configuration for enabling fuzzing was straightforward:

```cmake
cmake_minimum_required(VERSION 3.5)
find_package(cifuzz NO_SYSTEM_ENVIRONMENT_PATH)
enable_fuzz_testing()
add_subdirectory(cifuzz-spark)
```

**[DIAGRAM RECOMMENDATION: A toolchain architecture diagram showing the relationships between components would be helpful here. The diagram should show: Local Development (Mac with Podman, Ollama) connected to Cifuzz Spark, which connects to either Local LLM or Azure OpenAI. Output flows to LibFuzzer with ASan/UBSan, producing Coverage Reports and Crash Reports.]**

## 4.2 Phase 1: Local LLM Evaluation Setup

The first phase focused on understanding which models could actually generate useful fuzz drivers. We did not assume any model would work well. The literature suggested mixed results for LLM code generation, and fuzz drivers are a specialized task.

### 4.2.1 Benchmarked Models

We evaluated 14 models spanning a wide range of sizes and architectures. The selection included both general purpose models and code specialized variants.

**Large Models (25 to 35 billion parameters):**
- Qwen 2.5 Coder 32B
- DeepSeek Coder 33B  
- CodeLlama 34B Instruct
- WizardCoder 34B
- Yi 34B

**Extra Large Models (above 40 billion parameters):**
- Mixtral 46.7B

**Medium Models (7 to 14 billion parameters):**
- Qwen 2.5 Coder 7B and 14B
- DeepSeek Coder V2 Lite 16B
- Gemma 3 9B

**Small Models (below 7 billion parameters):**
- Qwen 2.5 Coder 1.5B
- Gemma 3 4B
- Phi 4 3.8B

The code specialized models like Qwen Coder and DeepSeek Coder were trained specifically on programming tasks. General purpose models like Mixtral were included to test whether raw scale could compensate for lack of specialization.

### 4.2.2 Target Repositories

We needed consistent test targets to compare models fairly. We selected open source C++ libraries with varying complexity:

**Primary Target: yaml-cpp**

Yaml-cpp is a YAML parsing library with approximately 35 source files and over 1000 API functions that could potentially be fuzzed. We chose it as our primary benchmark because it has enough complexity to challenge the models while being well documented and actively maintained.

**Secondary Targets:**

- **RapidJSON**: JSON parsing library, good test of format handling code
- **pugixml**: XML processing, similar parsing challenges to yaml-cpp
- **fmt**: Text formatting library, different API patterns than parsers
- **spdlog**: Logging library, many configuration options
- **jsoncons**: Another JSON library for comparison

These libraries cover different domains but share the characteristic of being C++ with clean public APIs. They all have existing fuzz targets in OSS Fuzz, which gave us baselines for comparison.

### 4.2.3 Evaluation Environment

The hardware for local evaluation was a workstation with an NVIDIA RTX 4090 GPU (24GB VRAM) and 64GB system RAM. The 24GB VRAM was sufficient for running 32B parameter models with 4-bit quantization. Larger models like Mixtral 46.7B required offloading to system RAM, which slowed inference significantly.

The evaluation process for each model followed these steps:

1. Start the model server (Ollama or llama.cpp)
2. Configure cifuzz spark environment variables
3. Run `cifuzz spark` on the target library
4. Attempt to compile the generated driver
5. If compilation succeeded, run the fuzzer for a fixed duration
6. Record coverage metrics using llvm-cov

We ran each model three times on yaml-cpp to check consistency. Some models produced different outputs on identical prompts, so multiple runs were necessary for reliable results.

**[DIAGRAM RECOMMENDATION: A flowchart showing the evaluation pipeline would clarify the process. Start with "Model Server Running", then "cifuzz spark generates driver", then decision diamond "Compiles?", if No record failure, if Yes run fuzzer, then "Record Coverage Metrics". Include timing annotations showing typical durations for each step.]**

## 4.3 Phase 2: Model Optimization with LoRA Fine Tuning

After the initial evaluation showed promising results from Qwen 2.5 Coder models, we investigated whether fine tuning could improve performance further. The goal was to see if a small model, specifically trained for fuzz driver generation, could match or exceed larger general purpose models.

### 4.3.1 Training Data Preparation

The most important part of fine tuning is the training data. Garbage in, garbage out. We needed high quality examples of fuzz drivers paired with the source code they were testing.

We extracted training examples from OSS Fuzz. Google has fuzz tested hundreds of open source projects through OSS Fuzz, and the fuzz drivers are publicly available. We focused on C and C++ drivers because those matched our target domain.

The extraction process:

1. Clone the OSS Fuzz repository
2. Identify projects with C/C++ fuzz drivers
3. For each driver, extract the header files it includes
4. Create input/output pairs: headers as input, driver as expected output

We created two datasets:
- **Small dataset**: 172 examples, focusing on quality over quantity
- **Extended dataset**: 709 examples, broader coverage but more noise

The examples included drivers for libraries like FreeType, LibPNG, SQLite, and zlib. These cover a range of API styles from simple functions to complex object oriented interfaces.

### 4.3.2 LoRA Configuration and Training

We used Low Rank Adaptation (LoRA) rather than full fine tuning. Full fine tuning updates all model weights, which requires enormous compute resources. LoRA adds small trainable matrices alongside the frozen base model. This reduces memory requirements dramatically while achieving similar results on specialized tasks.

The LoRA configuration we used:

```python
LORA_RANK = 16          # Complexity of adaptation
LORA_ALPHA = 32         # Scaling factor  
DROPOUT = 0.1           # Regularization
TARGET_MODULES = ["q_proj", "v_proj", "k_proj", "o_proj"]
DTYPE = "float16"       # Memory efficient
```

The rank of 16 was a balance between adaptation capacity and training speed. Higher ranks can capture more complex patterns but require more memory and training time. We tested ranks of 8, 16, and 32. Rank 16 gave the best results for our dataset size.

We trained on the Qwen 2.5 Coder 1.5B base model. This is a small model that runs quickly on modest hardware. Training took approximately 4 hours on a single RTX 4090.

The training used standard supervised fine tuning with the input being the header files and instructions, and the target being the complete fuzz driver. We used a learning rate of 2e-5 with cosine decay and trained for 3 epochs.

### 4.3.3 Fine Tuned Model Evaluation

After training, we evaluated the fine tuned models on the same yaml-cpp benchmark used in Phase 1. The comparison:

**Base Qwen 2.5 Coder 1.5B:**
- Token usage: ~4200 tokens per generation
- Generation time: ~45 seconds
- Coverage achieved: 42% on yaml-cpp

**Fine Tuned (172 examples):**
- Token usage: ~2900 tokens per generation (31% reduction)
- Generation time: ~30 seconds (33% reduction)  
- Coverage achieved: 44% on yaml-cpp

**Fine Tuned (709 examples):**
- Token usage: ~2700 tokens per generation (36% reduction)
- Generation time: ~28 seconds (38% reduction)
- Coverage achieved: 43% on yaml-cpp

The fine tuned models generated more concise code. They learned the structure of fuzz drivers and did not waste tokens on unnecessary explanations or boilerplate. Coverage was slightly better or equivalent to the base model, but the real win was efficiency. Generating each driver took 33% less time and used fewer tokens.

Interestingly, more training data did not always help. The 709 example dataset included some lower quality drivers, and this may have hurt the model's output. Quality matters more than quantity for fine tuning specialized tasks.

**[DIAGRAM RECOMMENDATION: A bar chart comparing the three model versions (Base, 172 examples, 709 examples) across three metrics (Token Usage, Generation Time, Coverage). This would clearly show the efficiency gains from fine tuning.]**

## 4.4 Phase 3: Enterprise CI/CD Integration

The final phase moved from controlled experiments to real world deployment. This is where things got complicated. Academic research happens in ideal conditions. Enterprise deployment happens in conditions dictated by security policies, legacy infrastructure, and organizational constraints.

### 4.4.1 Architectural Challenges

The first attempt at integration seemed straightforward. We had a working pipeline locally. The enterprise runners used similar Linux environments. How hard could it be?

Very hard, it turned out.

**Challenge 1: Network Isolation**

CARIAD operates with a zero trust security model. CI/CD runners cannot access the public internet by default. Every external connection requires explicit approval. This makes sense for security, but it creates problems for AI integration.

Our initial pipeline tried to call Azure OpenAI during the build. The request never reached the Azure endpoint. It was blocked by the firewall before leaving the network.

The error message was cryptic:
```
Error: fatal: unable to access 'https://git.hub.vwgroup.com/...': 
Failed to connect to 127.0.0.1 port 9000 after 0 ms
```

The proxy configuration we set for Azure was interfering with internal Git operations. The runner was trying to send Git traffic through a proxy that did not exist.

**Challenge 2: Container Networking**

The runners use Buildah containers for isolation. Each build step runs inside a container with its own network namespace. Even if the host could reach external services, the container might not be able to.

We tried several approaches:

Host network mode:
```bash
buildah run --network host ...
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

Each environment variable had to be passed explicitly. Missing one caused silent failures where the LLM call would not work but the pipeline continued.

### 4.4.2 Self Hosted Runner Solution

After weeks of debugging network issues, we concluded that the standard enterprise firewall would not allow direct Azure OpenAI access. The solution required infrastructure changes beyond what we could do ourselves.

**Azure Private Link Architecture**

The approved solution used Azure Private Link. This creates a private endpoint for Azure services that appears inside the corporate network. Traffic to the LLM endpoint never crosses the public internet.

The architecture:

```
CI/CD Runner (internal network)
    |
    v
Azure Private Link Endpoint (inside CARIAD network)
    |
    v
Azure OpenAI Service (Microsoft infrastructure)
```

From the runner's perspective, the Azure OpenAI endpoint has an internal IP address. The firewall allows internal traffic, so the LLM calls succeed.

Setting up Private Link required coordination with the infrastructure team. The process took approximately one month:

1. Request submission with business justification
2. Security review of the use case
3. Azure subscription configuration
4. DNS updates for private endpoint resolution
5. Network security group rules
6. Testing and validation

**Working Pipeline Configuration**

Once Private Link was operational, the pipeline worked reliably. The final configuration:

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
        run: |
          buildah login -u $ARTIFACTORY_USER -p $JFROG_API_KEY jfrog.hub.vwgroup.com
          
      - name: Download corpus if exists
        run: |
          HTTP_STATUS=$(curl -u "$ARTIFACTORY_USER:$JFROG_API_KEY" \
            -o /dev/null -w "%{http_code}" -s -I \
            "${ARTIFACTORY_CORPUS_URL}${PROJ_NAME}_corpus.gz")
          if [ "$HTTP_STATUS" -eq 200 ]; then
            echo "Corpus found. Downloading..."
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
            --env CIFUZZ_LLM_API_URL \
            --env CIFUZZ_LLM_AZURE_DEPLOYMENT_NAME \
            --env CIFUZZ_LLM_MAX_TOKENS \
            --env CIFUZZ_LLM_API_TOKEN \
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
            -T ${PROJ_NAME}_corpus.gz \
            "${ARTIFACTORY_CORPUS_URL}${PROJ_NAME}_corpus.gz"
```

The pipeline includes corpus persistence through JFrog Artifactory. Each run downloads the previous corpus, runs fuzzing to extend it, and uploads the updated corpus. This ensures coverage improvements accumulate over time rather than starting fresh each build.

**[DIAGRAM RECOMMENDATION: A network architecture diagram showing the Azure Private Link setup would be essential here. Show the CI/CD Runner in the CARIAD internal network, connected through Azure Private Link Endpoint to Azure OpenAI. Include firewall boundaries to illustrate why this approach works when direct connections fail.]**

### 4.4.3 Operational Considerations

With the technical implementation complete, several operational concerns remained.

**Cost Management**

Azure OpenAI charges per token. Each fuzz driver generation uses approximately 3000 to 8000 tokens depending on the target complexity. For a project with 10 fuzz targets running daily builds:

- Tokens per target: ~5000 average
- Targets per day: 10
- Days per year: 250 (working days)
- Total tokens: 12.5 million per year
- Cost at GPT 4o rates: approximately 400 to 500 euros per year

This is affordable for an enterprise. But costs could increase rapidly with more targets or more frequent builds. We implemented token usage monitoring to track spending.

**Failure Handling**

LLM outputs are not deterministic. Sometimes the generated driver does not compile. Sometimes it compiles but crashes immediately. The pipeline needed to handle these cases gracefully.

We added fallback logic: if the LLM generated driver fails, the pipeline falls back to any existing manually written drivers. The build does not fail just because AI generation had a bad day. Generated drivers that work are kept. Those that fail are logged for review.

**Security Review**

Generated code running in CI/CD pipelines poses security considerations. What if the LLM generates malicious code? The risk is low because:

1. The code runs inside containers with limited privileges
2. ASan and UBSan catch most dangerous behaviors
3. Generated code is compiled and executed, not given shell access
4. The LLM only sees public header files, not sensitive code

Still, the security team reviewed the architecture before approval. They requested additional logging of all generated code for audit purposes.

---

## Summary

This chapter covered the practical implementation of our AI driven fuzzing system. We selected tools based on compatibility with both research needs and enterprise constraints. The local evaluation phase established baseline performance across 14 models. Fine tuning demonstrated that small specialized models can match larger ones for specific tasks. Enterprise integration revealed that network security policies are often the hardest challenge, not the AI itself.

The key technical contributions are:

1. A complete toolchain for LLM assisted fuzz driver generation
2. Methodology for evaluating LLM performance on code generation tasks
3. LoRA fine tuning configuration optimized for fuzz driver generation
4. Enterprise deployment architecture using Azure Private Link
5. Production ready CI/CD pipeline configuration

The next chapter presents experimental results in detail, with quantitative analysis of model performance and coverage metrics.
