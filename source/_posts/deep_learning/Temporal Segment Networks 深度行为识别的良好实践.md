---
title:  TSM 论文翻译
date: 2022/12/6 23:00:00
categories:
- Deep Learning
tags:
- Deep Learning
---

<center><font size = "6"><b>Temporal Segment Networks: 深度行为识别的良好实践 </b></font> </center>
 
<div align=center>Limin Wang , Yuanjun Xiong , Zhe Wang , Yu Qiao , Dahua Lin, Xiaoou Tang , and Luc Van Gool <br> Computer Vision Lab, ETH Zurich, Switzerland <br> Department of Information Engineering, The Chinese University of Hong Kong <br> Shenzhen Institutes of Advanced Technology, CAS, China </div>



<b>摘要:</b> &ensp;深度卷积神经网络已经在静止图像的识别上取得了巨大成果。然而对于视频中的行为识别, 传统方法的效果并不理想。这篇文章旨在设计在视频中的高效的行为识别卷积网络结构，以及在有限的样本下训练它们。我们的首个成果是时间段网络 (**TSN**)，一个基于视频行为识别的全新框架。 它源自于长期时间结构建模，将稀疏时间采样的策略与视频级监督相结合，使网络可以高效地利用整个动作视频进行学习。另一个成果是我们借时间段网络在使用卷积神经网络学习视频数据领域的一系列良好实践。我们的方法在**HMDB51** 与 **UCF101**数据集上取得了领先的表现(**69.4%** 和 **94.2%**)。我们也可视化了训练的卷积神经网络模型，它量化地展示了时间段网络的高效性。 

<b>关键词:</b> &ensp; 行为识别；时间段网络；Good Practices；卷积神经网络

### 1  介绍

​	基于视频的行为识别已经在学术界引起了学术界强烈的重视，归功于他在安全分析以及行为分析等诸多领域的应用。在行为识别中，有两个至关重要且互补的方面：外观以及动态。一个行为识别系统的表现很大程度上取决于它是否能提炼并运用所得来的相关信息。然而，在高复杂度下提炼这样的信息并不简单，例如在目标尺寸变化， 视点变化以及相机运动等情况下。因此在保留行为类的分类信息的前提下，设计一个有效的方案以应对这些挑战变得至关重要。最近，卷积神经网络(**ConvNets**) 在关于物体，情景以及复杂事件的图像分类中取得了巨大的成功。同时，它也被引进视频行为识别的领域处理特定的问题。深度神经网络拥有极好的建模能力，并且能够借助大规模数据集，从原始视觉数据中学习到不同的表示。然而，不同于图像分类，端到端的深度卷积神经网络仍然无法在视频行为识别领域超越传统人工特征。

​	在我们看来，卷积神经网络在视频行为识别中的应用被两个关键因素阻碍着。首先，尽管长期的时间结构在理解行为视频动态中发挥重要作用，但是，主流的卷积神经网络框架总是关注外观以及短期动作，因此它们缺乏处理长期时间结构的能力。近期有不少处理这个问题的尝试。这些尝试的方法依靠于在提前定义好采样间隔后的，密集的时间采样。这个方法被应用在长视频序列上时，附带过高的计算代价，从而限制了其在真实世界的应用，而且带来丢失超过序列限制的重要信息的风险。其次，在实践中，训练深度卷积神经网络需要大量的训练采样以使模型取得最优的表现。然而由于数据收集以及数据标注的困难，公开可用的行为识别数据集在大小与多样化上有着限制。因此在图像识别取得显著成功的极深卷积神经网络，正面临着更高几率过拟合的风险。

​	这些挑战激励着我们研究这两个问题：(1)如何设计一个高效的视频级框架以学习视频表示，从而能够捕捉长期的时间结构(2)如何在有限的训练采样下训练卷积神经网络。通常在应对以上问题时，我们会在最成功的双流结构上构建我们自己的方案。就时间结构建模而言，有一个重要的观察结果，即连续的帧是高度冗余的。因此，通过高度相似的采样帧获得结果的密集时间采样，是不必要的。相反，稀疏时间采样的策略在这个情境下将会更受青睐。受到这一观察结果的启发，我们开发了一个视频级框架，时间段网络(**TSN**)。这个框架使用稀疏时间采样的方案，其中所有采样均匀分布在时间维度上，从长视频序列中提取小的片段。之后用一个段结构从采样片段中收集信息。在这个情境下，时间段网络能够在整个视频上进行长期段结构的建模。而且这样的稀疏采样策略能在保留相关信息的条件下，仅付出极低的代价，使得长视频序列上端到端的学习在时间以及计算资源上具有合理的开支。

​	为了挖掘时间段网络的全部潜力，我们采用了最近引进的极深卷积神经网络结构，并且研究了大量克服上述训练数据限制问题的例子，包括(1)交叉模态，预训练；(2)正则化；(3)数据增强。同时，为了充分利用视频中的视觉内容，我们根据经验研究了四个双流卷积神经网络的输入方式，即 **single RGB image, stacked RGB difference, stacked optical flow field** 和 **stacked warped, optical flow field**。

​	我们在两个具有挑战性的行为识别数据集上进行了实验，即 **UCF101** 和 **HMDB51** ，以检验我们方法的效率。在实验中，使用时间段网络的模型学习在这两个数据集上取得了最先进的表现。我们同时将我们的双流学习模型可视化，以为未来行为识别的研究提供信息。

### 2  相关工作

​		在过去的几年里，行为识别已经被广泛地研究。与我们的工作有关的先前的工作分成了两类：(1)行为识别的卷积神经网络(2)时间结构建模

​	<b>行为识别卷积神经网络。</b>一些研究已经尝试设计高效的卷积神经网络结构以应对视频中的行为识别。Karpathy 等人在一个大数据集(**Sports-1M**)上测试了深层卷积神经网络。Simonyan等人通过开发ImageNet，设计了包含时空网的卷积神经网络，用于预训练以及计算精确捕捉运动信息需要的光流。Tran等人在现实且大规模的视频数据集中探索 **3D** 的卷积神经网络，并尝试通过 **3D** 卷积操作同时学习外观和涌动特征。Sun等人提出了因子化的卷积神经网络并且采用了不同的方式以分解 **3D** 卷积核。最近，一些工作专注于用卷积神经网络对长期时间结构进行建模。然而，这些方法直接在更长的连续视频流上操作。受限于计算代价，这些方法通常只能处理 64 到 120 帧的固定长度的序列。对于这些时间覆盖范围受限的方法来说，学习整个视频并不简单。我们的方法与这些端到端的卷积神经网络不同，它采用了与众不同的稀疏时间采样策略，使它能够高效地学习完整视频，不受序列长度的限制。

​	<b>时间结构建模。</b>许多研究工作已经在为行为识别的时间结构建模上做出贡献。Gaidon等人将每个视频中原子级的动作进行标注，并提出了 **Actom Sequence Model**(**ASM**) 进行行为识别。Niebles等人提出使用隐变量对复杂动作的时间分解进行建模，并被分类到 **Latent SVM** 中，以在迭代中学习模型参数。Wang 等人以及 Pirsiavash 等人分别利用 Latent Hierachical Model(LHM) 以及 Segmental Grammar Model(SGM)，将复杂动作的时间分解扩展到层次结构。Wang等人设计了一个序列化骨架模型 (SSM) 以捕获动态偏序集中的关系，并进行时空行为检测。Fernando 将 **BoVW** 的行为识别表示中的时间演变进行建模。这些方法仍然不能集合成一个端到端的学习框架，以对时间结构进行建模。我们提出的时间段网络强调了以上原则，同时是第一个为完整视频进行端到端时间结构建模的框架。

### 3  Temporal Segment Networks 行为识别

​	在这个部分，我们会给出使用时间段网络进行行为识别的详细信息。明确地说，我们将首先介绍时间段网络框架中的基本概念。接着我们会研究在使用时间段网络框架学习双流卷积神经网络的良好实践。最后我们会描述训练的双流卷积神经网络的测试细节。

#### 3.1  时间段网络

​	正如我们已经在第一部分讨论过，当前的双流卷积神经网络存在一个明显的问题，即它们无法对长期时间结构进行建模。这主要是因为他们在具有时间先后的上下文处理中的局限性，因为他们被设计用于操作单个帧(空间网络)，或者一小批样本中的帧堆栈(时间网络)。然而，由多个状态组成的复杂行为，诸如运动行为，涵盖了相当长的实践。如果不能在这些行为中将长期时间结构应用到卷积神经网络的训练，将会是一个不小的损失。为了解决这个问题，我们提出了时间段网络，一个视频级框架，以能够对整个视频的动态进行建模，正如图片1中的展示。

​	具体而言，我们提出的时间段网络框架，以充分利用完整视频中的视觉信息从而做出视频级预测为目的，同样是由空间流卷积神经网络以及事件流卷积神经网络组成。与在单个帧或帧堆栈上工作不同，时间段网络操作从完整视频中通过稀疏采样得来的短片段组成的序列。每一个序列中的片段会各自提供对行为类的初步预测。接着这些片段生成统一的结果导出为视频级预测。在学习过程中，不同于那些双流神经网络中片段级的预测, 视频级预测的损失函数通过迭代更新模型参数进行优化。

​	正式地，给定一个视频 **V** ，我们把它分为 **K** 个等长段 **{S1, S2, ..., Sk}** 。接着时间段网络对片段组成的序列进行建模，如下所示:

​		![formula 1](/images/tsm/F1.png)				

​	在这里 **(T1, T2,...,TK)** 代表片段组成的序列，每一个片段 **TK** 从相对应的段 **Sk** 中随机采样。**F(TK; W)** 代表卷积神经网络，其中参数 **W** 在短片段 **TK** 上进行操作并为所有类别产生类别分数。段共识函数 **G** 将多个短片段的输出进行结合，以获得它们中类别假设的共识。基于这个共识，预测函数 **H** 为整个视频预测每一个行为类的可能性。这里我们为 **H** 选择了广泛使用的 **Softmax** 函数。与标准的分类交叉熵损失结合，最终的关于段共识的损失函数 **G = G(F(T1), F(T2), ···, F(TK))** 变为：

​	![formula 2](/images/tsm/F2.png)

其中 **C** 代表行为类的种类，**yi** 代表关于类别 **i** 的标签。根据先前在时间建模的工作，在实验中片段 **K** 的值被设置为3。共识函数 **G** 的形式仍然是一个开放式的问题。在这次工作中我们使用了共识函数 **G** 最简单的形式，即 **Gi = g(F(T1),...,F(TK))** 。在这里一个类别分数 **Gi**  ，是使用聚合函数 **g** 由所有片段中同类别分数推导出。按照经验我们评估了几个不同形式的聚合函数 **g** ，在实验中包括**evenly**平均值, 最大值以及加权平均值。在他们当中，**evenly** 平均值用于展示我们最终的识别精确度。

​	这个时间段网络是可微分的，或者说至少具有次梯度，取决于 **g** 的选择。这使我们能够用多个片段，结合标准反向传播算法，共同优化模型参数 **W** 。在反向传播算法的处理过程中，关于损失值 **L** 的模型参数 **W** 的梯度可以被导出为

![formula 3](/images/tsm/F3.png)

  其中 **K** 代表时间段网络使用的段的数量。

​	当我们使用诸如随机梯度下降(**SGD**) 等基于梯度的优化算法学习模型参数的时候公式 **(3)** 保证了参数的更新是在利用从所有片段级预测得来的段共识 **G** 。以上述方式进行优化的时间段网络可以从完整视频中学习模型参数，而不是从短片段中学习。

![Fig.1](/images/tsm/Fig.1.png)

<b>图 1。</b>时间段网络：一个输入视频被分为 **K** 段，并在每个段中随机选出一个短片段。不同片段的类别分数由段共识函数融合而成，以产生段共识，这即是一个视频级的预测。从各个形态得来的预测接着被融合以的到最终的预测。所有片段上的卷积神经网络分享参数。

​	同时，通过为所有视频确定 **K** 值，我们综合出一个稀疏时间采样策略，其中采样片段只包含帧的一小部分。相比起先前使用使用密集帧采样的工作，它彻底地降低了在帧上进行评估地计算成本。

#### 3.2  训练时间段网络

​	时间段网络提供了一个视频级学习的坚实框架，但是为了达到最优的表现，我们需要考虑一些实际问题，比如有限的训练样本。为此，我们研究了一系列在视频数据上训练深度卷积神经网络的良好实践，它们同样也可以被直接运用于训练时间段网络。

​	<b>网络结构。</b> 网络结构在神经网络设计中是一个重要的因素。一些工作已经表明更深的结构能够改善物体识别的表现。然而，原始的双流卷积神经网络运用了相当浅的网络结构(**ClarifaiNet**) 。在这次工作中，我们选择 **BN-Inception** 作为架构，因为它在精确度与效率之间有着良好的平衡性。我们采用了原始的 **BN-Inception**结构设计双流卷积神经网络。正如在原始的双流卷积神经网络中，空间流卷积神经网络在单张 **RGB** 图片上进行操作，而时间流卷积神经网络将堆叠的连续光流域作为输入。

​	<b>网络输入。</b> 同时我们也对探索更多输入方式从而增强时间段网络的分辨力感兴趣。起初，双流卷积神经网络为空间流使用 **RGB** 图片，为时间流使用堆叠的光流域。在这里我们提出研究两个额外的方式，即 **RGB** 差与 **warped optical flow fields** 。

![Fig 2](/images/tsm/Fig.2.png)

<b>图 2。</b> 四个类型输入方式的样例： **RGB** 图片， **RGB** 差, **optical flow fields** (x，y 方向) 和 **warped optical flow fields** (**x，y** 方向)

​	单张 **RGB** 图片通常编码在特定时间点下的静态外观，并且缺乏关于前后帧的上下文信息。如图2所示，两个连续帧的 **RGB** 差描述了外观的变化，它可能符合运动显著的区域。受到启发，我们将堆叠的 **RGB** 差作为另一个输入进行实验，并研究其在行为识别中的表现。

​	时间流卷积神经网络将光流域作为输入以捕捉运动信息。然而在实际的视频中通常会存在相机移动的问题，使得光流域可能无法聚焦人类的行为。如图片2所示，由于相机移动，明显有大量背景中的水平运动被高亮。受到改进密集轨迹的启发，我们提出将扭曲光流域作为额外的输入方式。按照 [2] ，我们通过第一估计单应矩阵以及抵消相机移动，提取出了扭曲光流域。如图2所示，扭曲光流域抑制了背景的移动并使运动聚焦在演员身上。

​	<b>网络训练。</b> 由于行为识别的数据集都相当小，训练深度卷积神经网络面临着过拟合的风险。为了缓解这个问题，我们设计了几个在时间段网络中训练卷积神经网络的策略，如后文所示。

​	跨模态预训练  当目标数据集的训练样本不足时，预训练成为了初始化深度卷积神经网络的有效方式。当空间网络将 **RGB** 图像作为输入，它自然会利用在 **ImageNet** 上训练的模型作为初始化。对于其他的方式，诸如光流域以及 **RGB** 差值，他们在本质上捕捉视频数据中不同的视觉部分，且他们的分配与使用 **RGB** 图像不同。我们想出了跨模态预训练技术，我们在其中利用了 **RGB** 图像初始化时间网络。开始，我们通过线性变化将光流域离散化至0到255的间隔。这个步骤是光流域的范围与 **RGB** 图像的范围相同。接着我们调整了 **RGB** 模型中的第一个卷积层的权重，以处理光流域的输入。具体而言，我们将 **RGB** 通道的权重进行平均并按照时间网络输入的数量对这个平均值进行复制。这个初始化方法在时间网络上效果良好，在实验中降低了过拟合的影响。

​	正则化技术  批量规范化是处理 **covariate,,shift** 问题的重要部分。在学习过程中，批量规范化将会估计每个批次的激活平均值与方差，并且用他们将激活值转换为标准高斯分布。这个操作将会加速训练集合的速度，但同时也会在转换过程中带来过拟合，由于来自有限样本的估计偏差导致。引起，在用预训练的模型进行初始化之后，我们选择冻结除了第一层以外，BN层的平均与方差参数。因为光流的分布与RGB图像不同，第一个卷积层的激活值将会有不同的分布所以我们需要根据情况重新调整平均值与方差。我们把这个策略称之为<b> partial BN</b> 。同时，我们在**BN-Inception**结构中的全局池化层后额外添加了一个 <b>dropout</b> 层，以为了降低过拟合带来的影响。**dropout** 的比例，对于空间流卷积神经网络被设置为0.8，对于时间流卷积神经网络设置为0.7。

​	数据增强  数据增强能生成多样的训练样本并且预防严重的过拟合，在原始的双流卷积神经网络中，随机的裁剪以及水平翻转被用于增强训练样本。我们应用了两个新的数据增强技术：边角裁剪与缩放抖动。在拐角裁剪技术中，只有边角或图像的中心会成为被提取的区域，以避免隐式关注一张图像的中心部分。在多缩放裁剪技术中，我们应用在**ImageNet**分类中使用的缩放抖动技术进行行为识别。我们展示了在缩放抖动上的高效实现。我们将图像输入或者光流域的大小设置为 **256 x 340** ，且裁剪区域的宽与长从 {**256**, **224**, **192**, **168**} 中挑选。最终这些被裁剪的区域将被重新设置为 **224 x 224** 以进行网络训练。事实上，这个实现不仅包含缩放抖动，它同样包括长宽比抖动。

####  3.3  测试时间段网络

​	最后，我们将展示时间段网络的测试方法。由于片段级卷积神经网络共享时间段网络的模型参数，训练的模型可以像普通卷积神经网络一样实施逐帧评估。这让我们能够公平地与不使用时间段网络框架的模型进行对比。具体而言，我们使用原始双流卷积神经网络的测试方案，在其中我们从行为视频中采样25张 **RGB** 帧或者光流叠片。同时我们裁剪了四个边角与一个中心，以及它们相对于采集帧的水平翻转以评估卷积神经网络。对于空间与时间流结合的网络，我们采用了加权平均值。当使用时间段网络框架进行学习的时候，空间流卷积神经网络与时间流卷积神经网络之技安的表现差距比起在原始双流卷积神经网络中的小得多。基于这个事实，将空间流的权重设置为 **1** ，时间流权重设置为 **1.5** ，能使其更加可靠。当普通以及扭曲光流域被同事使用时，时间流中光流的权重被划分为 **1** ，扭曲光流域被划分为 **0.5** 。在3.1节中描述过段共识函数被应用在 **Softmax** 归一化之前。为了测试模型的训练效果，我们将 **25** 帧的预测分数与在 **Softmax** 归一化之前的不同流进行融合。

### 4  实验

​	在这个部分，我们首先介绍评估数据集以及我们方法实现的细节。接着我们将会探索提出的，学习时间段网络的良好实践。在这之后我们将强调通过添加时间段网络框架进行长期时间结构建模的重要性。我们也将将我们方法的表现与当下最好的方法进行对比。最终，我们将学习的卷积神经网络模型可视化。

#### 4.1  数据集与实现细节

​	我们在两个大数据集上进行试验，即 **HMDB51** 与 **UCF101**。**UCF101** 数据集包含了**101** 个行为类别以及 **13, 320** 个视频片段。我们使用 **THUMOS13** 挑战的评估方案并采用三个训练或测试的分割以进行评估。 **HMD51** 数据集是一个多来源的真实视频集合，包括电影以及网页视频。该数据集包含了 **6, 766** 个视频，51个行为种类。我们的实验采用原始的评估方案，使用三个训练或测试分割，并且在这些分割上展示平均准确率。

​	我们使用小批量随机梯度下降算法以学习模型参数，其中批量大小被设置为 **256** ，栋梁被设置为 **0.9** 。我们用**ImageNet** 的预训练模型初始化网络权重。在实验中，我们设置了更小的学习率。对于空间网络，学习率被初始化为 **0.001** 并且每 **2, 000** 次迭代降低为原来的 **1/10** 。整个训练过程在 **4, 500** 次迭代停下。对于时间网络，我们将学习率设置为0.005，并且在 **12, 000** 次以及 **18, 000** 次迭代后降低为原来的 **1/10** 。最大迭代次数被设置为 **20, 000** 。关于数据增强，我们使用了位置抖动，水平翻转，边角裁剪以及缩放抖动等技术，这在 **3.2** 节中明确指出。对于提取光流域以及卷曲光流域，我们选择了 **OpenCV** 中使用 **CUDA** 的 **TVL1** 光流算法。为了加速训练过程，我们使用多个 **GPU**，采用了我们调整过的 **Caffe** 和 **OpenMPI** 实现的平行数据的策略。在 **UCF101** 数据集上，空间 **TSNs** 使用 **4** 个 **TITANX** **GPU** 训练的时间大约为 **2** 小时，时间 **TSNs** 为 **9** 小时。

#### 4.2 探索研究

​	在这个部分，我们着重与在3.2节中描述的良好时间的调查，包括训练策略以及输入方式。在探索研究中，我们使用[23]中的深度结构双流神经网络并在 **UCF101** 数据集的分割上进行所有实验。

​	我们在3.2节提出了两个训练策略，即跨模态预训练以及带 **dropout** 的部分**BN** 。具体而言，我们比较四个设置：(1)从头开始训练，(2)如[1]中仅仅预训练的空间流，(3)使用跨模态与训练，(4)将跨模态与训练与带 **dropout** 的部分 **BN** 结合。结果被总结在表1。首先，我们可以看见从头训练的表现要比原始双流卷积神经网络(**baseline**) 更差，这意味着精神的设计学习策略以预防过拟合是有必要的，特别是空间网络。接着，我们凭借预训练的时间流以及跨模态预训练的时间流初始化双流神经网络，它比基准线取得了更好的表现。进一步地，我们利用带 **dropout** 的部分 **BN** 正则化训练过程，这将识别效果增强到 **92.0%** 。 

<b>表 1</b> 在 数据集 **UCF101** 上探索不同的双流卷积神经网络训练策略。

![](/images/tsm/Table-1.png)

​	我们在 **3.2** 节提出了两个新类型的方式：**RGB** 差与卷曲光流域。在表2对比了不同方式的表现。这些实验都是在表1展示的良好实践的基础上进行的。我们首先观察到 **RGB** 图像与 **RGB** 差的结合使识别表现增强到 **87.3%** 。这个结果表明**RGB** 图像与 **RGB** 差可能会编码出互补的信息。接着我们可以发现，光流与扭曲光流产生的结果相似 **(87.2%,,vs.,,86.9%)**，并且他们它们的结合会将表现改善到 **87.8%** 。将这四个方式结合起来将会得到 **91.7%** 的精确度。由于**RGB** 差可能显得有些相似却能理解运动模式，我们同样评估了将其他三个方式结合的表现，这带来了更好的识别准确度 **(92.3%,,vs,,91.7%)**。我们猜想光流更适合捕捉运动信息，且有时 **RGB** 差不能稳定的描述运动。

<b>表 2</b> 在数据集 **UCF101** 上探索不同的双流卷积神经网络输入方式。

![](/images/tsm/Table-2.png)

<b>表 3</b> 在数据集 **UCF101** 上探索不同时间段网络的段共识函数。

![](/images/tsm/Table-3.png)

另一方面，**RGB** 差能为运动表示提供一个低质量，高速度的选择。

#### 4.3  评估时间段网络

​	在这个部分，我们重点关注于时间段网络框架的研究。我们首先研究段共识函数的影响，接着在**UCF101** 数据集上讲不通的卷积神经网络结构进行对比。为了公平进行对比，我们只使用了 **RGB** 图片以及光流域作为输入方式进行探索。在 **3.1** 节提到过，段落 **K** 的值被设为 **3** 。

​	在公式(1)中，段共识函数用它的聚合函数 **g** 进行定义。在这里我们对于2 **g** 的形式评估了三个候选：(1)最大池化，(2)平均池化，(3)带权平均值。这些实验性的结果在表 **3** 中被总结。我们可以看到平均池化函数达到了最好的效果。所以在接下来的实验中，我们选择了平均池化作为标准的聚合函数。接着我们将不同网络结构表现进行对比，结果被总结在表 **4** 。具体而言，我们比较了三个极深结构：**BN-Inception**，**GoogLeNet**，**VGGNet16** ，所有这些结构通过前面提到的良好实践进行训练。在这些比较的结构中，采用 **BN-Inception** 的极深双流卷积神经网络达到了最好的 **92.0%** 精确度。

<b>表 4</b> 在数据集 **UCF101** 上探索不同的极深卷积神经网络结构。"**BN-Inception+TSN**" 代表时间段网络框架被应用在表现最好的 **BN-Inception** 上的设置。 

![](/images/tsm/Table-4.png)

<b>表 5</b> 在数据集 **UCF101** 上对提出方法进行分量分析。从左到右我们一个一个地添加分量。**BN-Inception** 被用作卷积神经网络结构。

![Table 5](/images/tsm/Table-5.png)

这和它在图像分类任务中更好的表现一致。所以我们选择将 **BN-Inception** 作为时间段网络的卷积神经网络架构。

​	凭借所有的设计选择集，我们将时间段网络(**TSN**) 运用在行为识别上。结果在表4中描绘。就行为识别准确略而言的组建的份量分析在表5中展示。我们可以发现，即使当所有讨论到的良好实践已经被应用，时间段网络仍然能够提升模型的表现。这证明对长期时间段结构件我对于理解视频中的行为至关重要。并且它已经被时间段网络实现。

#### 4.4  与最先进的技术比较

​	在探索完良好实践并且理解了时间段网络的影响后，我们准备好建立我们最后的行为识别方法。具体而言，我们组合了三个输入方法以及所有描述过的技术作为我们最终的识别方法，并且在两个挑战性的数据集  **HMDB51** 与 **UCF101** 上进行测试。结果总结于表6，其中我们将我们的方法与 **improved trajectories(iDTs), MoFAP**表示的传统方法以及 **3D** 卷积网络(**C3D**), **trajectory**-**pooled** **deep**-**convolutional** **descriptions**(**TDD**)，**factorized** **spatio**-**temporal** **convolutional** **networks** (**FST CN**), 长期卷积网络(**LTC**), 

<b>表 6</b> 基于时间段网络将我们的方法与其他先进的方法进行比较。我们分别将使用两种输入(**RGB+Flow**)与三种输入(**RGB + Flow + Warped Flow**)的结果进行展示。

![Table 6](/images/tsm/Table-6.png)

**key,volume,mining,framework(KVMF)** 等深度学习表示进行比较。我们最好的结果在 **HMD51** 上比其他方法优 **3.9%** ，在 **UCF101** 上优 **1.1%** 。这样优越的表现证明了时间段网络的高效性，并且说明了长期时间模型建模的重要性。

#### 4.5  模型可视化

​	除了识别精确度以外，我们希望进一步地了解训练的卷积神经网络模型。在此情境下，我们采用了 **DeepDraw** 工具盒。这个工具能只用白噪声在输入图像上进行升序梯度迭代。因此在大量迭代之后的输出能被看作仅基于卷积神经网络模型内部类别知识的类别的可视化。这个工具原始的版本只能处理 **RGB** 数据。为了在光流模型上进行可视化，我们在时间卷积网络上采用了这个工具。结果是，我们第一次将行为识别中有趣的类别信息进行了可视化。我们随机地从数据集 **UCF101** 中挑选了五个类别，太极，拳击，跳水，跳远 以及 骑行 进行可视化。图3展示了结果，对于 **RGB** 以及光流，我们可视化了学习以下三个设置地卷积神经网络模型：(1)没有预训练；(2)仅进行预训练；(3)使用时间段网络。

​	通常而言，进行了预训练的模型比那些没有预训练的模型更能表示视觉概念。可以看出没有进行预训练的空间

![Fig.3](/images/tsm/Fig.3.png)

<b>图 3</b> 使用 **DeepDraw** 对行为识别的卷积神经网络进行可视化。我们将以下配置进行对比：(1)没有进行预训练；(2)进行了预训练；(3)使用时间段网络；对于空间卷积神经网络，我们将三个生成的可视化进行彩色图像的展示。对于时间卷积神经网络，我们将 **x** 和 **y** 方向的流图用灰阶图像展示。需要注意的是这些图像仅生成与随机的像素。

和时间模型几乎不能生成有任何意义的视觉结构。凭借从预训练过程前以来的知识，空间和时间模型能够捕捉结构化的视觉模式。

同时很容易发现，仅仅只用短期信息进行训练的模型，例如单一帧，更容易将视频中的场景模式和物体当作行为识别的关键依据。例如，在“跳水”类中，单帧空间流卷积神经网络主要寻找水和跳水平台，而不是进行跳水的人。它对应的在光流上运作的时间流，倾向于关注水面波纹的运动。使用时间段网络引进的长期时间建模，显而易见，训练模型更能关注视频中的人类，并且看起来是在对行为类进行长期结构建模。再将"跳水"作为例子，空间卷积神经网络以及时间段网络生成了以人类为主要视觉信息的图像。并且在图像中不同的姿势能被辨别出来，它描绘了了一个跳水动作的状态。这提示我们使用提出方法进行学习的模型可能会表现得更好，这在我们的定量实验中很好地被反映了。我们推荐读者阅读单独更多关于行为类可视化以及可视化过程的补充材料。

### 5  总结

​	在这篇论文中，我们展示了时间段网络(**TSN**)，一个进行长期时间结构建模的视频级的框架。正如在两个富有挑战性的数据集上证明的那样，这个工作已经把最先进的技术带到了新高度，同时维持了合理的计算代价。这主要归功于使用系数时间采样的段架构，以及一系列在这次工作中的良好实践。前者提供了高效的方式以捕获长期时间结构，而后者使在有限数据集下训练极深网络而不造成严重的过拟合成为可能。

<b>致谢。</b>  <font face ="times new roman" size="4">This  work  was  supported  by  the  **Big  Data  Collaboration**  **Research**  grant  from  SenseTime  Group (CUHK  Agreement  No. TS1610626),  Early  Career  Scheme  (ECS)  grant  (No. 24204215),  ERC  Advanced Grant  **VarCity**  (No.273940),  Guangdong  Innovative  Research  Program  (2015B010129013, 2014B050505017),  and  Shenzhen  Research  Program  (KQCX2015033117354153, JSGG20150925164740726,  CXZZ20150930104115529),  and  External  Cooperation  Program  of  BIC, Chinese  Academy  of  Sciences  (172644KYSB20150019).</font>

### 参考文献

1. Simonyan, K., Zisserman, A.: Two-stream convolutional networks for action recog

nition in videos. In: NIPS, pp. 568–576 (2014)

2. Wang, H., Schmid, C.: Action recognition with improved trajectories. In: ICCV,

pp. 3551–3558 (2013)

3. Wang, L., Qiao, Y., Tang, X.: Motionlets: mid-level  **3D**  parts for human motion

recognition. In: CVPR, pp. 2674–2681 (2013)

4. Ng, J.Y.H., Hausknecht, M., Vijayanarasimhan, S., Vinyals, O., Monga, R.,

Toderici, G.: Beyond short snippets: deep networks for video classifification. In:

CVPR, pp. 4694–4702 (2015)

5. Wang, L., Qiao, Y., Tang, X.: Action recognition with trajectory-pooled deep

convolutional descriptors. In: CVPR, pp. 4305–4314 (2015)

6. Gan, C., Wang, N., Yang, Y., Yeung, D.Y., Hauptmann, A.G.: Devnet: a deep

event network for multimedia event detection and evidence recounting. In: CVPR,

pp. 2568–2577 (2015)

7. LeCun, Y., Bottou, L., Bengio, Y., Haffffner, P.: Gradient-based learning applied to

document recognition. Proc. IEEE **86**(11), 2278–2324 (1998)

8. Krizhevsky, A., Sutskever, I., Hinton, G.E.: ImageNet classifification with deep con

volutional neural networks. In: NIPS, pp. 1106–1114 (2012)

9. Simonyan, K., Zisserman, A.: Very deep convolutional networks for large-scale

image recognition. In: ICLR, pp. 1–14 (2015)

10. Szegedy, C., Liu, W., Jia, Y., Sermanet, P., Reed, S., Anguelov, D., Erhan, D.,

Vanhoucke, V., Rabinovich, A.: Going deeper with convolutions. In: CVPR, pp.

1–9 (2015)

11. Xiong, Y., Zhu, K., Lin, D., Tang, X.: Recognize complex events from static images

by fusing deep channels. In: CVPR, pp. 1600–1609 (2015)

12. Karpathy, A., Toderici, G., Shetty, S., Leung, T., Sukthankar, R., Fei-Fei, L.: Large

scale video classifification with convolutional neural networks. In: CVPR, pp. 1725–

1732 (2014)

13. Tran, D., Bourdev, L.D., Fergus, R., Torresani, L., Paluri, M.: Learning spatiotem

poral features with  **3D**  convolutional networks. In: ICCV, pp. 4489–4497 (2015)

14. Zhang, B., Wang, L., Wang, Z., Qiao, Y., Wang, H.: Real-time action recognition

with enhanced motion vector CNNs. In: CVPR, pp. 2718–2726 (2016)

15. Niebles, J.C., Chen, C.-W., Fei-Fei, L.: Modeling temporal structure of decom

posable motion segments for activity classifification. In: Daniilidis, K., Maragos, P.,

Paragios, N. (eds.) ECCV 2010, Part II. LNCS, vol. 6312, pp. 392–405. Springer,

Heidelberg (2010)

16. Gaidon, A., Harchaoui, Z., Schmid, C.: Temporal localization of actions with

actoms. IEEE Trans. Pattern Anal. Mach. Intell. **35**(11), 2782–2795 (2013)

17. Wang, L., Qiao, Y., Tang, X.: Latent hierarchical model of temporal structure for

complex activity classifification. IEEE Trans. Image Process. **23**(2), 810–822 (2014)

18. Fernando, B., Gavves, E., O., MJ, Ghodrati, A., Tuytelaars, T.: Modeling video

evolution for action recognition. In: CVPR, pp. 5378–5387 (2015)

19. Varol, G., Laptev, I., Schmid, C.: Long-term temporal convolutions for action

recognition. CoRR abs/1604.04494 (2016)

20. Donahue, J., Anne Hendricks, L., Guadarrama, S., Rohrbach, M., Venugopalan,

S., Saenko, K., Darrell, T.: Long-term recurrent convolutional networks for visual

recognition and description. In: CVPR, pp. 2625–2634 (2015)

21. Soomro, K., Zamir, A.R., Shah, M.: UCF101: a dataset of 101 human actions

classes from videos in the wild. CoRR abs/1212.0402 (2012)

22. Kuehne, H., Jhuang, H., Garrote, E., Poggio, T.A., Serre, T.: HMDB: a large video

database for human motion recognition. In: ICCV, pp. 2556–2563 (2011)

23. Ioffffe, S., Szegedy, C.: Batch normalization: accelerating deep network training by

reducing internal covariate shift. In: ICML, pp. 448–456 (2015)

24. Gan, C., Yao, T., Yang, K., Yang, Y., Mei, T.: You lead, we exceed: labor-free

video concept learning by jointly exploiting web videos and images. In: CVPR, pp.

923–932 (2016)

25. Peng, X., Wang, L., Wang, X., Qiao, Y.: Bag of visual words and fusion methods for

action recognition: comprehensive study and good practice. Comput. Vis. Image

Underst. **150**, 109–125 (2016)

26. Gan, C., Yang, Y., Zhu, L., Zhao, D., Zhuang, Y.: Recognizing an action using its

name: a knowledge-based approach. Int. J. Comput. Vis. **120**(1), 61–77 (2016)

27. Ji, S., Xu, W., Yang, M., Yu, K.:  **3D**  convolutional neural networks for human

action recognition. IEEE Trans. Pattern Anal. Mach. Intell. **35**(1), 221–231 (2013)

28. Sun, L., Jia, K., Yeung, D., Shi, B.E.: Human action recognition using factorized

spatio-temporal convolutional networks. In: ICCV, pp. 4597–4605 (2015)

29. Pirsiavash, H., Ramanan, D.: Parsing videos of actions with segmental grammars.

In: CVPR, pp. 612–619 (2014)

30. Wang, L., Qiao, Y., Tang, X.: Video action detection with relational dynamic

poselets. In: Fleet, D., Pajdla, T., Schiele, B., Tuytelaars, T. (eds.) ECCV 2014,

Part V. LNCS, vol. 8693, pp. 565–580. Springer, Heidelberg (2014)

31. Felzenszwalb, P.F., Girshick, R.B., McAllester, D.A., Ramanan, D.: Object detec

tion with discriminatively trained part-based models. IEEE Trans. Pattern Anal.

Mach. Intell. **32**(9), 1627–1645 (2010)

32. Zeiler, M.D., Fergus, R.: Visualizing and understanding convolutional networks.

In: Fleet, D., Pajdla, T., Schiele, B., Tuytelaars, T. (eds.) ECCV 2014, Part I.

LNCS, vol. 8689, pp. 818–833. Springer, Heidelberg (2014)

33. Deng, J., Dong, W., Socher, R., Li, L., Li, K., Li, F.: ImageNet: a large-scale

hierarchical image database. In: CVPR, pp. 248–255 (2009)

34. Jiang, Y.G., Liu, J., Roshan Zamir, A., Laptev, I., Piccardi, M., Shah, M.,

Sukthankar, R.: THUMOS challenge: action recognition with a large number of

classes (2013)

35. Zach, C., Pock, T., Bischof, H.: A duality based approach for realtime TV-**L**1 opti

cal flflow. In: Hamprecht, F.A., Schn¨orr, C., J¨ahne, B. (eds.) DAGM 2007. LNCS,

vol. 4713, pp. 214–223. Springer, Heidelberg (2007)

36. Jia, Y., Shelhamer, E., Donahue, J., Karayev, S., Long, J., Girshick, R.B.,

Guadarrama, S., Darrell, T.: Caffffe: convolutional architecture for fast feature

embedding. CoRR abs/1408.5093

37. Cai, Z., Wang, L., Peng, X., Qiao, Y.: Multi-view super vector for action recogni

tion. In: CVPR, pp. 596–603 (2014)

38. Wang, H., Schmid, C.: LEAR-INRIA submission for the thumos workshop. In:

ICCV Workshop on THUMOS Challenge, pp. 1–3 (2013)

39. Wang, L., Qiao, Y., Tang, X.: MoFAP: a multi-level representation for action recog

nition. Int. J. Comput. Vis. **119**(3), 254–271 (2016)

40. Ni, B., Moulin, P., Yang, X., Yan, S.: Motion part regularization: improving action

recognition via trajectory group selection. In: CVPR, pp. 3698–3706 (2015)

41. Zhu, W., Hu, J., Sun, G., Cao, X., Qiao, Y.: A key volume mining deep framework

for action recognition. In: CVPR, pp. 1991–1999 (2016)

42. GitHub: Deep draw. https://github.com/auduno/deepdraw
