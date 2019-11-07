---
layout: post
#标题配置
title:   OpenCV-6-ANN_MLP神经网络模型训练
#时间配置
date:   2019-11-07 10:00:00 +0800
#大类配置
categories: 图像识别
#小类配置
tag: OpenCV
---

* content
{:toc}



---

> 使用语言：Java 1.8
> 操作系统：windows x64
> OpenCV：4.1.1

---

### ANN_MLP神经网络理论介绍

#### 神经网络介绍

神经网络是当前机器学习领域普遍所应用的，例如可利用神经网络进行图像识别、语音识别等，从而将其拓展应用于自动驾驶汽车。

目前在机器学习领域火爆的开源软件库中，**TensorFlow**算其中一个，里面有大量的机器学习和深度神经网络方面的研究，毕竟由Google 大脑小组开发出来。

神经网络的变种目前有很多，如**误差反向传播（Back Propagation，BP）神经网路**、**概率神经网络**、**卷积神经网络**（Convolutional Neural Network ，CNN-适用于图像识别，GOOGLetNet用的就是这个）、**时间递归神经网络**（Long short-term Memory Network ，LSTM-适用于语音识别）等。但**最简单的神经网络则是多层感知器（Muti-Layer Perception** **，MLP）**

#### MLP多层感知器神经网络

MLP是基于生物神经元的模拟和简化，得到多层感知器MLP的基本机构，就是**输入层、隐层和输出层**，**MLP神经网络不同层之间是全连接的**（全连接的意思就是：上一层的任何一个神经元与下一层的所有神经元都有连接）。

**大概就是这个样子的：**

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMyLnpoaW1nLmNvbS92Mi0yMTEwYTRkNjIzODRhMjc3YWI3MDA5MDdlNzNlODcyMV9iLndlYnA?x-oss-process=image/format,png)

> 这里的箭头啥的，表示对应的映射。然后这些映射的抉择用的是：**权重、偏置和激活函数** （具体算法怎么搞定的，我不会算法，大学没有认真学数学啊 ε=(´ο｀*)))唉）

MLP的最经典例子就是数字识别，即我们随便给出一张上面写有数字的图片并作为输入，由它最终给出图片上的数字到底是几。

![img](<https://pic2.zhimg.com/v2-ee03400324c26d1227f9d418490ae241_b.webp>)

![img](https://pic3.zhimg.com/v2-2f571aa0e9edf1edebafbc1e7388836e_b.webp)



> 视频请观看：B站的3Brown1Blue， B站视频号：av15532370 ，<https://www.bilibili.com/video/av15532370/>

**理论小结：**

MLP的处理方式就是，输入图片的数据，然后在中间好几次的隐藏层中依据训练好的权重偏移等参数，最终映射到结果。

这里的输入图像为图像的N xN需要被转换为1x（N x N）的数据入参，输出则为对应数字（为训练集名称下标）

> 在接下来，用代码实现的时候，可以很好的理解这个训练的出入参数。至于中间的算法过程，那是大神们考虑的╮(╯_╰)╭

### 理论参考

* <https://zhuanlan.zhihu.com/p/63184325> 【这篇写的很好，推荐】

---

### 训练一个ANN_MLP模型

由于玩的都是人脸识别，所以也想试一试自己训练个识别来预测特定的人。

虽然使用MLP在数据量不多的情况下，训练个数字就差不多了。不过代码是一样的，训练集大家可以瞎搞的。

> 代码上，有一个严重的问题没有考虑到，就是样本不规则的情况，这里使用的每个种类样本数量必须是一样的。是使用循环直接来构建数据标签的。。。后续有空会进行更新调整的 （恩有空的时候 罒ω罒

#### 训练代码

```java
public class AnnMlpTest {
    public static void main(String[] args) {
        //加载opencv库
        System.load("D:\\OpenCV\\opencv\\build\\java\\x64\\opencv_java411.dll");
        //训练
       training();
        //识别测试
        //annTest(null);
    }

    public static void annTest( Mat img_src){/*使用测试分开说明*/}

   /**
     * 训练的方法
     * 内容：加载读取的图片训练数据，加载构建的数据标签，然后调用方法生成模型
     * 搞定，就是这么简单的意思 (⊙o⊙)…
     */
    public static void training(){
        //默认训练集已近是人脸识别之后的的图像 最终我转成 40*40 的进行训练 像素太大参数过多了
        Mat lables = outLables();  // 数据标签 
        Mat trainData = prepareTrainingData("");  //读取训练数据
        //遍历所有数据集
        //进行训练
        ANN_MLP mlp  = ANN_MLP.create();
        // 这里就是把layer 这5个数字参数 转成Mat类型 作为训练的入参
        int[] layer = {40*40,128,128,128,3};
        Mat layerSizes = new Mat(1,layer.length, CvType.CV_32FC1);
        for(int i = 0;i<layer.length;i++){
            layerSizes.put(0,i,layer[i]);
        }
        mlp.setLayerSizes(layerSizes);
        mlp.setActivationFunction(ANN_MLP.SIGMOID_SYM);
        mlp.train(trainData, Ml.ROW_SAMPLE ,lables);
        //训练并保存训练后的模型，这个就是结果了
        mlp.save("D:\\ijworkspace\\meaen_test\\data\\mlptest.xml");
    }

    /**
     * 读取训练数据
     * Mat(sample_num_perclass*class_num,image_rows*image_cols,CvType.CV_32FC1),
     * sample_num_perclass为每种类型的图片数量，class_num为图片种类
     */
    public static Mat prepareTrainingData(String path){
        path = "D:\\ijworkspace\\meaen_test\\data\\training_data";
        Mat trainData = new Mat();
        //获取其file对象
        File file = new File(path);
        //遍历path下的文件和目录，放在File数组中
        File[] fs = file.listFiles();
        //遍历File[]数组 【对应的是 训练集每一个种类的文件夹】
        for(int i =0;i<fs.length;i++){
            if(fs[i].isDirectory())	{
                //若是目录(即文件)
                File file2 = new File(path+"\\"+fs[i].getName());
                //遍历path下的文件和目录，放在File数组中  
                //【对应训练接各个类型下的文件  我这里是每个训练集57个文件】
                File[] fs2 = file2.listFiles();
                for(int j =0;j<fs2.length;j++){
                    // 此循环内为 每一个训练的图片了
                    Mat srcImage = Imgcodecs.imread(fs2[j].toString());
                    Mat resizeImage = new Mat();
                    Mat trainImage = new Mat();
                    int image_col = 40;
                    int image_rows = 40;
                    // 将图片大小调整为40*40 并且转变图片的色彩格式 灰色图
                    Imgproc.resize(srcImage,resizeImage,new Size(image_col,image_rows));
                    Imgproc.cvtColor(resizeImage,trainImage,Imgproc.COLOR_BGR2GRAY);
                    // 接下来这一段就是最麻烦的一段了 处理图片数据集
                    if(j==0&&i==0) {trainData = new Mat(fs2.length*fs.length,image_rows*image_col,CvType.CV_32FC1);}
                    for(int row = 0;row<image_rows;row++){
                        for(int col = 0;col<image_col;col++){
                            double[] d = trainImage.get(row,col);
                            trainData.put(i*fs2.length+j,row*image_col+col,d);
                        }
                    }
                    // 这一段的意思是：就是构建输入的数据集合
                    // 已我的数据为例3个文件，每个文件57个图片，每个图片大小为40*40
                    // 结果就是：一个矩阵 像这个样子  57*1600 的数据矩阵
                    // ⎡  12,31,32,12,12,2,3,11........................22,12 ⎤  第一个图片
                    // ⎢  ........这里的数据为图片每一个像素的值.............. ⎥  一共有2*57 行
                    // ⎣  ........一共是有40*40 个像素(数据)................. ⎦  第3*57个图片
                }
            }
        }
        // 这里把这个数据矩阵以图片形式保存下来 查看（图片每一个像素即使数据）
        Imgcodecs.imwrite("D:\\ijworkspace\\meaen_test\\data\\trainData.jpg", trainData);
        return trainData;
    }

   /**
     * 数据标签 也是个矩阵 57*3 的矩阵  【与上面的图片数据矩阵对应的】【每一行的1 表示此行图片数据属于哪个分类】
     *  ⎡1,0,0⎤   第1个图片
     *  ⎢1,0,0⎥	  第2个图片
     *  ⎣.....⎦   一共有2*57 行
     *  ⎣0,0,1⎦   第3*57个图片
     * @return
     * 由于写法问题，这里需要每个类型的训练图片数量一致
     */
    public static Mat outLables(){
        int sample_num_perclass = 57 ;
        int class_num = 3;
        Mat  lables = new Mat(sample_num_perclass*class_num,class_num,CvType.CV_32FC1);
        for(int i = 0 ;i<class_num;i++){
            for(int j = 0 ;j<sample_num_perclass;j++){
                for(int k = 0 ;k<class_num;k++){
                    if(k==i){
                        lables.put(i*sample_num_perclass+j,k,1);
                    }else{
                        lables.put(i*sample_num_perclass+j,k,0);
                    }
                }
            }
        }
        return lables;
    }

}
```

这里给出一下，以上代码使用的入参已经中间的数据集序列成二维矩阵之后的图片：

![](https://img-blog.csdnimg.cn/20191107173514767.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)

> 这个是训练用的图片

![处理的训练集矩阵](https://img-blog.csdnimg.cn/20191107173524760.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)

> 这个就是 上面代码中的输出的中间处理的训练数据的矩阵，**把一张图片变成一行的像素了**。

#### 测试代码

模型搞出来了，来试一下吧

```java
public static void annTest( Mat img_src){
        ANN_MLP ann = ANN_MLP.load("D:\\ijworkspace\\meaen_test\\data\\mlptest.xml");
        //识别
        if(img_src==null){
            img_src = Imgcodecs.imread("D:\\ijworkspace\\meaen_test\\data\\1332.jpg");
        }
        Mat img_tmp = new Mat();
        Imgproc.resize(img_src,img_tmp,new Size(40,40));
        Imgproc.cvtColor(img_tmp,img_tmp,Imgproc.COLOR_BGR2GRAY);
        //将测试图像转化
        Mat sample = new Mat(1,1600,CvType.CV_32FC1);
        for(int row = 0;row<40;row++){
            for(int col = 0;col<40;col++){
                double[] d = img_tmp.get(row,col);
                sample.put(0,row*40+col,d);
            }
        }
        Mat predict = new Mat();
        ann.predict(sample,predict,ANN_MLP.UPDATE_MODEL);
        System.out.println("sample--"+sample.dump());
        System.out.println("outputs--"+predict.dump());
        Core.MinMaxLocResult maxLoc = Core.minMaxLoc(predict);
        System.out.println("zzzzzz  "+maxLoc.maxVal+":"+maxLoc.minVal);
        System.out.println("结果："+maxLoc.maxLoc.x+"置信度"+maxLoc.maxVal*100+"%");
    }
```

这里测试的结果，简直不能看ε=(´ο｀*)))唉

```java
outputs--[1.4029182, -0.40310526, -0.0076343976]
zzzzzz  1.4029182195663452:-0.4031052589416504
结果：0.0置信度140.29182195663452%
```

先说明一下我的参数：这里的57张的每个类型的人脸图片，基本上是只有10张左右的原始数据，其他是我copy出来的。

导致的结果就是，我用训练的图片进行测试识别，要么是极度匹配，要么就是不知道匹配到哪国去了 **w(ﾟДﾟ)w**

> 比如，训练集里面1和2 是两个眼镜男生，然后测试的时候，只有有个眼镜都匹配到1了 ε=(´ο｀*)))唉

#### 后续优化代码

额。。。想想有空的时候再说吧（改成支持随便数据量的图片训练集，各个类型的图片数量可以不同）

### 代码参考

* <https://blog.csdn.net/akadiao/article/details/79236458> 【使用C++】
* <https://blog.csdn.net/ypqqq/article/details/79024360>【使用C++】

---

### 小结一下

使用ANN_MLP神经网络来训练自己的模型，对于训练集的选择还是要注意才行的啊（包括数据量，数据类型）

最好还是用数字的图片来测试，每个数字几十张应该就可以有不错的识别率了。

至于我所使用的，直接十几张的训练集图片弄人脸识别，只能是试试水，玩一下的。不要期待准确度的 ε=(´ο｀*)))唉

虽说如此，**用MLP来了解神经网络还是很合适的**，可以很好的简单了解，训练的出入参数，和模糊的算法匹配模式。（大佬的可以忽略啊，那不是普通人的级别了

---

2019-11-07 小杭 

一天一篇，好累。。。ψ(*｀ー´)ψ

---
