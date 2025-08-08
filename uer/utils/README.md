## act_fun

作者实现了一些非线性函数，但是如文件名所表示的，这些函数全部没有使用，所以这里也不写注释。

## seed

随机数设置。

同时改变random，os.environ['PYTHONHASHSEED']，np.random，torch.manual_seed和torch.cuda.manual_seed的设置，保证实验可复现。

## misc

工具类。

### count_lines

统计参数所指向文件的行数，实现是通过对换行符的计数实现的。

### flip

将张量x按照dim维度翻转。

[1 2; 3 4] --1--> [3 4; 1 2]

[1 2; 3 4] --2--> [2 1; 4 3]

## config

### load_hyperparam

从指定文件读取参数，并且写入传入的args中。

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

将outputs赋值为inputs。

若remove_space=True，对inputs进行清理，将所有空白符分隔变为单个空格。

若python2且outputs为字符串，对outputs进行类型转换（若python3则都将视为str，不需要这一过程）。使用兼容性分解（NFKD）对outputs进行处理，再去除组合符号。

若lower=True，返回outputs的转换为小写的字符串，否则返回原字符串。

### encode_pieces

需要使用sp_model。

若python2且text为字符串，转化为字节串。

若sample=True，将text采样拆分，否则全部拆分成片。对于每一片piece，调用printable_text将其重新转化为字符串。若字符串由数字+逗号结尾，则去除逗号，将前缀的特殊字符替换成空，写入cur_pieces；若piece不以特殊字符开头但cur_pieces以特殊字符开头，则去除cur_pieces中的特殊字符，将逗号作为单独的token重新写入cur_pieces，最后new_pieces将cur_pieces的所有内容写入。若字符串不以数字+逗号结尾，则直接作为token写入new_pieces。

若为python2且return_unicode=True，将new_pieces转化为字符串列表。

### encode_ids

需要使用sp_model。

调用encode_pieces将传入的text转化为pieces，再使用模型查询id，返回id列表。

### convert_to_unicode

使用多条件判断确保输入转化为unicode text。

### printable_text

使用多条件判断确保输入转化为可被print调用的形式。

### convert_by_vocab

使用词典将键列表转化为值列表。

### convert_tokens_to_ids 和 convert_ids_to_tokens

调用convert_by_vocab，实现tokens和ids的相互转化。

### whitespace_tokenize

使用默认方式split切分文本为tokens。

### BertTokenizer 类

#### \_\_init\_\_

执行父类的初始化函数。若没有指定spm_model_path，则指定BasicTokenizer和WordpieceTokenizer，二者都是自建类。

#### tokenize

若有sp_model，使用其调用encode_pieces分词。若没有，使用BasicTokenizer将text分解为token，再使用WordpieceTokenizer分解为sub_token，返回所有sub_token列表。

### BasicTokenizer 类

#### \_\_init\_\_

设置初始参数do_lower_case，判断是否将字符转为小写。

#### tokenize

依次调用convert_to_unicode、self._clean_text、self._tokenize_chinese_chars处理输入text。使用whitespace_tokenize将其拆分为orig_tokens。对其中的token，若do_lower_case=True则调用lower和_run_strip_accents（去除标记符）对其进行处理。对所有token，调用_run_split_on_punc进行处理。最后返回所有token列表。

#### _run_strip_accents

将输入的text进行兼容性分解（NFD），对字符遍历，忽视类别为Mn（Mark Nonspacing）的字符，重组后返回。

#### _run_split_on_punc

按照分隔符（调用_is_punctuation判断）将输入text拆分为词语列表。

#### _tokenize_chinese_chars

对text进行遍历，若为汉字，则在其两端添加空格，处理成单词。

#### _is_chinese_char

判断字符是否在CJK Unicode block中（是否为广义的汉字）。实质上是静态方法。

#### _clean_text

将无效字符，控制字符从text中除去；将所有空白符都视为空格，输出清理后的text。

### WordpieceTokenizer 类

#### \_\_init\_\_

设置初始参数vocab，unk_token和max_input_chars_per_word。

#### tokenize

将单词拆解为词元。首先将text转化为unicode。对于每一个由空格分隔的token（理论上这里应该已经没有空格了，但是单独使用WordpieceTokenizer是可能的），将其拆解为字符列表。若长度超出max_input_chars_per_word，直接整体标记为unk_token。使用贪心算法进行匹配，尝试按照可能的最大长度在vocab中寻找，有则添加到sub_tokens，全部找到则输出sub_tokens，只要有一个未找到则is_bad=True，将编码设置为unk_token并输出。

### _is_whitespace

判断字符是否是空白符（空格、制表符、换行符、回车符或unicodedata.category Zs）

### _is_control

判断字符是否是控制字符（不是制表符、换行符或回车符，unicodedata.category Cc或Cf）

### _is_punctuation

判断字符是否是分隔符（**所有非数字和字母的ASCII字符**或unicodedata.category P）

## data （data-edit，data-new）

设置数据集。

data和data-edit的区别位于BertDataset worker方法的文字说明不同，MLmDataLoader删除了try except语句，最主要的区别是create_ins_from_doc方法中，edit添加了关于tov的处理。

data-new是前二者的一个中间版本，数据处理逻辑上有一些问题。我们主要关注data，因为项目默认使用的是data。

### mask_seq

BERT数据生成的核心函数，实现添加掩码的功能。

从tokenizer中获取vocab。对src进行处理，去除填充id（常量：0）。

调用create_index，获取tokens_index（掩码位置列表）和新的src_no_pad。若src_no_pad长度小于src，则填充PAD_ID（将填充重新补全）回src。随机打乱tokens_index，设定需要掩码计算的个数num_to_predict，初始化tgt_mlm列表。

对于tokens_index中的每一个index_set，若当前掩码数已达标则退出循环，若没有，则分三种情况：

若whole_word_masking=True（全词掩码），获取其位置为1，掩码长度为mask_len。如果即将掩码数达标则跳过。对于每一个mask_len内的位置，获取其token，将位置和token写入tgt_mlm。随机一个概率，有80%可能将src对应位置更换为MASK，10%更换为非特殊的其他token，（10%不变）。

若span_masking=True（跨度掩码），基本同上，但同时判断整个区间，同时对整个区间掩码。

若都不是，则无视tokens_index，对每一个Token进行这种替换和记录。

返回值为掩码修改过的src和记录了修改前位置和内容对的列表tgt_mlm。

### create_index

初始化tokens_index，获取vocab。初始化span_end_position=-1。

若whole_word_masking=True（全词掩码），首先判断有无CLS和SEP标记，有则删除并记录；使用tokenizer.convert_ids_to_tokens将src转化为tokens列表，去除了UNK和##并拼接在一起。使用jieba库进行切分。对于切分的结果，再使用tokenizer.convert_tokens_to_ids转为ids列表，加入src_wwm，同时将[位置，长度]储存在tokens_index中。补全CLS和SEP标记。若src_wwm超长度则截断到src的长度。

若不是，遍历src，若token是CLS,SEP,PAD，则跳过；若span_masking=False，将其索引加入tokens_index，若为True，则首先查看i是否小于span_end_position，是则跳过，否则get_span_len随机生成一个跨度，将当前位置和跨度写入tokens_index，并调整span_end_position。最后将实现将所有token覆盖的span分布。

返回值tokens_index为元素为[位置，长度]的列表的列表。src为原src或经过拼接再分词的新tokens集合（whole_word_masking=True）。

### get_span_len

初始化累计概率数组geo_prob_cum和概率geo_prob。使用循环计算概率，最终的geo_prob_cum将会保存几何概率的累积分布函数。prob在总概率之内随机采样，使用循环确定current_span_len。更短的长度概率更高，整体返回结果遵循几何分布。

### merge_dataset

将多个dataset-tmp-{i}.pt的内容写入dataset_path中。不是并行函数。

### truncate_seq_pair

输入的两个序列长度之和需要小于参数max_num_tokens。为此循环，选择较长的序列随机裁剪头部或尾部，直到长度达到要求。

### Dataset 类

data文件的核心类，定义了数据集的处理方式。在preprocess阶段被使用。

#### \_\_init\_\_

设置以下[参数](../../uer/README.md)：

- `vocab` 词汇库。
- `tokenizer` Tokenizer分词器。
- `corpus_path` 语料库路径。
- `dataset_path` 预处理数据集输出路径。
- `seq_length` 序列长度。
- `seed` 随机数。
- `dynamic_masking` 是否动态掩码。
- `whole_word_masking` 是否全词掩码。
- `span_masking` 是否连续掩码。
- `span_geo_prob` 几何分布参数。
- `span_max_length` 最大连续掩码长度。
- `docs_buffer_size` 内存中缓存的文档数。
- `dup_factor` 实例重复增强次数。

#### build_and_save

参数workers_num为同时工作的worker数量。平均分行对文件进行worker操作。

#### worker

抛出 NotImplementedError

### DataLoader

数据读取类。在pretrain阶段被使用。

#### \_\_init\_\_

设置以下[参数](../../uer/README.md)：

- `tokenizer` Tokenizer分词器。
- `batch_size` （未使用）
- `instances_buffer_size` 实例内存大小。
- `proc_id` 参数proc_id。
- `proc_num` 参数proc_num。
- `shuffle` 是否随机打乱。
- `dataset_reader` 读取参数dataset_path的reader。
- `read_count` 初始化为0。
- `start` 初始化为0。
- `end` 初始化为0。
- `buffer` 初始化为[]。
- `vocab` 词汇库。
- `whole_word_masking` 是否全词掩码。
- `span_masking` 是否连续掩码。
- `span_geo_prob` 几何分布参数。
- `span_max_length` 最大连续掩码长度。

#### _fill_buf

循环从dataset_reader使用pickle读取实例，增加read_count。若read_count - 1余proc_num为proc_id，则将实例加入buffer。当buffer的内容超过instances_buffer_size时跳出循环。若读到文件末尾，会从头开始，跳出循环的条件只有大小超标。

若shuffle=True，随机打乱buffer。设置start为0，end为buffer大小。

#### _empty

返回start是否大于等于end，以显示文件是否为空。

#### \_\_del\_\_

资源管理，销毁时关闭reader。

### BertDataset 类

继承Dataset类，用于处理Bert任务的类。本项目的核心。

#### \_\_init\_\_

除Dataset的全部参数外，还包含：

- `short_seq_prob` 短序列概率。

#### worker

设置随机数。设置待写入的文档dataset-tmp-{i}.pt。打开文件，将指针跳到需要读取的行，循环读取一行。当需要读的行全部读完，将剩余的instances写入文件，关闭writer并结束。在那之前，若读取到内容，则调用tokenizer.convert_tokens_to_ids和tokenizer.tokenize将其处理成id列表，加入document中。若读取到的行为空，则将document加入docs_buffer，清空document；若docs_buffer大小达到了设定的docs_buffer_size，将这一批instances写入文件。

#### build_instances

循环dup_factor（实例重复增强次数）次，从参数all_documents调用create_ins_from_doc读取instance，形成instances列表返回。

#### create_ins_from_doc

根据两个参数确定document。最大可以包含max_num_tokens个token，减去的3包含CLS，SEP，SEP。target_seq_length初始化为max_num_tokens，根据short_seq_prob再次赋值，若在概率内则改为2到max_num_tokens之间的随机数。

初始化instances，current_chunk列表，current_length=0。遍历document（all_documents是一个列表，其中每个元素是一个包含ids列表的列表，document形如[ids, ids]；在本项目中，每个document应该固定有两个ids）的每一个segment（ids），将其加入current_chunk，其长度加到current_length。

若现在在遍历后一个segment或当前长度大于target_seq_length（长度达标），则开始处理current_chunk。设置a_end=1，但若current_chunk有超过2个ids，则随机一个a_end。将a_end之前的ids均放入tokens_a，tokens_b初始化为空。若不止有一组或随机到大于0.5的值，将a_end之后的内容放入tokens_b，is_random_next设为0；否则is_random_next设为1，设置最大可接受的target_b_length，在all_documents中随机寻找另一个document（设置的10次循环出错的概率应该非常低，若出错，应该是数据结构问题，这个设计比较大胆），将这个document写入tokens_b中。回退i以保证数据利用率。

截断二者长度保证长度符合要求。构建 CLS+a+SEP+b+SEP+PAD 的src，保存两个SEP的位置，根据dynamic_masking判断保存instance为3元组或4元组。

### BertDataLoader

继承DataLoader类，用于读取Bert数据。

#### \_\_iter\_\_

迭代方法。

当buffer没有内容时，调用_fill_buf从文件读取实例。判断batch_size，查看buffer中内容是否过多，将合适数量的内容赋值给instances。将start赋值到batch_size之后。这一部分实现获取一个batch的实例列表。

初始化src，tgt_mlm，is_next和seg四个列表，masked_words_num=0。对于每一个instance有两种情况：

若instance长度为4(src, tgt_mlm, is_random_next, seg_pos)，将第一个元素（src）放入src，第二个元素（tgt_mlm）的长度加到masked_words_num，tgt_mlm会写入一个长为第一个元素（src）长度的全0列表作为元素。遍历第二个元素（tgt_mlm），我们已知tgt_mlm是被掩码的位置和原内容的列表的列表，将本函数的tgt_mlm刚刚设置的全0列表进行修改，在对应位置放入原内容。is_next放入第三个元素（is_random_next）。seg根据第四个元素（seg_pos）设置为段编码

若其他情况（长度为3）（src, is_random_next, seg_pos），现场调用mask_seq将第一个元素（src）进行掩码生成，然后和上述情况一致。

若出现异常，没有读取到数据，则再进行一次上述过程。每轮返回torch.LongTensor的四个列表。

### 其余类

其余类在这个项目中不重要，因此这里不进一步进行分析。

## optimizers

定义项目模型训练阶段的optimizer和scheduler的文件，来自谷歌AI团队和HuggingFace团队，专为BERT模型设计。

### get_constant_schedule

设定恒定学习率的调度器，对于所有情况，学习率乘数都为1。

### get_constant_schedule_with_warmup

设定包含warmup的调度器。在step到达num_warmup_steps之前，学习率乘数会线性增长到1。

### get_linear_schedule_with_warmup

设定包含warmup和线性损失的调度器。在step到达num_warmup_steps之前，学习率乘数会线性增长到1；此后在step到达num_training_steps之前，学习率乘数会线性减小到0。

### get_cosine_schedule_with_warmup

设定包含warmup和余弦损失的调度器。在step到达num_warmup_steps之前，学习率乘数会线性增长到1；此后在step到达num_training_steps之前，学习率乘数会余弦减小到0（先慢后快）。

### get_cosine_with_hard_restarts_schedule_with_warmup

