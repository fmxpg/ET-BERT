uer应该是直接使用了BERT项目的源码。

## opts

用于处理命令行参数的函数集合。

### model_opts

模型基本参数。

- `embedding` 词嵌入方式（`word` 纯词嵌入/`word_pos` 词嵌入+位置编码/`word_pos_seg` 词嵌入+位置编码+段落编码/`word_sinusoidalpos` 词嵌入+正弦位置编码），默认为`word_pos_seg`。
- `max_seq_length` 词嵌入序列最大长度，默认为512。
- `relative_position_embedding` 是否使用相对位置编码。
- `relative_attention_buckets_num` 相对位置编码分桶数，默认为32（将位置编码降维，仅使用桶数的位置编码，以节约成本）。
- `remove_embedding_layernorm` 是否弃用嵌入层layernorm（层归一化）。
- `remove_attention_scale` 是否移除attention的缩放因子。
- `encoder` 编码器（`transformer` 变压器/`rnn` 循环神经网络/`lstm` 长短期记忆网络/`gru` 门控循环单元/`birnn` 双向rnn/`bilstm` 双向lstm/`bigru` 双向gru/`gatedcnn` 门控卷积神经网络），默认为`transformer`。
- `mask` 掩码类型（`fully_visible` 全可见/`casual` 因果（单向）/`casual_with_prefix` 前缀部分双向、生成部分单向），默认为`fully_visible`。
- `layernorm_positioning` layernorm（层归一化）和残差层的相对位置（`pre` 前/`post` 后），默认为`post`。
- `feed_forward` fnn（前馈神经网络）类型，transformer可选项（`dense` 标准MLP/`gated` 带门控单元），默认为`dense`。
- `remove_transformer_bias` 是否移除transformer偏置。
- `layernorm` 层归一化类型（`normal` 标准/`t5` RMS Norm），默认为`normal`。
- `bidirectional` 是否使用双向，仅当rnn模型有效。
- `factorized_embedding_parameterization` 是否扩展嵌入。
- `parameter_sharing` 是否共享参数（减少模型参数，减小开销，降低模型能力）。

### optimization_opts

训练优化参数。

- `learning_rate` 学习率，默认为2e-5。
- `warmup` 学习率预热比例，默认为0.1（前10%逐渐增大学习率）。
- `fp16` 是否使用fp16混合精度训练，需要NVIDIA GPU。
- `fp16_opt_level` （`O0` 纯fp32/`O1` 保守混合精度/`O2` 激进混合精度/`O3` 纯fp16），默认为`O1`。
- `optimizer` 优化器类型（`adamw` Adam+权重衰减修正/`adamfactor` 自适应分解优化器），超大模型时后者可以节约空间。默认为`adamw`。
- `scheduler` 学习率调度器类型（`linear` 线性衰减/`cosine` 余弦衰减/`cosine_with_restarts` 带重启的余弦衰减	/`polynomial` 多项式衰减/`constant` 固定/`constant_with_warmup` 预热后固定），默认为`linear`。

### training_opts

训练基本参数。

- `batch_size` 每批样本数量，默认为32。
- `seq_length` 序列长度，默认为128。
- `dropout` 随机丢弃比例，默认为0.5。
- `epochs_num` 训练次数，默认为3。
- `report_steps` 每次报告的step间隔，默认为100。
- `seed` 随机种子，默认为7。

### finetune_opts

微调参数。

- `pretrained_model_path` 预训练模型路径。
- `output_model_path` 微调模型输出路径，默认为"models/finetuned_model.bin"。
- `vocab_path` 词表文件路径。
- `spm_model_path` 分词模型路径。
- `train_path` **（必需）** 训练集路径。
- `dev_path` **（必需）** 开发集路径。
- `test_path` 测试集路径。
- `config_path` 配置参数文件路径，默认为"models/bert/base_config.json"。

同时包含以上model_opts、optimization_opts、training_opts的所有参数。

### infer_opts

推断参数。

- `load_model_path` 模型输入路径。
- `vocab_path` 词表文件路径。
- `spm_model_path` 分词模型路径。
- `test_path` 测试集路径。
- `prediction_path` 预测文件路径。
- `config_path` 配置参数文件路径，默认为"models/bert/base_config.json"。

包含model_opts的所有参数。

- `batch_size` 每批样本数量，默认为**64**。
- `seq_length` 序列长度，默认为128。

## model_builder

通过参数确定模型。

### build_model

传入参数args。根据args确认embedding层为WordEmbedding、WordPosEmbedding、WordPosSegEmbedding还是WordSinusoidalposEmbedding（均在[uer.layers](./layers/README.md).embeddings中）；确认encoder是TransformerEncoder、RnnEncoder、LstmEncoder、GruEncoder、BirnnEncoder、BilstmEncoder、BigruEncoder还是GatedcnnEncoder（均在[uer.encoders](./encoders/README.md)中）；确认target是BertTarget、MlmTarget、LmTarget、BilmTarget、AlbertTarget、Seq2seqTarget、T5Target、ClsTarget、PrefixlmTarget还是NspTarget（均在[uer.targets](./targets/README.md)中）。

调用[uer.models](./models/README.md).model，将args和以上三部分组成model返回。

## model_loader

读取模型。

### laod_model

调用torch.load，若model有module参数，调用model.module的load_state_dict，否则调用model的load_state_dict。

## model_saver

保存模型。

### save_model

调用torch.save，若model有module参数，保存model.module.state_dict()，否则保存model.state_dict()。

## trainer (edit-trainer)

预训练的主要部分，实现对模型的预训练逻辑。edit-trainer在trainer的基础上添加了tqdm获取训练进度，加入了NspTarget类专门训练Nsp问题。除此之外两文件区别不大。

### train_and_validate

某种意义上文件的主函数，实现模型训练和验证。

若指定args.spm_model_path，则使用其获取vocab，构建tokenizer，对于seq2seq（序列到序列）任务设置tgt_vocab；否则先构建tokenizer，再使用其vocab作为vocab，对seq2seq设置从vocab文件中读取的tgt_vocab。

调用build_model构建model。

若指定args.pretrained_model_path，调用load_model读取model，否则使用model.named_parameters遍历所有参数，对名称不含有gamma和beta的参数进行随机初始化。

根据dist_train和single_gpu确认是多GPU训练，单GPU训练还是CPU训练。多GPU使用mp.spawn实现多线程。训练均通过调用worker实现。

### Trainer 类

所有预训练任务的基类。

#### \_\_init\_\_

初始化函数，初始化配置/读取类参数。

- `current_step` 当前step，初始化为1。
- `total_steps` 读取args的总训练step，默认为100000。
- `accumulation_steps` 读取args的梯度累积的step间隔，默认为1。
- `report_steps` 读取args的每次报告的step间隔，默认为100。
- `save_checkpoint_steps` 读取args的模型进行中间保存step间隔，默认为10000。
- `output_model_path` 读取args的模型输出路径。
- `start_time` 初始化为当前时间。
- `total_loss` 初始化为0，总loss。
- `dist_train` 读取args的是否分布式训练。
- `batch_size` 读取args的每批样本数量，默认为32。
- `world_size` 读取args的全局进程数，默认为1。

#### forward_propagation

抛出 NotImplementedError

#### report_and_reset_stats

抛出 NotImplementedError

#### train

模型训练过程，参数有args，gpu_id，rank，loader，model，optimizer和scheduler。

将模型设置为训练模式，将loader处理成迭代器模式。循环执行step直到达到total_steps，对于每一个step做如下操作。

通过loader_iter获取一个batch，将其序列长度记入self.seq_length。若gpu_id存在，将batch搬移到cuda。调用forward_propagation计算loss。根据args.fp16决定反向传播是否自动缩放。若达到指定的accumulation_steps，更新optimizer和scheduler（参数和学习率），清空梯度。对于主进程，若达到指定的report_steps，调用report_and_reset_stats输出状态并保存，修改start_time为当前时间；若达到指定的save_checkpoint_steps，保存当前模型状态到output_model_path。

### MlmTrainer 类

继承自Trainer，实现掩码语言模型训练。

#### \_\_init\_\_

除去父类初始化外，额外增加两个参数：

- `total_correct` 初始化为0。
- `total_denominator` 初始化为0。

#### forward_propagation

从batch拆分出src，tgt，seg。调用model，获取loss_info。从loss_info拆分出loss，correct，denominator并且加到类参数中。计算平均loss并返回。

#### report_and_reset_stats

已完成的tokens数等于一批所含序列数乘序列长度乘设定的报告间隔step数。若分布训练，则还需要乘进程数。输出当前步和总步数、训练速度、loss和acc。重置参数loss、correct和denominator。

### BertTrainer 类

继承自Trainer，实现掩码语言模型训练。

#### \_\_init\_\_

除去父类初始化外，额外增加6个参数：

- `total_loss_sp` nsp部分的loss。初始化为0。
- `total_correct_sp` nsp部分的correct，初始化为0。
- `total_instances` 句子对任务样本数，初始化为0。
- `total_loss_mlm` mlm部分的loss，初始化为0。
- `total_correct_mlm` mlm部分的correct，初始化为0。
- `total_denominator` 掩码数，初始化为0。

#### forward_propagation

从batch拆分出src，tgt_mlm，tgt_sp，seg。调用model，获取loss_info。从loss_info拆分出loss_mlm，loss_sp，correct_mlm，correct_sp，denominator并且加到类参数中。loss的计算为loss_mlm / 10 + loss_sp。计算平均loss并返回。

#### report_and_reset_stats

已完成的tokens数等于一批所含序列数乘序列长度乘设定的报告间隔step数。若分布训练，则还需要乘进程数。输出当前步和总步数、训练速度、总loss、两个子任务分别的loss和acc。重置多个参数。

### 其余类

暂时不研究。

### worker

设置随机数。根据dist_train和single_gpu确定rank和gpu_id（GPU和多进程参数）。根据dist_train判断[uer.utils](./utils/README.md).data.DataLoader的初始化参数。若启用GPU，将model和device都设置为cuda(gpu_id)。

设置参数衰减，先获取所有参数，除bias，gamma和beta不衰减外，其余参数设置L2衰减。根据args确定选择的[uer.utils](./utils/README.md).optimizers中的optimizer（Optimizer类的对象）和scheduler（函数，返回值是一个LambdaLR）。

（注释是因为我才疏学浅，需要理解二者可以调用step函数的理由）

若开启混合精度训练，需要导入apex库，使用其amp.initialize处理model和optimizer。

若dist_train=True（分布式训练），使用torch.distributed指定并行训练的model，否则不作操作。

根据args确定选择的trainer并开始训练。