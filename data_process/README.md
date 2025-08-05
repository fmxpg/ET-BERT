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

3. 数据集生成。
预处理完成后，需要调整参数以生成微调数据。`pcap_path` 应为切分后的数据，并且应设置`splitcap=False`。 
现在 `sample` 可以不受最小样本数限制。`open_dataset_not_pcap` 和 `file2dir` 都应为 False。微调数据集将生成并保存于 `dataset_save_path`。

---

以下依次阅读代码，理解每个部分的工作逻辑。

这个文件夹中的文件最让人困惑的点在于它没有很好地实现批处理的逻辑，虽然其代码能够完成全部的工作，但你不知道应该调用什么来实现它。引入“三个小小道”的prepare_dataset.py之后，这一部分所处理的内容就清晰了很多。

## dataset_generation.py

### convert_pcapng_2_pcap

使用editcap.exe（Wireshark自带），实现pcapng类型到pcap类型的转换。

cmd是含有通配符的命令，command使用参数进行拼接，交于system进行调用。

### split_cap

使用SplitCap.exe对pcap包进行分解。首先检查有无splitcap目录，没有则创建；其次查看有没有pcap_label，有则在上述目录下创建pcap_label目录和其下pcap_name目录，没有则直接在splitcap下创建pcap_name目录。根据dataset_level选定SplitCap的参数，对其进行切分。

### cut

按照步长sec将obj切分为List。remanent_count为切分长度余4，（异常情况基本上能确定是因为没有数据，这里修改了源代码）出现异常则余数赋值为0。若余数不为0，则重新设置分割长度，余数为几则分割长度加几。

### bigram_generation

将输入的packet_datagram切分为长度为2的子串，然后将前n-1个子串和其后一个子串拼接并以字符串形式输出。可以指定packet_length来获取一部分。

### get_burst_feature

（此函数无法正常运行，也不需要使用）

将pcap转化为BURST字符串。

首先使用scapy.rdpcap将输入的pcap转化为packet list。设置packet_direction列表，使用flowcontainer的extract方法获取pcap包信息，获取包的方向（ip_lengths具有正负号，表明了其方向）。

假定到这里一切正常，二者长度匹配，则将packets的每一个packet转换成16进制字符串，截取前payload_len字节为packet_string。若当前处理的是第一个包则直接将burst_data_string赋值，若不是，则判断方向是否和前一个包相同。出现不同，则应该分隔为不同的BURST，此时将burst_data_string先处理为bigram，然后两行输入到burst_txt（分句），然后输入空行，最后清空burst_data_string。将当前packet信息加入到burst_data_string。循环结束时将剩余内容也处理到burst_txt。最后输出burst_txt到指定位置。

### get_feature_packet

使用scapy.rdpcap将输入的pcap转化为packet list，获取第一个packet，去除报头，转化为bigram string并输出。

### get_feature_flow

使用scapy.rdpcap将输入的pcap转化为packet list。

抓取tcp报文，若没有抓取udp报文，若还没有则报错；若抓取到的报文长度太短（小于3），报错；然后同样对每个packet去除报头，转化为bigram输出。

### generation

若存在dataset.json，先输出generate完成。若clean_dataset为1，读取json内容，将113-118的keys换为pop_keys（1，10，16，23，25，71），原来的pop_keys对应内容删除。若re_write为1，则读取json，将json重命名为old_dataset.json，逐行读取new-samples.txt（描述见下），若一行中第二个元素大于9（指代类别），则将此数据放入新数据集中，最后将整理好的新数据集写入json。最后使用obtain_data获取pcap_path的X和Y。

若不存在json，则初始化dataset，label_name_list和session_pcap_path。遍历pcap_path目录，获取所有子目录名作为流量列别标签放入label_name_list。tls13判断是否对TLS 1.3作特殊处理，若为1，则从记录文件读取需要处理的pcap文件列表，创建目录结构，复制文件并重命名（一堆神仙数）。对每一个子目录进行补全，保存在session_pcap_path中。

对label_name_list进行编号。初始化r_file_record。

使用tqdm遍历session_pcap_path（也就是遍历所有文件位置），对于流级数据，若启用splitcap，则遍历并删除小于2KB的数据；初始化数据集信息，包含samples，payload，length，time，direction和message_type。对于包级数据，若启用splitcap，则遍历并删除空文件，小于0.14KB的TCP报文，小于0.1KB的UDP报文，处理错误的无效文件；初始化数据集信息，只包含samples和payload。若启用splitcap则继续循环。不启用时，随机抽样当前类别文件的完整路径列表，调用get_feature_flow或者get_feature_data获取特征，并保存在dataset中。

最后，统计并输出每个类别的样本数量和总数量，保存已处理文件记录并保存dataset.json。调用obtain_data获取X和Y。

get_feature_flow/get_feature_packet (bigram_generation (cut)) -> obtain_data

### read_data_from_json

从json读取内容，对于每一个feature，若开启消融，y会取1500和sample_num的小值个label，将其加入Y；x取随机1500个sample（sample总计多余1500个）或全部sample_num个sample（sample少于1500个），加入X。

### obtain_data

封装上一个函数，从指定的json_data或json文件中读取获得X和Y，简单作数值检验并输出。

### combine_dataset_json

将json进行合并。将8个dataset-{i}.json分别读入，将每个json的key进行搬移，0和1搬移9，2及以上搬移6，汇总到一起并且写入一个dataset.json。

### pretrain_dataset_generation

（不要使用）

总函数。如果pcap_output_path没有内容，则开始第一部分，将pcapng文件转化为pcap文件，将其余文件使用shutil.copy进行复制。若output_split_path没有splitcap文件夹，开始第二部分，将pcap处理成会话流。最后对每一个会话流调用get_burst_feature生成BURST特征。

### size_format

格式化输出文件大小，以三位小数表示，单位为KB。（KB除以1000是吧？）

## data_preprocess

这个文件的所有函数用法都在这个文件中，是一个独立运作的文件。

### combine_data_json

将target_path位置的dataset_1.json和dataset_2.json合并到dataset.json。我修改了参数，将target_path搬到参数部分。

首先result_samples初始化，设置`0`和`1`的`payload`为空字典，读取两个json文件。对两个json文件，都进行以下操作：获取其每一个内容的`payload`的内容，若长度大于100，将其放入`result_samples['0/1']['payload'][i]`；随机抽取其中的5000项。

combined_result初始化，设置`0`和`1`的`payload`为空字典，`samples`为之前采样的数量。将result_samples的内容对应放入combined_result的payload中。

将combined_result写入dataset.json。

### basic_process_1

（没有用法）

初始化x_len和x_time的train、test、valid列表。bind_data保存y_dict的train、test、valid；bind_len_data和bind_time_data保存x_dict中train、test、valid的len和time；bind_len_data_new和bind_time_data_new保存上述初始化内容。

对bind_data遍历，对每一个bind_data的一条信息，获取其对应的bind_len_data和bind_time_data的len和time信息，清除0数据（这一步的逻辑可能是错误的，但是可能会做到正确的效果），将time更改为差值形式，然后将二者存入new中。最后将两个new的list转化为np.array。

### time_process_1

将时间序列处理为差值形式，初值为0。`[1, 2, 4, 5] -> [0, 1, 2, 1]`

## open_dataset_deal

### fix_dataset

将文件夹下的pcap整合到一个pcap中，并且将所有整合后的pcap文件放在统一的位置。

### reverse_dir2file

将path内的所有文件移动到path的位置。

### dataset_file2dir

将path内的pcap文件套一层文件夹。

### file_2_pcap

使用tshark将source_file的文件转化成target_file的pcap文件。

### clean_pcap

按照筛选规则对pcap包进行处理，自动命名为“原名_clean.pcap”。

筛选规则："not arp and not dns and not stun and not dhcpv6 and not icmpv6 and not icmp and not dhcp and not llmnr and not nbns and not ntp and not igmp and frame.len > 80"

### statistic_dataset_sample_count

初始化列表dataset_label和dataset_lengths。tls13_flag默认置为1。

对于根目录下的目录，认为是label，写入到dataset_label列表。

对于非根目录下，当前位置没有文件只有目录的情况，若不是根目录下一级目录则跳过，若是则在当前路径进行遍历，将该路径下的所有文件都进行计数保存在dataset_lengths，路径保存在temp。

对于非根目录下的文件，当前位置有文件，当tls13_flag为1时进行处理，逻辑同上；为0时什么也不做。

最后输出dataset_lengths和dataset_label，temp没有用法。

## dataset_cleaning

### deal_label

[22, 23, 24, 28, 35, 44, 52, 62, 67, 76, 94, 95, 102, 104]

### deal_finetuning

默认参数excluding_label是deal_label，上面那个。

读取dataset_path处的train_dataset.tsv，valid_dataset.tsv和test_dataset.tsv。对于每一个excluding_label，初始化train_pop_index，valid_pop_index和test_pop_index，若`excluding_label+\t`在train_data的某一条中存在，则将这一条信息的索引写入train_pop_index，再读取此列表，将train_data的对应条目删除；对valid_data和test_data做同样的操作。

label_number初始化为120。循环，当label_number > 105时，若`label_number+\t`在train_data的某一条中存在，则将其number修改为excluding_label；对valid_data和test_data做同样的操作。每次循环label_number自减。

将修改的内容写入train_dataset.tsv，valid_dataset.tsv和test_dataset.tsv。对于test_dataset.tsv作特殊操作，人工删除其最后一行空行然后在控制台输入1，调用main中的方式去除这一文件中数据的标签。

## prepare_dataset （非原项目文件，来自“三个小小道”）

没有函数，文件本身是可执行python文件。指定path下要求按照文件夹对pcap文件进行分类，同种pcap放于同一文件夹，使用文件夹名称作为分类的标志。

进行了一定程度的修改。

遍历路径。对于路径内的一个文件夹，其名称应该对应pcap的label，其内应该有一个或数个pcap包，数量很少。对每个文件夹遍历其所有文件，首先处理pcapng格式转化为pcap；若exact_5为真，则使用tcpdump将源文件仅保留100000报文；使用过滤规则处理报文；调用SplitCap.exe切分报文；删除长度小于5的session（已删除此逻辑），若exact_5为真，则将大于5的报文也分割为长度为5的报文。

## main

### write_dataset_tsv

将label和data写入dataset_file，逐行写入_dataset.tsv。分隔符为制表符。

### unlabel_data

读取csv，仅将其每行的第二项写入nolabel_data，重命名后写入。

### cut_byte

（没有用法）

同dataset_generation的cut，修改了被余数4为2，删除了告警。

### pickle_save_data

（没有用法）

使用pickle而不是json保存数据。不知道是因为没有使用的必要还是什么原因没有使用。

### count_label_number

默认输入的samples是`[5000]`。首先按照种类数目_category（根据需要调整）去乘以samples，得到初始samples列表；调用open_dataset_deal.statistic_dataset_sample_count遍历目录，获取label和对应的pcap数据文件数量（这里统计的是已经过SplitCap之后的pcap数据，应该条目会有很多）；与默认samples比较，若数据不足则采用真实数据量替换对应samples。

### models_deal

遍历model数组（字符串数组），对于每一个model（作为参考，默认的model是`['pre-train']`），分隔train、test、valid。初始化x_train_dataset、x_test_dataset、x_valid_dataset。若model为pre-train，将三者的x_payload和label写入位于dataset_dir的_dataset.tsv中，再去除test_dataset.tsv的标签数据。将输入数据进行整理，就得到了X_dataset和Y_dataset。

（似乎实际上没有使用返回值的需要）

### dataset_extract

若已经存在npy数据，则读取数据并返回。若不存在，则进行数据生成，调用dataset_generation.generation。

初始化和label数目等长的dataset_statistic列表、x_payload和Y_all。Y_all随后使用循环将Y拉平到一维；dataset_statistic使用循环统计每种标签的数量（重构了samples，意义不明）；循环将X拉平到一维，存储在X_payload中。

x_payload和dataset_label分别表示X_payload和Y_all的numpy.array形式。初始化x_payload和y的train、test、valid，使用StratifiedShuffleSplit进行分割数据，按照0.8、0.1、0.1的比例分割。使用np.save保存到本地。最后使用models_deal整理数据，以两个字典的形式返回。

### 主逻辑

可选是否将文件全部处理成pcap文件。

可选是否使文件套入与文件名相同的文件夹中。

可选数据是否理解为经过splitcap切分。按照我引入了prepare_dataset.py的逻辑，应当置为1。

初始化`train_model = ["pre-train"]`，调用dataset_extract。