# 基于 DCN_V2 模型的点击率预估模型

以下是本例的简要目录结构及说明： 

```
├── models
    ├── rank
        ├── dcn_v2
           ├── data # 样例数据
                ├── sample_data # 样例数据
                    ├── train
                        ├── sample_train.txt # 训练数据样例
           ├── __init__.py
           ├── README.md # 文档
           ├── config.yaml # sample数据配置
           ├── config_bigdata.yaml # 全量数据配置
           ├── net.py # 模型核心组网（动静统一）
           ├── reader.py # 数据读取程序
           ├── dygraph_model.py # 构建动态图
├── tools # 
├── README.md # 文档
```

注：在阅读该示例前，建议您先了解以下内容：

[PaddleRec入门教程](https://github.com/PaddlePaddle/PaddleRec/blob/master/README.md)

## 内容

- [模型简介](#模型简介)
- [数据准备](#数据准备)
- [运行环境](#运行环境)
- [快速开始](#快速开始)
- [模型组网](#模型组网)
- [效果复现](#效果复现)
- [进阶使用](#进阶使用)
- [FAQ](#FAQ)

## 模型简介
`CTR(Click Through Rate)`，即点击率，是“推荐系统/计算广告”等领域的重要指标，对其进行预估是商品推送/广告投放等决策的基础。简单来说，CTR预估对每次广告的点击情况做出预测，预测用户是点击还是不点击。CTR预估模型综合考虑各种因素、特征，在大量历史数据上训练，最终对商业决策提供帮助。本模型实现了下述论文中的 DCN_V2 模型：

```text
@article{DCN_V2 2020,
  title={DCN V2: Improved Deep & Cross Network and Practical Lessons for Web-scale Learning to Rank Systems},
  author={Ruoxi Wang, Rakesh Shivanna, Derek Z. Cheng, Sagar Jain, Dong Lin, Lichan Hong, Ed H. Chi},
  journal={arXiv preprint arXiv:2008.13535v2},
  year={2020}，
  url={https://arxiv.org/pdf/2008.13535v2.pdf},
}
```

## 数据准备

#### 数据准备:1.下载Criteo原始数据和对应的Sparse特征数量文件

```
# 切换路径至数据处理脚本目录下
cd work/PaddleRec/datasets/criteo_dcn_v2/

# 下载、解压数据
wget --no-check-certificate https://paddlerec.bj.bcebos.com/deepfm%2Ffeat_dict_10.pkl2
wget --no-check-certificate https://fleet.bj.bcebos.com/ctr_data.tar.gz
tar -zxvf ctr_data.tar.gz
mv ./raw_data ./train_data_full
mkdir train_data && cd train_data
cp ../train_data_full/part-0 ../train_data_full/part-1 ./ && cd ..
mv ./test_data ./test_data_full
mkdir test_data && cd test_data
cp ../test_data_full/part-220 ./  && cd ..
echo "Complete data download."
echo "Full Train data stored in ./train_data_full "
echo "Full Test data stored in ./test_data_full "
echo "Rapid Verification train data stored in ./train_data "
echo "Rapid Verification test data stored in ./test_data "
```
####  数据准备:2.处理原始数据，对dense特征进行log化
```bash
# 在终端中运行数据处理脚本

# 确保目录在 work/PaddleRec/datasets/criteo_dcn_v2/
sh data_process.sh

```

## 运行环境
PaddlePaddle>=2.0

python 2.7/3.5/3.6/3.7

os : windows/linux/macos 

## 快速开始
本文提供了[DCNv2-Paddle复现 AiStudio项目](https://aistudio.baidu.com/aistudio/projectdetail/3406204)可以供您快速体验，进入项目快速开始。


## 模型组网
DCN_V2 模型的组网，代码参考 `net.py`。模型主要组成是 Embedding层，CrossNetwork 层，MLP 层，以及相应的分类任务的loss计算和auc计算。另外，DCN_V2在DCN的基础
上，根据CrossNetwork 层和MLP 层的堆叠方式，网络结构又分为Stacked和Parallel两种方式，模型架构如下：

<img align="center" src="https://wx4.sinaimg.cn/mw2000/0073e4AWgy1gyao6r7ovbj30ov0guqac.jpg">

### **CrossNetwork 层**
CrossNetwork的核心是创建显示的特征交叉，每一层都与原始特征进行特征交叉，每一层的输出是下一层的输入，计算方式如下公式（1）所示，计算方法可视化表达如下图所示：

<img align="center" src="https://wx2.sinaimg.cn/mw2000/0073e4AWgy1gyaotiqopbj30hh01nt8w.jpg">

<img align="center" src="https://wx3.sinaimg.cn/mw2000/0073e4AWgy1gyaohnd39zj30hb06i0ts.jpg">

CrossNetwork计算特征交叉时，随着层数和特征维度的增大，计算成本也比较高，论文中设计了降低计算成本的CrossMix网络结构，如下介绍。

### **Cost-Effective Mixture of Low-Rank DCN**
如上公式（1）所示，权重矩阵 W 是一个具有高秩的矩阵，论文中将W分解为两个低秩的矩阵U 和 V 有效降低了计算成本，如下公式（2）所示，但在精度性能上效果却略逊色前一种方式。

<img align="center" src="https://wx4.sinaimg.cn/mw2000/0073e4AWgy1gyap3vkyq1j30k301xwev.jpg">



### **Loss 及 Auc 计算**
- 为了得到每条样本分属于正负样本的概率，我们将预测结果和 `1-predict` 合并起来得到 `predict_2d`，以便接下来计算 `auc`。  
- 每条样本的损失为负对数损失值，label的数据类型将转化为float输入。  
- 该batch的损失 `avg_cost` 是各条样本的损失之和
- 我们同时还会计算预测的auc指标。

## 效果复现
为了方便使用者能够快速的跑通每一个模型，我们在每个模型下都提供了样例数据。如果需要复现 README 中的效果,请按如下步骤依次操作即可。
在全量数据下模型的指标如下：  

| 模型 | auc | logloss | batch_size | epoch_num| Time of each epoch |
| :------| :------ | :------ | :------| :------ | :------ | 
| DCN_V2 | 0.8026 | 0.4384 |512 | 1 | 约 3 小时 |

1. 确认您当前所在目录为 `PaddleRec/models/rank/dcn_v2`

```
# 动态图训练
python -u ../../../tools/trainer.py -m config_bigdata.yaml # 全量数据运行config_bigdata.yaml 
python -u ../../../tools/infer.py -m config_bigdata.yaml # 全量数据运行config_bigdata.yaml 
```

## 进阶使用
  
## FAQ