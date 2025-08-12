## layer_norm

归一化层的实现。

### LayerNorm 类

#### \_\_init\_\_

手动定义可学习的gamma（缩放参数）和beta（偏移参数）。eps是防止除零参数，可以指定。

#### forward

求输入的平均值和标准差，进行中心化和缩放。输出隐藏层和偏移参数beta的和。

### T5LayerNorm 类

#### \_\_init\_\_

手动定义可学习的weight（缩放参数）。variance_epsilon是防止除零参数，可以指定。没有偏移。

#### forward

使用二阶矩有偏估计，实现中心化和缩放。

## embedding

四种Embedding层的实现。

### WordEmbedding 类

继承nn.Module，最普通的词嵌入层实现。

#### \_\_init\_\_

调用nn.Embedding作为词嵌入层，调用nn.Dropout作为随机置零，根据参数判断是否弃用词嵌入层归一化，若为否则应使用，实例化LayerNorm类作为归一化层。

#### forward

对src进行查表、归一化（可选）、随机参数置零操作。

### WordPosEmbedding 类

继承nn.Module，GPT形式的词嵌入层实现，但位置编码也使用可学习的嵌入层。

#### \_\_init\_\_

调用nn.Embedding作为词汇和位置编码的词嵌入层，调用nn.Dropout作为随机置零，根据参数判断是否弃用词嵌入层归一化，若为否则应使用，实例化LayerNorm类作为归一化层。获取参数的max_seq_length。

#### forward

对src进行查表，生成其对应的位置编码并且也进行查表，最终的emb为二者之和。进行归一化（可选）、随机参数置零操作。

### WordPosSegEmbedding 类

继承nn.Module，BERT形式的词嵌入层实现。

#### \_\_init\_\_

调用nn.Embedding作为词汇、位置编码和段编码的词嵌入层，调用nn.Dropout作为随机置零，根据参数判断是否弃用词嵌入层归一化，若为否则应使用，实例化LayerNorm类作为归一化层。获取参数的max_seq_length。

#### forward

对src进行查表，生成其对应的位置编码并且也进行查表，对seg进行查表，最终的emb为三者之和。进行归一化（可选）、随机参数置零操作。

### WordSinusoidalposEmbedding 类

继承nn.Module，GPT形式的原版词嵌入层实现。

#### \_\_init\_\_

调用nn.Embedding作为词嵌入层，调用nn.Dropout作为随机置零。由于交替使用sin和cos，嵌入维度需要为偶数。获取参数的max_seq_length。位置编码矩阵pe不参与训练，偶数使用sin，奇数使用cos。

#### forward

对src进行查表，位置编码直接使用pe的嵌入，最终的emb为二者之和。进行归一化（可选）、随机参数置零操作。

## multi_headed_attn

实现多头注意力机制。

### MultiHeadedAttention 类

继承nn.Module，实现多头注意力。

#### \_\_init\_\_

从参数中获取注意力头数量heads_num，注意力头维度per_head_size，是否放缩，隐藏层大小通过注意力头数量和维度乘积得到。

设定三个线性层的列表（QKV），随机置零层和最终线性层。

#### forward

内置函数shape和unshape实现隐藏层大小和多头+头维度两种表示的相互转化。

通过列表推导式对Q、K、V线性投影和变形，得到多头结构；计算注意力偏置；若有偏置，添加偏置；若有放缩，进行放缩；添加掩码，屏蔽无效信息（实现上可以通过-inf做到）；连接Softmax层，转变为概率形式；随机置零；调用unshape恢复为原始维度，连接最终线性层输出。

## position_ffn

实现前馈网络的层。

### PositionwiseFeedForward 类

#### \_\_init\_\_

定义两层线性层和选定一个激活函数。

#### forward

线性层-激活函数-线性层

### PositionwiseFeedForward 类

#### \_\_init\_\_

定义两层线性层、选定一个激活函数、定义一个门控。

#### forward

线性层*门控-激活函数-线性层

## transformer

实现transformer的基本逻辑的层。

### TransformerLayer 类

继承nn.Module，Transformer encoder主要逻辑部分的实现。

#### \_\_init\_\_

从args中获取layernorm_positioning，这一选项决定了各层的相对位置。若args指定了attention_head_size，将其保存；否则，使用隐藏层大小和头的数目计算得到（这两个参数通过读取models中的json文件得到，不是通过命令交互指定）。获取has_bias和with_scale。

初始化多头注意力机制层和随机置零层。根据参数选择的feed_forward确定初始化的前馈网络类型。根据参数选择的layernorm决定归一化层的类型。

#### forward

若layernorm_positioning为post，则为后置层归一化，按照“多头注意力（随机置零）-残差连接-归一化-前馈层（随机置零）-残差连接-归一化”顺序连接，多头注意力输入为函数参数hidden；否则为pre，前置层归一化，先归一化hidden，按照“归一化-多头注意力（随机置零）-残差连接-归一化-前馈层（随机置零）-残差连接”顺序连接。

### TransformerDecoderLayer 类

继承nn.Module，Transformer decoder主要逻辑部分的实现。

#### \_\_init\_\_

基本同TransformerLayer类，但有三个归一化层，两个多头注意力。

#### forward

基本同TransformerLayer类，但多进行一次多头注意力。

## relative_position_embedding

相对位置编码实现。

### RelativePositionEmbedding 类

继承nn.Module，相对位置编码逻辑的实现。

#### \_\_init\_\_

保存参数中的桶数num_buckets，是否双向bidirectional和最大距离max_distance。初始化一个嵌入层命名为relative_attention_bias。

#### forward

获取Q和K的维度。计算出全局的位置编码，然后计算其相对的位置编码。调用relative_position_bucket分桶映射，通过嵌入层将其进行训练。最终目标是使得相对位置索引映射成一个可学习的偏置值。

#### relative_position_bucket

若双向注意力，则桶的数量减半，使用前一半用于负偏移，后一半用于正偏移；若单向，则仅允许负偏移。若桶的偏移量在指定的范围内（1/2），进行一一映射，否则进行对数缩放，确保所有桶都获得单调不减的位置编码。