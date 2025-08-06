## preprocess

负责预训练数据集的生成。

参数设置包含路径参数，预处理参数和掩码参数三部分。所有的“目标”参数针对序列到序列（seq2seq）任务设计，对于ET-BERT理论上都是不需要的。

路径参数：

- `corpus_path` **（必需）** 语料库路径。
- `vocab_path` 词表文件路径。
- `spm_model_path` 分词模型路径。
- `tgt_vocab_path` 目标词表文件路径。
- `tgt_spm_model_path` 目标分词模型路径。
- `dataset_path` 预处理数据集输出路径，默认为"dataset.pt"。

预处理参数：

- `tokenizer` 分词器（`bert` Google BERT/`char` 分割成字符/`space` 按空格分词），默认为`bert`。
- `tgt_tokenizer` 目标分词器，选项同上。
- `processes_num` 将整个数据集分成几份。每一部分会训练1个training step。
- `target` 预训练模型训练目标（`bert` BERT/`lm` 语言模型/`mlm` 掩码语言模型/`bilm` 双向语言模型/`albert` ALBERT/`seq2seq` 序列到序列/`t5` T5/`cls` 文本分类/`prefixlm` 前缀语言模型），默认为`bert`。
- `docs_buffer_size` 内存中缓存的文档数，默认为100000。
- `seq_length` 序列长度，默认为128。
- `tgt_seq_length` 目标序列长度，默认为128。
- `dup_factor` 实例重复增强次数，默认为5。
- `short_seq_prob` 使用短序列的概率，默认为0.1。
- `full_sentences` 是否强制完整句子。
- `seed` 随机数，默认为7。

掩码参数：

- `dynamic_masking` 是否动态掩码。
- `whole_word_masking` 是否对整个词掩码。
- `span_masking` 是否连续掩码。
- `span_geo_prob` 几何分布参数，默认为0.2（控制掩码长度的概率分布）。
- `span_max_length` 最大连续掩码长度，默认为10。

若动态掩码，将dup_factor置为1。调用str2tokenizer来获取tokenizer（必需的1个和可选的1个（若任务为seq2seq））。调用str2dataset构建数据集，将数据集进行保存。按照默认的情况，tokenizer会是一个[uer.utils.tokenizers](uer/utils/tokenizers.py).BertTokenizer，参数为args；dataset会是一个[uer.utils.data](uer/utils/data.py).BertDataset，参数为args，tokenizer.vocab，tokenizer。