## main

用于对pcap进行处理，生成语料库。

### pcap_preprocess

（没有用法）

对TLS 1.3文件进行特殊处理，仅用于适配目录格式，功能等同于preprocess。

### preprocess

对于非pcapng并且名称中含有`tls13_name`的文件作处理。对每一个文件，使用scapy.rdpcap得到包列表。对于每一个包，将其转换为文字信息，去除前38字节，分成两部分，按照字节切割成列表，拼接成bigram，写入words_txt，最后写入文件。将报文信息转化为字符串信息。

### cut

同data_process/dataset_generation -> cut。

### build_BPE

创建source_dictionary，将0到65535的十六进制文字作为键，数字作为值写入其中。初始化一个Tokenizer，使用WordPiece模型，将之前的source_dictionary作为词典，设置未知字符，设置输入的最大长度。配置其pre_tokenizer、decoder和post_processor，然后对preprocess的结果进行训练，最后保存到json中。

### build_vocab

读取build_BPE创建的json文件，设置vocab_txt为`["[PAD]","[SEP]","[CLS]","[UNK]","[MASK]"]`，将读入的json文件也放入vocab_txt，最后写入vocab.txt。

### bigram_generation

类似data_process/dataset_generation -> bigram_generation，但是限制每个报文段最长只保留256。

### read_pcap_flow

调用scapy.rdpcap将报文拆分为packets。若长度小于5，则返回-1；对仅前5个报文，将其转化为bigram，拼接在一起返回。

### split_cap

同data_process/dataset_generation -> split_cap。