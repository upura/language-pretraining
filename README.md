<div id="top"></div>

<h1 align="center">Pre-training Language Models for Japanese</h1>

<p align="center">
  <a href="https://github.com/retarfi/language-pretraining#licenses">
    <img alt="GitHub" src="https://img.shields.io/badge/license-MIT-brightgreen">
  </a>
  <a href="https://github.com/retarfi/language-pretraining/releases">
    <img alt="GitHub release" src="https://img.shields.io/github/v/release/retarfi/language-pretraining.svg">
  </a>
</p>

This is a repository of pretrained Japapanese transformer-based models.
BERT, ELECTRA, RoBERTa, DeBERTa, and DeBERTaV2 is available.

Our pre-trained models are available in Transformers by Hugging Face: [https://huggingface.co/izumi-lab](https://huggingface.co/izumi-lab).
BERT-small, BERT-base, ELECTRA-small, ELECTRA-small-paper, and ELECTRA-base models trained by Wikipedia or financial dataset is available in this URL.

**issue は日本語でも大丈夫です。**

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#usage">Usage</a>
      <ul>
        <li><a href="#train-tokenizer">Train Tokenizer</a></li>
        <li><a href="#create-dataset">Create Dataset</a></li>
        <li>
          <a href="#training">Training</a>
          <ul>
            <li><a href="#additional-pre-training">Additional Pre-training</a></li>
            <li><a href="#for-electra">For ELECTRA</a></li>
            <li><a href="#training-log">Training Log</a></li>
          </ul>
        </li>
      </ul>
    </li>
    <li>
      <a href="#pre-trained-models">Pre-trained Models</a>
      <ul>
        <li><a href="#model-architecture">Model Architecture</a></li>
      </ul>
    </li>
    <li>
      <a href="#training-data">Training Data</a>
      <ul>
        <li><a href="#wikipedia-model">Wikipedia Model</a></li>
        <li><a href="#financial-model">Financial Model</a></li>
      </ul>
    </li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li>
      <a href="#citation">Citation</a>
      <ul>
        <li><a href="#pre-trained-model">Pre-trained Model</a></li>
        <li><a href="#this-implementation">This Implementation</a></li>
      </ul>
    </li>
    <li><a href="#licenses">Licenses</a></li>
    <li><a href="#related-work">Related Work</a></li>
    <li><a href="#acknowledgements">Acknowledgements</a></li>
  </ol>
</details>


## Usage

### Train Tokenizer

In our pretrained models, the texts are first tokenized by [MeCab](https://taku910.github.io/mecab/) with [IPAdic](https://pypi.org/project/ipadic/) dictionary and then split into subwords by the WordPiece algorithm.

From v2.2.0, [jptranstokenizer](https://github.com/retarfi/jptranstokenizer) is required, which enables to use word tokenizers other than MeCab, such as Juman++, Sudachi, and spaCy LUW.

For subword tokenization, [SentencePiece](https://github.com/google/sentencepiece) is also available for subword algorithm.

```
$ python train_tokenizer.py \
--word_tokenizer mecab \
--input_file corpus.txt \
--model_dir tokenizer/ \
--intermediate_dir ./data/corpus_split/ \
--mecab_dic ipadic \
--tokenizer_type wordpiece \
--vocab_size 32768 \
--min_frequency 2 \
--limit_alphabet 2900 \
--num_unused_tokens 10
```

You can see all the arguments with `python train_tokenizer.py --help`

### Create Dataset

You can train any type of corpus in Japanese.  
When you train with another dataset, please add your corpus name with the line.  
The output directory name is `<dataset_type>_<max_length>_<input_corpus>`.  
In the following case, the output directory name is `nsp_128_wiki-ja`.  
``tokenizer_name_or_path`` will end with vocab.txt for wordpiece and with spiece.model for sentencepiece.

We show 3 examples to create dataset.

- When you use your trained tokenizer:

```
$ python create_datasets.py \
--input_corpus wiki-ja \
--max_length 512 \
--input_file corpus.txt \
--mask_style bert \
--tokenizer_name_or_path tokenizer/vocab.txt \
--word_tokenizer_type mecab \
--subword_tokenizer_type wordpiece \
--mecab_dic ipadic
```

- When you use the tokenizer existing in [HuggingFace Hub](https://huggingface.co/):

```
$ python create_datasets.py \
--input_corpus wiki-ja \
--max_length 512 \
--input_file corpus.txt \
--mask_style roberta-wwm \
--tokenizer_name_or_path izumi-lab/bert-small-japanese \
--load_from_hub
```

### Training

Distributed training is available.
For run command, please see the [PyTorch document](https://pytorch.org/docs/stable/distributed.html#launch-utility) in detail.
In official PyTorch implementation, different batch size between nodes is not available.
We improved PyTorch sampling implementation (utils/trainer_pt_utils.py).

For example, `bert-base-dist` model is defined in parameter.json:

```
"bert-base-dist" : {
    "number-of-layers" : 12,
    "hidden-size" : 768,
    "sequence-length" : 512,
    "ffn-inner-hidden-size" : 3072,
    "attention-heads" : 12,
    "warmup-steps" : 10000,
    "learning-rate" : 1e-4,
    "batch-size" : {
        "0" : 80,
        "1" : 80,
        "2" : 48,
        "3" : 48
    },
    "train-steps" : 1000000,
    "save-steps" : 50000,
    "logging-steps" : 5000,
    "fp16-type": 0,
    "bf16": false
}
```

In this case, node 0 and node 1 have 80 batch sizes and node 2 and node 3 have 48 respectively.
If node 0 has 2 GPUs, each GPU have a 40 batch size.
**10G or higher network speed** is recommended for training with multi-nodes.

`fp16-type` argument specifies which precision mode to use:

- 0: FP32 training
- 1: Mixed Precision
- 2: "Almost FP16" Mixed Precision
- 3: FP16 training

In detail, please see [NVIDIA Apex document](https://nvidia.github.io/apex/amp.html).

`bf16` argument determine whether bfloat16 is enabled or not.  
You cannot use `fp16-type` (1, 2 or 3) and `bf16` (true) simultaneously.

The whole word masking option is also available.

```
# Train with 1 node
$ python run_pretraining.py \
--dataset_dir ./datasets/nsp_128_wiki-ja/ \
--model_dir ./model/bert/ \
--parameter_file parameter.json \
--model_type bert-small \
--tokenizer_name_or_path tokenizer/vocab.txt \
--word_tokenizer_type mecab \
--subword_tokenizer_type wordpiece \
--mecab_dic ipadic \
(--use_deepspeed \)
(--do_whole_word_mask \)
(--do_continue)

# Train with multi-node and multi-process
$ NCCL_SOCKET_IFNAME=eno1 CUDA_VISIBLE_DEVICES=0,1 python -m torch.distributed.launch \
--nproc_per_node=2 --nnodes=2 --node_rank=0 --master_addr="10.0.0.1" \
--master_port=50916 run_pretraining.py \
--dataset_dir ./datasets/nsp_128_wiki-ja/ \
--model_dir ./model/bert/ \
--parameter_file parameter.json \
--model_type bert-small \
--tokenizer_name_or_path tokenizer/vocab.txt \
--word_tokenizer_type mecab \
--subword_tokenizer_type wordpiece \
--mecab_dic ipadic \
(--use_deepspeed \)
(--do_whole_word_mask \)
(--do_continue)
```

#### Additional Pre-training

You can train models additionally with existing pre-trained model.  
For example, `bert-small-additional` model is defined in parameter.json:

```
"bert-small-additional" : {
    "pretrained_model_name_or_path" : "izumi-lab/bert-small-japanese",
    "flozen-layers" : 6,
    "warmup-steps" : 10000,
    "learning-rate" : 5e-4,
    "batch-size" : {
        "-1" : 128
    },
    "train-steps" : 1450000,
    "save-steps" : 100000,
    "fp16-type": 0,
    "bf16": false
}
```

`pretrained_model_name_or_path` specifies a pretrained model in HuggingFace Hub or the path of a pretrained model.  
`flozen-layers` specifies the flozen (not trained) layers of transformer.  
When it is -1, train all layers (including embedding layer).  
When it is 3, train upper (near output layer) 9 layers.

When you train ELECTRA model additionally, you need to specify `pretrained_generator_model_name_or_path` and `discriminator_model_name_or_path` instead of `pretrained_model_name_or_path`.

```
$ python run_pretraining.py \
--tokenizer_name_or_path izumi-lab/bert-small-japanese \
--dataset_dir ./datasets/nsp_128_fin-ja/ \
--model_dir ./model/bert/ \
--parameter_file parameter.json \
--model_type bert-small-additional
```

### For ELECTRA

ELECTRA models generated by run_pretraining.py contain both generator and discriminator.
For general use, separation is needed.

```
$ python extract_electra_model.py \
--input_dir ./model/electra/checkpoint-1000000 \
--output_dir ./model/electra/extracted-1000000 \
--parameter_file parameter.json \
--model_type electra-small \
--generator \
--discriminator
```

In this example, the generator model is saved in `./model/electra/extracted-1000000/generator/` and discriminator model is saved in `./model/electra/extracted-1000000/discriminator/` respectively.

### Training Log

Tensorboard is available for the training log.

## Pre-trained Models

### Model Architecture

Following models are available now:

- BERT
- ELECTRA

The architecture of BERT-small, BERT-base, ELECTRA-small-paper, ELECTRA-base models are the same as those in [the original ELECTRA paper](https://arxiv.org/abs/2003.10555) (ELECTRA-small-paper is described as ELECTRA-small in the paper).
The architecture of ELECTRA-small is the same as that in [the ELECTRA implementation by Google](https://github.com/google-research/electra).

|    Parameter     | BERT-small | BERT-base | ELECTRA-small | ELECTRA-small-paper | ELECTRA-base |
| :--------------: | :--------: | :-------: | :-----------: | :-----------------: | :----------: |
| Number of layers |     12     |    12     |      12       |         12          |      12      |
|   Hidden Size    |    256     |    768    |      256      |         256         |     768      |
| Attention Heads  |     4      |    12     |       4       |          4          |      12      |
|  Embedding Size  |    128     |    512    |      128      |         128         |     128      |
|  Generator Size  |     -      |     -     |      1/1      |         1/4         |     1/3      |
|   Train Steps    |   1.45M    |    1M     |      1M       |         1M          |     766k     |

Other models such as BERT-large or ELECTRA-large are also available in this implementation.
You can also add your original parameters in parameter.json.

### Training Data

Training data are aggregated to a text file.
Each sentence is in one line and a blank line is inserted between documents.

#### Wikipedia Model

The normal models (not financial models) are trained on the Japanese version of Wikipedia, using [Wikipedia dump](https://dumps.wikimedia.org/jawiki/) file as of June 1, 2021.
The corpus file is 2.9GB, consisting of approximately 20M sentences.

#### Financial Model

The financial models are trained on Wikipedia corpus and financial corpus.
The Wikipedia corpus is the same as described above.
The financial corpus consists of 2 corpora:

- Summaries of financial results from October 9, 2012, to December 31, 2020
- Securities reports from February 8, 2018, to December 31, 2020

The financial corpus file is 5.2GB, consisting of approximately 27M sentences.


## Roadmap

See the [open issues](https://github.com/retarfi/language-pretraining/issues) for a full list of proposed features (and known issues).

## Citation


```
@article{Suzuki-etal-2023-ipm,
  title = {Constructing and analyzing domain-specific language model for financial text mining}
  author = {Masahiro Suzuki and Hiroki Sakaji and Masanori Hirano and Kiyoshi Izumi},
  journal = {Information Processing \& Management},
  volume = {60},
  number = {2},
  pages = {103194},
  year = {2023},
  doi = {10.1016/j.ipm.2022.103194}
}
```


## Licenses

The pretrained models are distributed under the terms of the [Creative Commons Attribution-ShareAlike 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

The codes in this repository are distributed under MIT.

## Related Work

- Original BERT model by Google Research Team
  - https://github.com/google-research/bert
- Original ELECTRA model by Google Research Team
  - https://github.com/google-research/electra
- Pretrained Japanese BERT models
  - Autor Tohoku University
  - https://github.com/cl-tohoku/bert-japanese
- ELECTRA training with PyTorch implementation
  - Author: Richard Wang
  - https://github.com/richarddwang/electra_pytorch

## Acknowledgements

This work was supported by JSPS KAKENHI Grant Number JP21K12010, JST-Mirai Program Grant Number JPMJMI20B1, and JST PRESTO Grand Number JPMJPR2267, Japan.
