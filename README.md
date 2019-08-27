# GroupSoftmax-SimpleDet
![](./doc/image/demo83.jpg)

|Model|Backbone|Head|GroupSoftmax|Num Classes|Train Schedule|FP16|AP|AP50|AP75|APs|APm|APl|Link|
|-----|--------|----|------------|-----------|--------------|----|--|----|----|---|---|---|----|
|Faster-SyncBN|R101v2-C4|C5-256ROI|no|80|1X|no|38.6|-|-|-|-|-|[model](https://simpledet-model.oss-cn-beijing.aliyuncs.com/faster_r101v2c4_c5_256roi_syncbn_1x.zip)|
|Faster-SyncBN|R101v2-C4|C5-256ROI|yes|83|1X|yes|39.3|59.9|42.3|21.0|44.1|53.3|-|
|Trident*|R101v2-C4|C5-128ROI|yes|83|1X|yes|44.0|64.9|48.4|29.0|47.8|57.6|-|

---
## GroupSoftmax Cross Entropy Loss Function
GroupSoftmax cross entropy loss function is implemented for training with multiple different benchmark datasets. We trained a 83 classes detection model by using COCO and CCTSDB.

---
## GroupSoftmax交叉熵损失函数
GroupSoftmax交叉熵损失函数能够支持不同标注标准的数据集进行联合训练，在工业界的实际生产环境中，能够有效解决下面四个问题
- 在不重新标注原有数据的情况下，对新的标注数据，增加某些新的类别
- 在不重新标注原有数据的情况下，对新的标注数据，修改原有的类别标准，比如将某个类别拆分成新的几个类别
- 已标注数据集中，出现概率较高的类别一般属于简单样本，对新的数据只标注出现概率较低的类别，能够显著降低标注成本
- 在标注大型目标检测数据集时，只标注矩形框，而不选择类别属性，能够有效降低标注成本

我们利用GroupSoftmax交叉熵损失函数在COCO和CCTSDB数据集上进行了联合训练，得到了一个83类检测器。有趣的是，模型不仅有83类检测效果，在coco_minival2014测试集上的表现比原来80类检测器反而会好一些。也就是说我们利用了一个与COCO无关的CCTSDB数据集，在相同参数下，Faster RCNN算法的检测效果由原来的38.6提高到了39.3，提高了0.7个点。为了验证GroupSoftmax交叉熵损失函数的有效性，我们同时训练了一个83类Trident*模型，最后在coco_minival2014测试集上mAP指标为44.0，所以从理论上而言，利用GroupSoftmax交叉熵损失函数，你可以无限添加不同标注标准的数据集，进行联合训练。
### [GroupSoftmax交叉熵损失函数详解见知乎](https://zhuanlan.zhihu.com/p/73162940)

### USAGE
GroupSoftmax用法参考[groupsoftmax_faster_r101v2c4_c5_256roi_syncbn_1x.py](./config/groupsoftmax_faster_r101v2c4_c5_256roi_syncbn_1x.py)中的GroupParam设置，下面举例说明用法。如果CCTSDB中同时标注了preson类别和3类交通标注，也即CCTSDB和COCO中都标注了person类别，则应该做出改动的地方有三个：
- RPN的分类任务应该由3分类修改为4分类，因为此时有4中情况，分别为：背景、前景1、前景2、前景3，其中前景1为COCO和CCTSDB中都标注了的person，前景2为COCO中的其他79个类别，前景3为CSTSDB中的3类交通标志。所以应该将GroupParam中的rpnvx信息，由原来的：
    ```
    rpnv0 = np.array([0, 1, 2], dtype=np.float32)     # rpn 3 classes
    rpnv1 = np.array([0, 1, 0], dtype=np.float32)     # COCO benchmark
    rpnv2 = np.array([0, 0, 2], dtype=np.float32)     # CCTSDB benchmark
    ```
    修改为：
    ```
    rpnv0 = np.array([0, 1, 2, 3], dtype=np.float32)     # rpn 4 classes
    rpnv1 = np.array([0, 1, 2, 0], dtype=np.float32)     # COCO benchmark
    rpnv2 = np.array([0, 1, 0, 3], dtype=np.float32)     # CCTSDB benchmark
    ```
    前景1在COCO和CCTSDB中都进行标注了。而前景3在COCO中未标注，所以前景3在COCO的样本中会与背景类组成一个group组类别。而前景2在CCTSDB中未标注，所以前景2在CCTSDB的样本中会与背景类组成一个group组类别。本质是某个类别未标注可以理解为将某个类别标注为背景。

- 修改RPN网络的类别映射方法gtclass2rpn，原来是3分类网络，类别映射为：{0 ==> 0, [1,2,...,80] ==> 1, [81,82,83] ==> 2}。现在是4分类网络，类别映射应该修改为：{0 ==> 0, 1 ==> 1, [2,3,...,80] ==> 2, [81,82,83] ==> 3}。对应的gtclass2rpn方法应该由原来的：
    ```
    def gtclass2rpn(gtclass):
        class_gap = 80
        gtclass[gtclass > class_gap] = -1
        gtclass[gtclass > 0] = 1
        gtclass[gtclass < 0] = 2
        return gtclass
    ```
    修改为：
    ```
    def gtclass2rpn(gtclass):
        class_gap = 80
        gtclass[gtclass > class_gap] = -1
        gtclass[gtclass == 1] = 1
        gtclass[gtclass > 1] = 2
        gtclass[gtclass < 0] = 3
        return gtclass
    ```
- 在Head网络中，跟原来唯一的区别是CCTSDB中标注了person类别，因为person类别在COCO中类别id为1，所以对应的，在CCTSDB的boxv2中，id为1的类别也由原来的0修改为1，因为已经标注了，不再与背景类0组成一个group进行训练。修改之后如下：
    ```
    # box 83 classes
    boxv0 = np.array([0,  1,  2,  3,  4,  5,  6,  7,  8,  9,  10, 11, 12, 13, 14, 15, \
                          16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, \
                          31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, \
                          46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, \
                          61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, \
                          76, 77, 78, 79, 80, 81, 82, 83], dtype=np.float32)
    #COCO benchmark
    boxv1 = np.array([0,  1,  2,  3,  4,  5,  6,  7,  8,  9,  10, 11, 12, 13, 14, 15, \
                          16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, \
                          31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, \
                          46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, \
                          61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, \
                          76, 77, 78, 79, 80, 0,  0,  0 ], dtype=np.float32)
    #CCTSDB benchmark
    boxv2 = np.array([0,  1,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  \
                          0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  \
                          0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  \
                          0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  \
                          0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  \
                          0,  0,  0,  0,  0,  81, 82, 83], dtype=np.float32)
    ```


---
## SimpleDet - A Simple and Versatile Framework for Object Detection and Instance Recognition
### Major Features
![](./doc/image/diagram.png)
- FP16 training for memory saving and up to **2.5X** acceleration
- Highly scalable distributed training available **out of box**
- Full coverage of state-of-the-art models including FasterRCNN, MaskRCNN, CascadeRCNN, RetinaNet, **[TridentNet](./models/tridentnet)**, **[NASFPN](./models/NASFPN)** and **[EfficientNet](./models/efficientnet)**
- Extensive feature set including **large batch BN**, **loss synchronization**, **automatic BN fusion**, deformable convolution, soft NMS, multi-scale train/test
- Modular design for coding-free exploration of new experiment settings

### Recent Updates
- Add RPN test (2019.05.28)
- Add [NASFPN](https://github.com/TuSimple/simpledet/tree/master/models/NASFPN) (2019.06.04)
- Add new ResNetV1b baselines from GluonCV (2019.06.07)
- Add Cascade R-CNN with FPN backbone (2019.06.11)
- Speed up FPN up to 70% (2019.06.16)
- Update [NASFPN](https://github.com/TuSimple/simpledet/tree/master/models/NASFPN) to include larger models (2019.07.01)
- Automatic BN fusion for fixed BN training, saving up to 50% GPU memory (2019.07.04)
- Speed up MaskRCNN by 80% (2019.07.23)
- Update MaskRCNN baselines (2019.07.25)
- Add EfficientNet and DCN (2019.08.06)

### Setup
#### Install
SimpleDet contains a lot of C++ operators not in MXNet offical repo, so one has to build MXNet from scratch. Please refer to [INSTALL.md](./doc/INSTALL.md) more details

#### Preparing Data
SimpleDet requires groundtruth annotation organized as following format
```
[
    {
        "gt_class": (nBox, ),
        "gt_bbox": (nBox, 4),
        "flipped": bool,
        "h": int,
        "w": int,
        "image_url": str,
        "im_id": int,

        # this fields are generated on the fly during test
        "rec_id": int,
        "resize_h": int,
        "resize_w": int,
        ...
    },
    ...
]
```

Especially, for experimenting on coco datatet, one can organize coco data in
```
data/
    coco/
        annotations/
            instances_train2014.json
            instances_valminusminival2014.json
            instances_minival2014.json
            image_info_test-dev2017.json
        images/
            train2014
            val2014
            test2017
```

and run the helper script to generate roidb
```bash
python3 utils/generate_roidb.py --dataset coco --dataset-split train2014
python3 utils/generate_roidb.py --dataset coco --dataset-split valminusminival2014
python3 utils/generate_roidb.py --dataset coco --dataset-split minival2014
python3 utils/generate_roidb.py --dataset coco --dataset-split test-dev2017
```

#### Deploy dependency and compile extension
1. setup mxnext, a wrapper of mxnet symbolic API
```bash
cd $SIMPLEDET_DIR
git clone https://github.com/RogerChern/mxnext
```
2. run make in simpledet directory to install cython extensions

#### Quick Start

```bash
# train
python3 detection_train.py --config config/detection_config.py

# test
python3 detection_test.py --config config/detection_config.py
```


### Project Design
#### Model Zoo
Please refer to [MODEL_ZOO.md](./MODEL_ZOO.md) for available models

#### Code Structure
```
detection_train.py
detection_test.py
config/
    detection_config.py
core/
    detection_input.py
    detection_metric.py
    detection_module.py
models/
    FPN/
    tridentnet/
    maskrcnn/
    cascade_rcnn/
    retinanet/
mxnext/
symbol/
    builder.py
```
#### Config
Everything is configurable from the config file, all the changes should be **out of source**.

#### Experiments
One experiment is a directory in **experiments** folder with the same name as the config file.
> E.g. r50_fixbn_1x.py is the name of a config file

```
config/
    r50_fixbn_1x.py
experiments/
    r50_fixbn_1x/
        checkpoint.params
        log.txt
        coco_minival2014_result.json
```

#### Models
The `models` directory contains SOTA models implemented in SimpletDet.

#### How is Faster-RCNN built
**Simpledet** supports many popular detection methods and here we take [**Faster-RCNN**](https://arxiv.org/abs/1506.01497) as a typical example to show how a detector is built.

- *Preprocessing*. The preprocessing methods of the detector is implemented through `DetectionAugmentation`.
  - Image/bbox-related preprocessing, such as `Norm2DImage` and `Resize2DImageBbox`.
  - Anchor generator `AnchorTarget2D`, which generates anchors and corresponding anchor targets for training RPN.
- *Network Structure*. The training and testing symbols of Faster-RCNN detector is defined in `FasterRcnn`. The key components are listed as follow:
  - *Backbone*. `Backbone` provides interfaces to build backbone networks, *e.g.* ResNet and ResNext.
  - *Neck*. `Neck` provides interfaces to build complementary feature extraction layers for backbone networks, *e.g.* `FPNNeck` builds Top-down pathway for [Feature Pyramid Network](https://arxiv.org/abs/1612.03144).
  - *RPN head*. `RpnHead` aims to build classification and regression layers to generate proposal outputs for RPN. Meanwhile, it also provides interplace to generate sampled proposals for the subsequent R-CNN.
  - *Roi Extractor*. `RoiExtractor` extracts features for each roi (proposal) based on the R-CNN features generated by `Backbone` and `Neck`.
  - *Bounding Box Head*. `BboxHead` builds the R-CNN layers for proposal refinement.

#### How to build a custom detector
The flexibility of **simpledet** framework makes it easy to build different detectors. We take [**TridentNet**](https://arxiv.org/abs/1901.01892) as an example to demonstrate how to build a custom detector simply based on the **Faster-RCNN** framework.

- *Preprocessing*. The additional processing methods could be provided accordingly by inheriting from `DetectionAugmentation`.
  - In TridentNet, a new `TridentAnchorTarget2D` is implemented to generate anchors for multiple branches and filter anchors for scale-aware training scheme.
- *Network Structure*. The new network structure could be constructed easily for a custom detector by modifying some required components as needed and
  - For TridentNet, we build trident blocks in the `Backbone` according to the descriptions in the paper. We also provide a `TridentRpnHead` to generate filtered proposals in RPN to implement the scale-aware scheme. Other components are shared the same with original Faster-RCNN.


### Distributed Training
Please refer to [DISTRIBUTED.md](./doc/DISTRIBUTED.md)


### Contributors
Yuntao Chen, Chenxia Han, Yanghao Li, Zehao Huang, Yi Jiang, Naiyan Wang


### License and Citation
This project is release under the Apache 2.0 license for non-commercial usage. For commercial usage, please contact us for another license.

If you find our project helpful, please consider cite our tech report.
```
@article{chen2019simpledet,
  title={SimpleDet: A Simple and Versatile Distributed Framework for Object Detection and Instance Recognition},
  author={Chen, Yuntao and and Han, Chenxia and Li, Yanghao and Huang, Zehao and Jiang, Yi and Wang, Naiyan and Zhang, Zhaoxiang},
  journal={arXiv preprint arXiv:1903.05831},
  year={2019}
}
```
