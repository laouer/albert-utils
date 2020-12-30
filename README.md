# Finetune Albert on french corpus

## Introduction

ALBERT is the "A Lite" version of BERT, a popular unsupervised language representation learning algorithm. ALBERT uses parameter reduction techniques that allow for large-scale configurations, overcoming previous memory limitations and achieving better performance  model degradation behavior.

For a detailed technical description of the algorithm, see the article :

[ALBERT: A Lite BERT for Self-supervised Learning of Language Representations](https://arxiv.org/abs/1909.11942) (Zhenzhong Lan, Mingda Chen, Sebastian Goodman, Kevin Gimpel, Piyush Sharma, Radu Soricut) and the official repository [Google ALBERT](https://github.com/google-research/ALBERT)

Google researchers have introduced three significant innovations with ALBERT. [1](https://medium.com/syncedreview/googles-albert-is-a-leaner-bert-achieves-sota-on-3-nlp-benchmarks-f64466dd583)

* Factorized integration parameterization: The researchers isolated the size of hidden layers from the size of vocabulary embedding by projecting one-hot vectors into a smaller embedding space and then into the hidden space. This made it easier to increase the size of hidden layers without significantly increasing the size of vocabulary embedding parameters.

* Sharing of parameters between layers: The researchers chose to share all parameters between layers to prevent the parameters from increasing with the depth of the network. As a result, the large ALBERT model has about 18x fewer parameters than the large BERT model.

* Loss of consistency between sentences: In the BERT document, Google proposed a technique for predicting the next sentence to improve the performance of the model in downstream tasks, but subsequent studies showed that this technique was not reliable. The researchers used a loss of sentence order prediction (SOP) to model inter-sentence consistency in ALBERT, which allowed the new model to perform better in multi-sentence coding tasks.

## Data preparation

To finetune Albert, we need to get the targetted corpus (here we detail the wikipedia data preparation)

* Download the French wikipedia corpus from [Wikipedia](https://dumps.wikimedia.org/)

* Data is preprocessed and extracted using [WikiExtractor](https://github.com/attardi/wikiextractor)

* Training SentencePiece model for producing vocab file, I used 30000 words from this model on French wikipedia corpus. The SPM model was trained by [SentencePiece](https://github.com/google/sentencepiece)
  * To train spm vocab model You need to build&Install sentencePiece not only the python module

### Preprocessing the wiki corpus dump

First step is to get your corpus: I have choosen the wikipedia corpus as it globally clean and rich. <https://dumps.wikimedia.org/mirrors.html>

1. Get your the articles-multistream (from a suitable mirror) example:

    ```bash
    wget https://ftp.acc.umu.se/mirror/wikimedia.org/dumps/frwiki/20201220/frwiki-20201220-pages-articles-multistream.xml.bz2 
    ```

2. We need to extract and clean the corpus,. So after installing wikiextractor (see the above link for details), run the following command on the wikipedia dump:

    ```bash
    python -m wikiextractor.WikiExtractor -o Output -b 100M \
         frwiki-20201220-pages-articles-multistream.xml.bz2
    ```

   * It will generate something like 480 xml files
   * For training we will need just the text of articles, so running the next commandline Will :
     * Concat all articles in one file
     * keep only articles text removing the html tags
     * Will keep lines that are longer than 5 words (really ampirical approach)
     * NB: we keep empty lines to separate documents / paragraphs

    ```bash
        awk '!/^<.*doc/ && (NF>=4 || NF==0)' $(find Output/ -type f) > wikip│anouar@corpus:~/sources/albert/working$ wc -l wikipedia_fr.txt 
    ```

* We will get some 24,4 Million lines: it's huge!

> Running this 2 steps , on my MacPro 16 (2020) with 6 core Intel i7, takes roughly 1h30

### Generate the spm vocal files

Train the spm files (As said above, you need to Build&Install to get the spm_train command). You can look to the rich [options](https://github.com/google/sentencepiece/blob/master/doc/options.md) of spm_train. 

```bash
spm_train --input=wikipedia_fr.txt --model_prefix=wikifr2M\
          --vocab_size=30000 \
          --num_threads=12 --input_sentence_size=2000000 \
          --pad_id=0 --unk_id=1 --bos_id=-1 --eos_id=-1 \
          --control_symbols="[CLS],[SEP],[MASK],(,),-,£,$,€"
```

Once run, It will generate wikifr2M.model and wikifr2M.vocab files that will be used later for training.

* Ex: Head from wikifr2M.vocab

    ```text
    <pad>	0
    <unk>	0
    [CLS]	0
    [SEP]	0
    [MASK]	0
    (	    0
    )	    0
    -       0
    £	    0
    $	    0
    €	    0
    ▁	   -2.94116
    ▁de	   -3.13249
    ...

    ```

## Pretraining

### Get needed tools

To run Albert finetuning, You will need tensorflow 1.5 and moreover you will need albert google code

```bash
git clone https://github.com/google-research/albert albert
```

### Creating data for pretraining

We WILL train ALBERT model on config version 2 of base and large the Other configurations in the folder [config](config/base/)

* Base Config

    ```Json
    {
    "attention_probs_dropout_prob": 0,
    "hidden_act": "gelu",
    "hidden_dropout_prob": 0,
    "embedding_size": 128,
    "hidden_size": 768,
    "initializer_range": 0.02,
    "intermediate_size": 3072,
    "max_position_embeddings": 512,
    "num_attention_heads": 12,
    "num_hidden_layers": 12,
    "num_hidden_groups": 1,
    "net_structure_type": 0,
    "gap_size": 0,
    "num_memory_blocks": 0,
    "inner_group_num": 1,
    "down_scale_factor": 1,
    "type_vocab_size": 2,
    "vocab_size": 30000
    }```

* Large Config

    ```Json
    {
    "attention_probs_dropout_prob": 0,
    "hidden_act": "gelu",
    "hidden_dropout_prob": 0,
    "embedding_size": 128,
    "hidden_size": 1024,
    "initializer_range": 0.02,
    "intermediate_size": 4096,
    "max_position_embeddings": 512,
    "num_attention_heads": 16,
    "num_hidden_layers": 24,
    "num_hidden_groups": 1,
    "net_structure_type": 0,
    "gap_size": 0,
    "num_memory_blocks": 0,
    "inner_group_num": 1,
    "down_scale_factor": 1,
    "type_vocab_size": 2,
    "vocab_size": 30000
    }```

### Create tfrecord for training data

As we said above the wikipedia_fr.txt is huge and we have to handle this problem, unless we will stuck. 
let's start by splitting the wikipedia_fr.txt to mini shards

```bash 
VOCAB_MODEL_FILE="wikifr2M.vocab"
SPM_MODEL_FILE="wikifr2M.model"
DATA_TASK_DIR="shard-wiki"
mkdir -p $DATA_TASK_DIR
split -d -l 1000000 "wikipedia_fr.txt" "$DATA_TASK_DIR/wikipedia_fr_shard"

python -m albert.create_pretraining_data \
    --dupe_factor=10 \
    --vocab_file=$VOCAB_MODEL_FILE \
    --spm_model_file=$SPM_MODEL_FILE \
    --input_file="$DATA_TASK_DIR/wikipedia_fr_shard00" \
    --output_file="$DATA_TASK_DIR/wikipedia_fr_shard00.tf_record" 
```
Be Patient it could take a long time