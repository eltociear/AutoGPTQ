# AutoGPTQ
An easy-to-use model quantization package with user-friendly apis, based on GPTQ algorithm.

## News or Update
- 2023-04-23 - (Update) - Support evaluation on multiple (down-stream) tasks such as: language-modeling, text-classification, text-summarization.
- 2023-04-22 - (News) - qwopqwop200's [AutoGPTQ-triton](https://github.com/qwopqwop200/AutoGPTQ-triton) provides faster speed to integrate with quantized model, for everyone who can access to triton, try and enjoy yourself!
- 2023-04-20 - (News) - AutoGPTQ is automatically compatible with Stability-AI's newly released `gpt_neox` type model family [StableLM](https://github.com/Stability-AI/StableLM).
- 2023-04-16 - (Update) - Support quantization and inference for `bloom`, `gpt_neox`, `gptj`, `llama` and `opt`.

## Installation
### Install from source
First, install `torch` with minimum version of 1.13.0 following [pytorch installation guide](https://pytorch.org/get-started/locally/)

Second, clone the source code:
```shell
git clone https://github.com/PanQiWei/AutoGPTQ.git && cd AutoGPTQ
```
Then, install from source:
```shell
pip instal .
```
For some people want to try LLaMa and whose `transformers` version not meet the newest one that supports it, using:
```shell
pip install .[llama]
```

## Supported Models
Currently, `auto_gptq` supports: `bloom`, `gpt_neox`, `gptj`, `llama` and `opt`; more CausalLMs will come soon!

## Supported Evaluation Tasks
Currently, `auto_gptq` supports: `LanguageModelingTask`, `SequenceClassificationTask` and `TextSummarizationTask`; more Tasks will come soon!

## Usage

### Basic
> warning: this is just a show case of the usage of basic apis in AutoGPTQ, which uses only one sample to quantize a much small model, thus may not performs as well as expected in LLMs.

Below is an example for the simplest use of auto_gptq: 
```python
from transformers import AutoTokenizer, TextGenerationPipeline
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig


pretrained_model_dir = "facebook/opt-125m"
quantized_model_dir = "opt-125m-4bit"


tokenizer = AutoTokenizer.from_pretrained(pretrained_model_dir, use_fast=True)
example = tokenizer(
    "auto_gptq is a useful tool that can automatically compress model into 4-bit or even higher rate by using GPTQ algorithm.",
    return_tensors="pt"
)

quantize_config = BaseQuantizeConfig(
    bits=4,  # quantize model to 4-bit
    group_size=128,  # it is recommended to set the value to 128
)

# load un-quantized model, the model will always be force loaded into cpu
model = AutoGPTQForCausalLM.from_pretrained(pretrained_model_dir, quantize_config)

# quantize model, the examples should be list of dict whose keys can only be "input_ids" and "attention_mask" 
# with value under torch.LongTensor type.
model.quantize([example])

# save quantized model
model.save_quantized(quantized_model_dir)

# save quantized model using safetensors
model.save_quantized(quantized_model_dir, use_safetensors=True)

# load quantized model, currently only support cpu or single gpu
model = AutoGPTQForCausalLM.from_quantized(quantized_model_dir, device="cuda:0")

# inference with model.generate
print(tokenizer.decode(model.generate(**tokenizer("auto_gptq is", return_tensors="pt").to("cuda:0"))[0]))

# or you can also use pipeline
pipeline = TextGenerationPipeline(model=model, tokenizer=tokenizer)
print(pipeline("auto_gptq is")[0]["generated_text"])
```

### Customize Model
Below is an example to extend `auto_gptq` to support `OPT` model, as you will see, it's very easy:
```python
from auto_gptq.modeling import BaseGPTQForCausalLM


class OPTGPTQForCausalLM(BaseGPTQForCausalLM):
    # chained attribute name of transformer layer block
    layers_block_name = "model.decoder.layers"
    # chained attribute names of other nn modules that in the same level as the transformer layer block
    outside_layer_modules = [
        "model.decoder.embed_tokens", "model.decoder.embed_positions", "model.decoder.project_out",
        "model.decoder.project_in", "model.decoder.final_layer_norm"
    ]
    # chained attribute names of linear layers in transformer layer module
    # normally, there are four sub lists, for each one the modules in it can be seen as one operation, 
    # and the order should be the order when they are truly executed, in this case (and usually in most cases), 
    # they are: attention q_k_v projection, attention output projection, MLP project input, MLP project output
    inside_layer_modules = [
        ["self_attn.k_proj", "self_attn.v_proj", "self_attn.q_proj"],
        ["self_attn.out_proj"],
        ["fc1"],
        ["fc2"]
    ]

    @staticmethod
    # the overriding of this method may not necessary for most other models
    def _resize_attention_mask(attention_mask):
        attention_mask = [each.unsqueeze(1) for each in attention_mask]
        return attention_mask
```
After this, you can use `OPTGPTQForCausalLM.from_pretrained` and other functions

### Evaluation on Downstream Tasks
One can use tasks defined in `auto_gptq.eval_tasks` to evaluate model's performance on specific down-stream task before and after quantization.

The predefined tasks support all causal-language-models implemented in [huggingface transformers](https://github.com/huggingface/transformers) and in this project.

Below is an example to evaluate `EleutherAI/gpt-j-6b` on sequence-classification task using `cardiffnlp/tweet_sentiment_multilingual` dataset:
```python
from argparse import ArgumentParser
from functools import partial

import datasets
from transformers import AutoTokenizer, AutoModelForCausalLM, GenerationConfig

from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig
from auto_gptq.eval_tasks import SequenceClassificationTask


MODEL = "EleutherAI/gpt-j-6b"
DATASET = "cardiffnlp/tweet_sentiment_multilingual"
TEMPLATE = "Question:What's the sentiment of the given text? Choices are {labels}.\nText: {text}\nAnswer:"
ID2LABEL = {
    0: "negative",
    1: "neutral",
    2: "positive"
}
LABELS = list(ID2LABEL.values())


def ds_refactor_fn(samples):
    text_data = samples["text"]
    label_data = samples["label"]

    new_samples = {"prompt": [], "label": []}
    for text, label in zip(text_data, label_data):
        prompt = TEMPLATE.format(labels=LABELS, text=text)
        new_samples["prompt"].append(prompt)
        new_samples["label"].append(ID2LABEL[label])

    return new_samples


#  model = AutoModelForCausalLM.from_pretrained(MODEL).eval().half().to("cuda:0")
model = AutoGPTQForCausalLM.from_pretrained(MODEL, BaseQuantizeConfig())
tokenizer = AutoTokenizer.from_pretrained(MODEL)

task = SequenceClassificationTask(
        model=model,
        tokenizer=tokenizer,
        classes=LABELS,
        data_name_or_path=DATASET,
        prompt_col_name="prompt",
        label_col_name="label",
        **{
            "num_samples": 1000,  # how many samples will be sampled to evaluation
            "sample_max_len": 1024,  # max tokens for each sample
            "block_max_len": 2048,  # max tokens for each data block
            "load_fn": partial(datasets.load_dataset, name="english"),  # function to load dataset, one must only accept data_name_or_path as input and return datasets.Dataset
            "preprocess_fn": ds_refactor_fn,  # function to preprocess dataset, which is used for datasets.Dataset.map, must return Dict[str, list] with only two keys: [prompt_col_name, label_col_name]
            "truncate_prompt": False  # truncate label when sample's length exceed sample_max_len
        }
    )

# note that max_new_tokens will be automatically specified internally based on given classes
print(task.run())

# self-consistency
print(
    task.run(
        generation_config=GenerationConfig(
            num_beams=3,
            num_return_sequences=3,
            do_sample=True
        )
    )
)
```

### More Examples
For more examples, please turn to [examples](examples/README.md)

## Side Notes
### VRAM
Currently, I put everything (data, model, etc.) into CPU util one is required to be used or executed on GPU (and will back to CPU once the execution finished). Though I didn't run any benchmark to this date, but the maximum VRAM usage for GPTJ is about 6GB, which may be considered as a reference.

## Acknowledgement
- Specially thanks **Elias Frantar**, **Saleh Ashkboos**, **Torsten Hoefler** and **Dan Alistarh** for proposing **GPTQ** algorithm and open source the [code](https://github.com/IST-DASLab/gptq).
- Specially thanks **qwopqwop200**, for code in this project that relevant to quantization are mainly referenced from [GPTQ-for-LLaMa](https://github.com/qwopqwop200/GPTQ-for-LLaMa/tree/cuda).
