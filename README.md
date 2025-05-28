# instructLab을 사용하여 nmon Wrangler 만의 LLM 생성

## 참고 가이드
- 전체적인 과정은 아래 url의 가이드를 참고 하였습니다.
* [https://docs.instructlab.ai/getting-started/linux\_amd/](https://docs.instructlab.ai/getting-started/linux_amd/)
* [https://github.com/instructlab/instructlab](https://github.com/instructlab/instructlab)


## 개요

* **진행 개요** : 나만의 데이터로 LLM을 튜닝해서, 목적에 맞게 사용할 수 있는 모델을 만드는 전체 파이프라인 구성.

* **목적** : nmon sample data를 사용하여 input nmon file에 대해 분석 또는 개선점을 제안해주는 모델을 만든다. 이 모델을 사용하여 실제 nmon Wrangler에 data input 시 분석해주는 기능을 추가 할 예정.(1차 테스트는 chatbot 형식으로 진행 하여 tuned 모델 검증 후 최종 완성된 model을 사용하여 nmon Wrangler에 enable 할 수 있는 방법을 다시 찾을 예정)

* **한계점** : model fine tuning 시 GPU 가 없는 pc를 사용할 경우 시간이 너무 많이 소요됨.


## 개발 환경

* Windows WSL
* Ubuntu 22.04
* Python 3.11


## Step 1: 가상 환경 생성

```bash
python3.11 -m venv myenv
source myenv/bin/activate
```


## Step 2: 패키지 설치

```bash
pip install --upgrade pip setuptools wheel

sudo apt update
sudo apt install -y build-essential cmake
```


## Step 3: instructLab 설치

```bash
pip install instructlab
```


## Step 4: instructLab 환경 초기화

```bash
ilab config init
```

* GPU 없는 경우 `0` (No Profile) 선택
* GPU가 없는 경우, 0 no profile 선택하라는 document 내용. 
“When prompted, please choose a train profile. Train profiles are GPU specific profiles that enable accelerated training behavior. If you are on MacOS or a Linux machine without a dedicated GPU, please choose No Profile (CPU, Apple Metal, AMD ROCm) by hitting Enter. There are various flags you can utilize with individual ilab commands that allow you to utilize your GPU if applicable.”

```bash
(venv) root@junghyun:~/instructlab# ilab config init
Existing config file was found in:
    /root/.config/instructlab/config.yaml
Do you still want to continue? [y/N]: y

Existing system profiles were found in:
    /root/.local/share/instructlab/internal/system_profiles
Do you want to restore these profiles to the default values? [y/N]: y

----------------------------------------------------
         Welcome to the InstructLab CLI
  This guide will help you to setup your environment
----------------------------------------------------

Please provide the following values to initiate the environment [press 'Enter' for default options when prompted]
Path to taxonomy repo [/root/.local/share/instructlab/taxonomy]:
Path to your model [/root/.cache/instructlab/models/granite-7b-lab-Q4_K_M.gguf]:
INFO 2025-05-27 14:22:44,945 instructlab.config.init:64:
Generating config file and profiles:
    /root/.config/instructlab/config.yaml
    /root/.local/share/instructlab/internal/system_profiles

WARNING 2025-05-27 14:22:47,380 instructlab.config.init:125: ilab is only officially supported on Linux and MacOS with M-Series Chips
Please choose a system profile.
Profiles set hardware-specific defaults for all commands and sections of the configuration.
First, please select the hardware vendor your system falls into
[0] NO SYSTEM PROFILE
[1] APPLE
[2] INTEL
[3] AMD
[4] NVIDIA
Enter the number of your choice [0]: 0
No profile selected - ilab will use generic code defaults - these may not be optimized for your system.

--------------------------------------------
    Initialization completed successfully!
  You're ready to start using `ilab`. Enjoy!
--------------------------------------------
(venv) root@junghyun:~/instructlab# ilab config show

```


## Step 5: 모델 다운로드

```bash
(venv) root@junghyun:~/instructlab# ilab model download
```

* 기본으로 instructLab에서 사용되는 모델은 아래와 같습니다.
* 이 중 flag를 사용하여 특정 모델을 사용할 수 있습니다.
* 우리는 instruct를 사용 해야 하기 때문에 mistral-7b-instruct 또는 granite-7b를 사용 하고자 합니다. (7b는 너무 커서(파라미터 개수가 너무 많음 70억개) gpu가 없는 환경에서 사용하기에 무리가 있긴 합니다... 그런데 언어모델은 모두 커서.. hugging face에서 양자화 된 모델을 찾아 이를 다운받아 사용 하면 될 것 같습니다. 일단 테스트는 mistral-7b로 진행 중입니다.)

```bash
(venv) root@junghyun:~/instructlab# ilab model download
INFO 2025-05-27 14:53:33,080 instructlab.model.download:302: Available models (`ilab model list`):
+--------------------------------------------+---------------------+----------+----------------------------------------------------------------------+
| Model Name                                 | Last Modified       | Size     | Absolute path                                                        |
+--------------------------------------------+---------------------+----------+----------------------------------------------------------------------+
| mistral-7b-instruct-v0.2.Q4_K_M.gguf       | 2025-05-27 14:53:14 | 4.1 GB   | /root/.cache/instructlab/models/mistral-7b-instruct-v0.2.Q4_K_M.gguf |
| merlinite-7b-lab-Q4_K_M.gguf               | 2025-05-27 14:51:59 | 4.1 GB   | /root/.cache/instructlab/models/merlinite-7b-lab-Q4_K_M.gguf         |
| ibm-granite/granite-embedding-125m-english | 2025-05-27 14:53:33 | 479.2 MB | /root/.cache/instructlab/models/ibm-granite                          |
| granite-7b-lab-Q4_K_M.gguf                 | 2025-05-27 14:50:42 | 3.8 GB   | /root/.cache/instructlab/models/granite-7b-lab-Q4_K_M.gguf           |
+--------------------------------------------
```


## Step 6: 모델 서빙 및 Chat Test

모델 선택. (granite-7b-lab-Q4_K_M.gguf)
```bash
(venv) root@junghyun:~/instructlab# ilab model serve --model-path /root/.cache/instructlab/models/granite-7b-lab-Q4_K_M.gguf
INFO 2025-05-27 15:35:12,095 instructlab.model.serve_backend:80: Setting backend_type in the serve config to llama-cpp
INFO 2025-05-27 15:35:12,108 instructlab.model.serve_backend:86: Using model '/root/.cache/instructlab/models/granite-7b-lab-Q4_K_M.gguf' with -1 gpu-layers and 4096 max context size.llama_new_context_with_model: n_ctx_pre_seq (4096) > n_ctx_train (2048) -- possible training context overflowINFO 2025-05-27 15:35:20,380 instructlab.model.backends.llama_cpp:306: Replacing chat template: {% set eos_token = "<|endoftext|>" %}{% set bos_token = "<|begginingoftext|>" %}{% for message in messages %}{% if message['role'] == 'pretraining' %}{{'<|pretrain|>' + message['content'] + '<|endoftext|>' + '<|/pretrain|>' }}{% elif message['role'] == 'system' %}{{'<|system|>'+ '' + message['content'] + ''}}{% elif message['role'] == 'user' %}{{'<|user|>' + '' + message['content'] + ''}}{% elif message['role'] == 'assistant' %}{{'<|assistant|>' + '' + message['content'] + '<|endoftext|>' + ('' if loop.last else '')}}{% endif %}{% if loop.last and add_generation_prompt %}{{ '<|assistant|>' + '' }}{% endif %}{% endfor %}
INFO 2025-05-27 15:35:20,388 instructlab.model.backends.llama_cpp:233: Starting server process, press CTRL+C to shutdown server...INFO 2025-05-27 15:35:20,389 instructlab.model.backends.llama_cpp:234: After application startup complete see http://127.0.0.1:8000/docs for API.
```

------------새로운 터미널 오픈------------

```bash
wsl
```
```bash
sudo -i
```
```bash
python3 -m venv venv
```
```bash
source venv/bin/activate
```

```bash
(venv) root@junghyun:~# ilab model chat

╭─── system  Welcome to InstructLab Chat : GRANITE-7B-LAB-Q4_K_M.GGUF (type /h for help) 

Q : what is the capital of south Korea? [S][default]

granite-7b-lab-Q4_K_M.gguf 

The capital city of South Korea is Seoul, a vibrant metropolis known for its rich history, bustling culture, and technological advancements. If you have any questions about Seoul or other parts of South Korea, I'd be happy to help! 
─── elapsed 10.470 seconds ─╯>>>
```

->한국의 수도를 잘 알려줍니다.



## Step 7: instruction yaml 구성 및 GitHub 업로드

* instructLab은 taxonomy를 사용하여 모델이 학습 하게 될 데이터를 사용합니다. (taxonomy는 모델이 학습하게 될 지식, 기능, 개념들을 계층적으로 구조화한 분류 체계 입니다.)
* 다음 과정에서 보시겠지만 ilab data generate 과정을 거쳐 .jsonl 파일을 생성하는데 이때 ~/.local/share/instructlab/taxonomy 디렉토리 경로를 자동 스캔합니다. 그래서 새로운 .yaml 파일이 있는지 검토 합니다.
* 이 instruction yaml 파일은 단순한 데이터 파일이 아니라 taxonomy의 leaf node역할을 합니다. 그래서 파일을 작성하는데 나름의 규칙도 있고 꼭 들어 가야 하는 항목이 존재합니다.
* 저는 nmon data 중 memuse 항목을 사용하여 데이터를 구성 하였습니다.

### step 7-1: git 에 repository 생성 후 .md 파일 생성
https://github.com/junghyuncha/instructlab/blob/main/nmon_memuse.md

* memuse 항목에 대한 설명과 간단한 예시, 분석 내용이 담긴 내용입니다.
* 저는 memuse에 대해 정리 하였지만 cpuuse, page 등 현재 우리가 수집하는 nmon data에 대해 각각 생성하여 만들어야 합니다.(모델의 학습을 위해)

### step 7-2: yaml file 작성
* 이 yaml file을 사용하여 .jsonl 파일을 생성합니다. (model train을 이 .jsonl 파일을 사용하여 진행하기 때문)
* 저는 nmonWrangler 도메인을 사용하였고 파일 위치는 /root/.local/share/instructlab/taxonomy/knowledge/nmonWrangler/collections/memuse/qna.yaml 로 지정 하였습니다.
* 파일 작성에는 여러 규칙이 있습니다. 아래 항목이 필드 구문에 맞게 들어가야 합니다.
    * domain, created_by, document, seed_examples, context, questions_and_answers, repo(github주소), commit(commit id), patterns(yaml 파일 위치) 
    * seed_examples 를 최소 5개 작성
    * 각 questions_and_answers 블록에 질문 3개 이상 작성

[yaml 파일의 일부]

```yaml
version: 3
domain: nmonWrangler
created_by: junghyun
seed_examples:
  - context: |
      The following is a snapshot of the `memuse` section from an AIX system collected using nmon:
      - %numperm: 91.2
      - %minperm: 3.0
      - %maxperm: 90.0
      - %numclient: 92.5
      - %maxclient: 90.0
      - minfree: 960
      - maxfree: 1088
      - lruable pages: 130171072

      These metrics reflect the system’s memory usage. The `perm` values indicate file system cache, while `client` values relate to application cache.
    questions_and_answers:
      - question: |
          What problems are indicated by this memuse data?
        answer: |
          Both %numperm and %numclient exceed their respective %maxperm and %maxclient thresholds. This suggests that the file system and applications are overusing cache memory, which could lead to page stealing and system performance degradation. You should consider tuning cache settings or increasing physical memory.

      - question: |
          What does the %numperm field represent?
        answer: |
          %numperm indicates the percentage of memory used by the persistent file system cache. It is recommended to keep this value below %maxperm to avoid I/O bottlenecks and page stealing issues.

      - question: |
          What configuration or tuning can be done to improve this?
        answer: |
          Use the `vmo` command to adjust maxperm and maxclient thresholds. Additionally, tuning `minfree` and `maxfree` can help ensure adequate free pages. It’s also advisable to review application memory usage and minimize unnecessary cache retention.

# 비슷하게 context 4개 더 추가

document:
  repo: https://github.com/junghyuncha/instructlab
  commit: cdb021ba10aa638c8f91c746aad8bc9761ee2073
  patterns:
    - nmon_memuse.md
```

### 7-3: yaml 파일 검토
* 파일 작성을 완료하면 ilab taxonomy diff 명령어를 사용하여 검토합니다.
* 파일 작성에 문제가 없다면 아래와 같이 로그가 출력됩니다.
```bash
(venv) root@junghyun:~/instructlab# ilab taxonomy diff
```

결과 예:

```
Taxonomy in /root/.local/share/instructlab/taxonomy is valid :)
```


## Step 8: .jsonl 학습 데이터 생성

* 모델을 train 시킬때 사용하는 .jsonl 파일을 생성합니다. 
* 위에서 만들어 놓은 yaml을 참고하여 생성 하게 됩니다.
* 시간이 조금 소요됩니다.. 저의 경우 ilab config init 시 gpu가 없는 환경인 0 no profile을 선택 하였기 때문에 cpu로 이 데이터를 만드는데 18시간 정도 걸렸습니다.

```bash
(venv) root@junghyun:~/.local/share/instructlab/taxonomy/knowledge/nmonWrangler/collections/memuse# ilab data generate
INFO 2025-05-27 19:55:32,321 instructlab.process.process:300: Started subprocess with PID 9860. Logs are being written to /root/.local/share/instructlab/logs/generation/generation-1c306fba-3ae9-11f0-b0dd-0b4567577773.log.
INFO 2025-05-27 19:55:32,622 instructlab.model.backends.llama_cpp:126: Trying to connect to model server at http://127.0.0.1:8000/v1
llama_new_context_with_model: n_ctx_per_seq (4096) < n_ctx_train (32768) -- the full capacity of the model will not be utilized
WARNING 2025-05-27 19:55:43,037 instructlab:188: Disabling SDG batching - unsupported with llama.cpp serving
INFO 2025-05-27 19:55:43,271 numexpr.utils:162: NumExpr defaulting to 12 threads.
INFO 2025-05-27 19:55:43,561 datasets:54: PyTorch version 2.7.0 available.
INFO 2025-05-27 19:55:43,965 instructlab:206: Generating synthetic data using 'full' pipeline, '/root/.cache/instructlab/models/mistral-7b-instruct-v0.2.Q4_K_M.gguf' model, '/root/.local/share/instructlab/taxonomy' taxonomy, against http://127.0.0.1:38241/v1 server
INFO 2025-05-27 19:55:43,966 root:356: Converting taxonomy to samples
INFO 2025-05-27 19:55:44,791 instructlab.sdg.utils.taxonomy:143: Processing files...
INFO 2025-05-27 19:55:44,791 instructlab.sdg.utils.taxonomy:148: Pattern 'nmon_memuse.md' matched 1 files.
INFO 2025-05-27 19:55:44,791 instructlab.sdg.utils.taxonomy:152: Processing file: /root/.local/share/instructlab/datasets/2025-05-27_195532/preprocessed_2025-05-27T19_55_43/documents/knowledge_nmonWrangler_collections_memuse_u5hs49g3/nmon_memuse.md
INFO 2025-05-27 19:55:44,791 instructlab.sdg.utils.taxonomy:156: Added file path: /root/.local/share/instructlab/datasets/2025-05-27_195532/preprocessed_2025-05-27T19_55_43/documents/knowledge_nmonWrangler_collections_memuse_u5hs49g3/nmon_memuse.md
INFO 2025-05-27 19:55:48,807 instructlab.sdg.utils.chunkers:141: Docling models not found on disk, downloading models...
INFO 2025-05-27 19:55:48,808 docling.utils.model_downloader:45: Downloading layout model...
INFO 2025-05-27 19:56:03,965 docling.utils.model_downloader:53: Downloading tableformer model...
INFO 2025-05-27 19:56:10,752 docling.utils.model_downloader:61: Downloading picture classifier model...
INFO 2025-05-27 19:56:12,811 docling.utils.model_downloader:69: Downloading code formula model...
INFO 2025-05-27 19:56:29,439 docling.utils.model_downloader:114: Downloading easyocr models...
WARNING 2025-05-27 19:56:35,156 instructlab.sdg.utils.chunkers:60: Tesseract not found, falling back to EasyOCR.
INFO 2025-05-27 19:56:35,163 docling.utils.accelerator_utils:67: Accelerator device: 'cpu'
WARNING 2025-05-27 19:56:35,166 easyocr.easyocr:251: Downloading detection model, please wait. This may take several minutes depending upon your network connection.
INFO 2025-05-27 19:56:38,534 easyocr.easyocr:255: Download complete
WARNING 2025-05-27 19:56:38,534 easyocr.easyocr:176: Downloading recognition model, please wait. This may take several minutes depending upon your network connection.
INFO 2025-05-27 19:56:40,172 easyocr.easyocr:180: Download complete.
You are using the default legacy behaviour of the <class 'transformers.models.llama.tokenization_llama_fast.LlamaTokenizerFast'>. This is expected, and simply means that the `legacy` (previous) behavior will be used so nothing changes for you. If you want to use the new behaviour, set `legacy=False`. This should only be set if you understand what it means, and thoroughly read the reason why this was added as explained in https://github.com/huggingface/transformers/pull/24565 - if you loaded a llama tokenizer from a GGUF file you can ignore this message.
Merges were not in checkpoint, building merges on the fly.
100%|##########| 32000/32000 [00:20<00:00, 1564.48it/s]
INFO 2025-05-27 19:57:04,583 instructlab.sdg.utils.chunkers:249: Successfully loaded tokenizer from: /root/.cache/instructlab/models/mistral-7b-instruct-v0.2.Q4_K_M.gguf
INFO 2025-05-27 19:57:04,625 docling.document_converter:271: Going to convert document batch...
INFO 2025-05-27 19:57:04,625 docling.document_converter:306: Initializing pipeline for SimplePipeline with options hash 4cc01982ae99b46a2a63fcda46c47c35
INFO 2025-05-27 19:57:04,625 docling.pipeline.base_pipeline:39: Processing document nmon_memuse.md
INFO 2025-05-27 19:57:04,671 docling.document_converter:286: Finished converting document nmon_memuse.md in 0.05 sec.
INFO 2025-05-27 19:57:04,728 instructlab.sdg.generate_data:405: Taxonomy converted to samples and written to /root/.local/share/instructlab/datasets/2025-05-27_195532/preprocessed_2025-05-27T19_55_43
INFO 2025-05-27 19:57:04,739 instructlab.sdg.generate_data:441: Synthesizing new instructions. If you aren't satisfied with the generated instructions, interrupt training (Ctrl-C) and try adjusting your YAML files. Adding more examples may help.
INFO 2025-05-27 19:57:05,414 instructlab.sdg.checkpointing:59: No existing checkpoints found in /root/.local/share/instructlab/datasets/checkpoints/knowledge_nmonWrangler_collections_memuse, generating from scratch
INFO 2025-05-27 19:57:05,414 instructlab.sdg.pipeline:156: Running pipeline single-threaded
INFO 2025-05-27 19:57:05,414 instructlab.sdg.pipeline:199: Running block: duplicate_document_col
INFO 2025-05-27 19:57:05,414 instructlab.sdg.pipeline:203: Batching disabled; processing block 'duplicate_document_col' single-threaded.
INFO 2025-05-27 19:57:05,422 instructlab.sdg.blocks.llmblock:56: LLM server supports batched inputs: False
INFO 2025-05-27 19:57:05,422 instructlab.sdg.pipeline:199: Running block: gen_spellcheck
INFO 2025-05-27 19:57:05,422 instructlab.sdg.pipeline:203: Batching disabled; processing block 'gen_spellcheck' single-threaded.
gen_spellcheck Prompt Generation:   0%|          | 0/35 [00:00<?, ?it/s]/root/myenv/lib/python3.11/site-packages/llama_cpp/llama.py:1237: RuntimeWarning: Detected duplicate leading "<s>" in prompt, this will likely reduce response quality, consider removing it...
  warnings.warn(
gen_spellcheck Prompt Generation:  23%|##2       | 8/35 [03:35<12:04, 26.83s/it]



Filter (num_proc=8): 100%|##########| 453/453 [00:00<00:00, 2176.96 examples/s]
INFO 2025-05-28 10:26:20,576 instructlab.sdg.pipeline:199: Running block: eval_relevancy_qa_pair
INFO 2025-05-28 10:26:20,576 instructlab.sdg.pipeline:203: Batching disabled; processing block 'eval_relevancy_qa_pair' single-threaded.
eval_relevancy_qa_pair Prompt Generation: 100%|##########| 158/158 [1:12:17<00:00, 27.45s/it]
INFO 2025-05-28 11:38:38,265 instructlab.sdg.pipeline:199: Running block: filter_relevancy
INFO 2025-05-28 11:38:38,266 instructlab.sdg.pipeline:203: Batching disabled; processing block 'filter_relevancy' single-threaded.
Map (num_proc=8): 100%|##########| 158/158 [00:00<00:00, 556.93 examples/s]
Filter (num_proc=8): 100%|##########| 158/158 [00:00<00:00, 952.41 examples/s]
INFO 2025-05-28 11:38:39,052 instructlab.sdg.pipeline:199: Running block: eval_verify_question
INFO 2025-05-28 11:38:39,052 instructlab.sdg.pipeline:203: Batching disabled; processing block 'eval_verify_question' single-threaded.
eval_verify_question Prompt Generation: 100%|##########| 155/155 [1:02:47<00:00, 24.31s/it]
INFO 2025-05-28 12:41:27,144 instructlab.sdg.pipeline:199: Running block: filter_verify_question
INFO 2025-05-28 12:41:27,146 instructlab.sdg.pipeline:203: Batching disabled; processing block 'filter_verify_question' single-threaded.
Map (num_proc=8): 100%|##########| 149/149 [00:00<00:00, 547.72 examples/s]
Filter (num_proc=8): 100%|##########| 149/149 [00:00<00:00, 760.13 examples/s]
INFO 2025-05-28 12:41:27,965 instructlab.sdg.generate_data:478: Generated 134 samples
INFO 2025-05-28 12:41:28,029 instructlab.sdg.pipeline:156: Running pipeline single-threaded
INFO 2025-05-28 12:41:28,041 instructlab.sdg.pipeline:199: Running block: gen_mmlu_knowledge
INFO 2025-05-28 12:41:28,041 instructlab.sdg.pipeline:203: Batching disabled; processing block 'gen_mmlu_knowledge' single-threaded.
gen_mmlu_knowledge Prompt Generation: 100%|##########| 35/35 [31:42<00:00, 54.36s/it]
Filter: 100%|##########| 13/13 [00:00<00:00, 455.87 examples/s]
Filter: 100%|##########| 10/10 [00:00<00:00, 473.17 examples/s]
Flattening the indices: 100%|##########| 10/10 [00:00<00:00, 1688.12 examples/s]
Map: 100%|##########| 10/10 [00:00<00:00, 2463.61 examples/s]
Map: 100%|##########| 10/10 [00:00<00:00, 671.91 examples/s]
Map: 100%|##########| 10/10 [00:00<00:00, 2309.64 examples/s]
Filter: 100%|##########| 10/10 [00:00<00:00, 2248.72 examples/s]
Filter: 100%|##########| 10/10 [00:00<00:00, 4698.97 examples/s]
Filter: 100%|##########| 9/9 [00:00<00:00, 4457.28 examples/s]
Flattening the indices: 100%|##########| 9/9 [00:00<00:00, 2173.21 examples/s]
Casting to class labels: 100%|##########| 9/9 [00:00<00:00, 799.24 examples/s]
INFO 2025-05-28 13:13:10,878 instructlab.sdg.eval_data:126: Saving MMLU Dataset /root/.local/share/instructlab/datasets/2025-05-27_195532/node_datasets_2025-05-27T19_55_43/mmlubench_knowledge_nmonWrangler_collections_memuse.jsonl
Creating json from Arrow format: 100%|##########| 1/1 [00:00<00:00, 32.45ba/s]
INFO 2025-05-28 13:13:10,965 instructlab.sdg.eval_data:130: Saving MMLU Task yaml /root/.local/share/instructlab/datasets/2025-05-27_195532/node_datasets_2025-05-27T19_55_43/knowledge_nmonWrangler_collections_memuse_task.yaml
Map: 100%|##########| 134/134 [00:00<00:00, 4157.88 examples/s]
Map: 100%|##########| 134/134 [00:00<00:00, 12437.19 examples/s]
Filter: 100%|##########| 134/134 [00:00<00:00, 18549.07 examples/s]
Map: 100%|##########| 12/12 [00:00<00:00, 4207.28 examples/s]
Map: 100%|##########| 12/12 [00:00<00:00, 3835.96 examples/s]
Creating json from Arrow format: 100%|##########| 1/1 [00:00<00:00, 134.98ba/s]
Map: 100%|##########| 134/134 [00:00<00:00, 9055.47 examples/s]
Map: 100%|##########| 134/134 [00:00<00:00, 5589.40 examples/s]
Map: 100%|##########| 134/134 [00:00<00:00, 9178.81 examples/s]
Map: 100%|##########| 134/134 [00:00<00:00, 33150.69 examples/s]
Filter: 100%|##########| 134/134 [00:00<00:00, 37697.82 examples/s]
Map: 100%|##########| 12/12 [00:00<00:00, 5622.39 examples/s]
Creating json from Arrow format: 100%|##########| 1/1 [00:00<00:00, 52.31ba/s]
INFO 2025-05-28 13:13:11,258 instructlab.sdg.datamixing:158: Loading dataset from /root/.local/share/instructlab/datasets/2025-05-27_195532/node_datasets_2025-05-27T19_55_43/knowledge_nmonWrangler_collections_memuse_p10.jsonl ...
Generating train split: 280 examples [00:00, 9201.78 examples/s]
INFO 2025-05-28 13:13:12,308 instructlab.sdg.datamixing:160: Dataset columns: ['messages', 'metadata', 'id', 'unmask']
INFO 2025-05-28 13:13:12,308 instructlab.sdg.datamixing:161: Dataset loaded with 280 samples
Map (num_proc=8): 100%|##########| 280/280 [00:00<00:00, 1688.90 examples/s]
Map (num_proc=8): 100%|##########| 280/280 [00:00<00:00, 1702.51 examples/s]
Creating json from Arrow format: 100%|##########| 1/1 [00:00<00:00, 122.59ba/s]
INFO 2025-05-28 13:13:12,968 instructlab.sdg.datamixing:235: Mixed Dataset saved to /root/.local/share/instructlab/datasets/2025-05-27_195532/skills_train_msgs_2025-05-27T19_55_43.jsonl
INFO 2025-05-28 13:13:12,970 instructlab.sdg.datamixing:158: Loading dataset from /root/.local/share/instructlab/datasets/2025-05-27_195532/node_datasets_2025-05-27T19_55_43/knowledge_nmonWrangler_collections_memuse_p07.jsonl ...
Generating train split: 146 examples [00:00, 44377.74 examples/s]
INFO 2025-05-28 13:13:13,187 instructlab.sdg.datamixing:160: Dataset columns: ['messages', 'metadata', 'id', 'unmask']
INFO 2025-05-28 13:13:13,187 instructlab.sdg.datamixing:161: Dataset loaded with 146 samples
Map (num_proc=8): 100%|##########| 146/146 [00:00<00:00, 1136.92 examples/s]
Map (num_proc=8): 100%|##########| 146/146 [00:00<00:00, 1020.67 examples/s]
Creating json from Arrow format: 100%|##########| 1/1 [00:00<00:00, 246.45ba/s]
INFO 2025-05-28 13:13:13,659 instructlab.sdg.datamixing:235: Mixed Dataset saved to /root/.local/share/instructlab/datasets/2025-05-27_195532/knowledge_train_msgs_2025-05-27T19_55_43.jsonl
INFO 2025-05-28 13:13:13,659 instructlab.sdg.generate_data:757: Generation took 62249.69s
ᕦ(òᴗóˇ)ᕤ Data generate completed successfully! ᕦ(òᴗóˇ)ᕤ

```

## Step 9: 모델 Fine-tuning
* 실제 우리의 memuse 데이터를 사용하여 모델을 Fine-tuning 하는 과정.
* 시간이 가장 오래 걸리는 작업.
* 위의 과정에서 .jsonl 파일이 두 개가 만들어 져서 이 파일을 합쳐서 train 하는데 사용 하기로 결정.


생성된 데이터 파일


skills_train_msgs_*.jsonl	: 일반적인 Q&A 형식의 skills 데이터.


knowledge_train_msgs_*.jsonl : knowledge/instruction 유형으로 재구성된 학습 데이터.


```bash
(venv) root@junghyun:~/.local/share/instructlab/datasets/2025-05-27_195532# cat knowledge_train_msgs_2025-05-27T19_55_43.jsonl skills_train_msgs_2025-05-27T19_55_43.jsonl > combined_train.jsonl
(venv) root@junghyun:~/.local/share/instructlab/datasets/2025-05-27_195532# pwd
/root/.local/share/instructlab/datasets/2025-05-27_195532
(venv) root@junghyun:~/.local/share/instructlab/datasets/2025-05-27_195532# ll
total 2260
drwxr-xr-x 5 root root   4096 May 28 13:20 ./
drwxr-xr-x 4 root root   4096 May 27 19:55 ../
-rw-r--r-- 1 root root 992549 May 28 13:20 combined_train.jsonl
drwxr-xr-x 2 root root   4096 May 28 12:41 generated_2025-05-27T19_55_43/
-rw-r--r-- 1 root root    476 May 28 13:13 knowledge_recipe_2025-05-27T19_55_43.yaml
-rw-r--r-- 1 root root 286892 May 28 13:13 knowledge_train_msgs_2025-05-27T19_55_43.jsonl
-rw-r--r-- 1 root root 103986 May 28 13:13 messages_2025-05-27T19_55_43.jsonl
drwxr-xr-x 2 root root   4096 May 28 13:13 node_datasets_2025-05-27T19_55_43/
drwxr-xr-x 3 root root   4096 May 27 19:57 preprocessed_2025-05-27T19_55_43/
-rw-r--r-- 1 root root    476 May 28 13:13 skills_recipe_2025-05-27T19_55_43.yaml
-rw-r--r-- 1 root root 705657 May 28 13:13 skills_train_msgs_2025-05-27T19_55_43.jsonl
-rw-r--r-- 1 root root  86072 May 27 19:57 test_2025-05-27T19_55_43.jsonl
-rw-r--r-- 1 root root  93802 May 28 13:13 train_2025-05-27T19_55_43.jsonl
(venv) root@junghyun:~/.local/share/instructlab/datasets/2025-05-27_195532# ilab model train --data-path /root/.local/share/instructlab/datasets/2025-05-27_195532/combined_train.jsonl
```
진행하다가 너무 오래 걸려서 중간에 interrupt 하였습니다..


## 정리

위 과정을 통해 memuse 데이터를 기반으로 nmon 분석 특화 LLM을 생성할 예정이며, 이 모델을 nmon Wrangler에 통합하여 실 사용에 적합한 분석 기능을 제공할 계획입니다.

> 추가 작업: `cpuuse`, `page`, 기타 nmon 항목에 대한 yaml 및 문서도 별도로 작성 필요

## 이후 과정

주요 단계별 설명

1. ilab data generate
2. ilab model train
3. ilab model convert
4. ilab model serve


1️⃣ ilab data generate
목적 : taxonomy 기반 YAML 파일을 분석해 학습용 .jsonl 데이터 생성
이 단계에서 하는 일 : taxonomy 안의 seed_examples, Q&A 등 분석, instruction-tuned 형식으로 재구성, jsonl 포맷의 학습 데이터 생성 (예: knowledge_train_msgs_*.jsonl)
결과:
/root/.local/share/instructlab/datasets/2025-05-27_XXXXXX/*.jsonl 생성


2️⃣ ilab model train
목적 : 앞에서 생성된 .jsonl 데이터를 이용해 LLM을 미세 조정 (fine-tuning)
이 단계에서 하는 일 : 선택한 모델 (예: mistral, granite 등)을 기반으로 .jsonl 내 문장들을 학습 -> 새로운 체크포인트 생성
결과:
checkpoints/instructlab-<model-name>/...이 단계는 가장 시간이 오래 걸리는 부분


3️⃣ ilab model convert
목적:학습된 모델을 .gguf 포맷으로 변환하여 서빙 가능한 형태로 만들기
이 단계에서 하는 일:checkpoints/에 저장된 모델 파일들을 llama.cpp가 읽을 수 있는 .gguf 포맷으로 변환
결과:
instructlab-<model-name>-trained/<model-name>.gguf 생성


4️⃣ ilab model serve목적: 변환된 .gguf 모델을 로컬 API 서버로 실행
이 단계에서 하는 일 : llama.cpp 서버를 실행, /v1/completions 같은 OpenAI API 호환 인터페이스 제공
결과:
localhost:8000/v1/→ 이제 나만의 fine-tuned LLM에 API 요청을 날릴 수 있게 됨.


🔹요약 


ilab data generate 
역할 : 학습용 JSONL 생성
생성물 : .jsonl


ilab model train
역할 :모델 미세 조정 (Fine-tuning)
생성물 : checkpoints/


ilab model convert
역할 : .gguf 포맷으로 변환
생성물 : *.gguf


ilab model serve
역할 : API로 모델 실행
생성물 : OpenAI 호환 서버