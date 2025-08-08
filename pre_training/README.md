实现预训练部分的对命令行交互接口和基本逻辑，是程序的基本部分。

[部分参数参见 /uer/README.md](../uer/README.md)

参数设置包含路径参数、训练和保存参数、预训练参数、模型参数、掩码参数、优化参数和GPU参数7部分。

路径参数：

- `dataset_path` 预处理数据集路径，默认为"dataset.pt"。
- `vocab_path` 词表文件路径。
- `spm_model_path` 分词模型路径。
- `tgt_vocab_path` 目标词表文件路径。
- `tgt_spm_model_path` 目标分词模型路径。
- `pretrained_model_path` 预训练模型路径。
- `output_model_path` 模型输出路径。
- `config_path` 模型配置文件路径，默认为"models/bert/base_config.json"。

训练和保存参数：

- `total_steps` 总训练step，默认为100000。
- `save_checkpoint_steps` 设定模型进行中间保存step间隔，默认为10000。
- `report_steps` 每次报告的step间隔，默认为100。
- `accumulation_steps` 梯度累积的step间隔，默认为1。
- `batch_size` 每批样本数量，默认为32。
- `instances_buffer_size` 内存数据缓冲大小。默认为25600。
- `labels_num` 预测标签数量。
- `dropout` 随机丢弃比例，默认为0.1。
- `seed` 随机种子，默认为7。

预训练参数：

- `tokenizer` 分词器（`bert` Google BERT/`char` 分割成字符/`space` 按空格分词），默认为`bert`。

模型参数：

包含model_opts中的所有参数。

- `tgt_embedding` 目标词嵌入方式（`word` 纯词嵌入/`word_pos` 词嵌入+位置编码/`word_pos_seg` 词嵌入+位置编码+段落编码/`word_sinusoidalpos` 词嵌入+正弦位置编码），默认为`word_pos_seg`。
- `decoder` 解码器（`transformer` 变压器），默认为`transformer`（只有这一种）。
- `pooling` 池化方式，从哪一位置获取全局信息（`mean` 平均池化/`max` 最大池化/`first` CLS/`last` SEP），默认为`first`。
- `target` 预训练模型训练目标（`bert` BERT/`lm` 语言模型/`mlm` 掩码语言模型/`bilm` 双向语言模型/`albert` ALBERT/`seq2seq` 序列到序列/`t5` T5/`cls` 文本分类/`prefixlm` 前缀语言模型），默认为`bert`。
- `tie_weights` 是否绑定输入层嵌入和输出层权重。
- `has_lmtarget_bias` 是否为部分LM任务在模型中添加额外偏置。

掩码参数：

- `whole_word_masking` 是否全词掩码。
- `span_masking` 是否连续掩码。
- `span_geo_prob` 几何分布参数，默认为0.2（控制掩码长度的概率分布）。
- `span_max_length` 最大连续掩码长度，默认为10。

优化参数：

包含optimization_opts中的所有参数。

GPU参数：

- `world_size` 全局进程数，默认为1。
- `gpu_ranks` 指定每个进程对应的唯一排名（rank）和绑定的GPU，输入一个整数列表。
- `master_ip` 分布式训练主节点地址，默认为"tcp://localhost:12345"。
- `backend` 分布式训练后端（`nccl` NVIDIA后端/`gloo` 通用后端），默认为`nccl`。

以下是逻辑部分。

若分类目标为cls，确认是否存在args.labels_num，如果没有则告警。调用[uer.utils](../uer/utils/README.md).config.load_hyperparam读取模型配置文件（默认是"models/bert/base_config.json"），并写入args。将args.gpu_ranks整理成ranks_num列表，进行条件判断，决定进行分布式训练、单GPU训练或CPU训练。设置参数args.dist_train（是否分布式训练）和args.single_gpu（是否单GPU训练）。如果是单GPU训练，设置参数args.gpu_id为args.gpu_ranks的唯一元素。

调用[uer](../uer/README.md).trainer.train_and_validate进行训练。