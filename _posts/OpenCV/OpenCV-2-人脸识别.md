## OpenCV -2 -人脸识别

[TOC]

---

> 使用语言：Java 1.8
> 操作系统：windows x64
> OpenCV：4.1.1

---

### 人脸识别的介绍

人脸识别是一个平常很经常看到，却又很不了解的技术。各种手机的摄像头，自拍或者监控上面经常会出现这个东东，但是关于如何实现的，可能了解的并不多，只是觉得挺高级的。

> 这里需要一个小抄：
>
> 人脸识别(Face Recognition)是一种依据人的面部特征(如统计或几何特征等)，自动进行身份识别的一种生物识别技术，又称为面像识别、人像识别、相貌识别、面孔识别、面部识别等。通常我们所说的人脸识别是基于光学人脸图像的身份识别与验证的简称。
>
> 人脸识别基本步骤：图像采集、图像预处理、特征提取、降维、特征匹配。

### 实现人脸识别【理论】

* 这里为了完成人脸识别，使用的是OpenCV中的目标检测模块：opencv_objdetect ；

> 就是一个依据输入图片，和分类器模型，然后进行检查匹配的东西

* OpenCV有自带的人脸Haar特征分类器：\data\ haarcascades目录下的haarcascade_frontalface_alt.xml；

> 这个haarcascades目录下还有人的全身，眼睛，嘴唇的Haar分类器。

* 实现的过程就是：在对应的图片中，按照训练好的人脸识别特征分类器，使用OpenCV的检测模块功能赛选出来。

> 这里说明一下，特征分类器并不是深度学习或者机器学习、神经网络之类的东西，还是有一点区别的。后者那些高级多了。

### 使用OpenCV来实现人脸识别【直接上代码实现】

代码如下:

```java
public class faceTest extends JPanel {
  /** 参数为需要进行识别的图像 **/
public static Mat detectFace(Mat img) {
		
        System.out.println("Running DetectFace ... ");
        // 从配置文件lbpcascade_frontalface.xml中创建一个人脸识别器，该文件位于opencv安装目录中
        CascadeClassifier faceDetector = new CascadeClassifier("D:\\OpenCV\\opencv\\sources\\data\\haarcascades\\haarcascade_frontalface_alt.xml");

        // 在图片中检测人脸
        MatOfRect faceDetections = new MatOfRect();
        // 灰度化 只是为了加快识别数据和准确性
        Mat imgTmp = new Mat();
        Imgproc.cvtColor(img, imgTmp, Imgproc.COLOR_BGR2GRAY);
		//进行识别分类
        faceDetector.detectMultiScale(imgTmp, faceDetections);

        System.out.println(String.format("Detected %s faces", faceDetections.toArray().length));
		//遍历特征分类器 的结构 【这里就是每一个的脸了】
        Rect[] rects = faceDetections.toArray();
        Random r = new Random();
        int i =0;  //只是个记数用的
        if (rects != null && rects.length >= 1) {
            for (Rect rect : rects) {
                System.out.println(i++);
                //画矩形
                Imgproc.rectangle(img, new Point(rect.x, rect.y), new Point(rect.x + rect.width, rect.y + rect.height), new Scalar(0, 255, 0), 1);
                //保存一下识别到的脸
                 save(img, rect, "D:\\ijworkspace\\meaen_test\\data\\" + r.nextInt(2000) + ".jpg");

                //年龄性别识别  并 画在图片上
                String age = OpenCVTools.predict_age(img.submat(rect));
                System.out.println(age);
                Imgproc.putText(img, "age:"+age, new Point(rect.x, rect.y), Imgproc.FONT_HERSHEY_PLAIN,0.8, new Scalar(0, 255, 0), 1);
                String sex = OpenCVTools.predict_gender(img.submat(rect));
                System.out.println(sex);
                Imgproc.putText(img, "sex:"+sex, new Point(rect.x, rect.y+10), Imgproc.FONT_HERSHEY_PLAIN,0.8, new Scalar(0, 255, 0), 1);

                //图像对比
                Mat faceImg = Imgcodecs.imread("D:\\ijworkspace\\meaen_test\\data\\1625.jpg");
                if(OpenCVTools.compare_image(faceImg,img.submat(rect)) > 0.9 ){
                    Imgproc.putText(img, "匹配:"+"WuBinHong", new Point(rect.x, rect.y+20), Imgproc.FONT_HERSHEY_PLAIN,0.8, new Scalar(0, 255, 0), 1);
                }
            }
        }
        return img;
    }
    
     /**
     * opencv将人脸进行截图并保存
     * @param img
     */
    private static void save(Mat img, Rect rect, String outFile) {
        Mat sub = img.submat(rect);
        Mat mat = new Mat();
        Size size = new Size(300, 300);
        Imgproc.resize(sub, mat, size);
        Imgcodecs.imwrite(outFile, mat);
    }

 public static void main(String[] args) {
        try {
            //加载opencv库  地址为我本地上下文环境
            System.load("D:\\OpenCV\\opencv\\build\\java\\x64\\opencv_java411.dll");
            Mat img = Imgcodecs.imread("D:\\ijworkspace\\meaen_test\\data\\test6.jpg");
            Imgcodecs.imwrite("D:\\ijworkspace\\meaen_test\\data\\result.jpg",detectFace(img));
            img.release();
        } catch (Exception e) {
            System.out.println("有问题");
            StringWriter sw = new StringWriter();
            PrintWriter pw = new PrintWriter(sw);
            e.printStackTrace(pw);
            System.out.println(sw.toString());
        } finally {
            System.out.println("Exit");
        }
    }
}
```

**以上的代码中的年龄性别预测的代码块可以直接注释，这个在后一个章节的时候会介绍。** 

### 图像对比

> 上面代码中有部分图像对比的代码，这里提出来说明一下

这个是使用了基于直方图的图片相似度计算函数，这个对比的识别度条件限制很高。

是直接图片对比，并不是预测。

```java
/**
     *  基于直方图的图片相似度计算函数
     * CV_COMP_CORREL 相关性比较
     * CV_COMP_CHISQR  卡方比较
     * CV_COMP_BHATTACHARYYA 巴氏距离
     * CV_COMP_INTERSECT 十字交叉性
     * @param img_1
     * @param img_2
     * @return
     */
    public static double compare_image(Mat img_1, Mat img_2) {
        // 灰度化
        Mat mat_1 = new Mat();
        Imgproc.cvtColor(img_1, mat_1, Imgproc.COLOR_BGR2GRAY);
        Mat mat_2 = new Mat();
        Imgproc.cvtColor(img_2, mat_2, Imgproc.COLOR_BGR2GRAY);
        Mat hist_1 = new Mat();
        Mat hist_2 = new Mat();

        //颜色范围
        MatOfFloat ranges = new MatOfFloat(0f, 256f);
        //直方图大小， 越大匹配越精确 (越慢)
        MatOfInt histSize = new MatOfInt(1000);

        Imgproc.calcHist(Arrays.asList(mat_1), new MatOfInt(0), new Mat(), hist_1, histSize, ranges);
        Imgproc.calcHist(Arrays.asList(mat_2), new MatOfInt(0), new Mat(), hist_2, histSize, ranges);
// 这里的输出也是为了更好的在玩的时候，看出对比的情况。反正跟我们靠感觉对比不一样，我是看不懂了╮(╯_╰)╭
//        Imgcodecs.imwrite("D:\\ijworkspace\\meaen_test\\data\\1.jpg", hist_1);
//        Imgcodecs.imwrite("D:\\ijworkspace\\meaen_test\\data\\2.jpg", hist_2);
        // CORREL 相关系数
        double res = Imgproc.compareHist(hist_1, hist_2, Imgproc.CV_COMP_CORREL);

        return res;
    }
```

### 小结

这里简单试了一下，使用OpenCV的人脸分类器，进行人脸识别，然后再脸上画个框框显示出来。并保存操作后的图像信息到本地进行查看。

其中有一小段，图片对比的，可以截取人脸作为对比图像进行测试。【相差一点就对比不出来了的 ╮(╯_╰)╭】

---

2019-11-01 小杭

目前识别只是人脸简单识别，不涉及预测。识别率也就那样子，还行的吧！

之后学习的就是，年龄性别预测，以及深度学习训练之类的。最后可能还是需要慢慢研究一下对应的识别算法和模型学习算法，虽然肯定是看不懂的。ε=(´ο｀*)))唉

---

