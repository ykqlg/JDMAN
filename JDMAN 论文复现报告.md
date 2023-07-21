# JDMAN 论文复现报告

## 数据集

网上找不到RAF-DB 2.0数据，通过松岭师兄要到了英建师兄的微信，他将使用的数据集全部发给了我，且**建议我使用vgg网络以获得更稳定的结果**

<img src="https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211955292.png" alt="image-20230721195534262" style="zoom:67%;" />

<img src="https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307212007231.png" alt="image-20230721200718200" style="zoom: 80%;" />

## 训练情况

- 资源：1卡GPU，A30，20GB（from学校高性能服务中心）
- 和论文实验同样的配置，训练vgg网络：用时8h
- epoch减半，训练resnet网络：用时4h

## 代码问题

- 项目给出的 数据集示例 和 实际数据集 格式不一致。更改了dataset.py的代码。
  ![image.png](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211927807.png)


- ResNet50和VGG16的output的维度不一致，但是代码层面没有设置相关的命令行参数入口，只能从model入参那边改数字（resnet是3072,vgg是4096）
  ![image.png](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211927884.png)

- vgg架构没有使用adaptation layer，跟resnet不一致
- test_only阶段，没有对应的导入模型参数的代码
  ![image.png](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211927855.png)
- 当仅测试阶段(test_only)的时候，还是跑了三次实验，但是test_only阶段只跑一次就够了。可优化。


## 困惑

- 论文3.4.1（SML with common centers）提出的损失函数，在代码()中好像没有使用上
- 在代码中，图中这部分的求法是使用Entropy(softmax)来近似P(h,d)？这个逻辑并不理解
  ![image.png](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211927831.png)

> 以上疑惑均已发送给英建师兄，由于师兄出差，下周返深才能答复，故同样以报告形式整理于此

## 实验结果分析

### 论文实验结果

![image.png](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211927934.png)

### 本次实验结果

| Method                               | JAFFE  | MMI    | CK+    | FER2013 |
| ------------------------------------ | ------ | ------ | ------ | ------- |
| Original JDMAN (resnet with adlayer) | 68.54  | 71.88  | 88.51  | 58.63   |
| my vgg (without adlayer)             | 52.582 | 67.045 | 88.188 | 52.076  |
| my_resnet (half epoch)               | 46.009 | 63.636 | 89.887 | 55.017  |

### 原因分析

1. 源码中，CK+ 和 JAFFE 在训练和测试的时候是不同的，但由于英建师兄发来的数据集中并未对 CK+ 和 JAFFE 有更具体的分类，所以这里我都用了同一个数据集。可能训出来的模型会因训练数据的区别而有所差异。
	![image-20230721194934232](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211949280.png)

	![image-20230721195227596](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211952633.png)

2. 可以看到，vgg在近乎和原文一样的配置下，只在CK+（目标域）作为测试集的情况下取得近似的结果，而在其他数据集取得的结果甚至不如Baseline。也许是adlayer没有使用上？
3. 由于训练成本较高，resnet只用了一半的epoch（30）训，出来的效果只是在CK+（目标域）作为测试集的情况下取得近似甚至超出的结果（有点奇怪，该不会是[分析1]里提到的原因，导致某种程度的过拟合？），其他在jaffe上的效果尤其差，按照论文4.3.2的解释，模型应该学到了constrained和unconstrained数据集之间的domain shift。这个性能有点反常。

:star:<u>因为师兄一开始推荐我用vgg，所以我vgg是按原文配置训练的。24小时GPU使用时间即将到期，等我下一次申请到时，我会按照相同配置训一个resnet看看问题出在哪。</u>



### （部分实验过程截图）

- vgg, test on MMI

![image-20230721193144479](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211931542.png)

- vgg, test on CK+

![image-20230721193616774](https://siyuandemo.oss-cn-shenzhen.aliyuncs.com/img/202307211936816.png)



## 总结

- 排队申请GPU资源（15h）等待比较长时间。
- 在自己windows系统上试跑，出现了跟num_workers相关的多线程问题，排查后发现，是因为win和linux创建子进程的方式有所不同。所以这只是win上特有的错误。
- resnet和vgg切换时，模型的维度接口需要手动改数字。因为这个问题，几乎把整个项目的代码彻底理解了一遍。
- 代码中很多变量的命名跟论文不一致，有些缩写也没有一些注释辅助理解，只能硬看，较为吃力
