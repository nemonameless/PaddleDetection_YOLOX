# 行为识别任务二次开发

在产业落地过程中应用行为识别算法，不可避免地会出现希望自定义类型的行为识别的需求，或是对已有行为识别模型的优化，以提升在特定场景下模型的效果。鉴于行为的多样性，PP-Human支持抽烟、打电话、摔倒、打架、人员闯入五种异常行为识别，并根据行为的不同，集成了基于视频分类、基于检测、基于图像分类以及基于骨骼点的四种行为识别技术方案，可覆盖90%+动作类型的识别，满足各类开发需求。我们在本文档通过案例来介绍如何根据期望识别的行为来进行行为识别方案的选择，以及使用PaddleDetection进行行为识别算法二次开发工作，包括：方案选择、数据准备、模型优化思路和新增行为的开发流程。


## 方案选择

在PaddleDetection的PP-Human中，我们为行为识别提供了多种方案：基于视频分类、基于图像分类、基于检测、以及基于骨骼点的行为识别方案，以期望满足不同场景、不同目标行为的需求。对于二次开发，首先我们需要确定要采用何种方案来实现行为识别的需求，其核心是要通过对场景和具体行为的分析、并考虑数据采集成本等因素，综合选择一个合适的识别方案。我们在这里简要列举了当前PaddleDetection中所支持的方案的优劣势和适用场景，供大家参考。

| 技术方案 | 方案说明 | 方案优势 | 方案劣势 | 适用场景 |
| :--: | :--: | :--: | :--: | :--: |
| 基于人体骨骼点的行为识别 | 1. 通过目标检测技术识别出图像中的人；<br> 2. 针对每个人，基于关键点检测技术识别出关键点；<br>3. 基于关键点序列变化识别出具体行为。 | 1. 可识别出每个人的行为；<br>2. 聚焦动作本身，鲁棒性和泛化性好； | 1. 对关键点检测依赖较强，人员较密集或存在遮挡等情况效果不佳；<br>2. 无法准确识别多人交互动作；<br>3. 难以处理需要外观及场景信息的动作；<br>4. 数据收集和标注困难； | 适用于根据人体结构关键点能够区分的行为，背景简单，人数不多场景，如健身场景。 |
| 基于人体id的分类 | 1. 通过目标检测技术得到图像中的人；<br>2. 针对每个人通过图像分类技术得到具体的行为类别。 | 1.通过检测技术可以为分类剔除无关背景的干扰，提升最终识别精度；<br>2. 方案简单，易于训练；<br>3. 数据采集容易；<br>4. 可结合跳帧及结果复用逻辑，速度快； | 1. 缺少时序信息；<br>2. 精度不高； | 对时序信息要求不强的动作，且动作既可通过人也可通过人+物的方式判断，如打电话。 |
| 基于人体id的检测 | 1. 通过目标检测技术得到画面中的人；<br>2. 根据检测结果将人物从原图中抠出，再在扣得的图像中再次用目标检测技术检测与行为强相关的目标。 | 1. 方案简单，易于训练；<br> 2. 可解释性强；<br> 3. 数据采集容易；<br> 4. 可结合跳帧及结果复用逻辑，速度快； | 1. 缺少时序信息；<br>2. 分辨率较低情况下效果不佳；<br> 3. 密集场景容易发生动作误匹配 | 行为与某特定目标强相关的场景，且目标较小，需要两级检测才能准确定位，如吸烟。 |
| 基于视频分类的行为识别 | 应用视频分类技术对整个视频场景进行分类。 | 1.充分利用背景上下文和时序信息；<br>2. 可利用语音、字幕等多模态信息；<br>3. 不依赖检测及跟踪模型；<br>4. 可处理多人共同组成的动作； | 1. 无法定位到具体某个人的行为；<br>2. 场景泛化能力较弱；<br>3.真实数据采集困难； | 无需具体到人的场景的判定，即判断是否存在某种特定行为，多人或对背景依赖较强的动作，如监控画面中打架识别等场景。 |


下面以PaddleDetection目前已经支持的几个具体动作为例，介绍每个动作方案的选型依据：

### 吸烟

方案选择：基于人体id的检测

原因：吸烟动作中具有香烟这个明显特征目标，因此我们可以认为当在某个人物的对应图像中检测到香烟时，该人物即在吸烟动作中。相比于基于视频或基于骨骼点的识别方案，训练检测模型需要采集的是图片级别而非视频级别的数据，可以明显减轻数据收集与标注的难度。此外，目标检测任务具有丰富的预训练模型资源，整体模型的效果会更有保障，

### 打电话

方案选择：基于人体id的分类

原因：打电话动作中虽然有手机这个特征目标，但为了区分看手机等动作，以及考虑到在安防场景下打电话动作中会出现较多对手机的遮挡（如手对手机的遮挡、人头对手机的遮挡等等），不利于检测模型正确检测到目标。同时打电话通常持续的时间较长，且人物本身的动作不会发生太大变化，因此可以因此采用帧级别图像分类的策略。
    此外，打电话这个动作主要可以通过上半身判别，可以采用半身图片，去除冗余信息以降低模型训练的难度。

### 摔倒

方案选择：基于人体骨骼点的行为识别

原因：摔倒是一个明显的时序行为的动作，可由一个人物本身进行区分，具有场景无关的特性。由于PP-Human的场景定位偏向安防监控场景，背景变化较为复杂，且部署上需要考虑到实时性，因此采用了基于骨骼点的行为识别方案，以获得更好的泛化性及运行速度。

### 打架

方案选择：基于视频分类的行为识别

原因：与上面的动作不同，打架是一个典型的多人组成的行为。因此不再通过检测与跟踪模型来提取行人及其ID，而对整体视频片段进行处理。此外，打架场景下各个目标间的互相遮挡极为严重，关键点识别的准确性不高，采用基于骨骼点的方案难以保证精度。


下面详细展开四大类方案的数据准备、模型优化和新增行为识别方法

1. [基于人体id的检测](./idbased_det.md)
2. [基于人体id的分类](./idbased_clas.md)
3. [基于人体骨骼点的行为识别](./skeletonbased_rec.md)
4. [基于视频分类的行为识别](./videobased_rec.md)
