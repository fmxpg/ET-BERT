## misc

工具类。

### count_lines

统计参数所指向文件的行数，实现是通过对换行符的计数实现的。

### flip

将张量x按照dim维度翻转。

[1 2; 3 4] --1--> [3 4; 1 2]

[1 2; 3 4] --2--> [2 1; 4 3]

## vocab

生成词汇集的类文件。

### Vocab 类

#### \_\_init\_\_

初始化w2i，w2c 2个字典和i2w 1个列表，和reserved_vocab.txt的位置。这个文件在models文件夹中。

#### load

参数vocab_path, is_quiet=False。

打开vocab_path，逐行进行读取，去除两端的换行符并且智能根据空格符分割，取第一个元素。将元素和行号作为键和值写入w2i，将元素写入i2w。

若is_quiet=False，输出词库的大小（i2w的长度）。

#### save

将i2w逐行写入参数对应的文件。

#### get

获取参数对应的索引。

#### \_\_len\_\_

返回i2w的长度。

#### worker

从语料库的部分行创建词汇库。实质上是静态方法。

对于corpus的start行到end-1行，调用参数的tokenizer.tokenize对一行进行分词，获得tokens。对每一个token，若不在w2i中，则将token加入w2i和w2c作为key，值为len(i2w)和1，并且将token加入i2w；若存在，将w2c对应栏目+1。

最终返回w2i，i2w，w2c，分别为这一部分词到索引的字典，索引到词的列表和词的计数字典。

#### union

将一个不同worker工作返回值的列表进行整理，归拢到一个w2i，i2w，w2c三元组中。实质上是静态方法。

#### build

从语料库创建词汇库。

使用multiprocessing.Pool进行调度，并行进行多个worker方法，对语料库的行进行分工读取。使用union方法将不同worker的工作归拢。 sorted_w2c得到词频统计排序的token和计数的二元组组成的列表。

从初始化设定的reserved_vocab_path读取保留词元，写入self.i2w，再根据其写入self.w2i和self.w2c，计数被设置为-1。遍历sorted_w2c，若已经遍历到计数小于参数min_count的项（低频词）则退出循环，否则将其加入self的三元组。最后会得到整个文件非低频词和代用词组成的vocab。

## tokenizers

SPIECE_UNDERLINE为特殊下划线字符“▁”的utf-8字节编码。

### Tokenizer 类

三种Tokenizer的父类，用于实现分词。

#### \_\_init\_\_

读入args。[有关参数的描述，点这里](../../README_preprocess.md)。根据第二个参数判断是否为目标tokenizer，从而选取args中的spm_model_path（分词模型路径）和vocab_path（词表文件路径）。如果设定了spm_model_path，导入sentencepiece.SentencePieceProcessor作为模型，生成词汇集；如果没有设定，则根据vocab_path，使用uer.utils.vocab.Vocab实现词汇集生成。生成翻转的词汇集。也就是说，spm_model_path和vocab_path只有一个会被使用。最终初始化的vocab和inv_vocab是两个字典。

（models文件夹中有xlmroberta_spm.model）

#### tokenize

抛出 NotImplementedError

#### convert_tokens_to_ids 和 convert_ids_to_tokens

调用sp_model或convert_by_vocab获取tokens对应的ids或反之。

### CharTokenizer

#### tokenize(override)

将输入文本完全拆分成单个字符。若使用词典，不在词典中的部分使用`[UNK]`替代。

### SpaceTokenizer

#### tokenize(override)

将输入文本按照空格进行拆分。若使用词典，不在词典中的部分使用`[UNK]`替代。

### preprocess_text

