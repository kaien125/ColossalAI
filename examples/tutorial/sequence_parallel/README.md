# Sequence Parallelism with BERT

In this example, we implemented BERT with sequence parallelism. Sequence parallelism splits the input tensor and intermediate
activation along the sequence dimension. This method can achieve better memory efficiency and allows us to train with larger batch size and longer sequence length.

Paper: [Sequence Parallelism: Long Sequence Training from System Perspective](https://arxiv.org/abs/2105.13120)

## How to Prepare WikiPedia Dataset

First, let's prepare the WikiPedia dataset from scratch. To generate a preprocessed dataset, we need four items:
1. raw WikiPedia dataset
2. wikipedia extractor (extract data from the raw dataset)
3. vocabulary file
4. preprocessing scripts (generate final data from extracted data)

For the preprocessing script, we thank Megatron-LM for providing a preprocessing script to generate the corpus file.

```python
# download raw data
mkdir data && cd ./data
wget https://dumps.wikimedia.org/enwiki/latest/enwiki-latest-pages-articles.xml.bz2

# install wiki extractor
git clone https://github.com/FrankLeeeee/wikiextractor.git
pip install ./wikiextractor

# extractmodule
wikiextractor --json enwiki-latest-pages-articles.xml.bz2
cat text/*/* > ./corpus.json
cd ..

# download vocab file
mkdir vocab && cd ./vocab
wget https://s3.amazonaws.com/models.huggingface.co/bert/bert-large-uncased-vocab.txt
cd ..

# preprocess some data
git clone https://github.com/NVIDIA/Megatron-LM.git
cd ./Megatron-LM
python tools/preprocess_data.py \
    --input ../data/corpus.json \
    --output-prefix my-bert \
    --vocab ../vocab/bert-large-uncased-vocab.txt \
    --dataset-impl mmap \
    --tokenizer-type BertWordPieceLowerCase \
    --split-sentences \
    --workers 24
```

After running the preprocessing scripts, you will obtain two files:
1. my-bert_text_sentence.bin
2. my-bert_text_sentence.idx

If you happen to encouter `index out of range` problem when running Megatron's script,
this is probably because that a sentence starts with a punctuation and cannot be tokenized. A work-around is to update `Encoder.encode` method with the code below:

```python
class Encoder(object):
    def __init__(self, args):
        ...

    def initializer(self):
        ...

    def encode(self, json_line):
        data = json.loads(json_line)
        ids = {}
        for key in self.args.json_keys:
            text = data[key]
            doc_ids = []

            # lsg: avoid sentences which start with a punctuation
            # as it cannot be tokenized by splitter
            if len(text) > 0 and text[0] in string.punctuation:
                text = text[1:]

            for sentence in Encoder.splitter.tokenize(text):
                sentence_ids = Encoder.tokenizer.tokenize(sentence)
                if len(sentence_ids) > 0:
                    doc_ids.append(sentence_ids)
            if len(doc_ids) > 0 and self.args.append_eod:
                doc_ids[-1].append(Encoder.tokenizer.eod)
            ids[key] = doc_ids
        return ids, len(json_line)
```

## How to Train with Sequence Parallelism

We provided `train.py` for you to execute training. Before invoking the script, there are several
steps to perform.

### Step 1. Set data path and vocab path

At the top of `config.py`, you can see two global variables `DATA_PATH` and `VOCAB_FILE_PATH`.

```python
DATA_PATH = <data-path>
VOCAB_FILE_PATH = <vocab-path>
```

`DATA_PATH` refers to the path to the data file generated by Megatron's script. For example, in the section above, you should get two data files (my-bert_text_sentence.bin and my-bert_text_sentence.idx). You just need to `DATA_PATH` to the path to the bin file without the file extension.

For example, if your my-bert_text_sentence.bin is /home/Megatron-LM/my-bert_text_sentence.bin, then you should set

```python
DATA_PATH = '/home/Megatron-LM/my-bert_text_sentence'
```

The `VOCAB_FILE_PATH` refers to the path to the vocabulary downloaded when you prepare the dataset
(e.g. bert-large-uncased-vocab.txt).

### Step 3. Make Dataset Helper

Build BERT dataset helper. Requirements are `CUDA`, `g++`, `pybind11` and `make`.

```python
cd ./data/datasets
make
```

### Step 3. Configure your parameters

In the `config.py` provided, a set of parameters are defined including training scheme, model, etc.
You can also modify the ColossalAI setting. For example, if you wish to parallelize over the
sequence dimension on 8 GPUs. You can change `size=4` to `size=8`. If you wish to use pipeline parallelism, you can set `pipeline=<num_of_pipeline_stages>`.

### Step 4. Invoke parallel training

Lastly, you can start training with sequence parallelism. How you invoke `train.py` depends on your
machine setting.

- If you are using a single machine with multiple GPUs, PyTorch launch utility can easily let you
  start your script. A sample command is like below:

  ```bash
    colossalai run --nproc_per_node <num_gpus_on_this_machine> --master_addr localhost --master_port 29500 train.py
  ```

- If you are using multiple machines with multiple GPUs, we suggest that you refer to `colossalai
  launch_from_slurm` or `colossalai.launch_from_openmpi` as it is easier to use SLURM and OpenMPI
  to start multiple processes over multiple nodes. If you have your own launcher, you can fall back
  to the default `colossalai.launch` function.