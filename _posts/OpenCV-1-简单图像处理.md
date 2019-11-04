## OpenCV -1 -简单图像处理

[TOC]

---

> 使用语言：Java 1.8
> 操作系统：windows x64
> OpenCV：4.1.1

---

### 项目环境的搭建

* 官网下载opencv对应版本环境的文件，安装：opencv-4.1.1-vc14_vc15.exe
* idea加载jar包：......\opencv\build\java\opencv-411.jar
* 然后就可以使用了。

### 加载OpenCV并简单处理图片

#### 加载OpenCV

> 这里需要一些小抄介绍OpenCV：
>
> 该库具有2500多种优化算法，其中包括一整套经典和最新的计算机视觉和机器学习算法。这些算法可用于检测和识别人脸，识别物体，对视频中的人类动作进行分类，跟踪相机运动，跟踪运动物体，提取物体的3D模型，从立体相机产生3D点云，将图像缝合在一起以产生高分辨率整个场景的图像，从图像数据库中查找相似的图像，从使用闪光灯拍摄的图像中消除红眼，跟随眼睛的运动，识别风景并建立标记以将其与增强现实叠加在一起等。

在使用OpenCV之前需要加载它的本地库。

> OpenCV用C ++原生编写，虽然加载了jar包，但是真正的处理不在这里。所以需要加载opencv_java411.dll的运行库。

加载代码：

```java
 System.load("D:\\OpenCV\\opencv\\build\\java\\x64\\opencv_java411.dll");
// 没错。就这么调用一个方法就可以了 ╮(╯_╰)╭
```

#### 加载图片处理

> OpenCV内置了很多的API方法，旋转，过滤，调通道等等

```java
 public void run() {
     	//加载图片
        Mat img = Imgcodecs.imread("D:\\ijworkspace\\meaen_test\\data\\test.jpg");
        //中值滤波将图像的每个像素用邻域 (以当前像素为中心的正方形区域)像素的 中值 代替
        //图像平滑处理：中值滤波：输入、输出、基数
        Imgproc.medianBlur(img, img, 7);
        //旋转
        Point center = new Point(img.width() / 2.0, img.height() / 2.0);
        Mat affineTrans = Imgproc.getRotationMatrix2D(center, 90.0, 1.0);
        Imgproc.warpAffine(img, img, affineTrans, img.size(), Imgproc.INTER_NEAREST);
     	//图片输出
        Imgcodecs.imwrite("D:\\ijworkspace\\meaen_test\\data\\test-end.png", img);
     	//释放资源
        img.release();
    }
```

执行之后就可以有对应的结果图片查看效果了！！！！！

---

2019-11-1 小杭
OpenCV从入门到放弃现在正式开始了。。。

---

