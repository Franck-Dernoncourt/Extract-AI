# Enhancing Coherence in Extractive Summarization using LLMs with Human Feedback

Coherence plays a pivotal role in creating a high-quality summary of a document. In recent times, neural extractive summarization has become increasingly popular, yet, most of them ignore the coherence of summaries when extracting sentences. Coherence emerges as a crucial attribute of text summarization since It holds a significant connection to user experience and profoundly influences the comprehensibility of the generated or extracted summaries. Within the context of extractive summarization, coherence becomes quantifiable through the interconnection among sentences and ease of readability. However, attaining this coherence within an extractive summary presents a challenge, given that the extracted sentences don't inherently assure coherence.

<p align="center"> <img width="500" alt="approach" src="https://github.com/Mihir3009/Extract-AI/assets/47143544/b3fef723-bc9a-4192-acb3-9ed366008ef1"> </p>

To improve the coherence in extractive summarization, we approach the concept of coherence within summaries through the lens of user-specific intent. With this user-centric perspective, our approach entails training LLMs using human feedback, a tactic aimed at refining the coherence in the generated summaries. Drawing inspiration from InstructGPT, our strategy involves fine-tuning the model to align with user intent and ensure the production of coherent summaries. Thus, our approach comprises two primary components: firstly, the aggregation of human feedback to enhance the coherence of generated summaries, and secondly, the supervised fine-tuning of open-source LLMs based on user feedback to amplify their capacity for coherent summary generation. Figure provides the schematic represntation of our proposed approach.

## Data

In our pursuit of acquiring human feedback to refine the coherence of extractive summaries, we enlisted the expertise of annotation specialists to accurately annotate data dedicated to our task. 

***Details about data and annotation instruction can be found in the `/data` folder. Please see this folder for further details.***

Full dataset is provided in `/data/annotated_data/data.json` file. The file is provided in following format:

```Json
{
  "type": "[Type of the document]",
  "document": "[Text corresponding to source document]",
  "falcon40B_summary": "[Extractive summary generated by prompting a Falcon-40B-Instruct model]",
  "annotation_1": {
         "coherent_summary": "[Human annotated coherent summary of document]",
         "summary_feedback": "[Feedback on the steps to go from the model summary to the gold summary]",
         "additional_feedback": "[Additional feedback if user wants to provide][Optional]",
         "scores": {
               "relevance": "[int]",
               "coherence": "[int]",
               "consistency": "[int]"
         }
  },
  "annotation_2": "[Similar format as annotation 1]",
  "annotation_3": "[Similar format as annotation 1]"
}
```

## Quick Start

Clone this repository to your project folder:

```shell
git clone git@git.corp.adobe.com:mihirp/Extractive_LLMs.git
```

Create a conda environment to install all required depedencies:

```shell
conda env create -f environment.yml
conda activate myenv
```

Now, we will provide fine-tuning scripts in different settings. All fine-tuning scripts and datasets are given in `/src`.

## Fine-tuning with Quantization

We fine-tune two models using LoRA: (1) Falcon-40B-Instruct and (2) Llama-2-13B. 

All fine-tuning scripts for LoRA are provided at `/src/finetune_quantization/utils`. Please create `train.csv` and `test.csv` file with "text" column for performing this training. Data for training and evaluation are also provided at `/src/finetune_quantization/datasets`. Within the dataset folder, the `raw_data` contains pairs of <Document, Summary>, while the `feedback_data` comprises pairs of <Document + Feedback, Summary>.

### Training with LoRA

To run the Falcon-40B-Instruct model, please use below command:

```shell
export TRANSFORMERS_CACHE=[path where you want to store the model weights]
export CUDA_VISIBLE_DEVICES=[GPUs]

cd ./src/finetune_quantization/utils

python run_model_falcon.py \
        --model_name "tiiuae/falcon-40b-instruct" \
        --train_file [path to train file] \
        --output_dir [path to output directory where trained model get stored] \
        --bf16 True
```

Before running Llama-2-13B model, make sure that you have applied for access from huggingface (https://huggingface.co/meta-llama/Llama-2-13b-hf), and also you have huggingface logic in your system through cli. Please use below command for huggingface cli login:

```shell
huggingface-cli login
```

To run the Llama-2-13B model, please use below command:

```shell
export TRANSFORMERS_CACHE=[path where you want to store the model weights]
export CUDA_VISIBLE_DEVICES=[GPUs]

cd ./src/finetune_quantization/utils

python run_model_llama2.py \
        --model_name "meta-llama/Llama-2-13b-hf" \
        --train_file [path to train file] \
        --output_dir [path to output directory] \
        --bf16 True
```

### Inference with LoRA

To run the inference for both models above, please use below command:

```shell
export TRANSFORMERS_CACHE=[path where you want to store the model weights]
export CUDA_VISIBLE_DEVICES=[GPUs]

cd ./src/finetune_quantization/inference

python run_inference.py \
        --model_name "meta-llama/Llama-2-13b-hf" [provide a model name] \
        --new_model [path to the fine-tuned model] \
        --test_file [path to test file] \
        --output_dir [path where you want to store predictions] \
        --bf16 True
```

### GPU Configuration for LoRA

Falcon-40B-Instruct: 3 GPUs A100 (80GB)

Llama-2-13B: 2 GPUs A100 (80GB)

## Full Parametric Training

We only fine-tune Llama-2-7b-hf model for full parametric training. Below is a step-by-step guide to fine-tune it:

1. `git clone https://github.com/facebookresearch/llama-recipes.git`
2. Go to the llama-reciepe Github, please make sure to install from source (https://github.com/facebookresearch/llama-recipes#install-from-source)
3. Once you install from the source, replace the samsum dataset file from `/src/llama_recipes/datasets/samsum_dataset.py` with the file provided in `/src/finetune_full_param/dataset/samsum.py`.
4. Make sure to change line 15 and 18 with path of relevant training and testing files.
5. Run below command to perform training:

```shell
export TRANSFORMERS_CACHE=[path where you want to store the model weights]
export CUDA_VISIBLE_DEVICES=[GPUs]

torchrun --nnodes 1 --nproc_per_node 4 ./examples/finetuning.py --enable_fsdp --model_name meta-llama/Llama-2-7b-hf --dist_checkpoint_root_folder [path to output folder] --dist_checkpoint_folder [name of output folder] --use_fast_kernels 
```

6. Once trainng is completed, please use below command for model conversion to Hugging Face:

```shell
python -m llama_recipes.inference.checkpoint_converter_fsdp_hf --fsdp_checkpoint_path {path}/Llama-2-7b-hf --consolidated_model_path {save-path}/Llama-2-7b-hf-converted/  --HF_model_path_or_name meta-llama/Llama-2-7b-hf
```

7. Peform inference using default script.

### GPU Configuration

All training and evaluation is conducted on 4 GPUs A100 (40GB)


## Fine-tune Encoder Models

We fine-tune three encoder-based LLMs: (1) T5-large (https://huggingface.co/t5-large), (2) FLAN-T5-large (https://huggingface.co/google/flan-t5-large), and (3) Tk-instruct-large-def-pos (https://huggingface.co/allenai/tk-instruct-large-def-pos). 

We fine-tuned these models using `deepspeed`. Config file for deepspeed will be found at `/src/finetune_enc/stage3.config`.

### Training and Evaluation

This script computes `Rouge-L` metric by default.

```shell
set -x

export TRANSFORMERS_CACHE=/sensei-fs/users/mihirp/hf_cache
export CUDA_DEVICE_ORDER="PCI_BUS_ID"
export NCCL_P2P_DISABLE=1

port=$(shuf -i25000-30000 -n1)

deepspeed --include localhost:0,1,2,3 --master_port $port run_model_enc.py \
        --model_name_or_path [Provide name or path of the model, you want to finetune. To finetune on BART use - "facebook/bart-base" (default: None)] \
        --cache_dir [Provide path to store transformer cache (default: home directory)] \
        --do_train  [Provide True or False (default: False)] \
        --do_eval   [Provide True or False (default: False)] \
        --do_predict  [Provide True or False (default: False)] \
        --train_file  [Path of an optional input training data file (a JSON or CSV file), if do_train argument is true. (default: None)] \
        --validation_file  [Path of an optional input evaluation data file to evaluate the metrics (rouge) on (a JSON or CSV file), if do_eval argument is true. (default: None)] \
        --test_file [Path of an optional input test data file to evaluate the metrics (rouge) on (a JSON or CSV file), if do_predict argument is true (default: None)] \
        --output_dir [The output directory where the model predictions and checkpoints will be written. (default: None)] \
        --per_device_train_batch_size   [Batch size per GPU/TPU core/CPU for training. (default: 8)] \
        --per_device_eval_batch_size    [Batch size per GPU/TPU core/CPU for evaluation. (default: 8)] \
        --gradient_accumulation_steps   [Number of updates steps to accumulate before performing a backward/update pass. (default: 1)] \
        --predict_with_generate   [Whether to use generate to calculate generative metrics (ROUGE, BLEU). (default: False)] \
        --save_strategy   [The checkpoint save strategy to use. (no, steps, epoch) (default: steps)] \
        --max_source_length 2048 \
        --max_target_length 512 \
        --learning_rate 5e-05 \
        --num_train_epochs [Number of training epochs (default: 3)] \
        --deepspeed [path to deepspeed config file]
```

### GPU Configuration

All training and evaluation is conducted on 4 GPUs A100 (40GB)

### Point of contact 

Mihir Parmar (Adobe Intern): mihirp@adobe.com \
Hanieh Deilamsalehy: deilamsa@Adobe.com

## License

The code and model are licensed under the [Adobe Research License](./LICENSE.md). The license prohibits commercial use and allows non-commercial research use. The DialogueSum dataset is under CC BY-NC-SA 4.0 license. The DebateSum dataset is under MIT license. The MeetingBank dataset is under CC-BY-NC-ND 4.0 International license.
