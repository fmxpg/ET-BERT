## 用于生成数据集的PCAP文件处理过程
推荐处理PCAP数据之前先进行数据清洗。由于程序逻辑较为复杂，以下详细说明预训练与微调阶段的数据预处理步骤。

### 预训练阶段（位于data_process）
*主程序*：dataset_generation.py

*函数*：pretrain_dataset_generation，get_burst_feature

1. 初始化。
将变量 `pcap_path` （第488行）修改为待处理的PCAP数据目录。
将变量 `word_dir` （第15行）和 `word_name` （第16行）修改为预训练数据集的存储目录。

2. PCAP预处理。
修改变量 `output_split_path` （第455行）和 `pcap_output_path` （第456行）。
`pcap_output_path` 指定从pcapng转换为pcap格式的数据存储目录。
`output_split_path` 表示按会话格式切分后的pcap数据存储目录。

3. 生成预训练模型。
PCAP数据集处理完成后，程序将生成由BURST构成的预训练数据集。

### 微调阶段
*主程序*：main.py

*函数*：data_preprocess.py, dataset_generation.py, open_dataset_deal.py, dataset_cleanning.py

在处理公开PCAP数据集时，微调阶段的核心逻辑是首先将数据集中不同标签的的数据按文件夹区分，然后对数据进行切分，最后生成包级别（packet-level）或流级别（flow-level）的数据集。

**注意**：由于原始PCAP数据的复杂性，如果报错，推荐按照以下流程进行检查。

1. 初始化。
`pcap_path`，`dataset_save_path`，`samples`，`features`，`dataset_level` （第16行）是最基础的变量，分别代表原始数据目录，数据集保存目录，样本数量，特征类型和数据集级别。 `open_dataset_not_pcap` （第189行）表示将非标准PCAP格式数据转化为pcap，例如pcapng转换为pcap。
`file2dir` （第200行）表示若若PCAP文件代表一个类别，生成存储PCAP数据的类别目录。

2. 预处理。
数据预处理过程主要将PCAP数据切分为会话数据。
请将参数 `splitcap_finish` 设置为0以初始化样本数数组，以及 `sample` 的值设置应不超过最小样本数。
然后可以设置 `splitcap=True` （第42行）然后运行代码来切分PCAP数据。切分后的数据将保存于 `pcap_path\\splitcap`。

3. 数据集生成
预处理完成后，需要调整参数以生成微调数据。`pcap_path` 应为切分后的数据，并且应设置`splitcap=False`。 
现在 `sample` 可以不受最小样本数限制。`open_dataset_not_pcap` 和 `file2dir` 都应为 False。微调数据集将生成并保存于 `dataset_save_path`。

---

