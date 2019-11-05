---
layout: post
#标题配置
title:   OpenCV-4-视频流整合识别预测模型
#时间配置
date:   2019-11-05 10:00:00 +0800
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

### 说明一下

在此之前，已经试过了图片的简单处理，人脸识别，年龄性别预测。

而视频的处理呢，其实就是吧视频流对应的每一帧的图片拿出来识别一下，然后画点东西上去，再显示出来。

大部分的解释内容都在代码的注释里面了，这种学习使用方法的看代码是最直观的。

> 显示使用swing的就行了。
>
> （由于电脑性能是视频质量的问题，限制为5帧识别一次，不然好卡）

### 代码：加载视频输出，调用图像识别

> 代码是拆分进行说明的，但是所有代码都会完整提供的。
>
> 之后也会上传到资源，不过那个要积分的。在博客里面copy出来整理一下也就可以了。

```java
public class videoTest extends JPanel {
    private BufferedImage mImg;
    
    public static void main(String[] args) {
        //使用Swing生成GUI
        JFrame frame = new JFrame("camera");
        try {
            //加载opencv库  本地上下文
            System.load("D:\\OpenCV\\opencv\\build\\java\\x64\\opencv_java411.dll");

            //加载普通视频
//            VideoCapture capture = new VideoCapture("D://VID_20181110_213415.mp4");
            //获取摄像头视频流 参数为第几个视频设备
            VideoCapture capture = new VideoCapture(0);
            System.out.println("获取视频流情况：" + capture.isOpened());
            int height = (int) capture.get(Videoio.CAP_PROP_FRAME_HEIGHT);
            int width = (int) capture.get(Videoio.CAP_PROP_FRAME_WIDTH);
            if (height == 0 || width == 0) {
                throw new Exception("camera not found!");
            }
            // 创建swing 窗口
            frame.setDefaultCloseOperation(WindowConstants.DISPOSE_ON_CLOSE);
            videoTest panel = new videoTest();
            frame.setContentPane(panel);
            frame.setVisible(true);
            frame.setSize(width + frame.getInsets().left + frame.getInsets().right,
                    height + frame.getInsets().top + frame.getInsets().bottom);

            Mat capImg = new Mat();
            Mat temp = new Mat();
            long num = 1;
            while (frame.isShowing()) {
                num++;
                //获取视频帧
                capture.read(capImg);
                //旋转 用于调整治疗颈椎病的视频
//                Point center = new Point(capImg.width() / 2.0, capImg.height() / 2.0);
//                Mat affineTrans = Imgproc.getRotationMatrix2D(center, 90.0, 1.0);
//                Imgproc.warpAffine(capImg, capImg, affineTrans, capImg.size(), Imgproc.INTER_NEAREST);

                // 由于电脑性能问题，只是每n帧进行一次识别
                if (num % 10 == 0) {
                    //转换为灰度图
                    Imgproc.cvtColor(capImg, temp, Imgproc.COLOR_RGB2GRAY);
                    //识别人脸
                    Mat image = detectFace(capImg);
                    //转为图像显示
                    panel.mImg = panel.mat2BI(image);
                    panel.repaint();
                } else {
                    //对未进行识别的帧，进行之前一次识别数据的重绘 参数太多很乱，建议大屏幕查看
                    if (OpenCVTools.getRect() != null) {
//                        for (Rect rect : OpenCVTools.getRect()) {
                        for (int i = 0; i < OpenCVTools.getRect().size(); i++) {

                            Imgproc.rectangle(capImg, new Point(OpenCVTools.getRect().get(i).x, OpenCVTools.getRect().get(i).y),
                                    new Point(OpenCVTools.getRect().get(i).x + OpenCVTools.getRect().get(i).width,
                                            OpenCVTools.getRect().get(i).y + OpenCVTools.getRect().get(i).height),
                                    new Scalar(0, 255, 0), 1);

                            Imgproc.putText(capImg, "age:" + OpenCVTools.rest_age_sex.get(i).get(0),
                                    new Point(OpenCVTools.getRect().get(i).x, OpenCVTools.getRect().get(i).y), Imgproc.FONT_HERSHEY_PLAIN, 0.8,
                                    new Scalar(0, 255, 0), 1);

                            Imgproc.putText(capImg, "sex:" + OpenCVTools.rest_age_sex.get(i).get(1),
                                    new Point(OpenCVTools.getRect().get(i).x, OpenCVTools.getRect().get(i).y - 10), Imgproc.FONT_HERSHEY_PLAIN, 0.8,
                                    new Scalar(0, 255, 0), 1);

                            if(Double.valueOf(OpenCVTools.rest_age_sex.get(i).get(2) ) > 0.8){
                                Imgproc.putText(capImg, "like:"+"WuBinHong",
                                        new Point(OpenCVTools.getRect().get(i).x, OpenCVTools.getRect().get(i).y-20), Imgproc.FONT_HERSHEY_PLAIN,0.8,
                                        new Scalar(0, 255, 0), 1);
                            }
                        }
                    }
                    //转为图像显示
                    panel.mImg = panel.mat2BI(capImg);
                    panel.repaint();
                }
            }
            capture.release();
            frame.dispose();
        } catch (Exception e) {
            System.out.println("有问题");
            StringWriter sw = new StringWriter();
            PrintWriter pw = new PrintWriter(sw);
            e.printStackTrace(pw);
            System.out.println(sw.toString());
        } finally {
            System.out.println("Exit");
            frame.dispose();
        }
        System.exit(0);
    }

 /**
     * 转换图像
     *
     * @param mat
     * @return
     */
    private BufferedImage mat2BI(Mat mat) {
        int dataSize = mat.cols() * mat.rows() * (int) mat.elemSize();
        byte[] data = new byte[dataSize];
        mat.get(0, 0, data);

        int type = mat.channels() == 1 ? BufferedImage.TYPE_BYTE_GRAY : BufferedImage.TYPE_3BYTE_BGR;
        if (type == BufferedImage.TYPE_3BYTE_BGR) {
            for (int i = 0; i < dataSize; i += 3) {
                byte blue = data[i + 0];
                data[i + 0] = data[i + 2];
                data[i + 2] = blue;
            }
        }
        BufferedImage image = new BufferedImage(mat.cols(), mat.rows(), type);
        image.getRaster().setDataElements(0, 0, mat.cols(), mat.rows(), data);

        return image;
    }

    @Override
    public void paint(Graphics g) {
        if (mImg != null) {
            g.drawImage(mImg, 0, 0, mImg.getWidth(), mImg.getHeight(), this);
        }
    }
}
```

> OpenCV的视频处理预设方法，还要很多，比如每一帧的回调函数，延迟信息，运行状态等等。由于这里想弄的不在视频处理，所以只是用最简单的获取每一帧进行处理。

### 代码：图像中人脸识别

由于使用摄像头拍摄的时候，很多时候拍摄到的都是侧脸，所以只使用一个正脸识别感觉不够。如果还有其他的识别需求，可以看看OpenCV的模型文件夹下，还有很多现成的识别模型（眼睛，身体，等等）。

**这里只做了侧脸+正脸的识别。**

```java
/**
     * opencv实现人脸识别，同时检测到人脸和人眼时才截图
     * @param img  需要识别的图像
     */
    public static Mat detectFace(Mat img) {
        OpenCVTools.setRect(new ArrayList<Rect>());
        OpenCVTools.rest_age_sex = new ArrayList<ArrayList<String>>();
        System.out.println("Running DetectFace ... ");
        // 从配置文件lbpcascade_frontalface.xml中创建一个人脸识别器，该文件位于opencv安装目录中
        CascadeClassifier faceDetector = new CascadeClassifier("D:\\OpenCV\\opencv\\sources\\data\\haarcascades\\haarcascade_frontalface_alt.xml");
        CascadeClassifier profilefaceDetector = new CascadeClassifier("D:\\OpenCV\\opencv\\sources\\data\\haarcascades\\haarcascade_profileface.xml");

        // 在图片中检测人脸
        MatOfRect faceDetections = new MatOfRect();
        faceDetector.detectMultiScale(img, faceDetections);
        System.out.println(String.format("Detected %s faces", faceDetections.toArray().length));
        Rect[] rects = faceDetections.toArray();
        // 独立出来的，进行对识别人脸图像的处理
        pointRect(rects, img);  

        // 在图片中检测侧脸
        MatOfRect profilefaceDetections = new MatOfRect();
        profilefaceDetector.detectMultiScale(img, profilefaceDetections);
        System.out.println(String.format("Detected %s profileface", profilefaceDetections.toArray().length));
        Rect[] profilefaceRects = profilefaceDetections.toArray();
        pointRect(profilefaceRects, img);

        return img;
    }
```



### 代码：人脸年龄性别预测，绘制信息

这里的性别预测使用的方法也就是上一篇的内容。

```java
/**
     * 绘制人脸以及预测年龄图像信息
     * 也可以进行人脸截取保存
     * @param rects 识别到的人脸列表
     * @param img   原始识别图片
     */
    private static void pointRect(Rect[] rects, Mat img) {
        if (rects != null && rects.length >= 1) {
            for (Rect rect : rects) {
                OpenCVTools.getRect().add(rect);
                //给脸 画矩形
                Imgproc.rectangle(img, new Point(rect.x, rect.y), 
                                  new Point(rect.x + rect.width, rect.y + rect.height),
                        new Scalar(0, 255, 0), 1);

                //年龄识别
                String age = OpenCVTools.predict_age(img.submat(rect));
                Imgproc.putText(img, "age:" + age, new Point(rect.x, rect.y), Imgproc.FONT_HERSHEY_PLAIN, 0.8, new Scalar(0, 255, 0), 1);
                //性别识别
                String sex = OpenCVTools.predict_gender(img.submat(rect));
                Imgproc.putText(img, "sex:" + sex, new Point(rect.x, rect.y - 10), Imgproc.FONT_HERSHEY_PLAIN, 0.8, new Scalar(0, 255, 0), 1);
                ArrayList<String> age_sex = new ArrayList<String>();
                age_sex.add(age);
                age_sex.add(sex);

                //图像对比
                Double compareNum = OpenCVTools.compare_image(OpenCVTools.faceImg, img.submat(rect));
                Double compareNum2 = OpenCVTools.compare_image(OpenCVTools.faceImg2, img.submat(rect));
                age_sex.add(compareNum > compareNum2 ? compareNum.toString() : compareNum2.toString());
                if (compareNum > 0.8 || compareNum2 > 0.8) {
                    Imgproc.putText(img, "like:" + "One People", new Point(rect.x, rect.y - 20), Imgproc.FONT_HERSHEY_PLAIN, 0.8, new Scalar(0, 255, 0), 1);
                } else {
                    //保存不被识别的人脸照片
//                    save(img, rect, "D:\\ijworkspace\\meaen_test\\data\\face\\" + new Random().nextInt(2000) + ".jpg");
                }
                //保存 识别的信息，由于其他帧的重绘
                OpenCVTools.rest_age_sex.add(age_sex);
            }
        }
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
```



### 代码：补充的一下代码

> 这里的参数是会用到的，作为识别帧保存的识别数据，用于重绘。

```java
public  class OpenCVTools {
    public static Mat faceImg = Imgcodecs.imread("D:\\ijworkspace\\meaen_test\\data\\1625.jpg");
    public static Mat faceImg2 = Imgcodecs.imread("D:\\ijworkspace\\meaen_test\\data\\914.jpg");

    private static List<Rect> rect = null;
    public static ArrayList<ArrayList<String>> rest_age_sex =  new ArrayList<ArrayList<String>>();

    public static  java.util.List<Rect> getRect() {
        return rect;
    }

    public static void setRect( java.util.List<Rect> rect) {
        OpenCVTools.rect = rect;
    }
    // 文章的一下封装的静态方法都是放在这个类里面的，只是为了文章需要拆分开了。
}
```



### 小结一下

这个视频流进行图像识别的代码，的结果就是**摄像头实时图像识别**，代码中还有一个简单的图像对比，这样就可以蛮当做一个识别摄像头的程序使用了。

比如，使用老板的头像数据做对比，在检查到比配度很好的情况下，提示些信息，或者切换到桌面 罒ω罒

---

2019-11-05 小杭 

使用摄像头来进行实时图像识别完成了，学习结束了。（然而并没有

接下来，就要研究一下识别模型和深度学习训练和算法了。。。

---
