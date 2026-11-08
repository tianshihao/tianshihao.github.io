+++
date = '2026-08-24T10:36:48+08:00'
title = 'BEVFormer Quantization Notes 2'
tags = ['Quantization']
series = ['BEVFormer量化']
toc = true
+++

# BEVFormer量化笔记2

这周尝试重新量化新版本BEVFormer模型，重新走通上一次的流程同时对流程进行优化。我会重新梳理，更详细地记录。模型导出量化主要关心这几个事情：

1. ONNX导出精度
1. 模型量化精度
1. 端侧部署精度

我目前获得的输入，即模型，是一个目录，该目录包含：

1. 若干.pth文件，命名`epoch_10.pth`、`epoch_20.pth` …… `epoch_120.pth`，最后有个`latest.pth`。

> .pth 就是 PyTorch 的模型权重文件，类似于一个压缩包，里面存了神经网络所有层的参数（权重、偏置等）。用 torch.load() 就能加载。

> epoch：训练时，模型把所有训练数据完整看一遍，叫一个 epoch。epoch_10.pth 就是训练了 10 轮后保存的模型，epoch_120.pth 就是 120 轮后的。

2. 模型配置文件`<model_name.py>`。这是一个基于 OpenMMLab 体系（MMDetection3D） 的标准配置文件，采用注册机制（Registry Pattern）来组装整个模型。它本质上是一个声明式的流水线定义，覆盖了从数据输入到训练推理的全流程：
    1. 模型架构 — BEVFormer，基于 Transformer 的纯视觉鸟瞰图感知模型
    1. 视觉主干 — ResNet50（ImageNet 预训练），输出特征图送入 FPN 做多尺度融合
    1. 检测目标 — 15 类（包括宽体车、轿车、装载机、洒水车、挖掘机、压路机等矿用/工程车辆及行人、交通标志等）
    1. 输入模态 — 纯视觉方案，3 路环视相机，不使用激光雷达/雷达/地图
    1. 感知范围 — 沿 X 轴 -51.2m ~ 153.6m，Y 轴 -51.2m ~ 51.2m，Z 轴 -5m ~ 10m，覆盖前方远距离检测区域
    1. BEV 网格 — 64×128 分辨率，每个网格对应实际空间中的 0.2m×0.2m
    1. 训练策略 — AdamW 优化器，CosineAnnealing 学习率衰减，训练 120 个 epoch
    1. 数据预处理 — 多视图图像归一化（均值 [150.7, 147.3, 147.2]）、随机缩放（0.5 倍）、填充对齐至 32 倍数

## 1. Export Pytorch GT

每次模型更新都需要获取Pytorch数据作为GT。方法就是读取模型配置，构建模型，读取模型需要的数据，完成数据预处理，跑一次前向，拿到模型输出，然后将实际喂给模型的输入以及模型输出保存下来，这样就拿到了该模型的Pytorch基线。

> 保存下来的数据格式可以是.npy，也可以是.bin。

一般而言，给模型的数据分为两类。两者的区别主要在数据来源、预处理方式和标定来源上。命令里可以使用`--from-val` 和 `--scene-dir`区分，是互斥的。


| 维度              | --from-val                                                                                | --scene-dir                                                                                                                                             |
| :---------------- | :---------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 数据来源          | val 数据集（pkl，/data_bevformer/syzk_data/kitti_infos_val.pkl），按索引 dataset[i] 取    | 原始场景目录里的图片（camera_long/short/back/*.jpg），不需要 pkl                                                                                        |
| 预处理            | 直接用 val 数据集的 pipeline（LoadMultiViewImageFromFiles→Normalize→MultiScaleFlipAug3D） | 脚本自己用 load_and_preprocess_image() 按 config 手动复刻（Normalize→Resize(0.5)→Pad），参数来自 extract_preprocess_params()                            |
| 标定（lidar2img） | 不需要额外参数，img_metas 由 get_img_metas_template() 从 val 数据里取                     | 必须传 --calib-dir，从 JSON 标定文件（center_camera-intrinsic.json + top_center_lidar-to-center_camera-extrinsic.json）解析出 lidar2img；can_bus 填全 0 |
| 取哪些帧          | 取数据集前 --num-samples 个                                                               | 扫描 camera_long 里的 .jpg，帧数多于需求时均匀抽样（np.linspace）                                                                                       |
| 适用场景          | 最省事：快速对已知 val 集导出 GT                                                          | 处理 val pkl 之外的新原始场景/特定场景（如示例里的 061586车扬尘... 场景 + calib/car_86）                                                                |


> .pkl是元数据索引

> **注**：本篇在第 1 节后中断，第 2–4 节内容遗失；笔记 3 直接从第 5 节「Export Engine」开始。
