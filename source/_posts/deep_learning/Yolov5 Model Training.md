---
title:  简单使用YOLOv5框架进行模型训练
date: 2022/12/7 23:05:00
categories:
- Deep Learning
tags:
- Deep Learning
---
<center><font size = "6"><b>简单使用YOLOv5框架进行模型训练 </b></font> </center>

<div align=center>azusaings@gmail.com</div>



<b>摘要:</b>&ensp;本课程论文将从搭建环境，准备数据集，模型训练等三个方面进行一次 **YOLOv5** 训练图像识别模型的探索学习，并进行总结，为接下来的学习铺垫。

<b>关键词:</b> &ensp;计算机视觉，图像分类，**YOLOv5** ，深度学习网络框架

### 1  引言

​	从上世纪五十年代开始，从图像处理的文字识别到数字图像处理与识别，再到近年的物体三维识别，图像识别的发展已经经历了半个多世纪。人们在图像识别的领域中积累了不少方法与经验，也开发出了一些图像识别的框架。目前主流的依靠深度学习实现的图像识别框架有 **YOLO** ，**Caffe**，**Theano**等。本文将围绕 **YOLOv5** 的使用以及模型训练等方面对 **YOLOv5** 框架进行探索学习，并进行经验总结。

​	 **YOLO** 是一个使用 **C** 与 **CUDA** 开发的开源神经网络框架，它支持 **CPU** 与 **GPU** 计算。其物体检测算法拥有极高的速度与精确度，并且开放了不同规模预训练模型（ **.pt** 文件）以支持不同情景下的开发。从 2015 年 7 月 **YOLOv1** 到 2020 年 4 月最新的 **YOLOv5** ， **YOLO** 框架已经成为最受人们欢迎的图像识别框架之一。

### 2  使用yolov5训练图像识别模型

​		本节将会介绍使用 **YOLOv5** 训练模型所需要的环境，顺序从 **CUDA** 以及 **cuDNN** 到 **PyTorch** ，再到 **YOLOv5** 。并准备数据集，进行模型训练，最后进行模型测试。

#### 2.1  环境搭建

​	<b>CUDA 与 cuDNN</b>

​		 **CUDA** （**Compute Unified Device Architecture**，统一计算设备架构） 是显卡厂商 **NVIDIA** 推出的通用并行计算架构，该架构包含了CUDA指令集架构（ **Instruction Set Architecture**， **ISA**）以及 **GPU** 内部的并行计算引擎，使 **GPU** 能解决复杂的计算问题。大部分深度学习框架例如 **TensorFlow** 与 **PyTorch** 都依赖于该框架，我们可以在[CUDA官网](https://developer.nvidia.com/cuda-downloads)下载安装。

​		**cuDNN**（**CUDA Deep Neural Network**， **CUDA** 深度神经网络）是用在 **CUDA** 上的深度神经网络的 **GPU** 加速库，使 **CUDA** 更适用于深度神经网络的开发。安装时与 **CUDA** 的版本有严格的对应，可以在[官网](https://developer.nvidia.com/rdp/cudnn-archive)下载。本文选择了 **CUDA 11.7** 与 **cuDNN 8.5.0** 的组合。

​	<b>PyTorch</b>

​		**PyTorch** 是由 **Facebook** 公司使用 **Python** 开发的深度学习框架。2017年1月， **Facebook**人工智能研究院 **FAIR**（**Facebook Artificial Intelligence Research**）在 **GitHub** 上开源了 **PyTorch** 。安装了 **CUDA** 与 **cuDNN** 环境之后，我在 **Anaconda** 环境下安装 **PyTorch** 。在 [PyTorch官网](https://pytorch.org/) 选择配置后将生成的命令在虚拟环境下运行进行安装。

```powershell
	conda install pytorch torchvision torchaudio pytorch-cuda=11.7 -c pytorch -c nvidia
```

按照上述过程安装将得到一个可使用GPU运算的 **PyTorch** 版本。

​	<b>YOLOv5</b>

​	如上文所提到，[YOLOv5](https://github.com/ultralytics/yolov5)已经成为图像识别的主流框架之一。除了能训练自定义模型，官方给出了不同规模问题的预训练模型供使用，包括 **YOLOv5s** ， **YOLOv5m** 与 **YOLOv5x** 等。更大的模型意味着更高的识别准确度，但也意味着需要更长的时间进行识别。图1中，横轴代表在半精度（**FP16**） **RTX3060** 以及 **PyTorch** 环境下模型的延迟（毫秒），纵轴表示 **COCO** 数据集下模型的**Mask AP**值（**Mask Average Precision Val**）。相比之下，可以看到 **YOLOv5s** 具有延迟更小，但准确度更高的特性，而 **YOLOv5x** 恰好相反。图 **1** 同时对比展示了 **sparseinst** 以及 **solov2** 模型的表现。![image-20221127232315991](/images/y5/F1.png)

<b>图 1</b>  **YOLOv5** 的部分模型以及与其他模型的对比。

​	我们可以通过以下终端命令在 **GitHub** 上下载 **YOLOv5** 的文件，并且通过 `pip` 指令安装 **YOLOv5** 的其他依赖

```powershell
	git clone https://github.com/ultralytics/yolov5  # clone
	cd yolov5
	pip install -r requirements.txt  # install
```

​	根据官方文档提示，安装时应该注意保证 **Python** 版本在 **3.7.0** 及以上，**PyTorch** 版本在 **1.7** 及以上。

#### 2.2  训练目标

​	本次实验的目标识别出包含 _ariplane_ ，_automobile_ ，_truck_ 在内的十个类。训练后的模型在检测完成后应该将给定图片中的目标类用颜色框标注。

#### 2.3  数据集准备

​	本文将使用[CIFAR-10数据集](http://www.cs.toronto.edu/~kriz/cifar.html)作为训练集与验证集。**CIFAR-10** 数据集中包含十个类别，每个类别包含 **6000** 张 **32x32** 的.png彩色图片，人工标注的正确率为 **94\%** 。为了能使 **CIFAR-10** 满足 **YOLOv5** 数据集需要的格式，我使用 **imageio** ，**pickle** 与 **numpy** 包编写IO.py脚本预先进行了一些数据整理工作，脚本可访问 [个人网页](http://120.77.13.22/downloads/YOLOv5_Model_Training/) 下载，本文不再赘述。

<img src="/images/y5/F2.png" style="zoom: 80%;" />

<b>图 2</b> **CIFAR-10** 数据集

​	使用 **YOLOv5** 框架进行训练，数据集有固定的格式。其中包含.yaml格式文件，包含.jpg/.png格式图片的训练与验证文件夹，以下给出了数据集的格式，注意子目录下 images 与 labels 文件名不可修改。

```powershell
								+-- images Dir -- .jpg/.png files
				+---- val Dir --+ 
				|				+-- labels Dir -- .txt files
				|
	Root Dir----+---- train Dir (same as 'val Dir')  
				|
				+---- .yaml
```

​	<b>.yaml文件</b>  .yaml文件包含了整个测试集的总览信息，其中包括数据集的根目录，训练数据以及验证数据的目录和数据集定义的各个类。 下面给出了官方文档中的实例。其中 train 以及 val 应指明包含图像文件夹与标签文件夹的目录。
```yaml
	# Train/val/test sets as 1) dir: path/to/imgs, 2) file: path/to/imgs.txt, or 3) list: 	[path/to/imgs1, path/to/imgs2, ..]
	path: ../datasets/coco128  # dataset root dir
	train: images/train2017  # train images (relative to 'path') 128 images
	val: images/train2017  # val images (relative to 'path') 128 images
	test:  # test images (optional)

	# Classes (80 COCO classes)
	names:
 	 0: person
 	 1: bicycle
 	 2: car
 	 ...
 	 77: teddy bear
  	 78: hair drier
  	 79: toothbrush
```

​	<b>训练数据与测试数据文件夹</b>  训练数据文件夹一样，文件夹都保存 **.jpg/.png** 原始数据文件夹以及 **.txt** 标注信息文件夹。需要注意的是在 **.txt** 标注信息文件中，每一行数据对应一张图片，且按照空格符分隔，包含目标的五个严格按照次序排列的数值参数：类别，横坐标，纵坐标，宽和高（class，x_center,  y_center,  width,  height），所有参数都应该标准化（缩放至 **0** 到 **1** 之间）。为了使每一张图片与其标签文件能够对应，图片与其标签的文件名称最后可以用相同的数字标识， **YOLOv5** 将会自动识别。

```powershell
	../datasets/coco128/images/im0.jpg  # image
	../datasets/coco128/labels/im0.txt  # label
```

![image-20221128095522541](/images/y5/F3.png)

<b>图 3</b>  .yaml文件中 x_center,  y_center,  width,  height 含义展示

#### 2.4  模型训练

​	上文提到 **YOLOv5** 提供了不同的模型下载使用，这里我选择 **YOLOv5s** 作为训练模型，并在 **Anaconda\,\,Prompt** 中参考官方的说明进行训练

```powershell
	# Train YOLOv5s on COCO128 for 3 epochs
	python train.py --img 640 --batch 16 --epochs 3 --data coco128.yaml --weights yolov5s.pt
```

​	其中 `--img` 参数指明了输入图片在模型中训练时的尺寸，会自动进行调整。`--data` 指出上文所述.yaml文件路径，而`--weights` 指出使用的模型路径 

​	我将batch设置为 **48** ，图像尺寸为 **320** ，且总共进行十轮训练，平均每轮训练耗费五分钟。训练过程中，从第一轮开始每一轮的 **mAP** 值都达到了 **100\%** ，训练结束后.pt后缀的模型将会被保存在指定文件夹（默认在runs/train下）。同样，我已将训练好的模型 best.pt 上传[个人网页](http://120.77.13.22/downloads/YOLOv5_Model_Training)。

#### 2.5  模型测试

​	要将训练好的模型进行测试，可使用以下命令。其中 `--conf` 参数可以指定置信度阈值 ，更多的参数可以参考官方文档进行设置。我将分别在 **CIFAR-10** 的数据与网上随机搜索得到的数据上进行测试。

```powershell
	python detect.py --weights .pt --source .jpg --conf 0.25
```

​	先在 **CIFAR-10** 的数据上进行测试，可以看到 **16** 张图片中有 **15** 张被识别。

<img src="/images/y5/F4.jpg" alt="val_batch1_pred" style="zoom: 50%;" />

<b>图 4</b>  在**CIFAR-10** 数据集上进行测试

​	接着我随即从网络挑选了 **16** 张图片， **13** 张图片被识别。其中包含猫特征的动漫图片也被识别出来，可以看出模型提取到了猫类的特征。但是置信阈值设置的不太合适，使物体被多次识别，出现了多个识别框与不同的置信度。

<img src="/images/y5/F5.jpg" alt="my_val" style="zoom: 50%;" />

<b>图 6</b>  在从网络随机搜索的得到的图片上进行测试

### 3  总结

​	这次实验，我使用 **YOLOv5** 框架与 **CIFAR-10** 数据集在 **GTX\,\,1650Ti** 上进行了大约一个小时的训练。训练后的模型已经可以大概进行图像识别分类，并在陌生的数据集上表现出了不错的准确度。

​	但由于经验不足，在训练模型的时候对于各个参数的设置，包括批量大小与图像大小，不太熟悉。同时这次实验缺乏科学严谨的模型分析方法，使得后期在模型改进与模型测试的过程中缺乏方向，仅仅只在极少的数据上进行实验并得出结论。后续将参考包含 [A Karpathy's Recipe for Training Neural Networks](https://courses.cs.cornell.edu/courses/cs5740/2021sp/resources/nn-tips.pdf) 在内的更多训练深度学习模型的资料进行研究学习。

### 参考文献

​	1. **YOLOv5** :  **Documentation**. https://docs.ultralytics.com/ 

​	2. **CSDN** : 机器&深度学习中的模型评估. https://blog.csdn.net/buchidanhuang/article/details/90172119

​	3. **CSDN** : YOLOv5的详细使用教程. https://blog.csdn.net/sinat_28371057/article/details/120598220
