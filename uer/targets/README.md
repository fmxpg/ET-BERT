## mlm_target

### MlmTarget 类

继承nn.Module，实现词掩码任务效果的评估。

#### \_\_init\_\_

首先保存参数，包括词库大小vocab_size、隐藏层大小hidden_size、掩码大小emb_size，是否扩展嵌入factorized_embedding_parameterization和激活函数act。

根据factorized_embedding_parameterization确认线性层输入输出的大小，初始化两个线性层和一个归一化层、一个全连接层和一个负对数似然损失NLLLoss（注释掉了一个交叉熵损失，实质上LogSoftmax+NLLLoss就是CrossEntropyLoss）。

#### mlm

通过线性层+激活函数+归一化，提取memory_bank（encoder输出）的MLM特征信息output_mlm。根据factorized_embedding_parameterization对输出维度进行整理。输入的tgt_mlm是重建的id列表，非0项位置为掩码位置，内容为原id，output_mlm和tgt_mlm只保留包含掩码的部分。将output_mlm再通过线性层+Softmax，进行词汇预测。计算掩码数量（分母）denominator、预测正确数量correct_mlm和误差loss_mlm。

#### forward

调用mlm。

### BertTarget 类

继承MlmTarget类，在词掩码任务的基础上增加了连续句子判断部分，实现两种任务效果的评估，构成完整的BERT误差计算。

#### \_\_init\_\_

除去MlmTarget的初始化外，额外初始化两个nsp任务的线性层。

#### forward

tgt包含两部分，第一部分是重构的mlm任务列表，是一个id列表，第二部分是is_next，一个0或1的值。确认输入tgt符合格式，分离两部分，将第一部分输入mlm任务部分，得到mlm部分的总loss，correct和数量denominator。

NSP部分，output_nsp初始化为memory_bank每个批次的第一部分的全部信息，即CLS标记处经过Transformer注意力结构后提取的特征信息。输入线性层+Softmax，使用NLLLoss计算nsp部分的loss，eq判断得到correct。返回两部分分别的loss，correct和mlm任务的denominator。