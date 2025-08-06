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
- `factorized_embedding_parameterization` 是否分解嵌入矩阵（大幅减少模型参数，可能降低模型能力）。
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