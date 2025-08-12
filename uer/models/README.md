定义torch模型的文件，具体来说，定义项目预训练部分模型的文件。

## model

### Model 类

项目中最核心的类，将预训练的各个部分拼接起来。微调部分见[这里](../../fine_tuning/README.md)。

#### \_\_init\_\_

nn.Module的初始化函数。Model整体包含词嵌入层，编码器层和目标头部层。

如果启用了args.tie_weights（权重共享）并且任务是bert或mlm，将mlm线性变换第二层和词嵌入层的权重绑定。

如果启用了args.tie_weights（权重共享）并且任务是lm或t5，将输出层和词嵌入层的权重绑定。

如果启用了args.share_embedding（项目中不存在这个配置？）并且任务是t5，将编码器和解码器的词嵌入矩阵绑定。

#### forward

输入src为ids列表；目标tgt为二元组，前者为还原的ids列表（非0项），后者为is_next；分段为一串1、一串2、一串0，对应子串1、子串2和填充。

嵌入层使用src和seg作为参数嵌入（根据ids列表和seg段落编号列表进行嵌入），输出emb；encoder使用嵌入结果emb和seg作为参数，输出output（seg结合掩码类型得到权重）；target使用output和tgt作为参数判断预测结果，返回loss_info，作为整体的返回值。