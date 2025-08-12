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

instances_num获取总样本数。



