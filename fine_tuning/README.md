## run_classifier

微调部分的代码实现，全部集中在一个文件中。

### Classifier 类

继承nn.Module，微调的分类器模型的实现。与之直接对应的是[预训练模型Model](../uer/models/README.md)。

#### \_\_init\_\_

初始化参数，包含嵌入层embedding，编码器encoder，标签总数labels_num，池化方式pooling，是否使用软标签soft_targets，软标签损失权重soft_alpha，两个线性层output_layer_1和output_layer_2。

#### forward

输入除Model的输入外，还包含可选的软标签soft_tgt。

嵌入层和编码器部分同Model，任务目标在这里实现。首先判断池化方式，确定使用Transformer输出的头部，尾部，平均还是最大向量来作为输入序列的特征向量。将此特征向量通过线性层-tanh-线性层得到分类头。

若tgt指定（训练模式），判断是否有软标签。若有软标签，总loss为软标签的MSELoss和硬标签的NLLLoss；否则只有硬标签的NLLLoss。

若tgt不指定（预测模式），直接返回分类头。

### count_labels_num

从路径中读取数据文件。数据有两列，每行第一个为label，第二个为bigram字符串text_a。通过将label加入到set，返回set的大小来得到原始数据真正的类别数量。

### load_or_initialize_parameters

若给定预训练模型路径，读取其参数；否则对model除gamma和beta外的参数进行手动随机初始化。

### build_optimizer

设置参数衰减，先获取所有参数，除bias，gamma和beta不衰减外，其余参数设置L2衰减。根据args确定选择的[uer.utils](../uer/utils/README.md).optimizers中的optimizer（Optimizer类的对象）和scheduler（函数，返回值是一个LambdaLR）。

（注释是因为我才疏学浅，需要理解二者可以调用step函数的理由）

这个函数是从trainer.py整理出来的。

### batch_loader

将输入的src、tgt、seg处理成批格式。

instances_num获取总样本数。循环进行分割，每次yield各参数长为batch_size的一部分。最后对剩余数据进行处理。

### read_dataset

初始化dataset列表和columns字典。打开path指定的文件，对每一行进行遍历。第一行获取columns的keys，也即tsv文件首行的标签。对于其他行，首先通过split分隔出各个元素，获取其硬标签tgt。若有logits列，将logits列读取成浮点数列表，作为软标签soft_tgt。

若没有text_b，仅读取text_a拼接CLS（和SEP？）作为src，seg为等长的全1；若有，则二者拼接CLS和两个SEP作为src，seg为1串和2串。若超出参数给定的序列长度，进行阶段；否则进行填充。将src、tgt、seq和可能的soft_tgt进行返回。

### train_model

将model的梯度归零。将src、tgt、seq和可能的soft_tgt搬移到device上。计算loss、反向传播、更新optimizer和scheduler。返回loss。

### evaluate

从元组中分离src、tgt、seq。获取batch_size。混淆矩阵初始化，长宽均为labels_num的全0矩阵。模型设置为eval模式。

使用batch_loader遍历src、tgt、seq，对于每一个batch，搬移到device上，调用model计算logits。预测值为logits链接Softmax+argmax，真实值为tgt_batch，遍历并计入混淆矩阵，将正确的数量加到correct中。

若print_confusion_matrix=True，则输出混淆矩阵。将混淆矩阵写入文件。对每个label，计算其预测的准确率、召回率和f1。

计算并输出预测的总准确率，正确数目和总数目。返回总准确率和混淆矩阵。

### main

首先参数设置，除[uer](../uer/README.md).opts中的finetune_opts外，还包括：

- `pooling` 池化方式，从哪一位置获取全局信息（`mean` 平均池化/`max` 最大池化/`first` CLS/`last` SEP），默认为`first`。
- `tokenizer` 分词器（`bert` Google BERT/`char` 分割成字符/`space` 按空格分词），默认为`bert`。
- `soft_targets` 是否使用软标签训练。
- `soft_alpha` 软标签损失权重，默认为0.5（硬标签和软标签的loss各一半）。

设置随机数。初始化model为Classifier对象。读取或初始化参数。搬移device。读取训练集，随机打乱，获取数据集大小和参数batch_size。分离出src、tgt、seq和可能的soft_tgt。设定参数train_steps，调用build_optimizer设置optimizer和scheduler。设置混合精度和GPU训练。

训练循环epochs_num次，从batch_loader读取数据，调用train_model进行训练，统计loss并在固定steps打印。使用dev_path对应的验证数据集，调用不带输出的evaluate进行评估，若效果好于之前的最佳，则进行替换，保存模型数据。

若指定测试集，则对测试集进行评估，对模型进行验证，调用evaluate计算测试集的混淆矩阵并且输出。
