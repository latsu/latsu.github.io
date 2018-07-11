---
layout: post
category: ['Python'，'OpenCv']
title: 基于Python-OpenCv的视频实时人脸检测功能
---

# python2.7和OpenCv2.4 环境配置
## 1.windows 10

**下载地址：** [https://www.python.org/downloads/windows/](https://www.python.org/downloads/windows/)

![](/res/img/in_posts/2018/pydl.png)

下载完成后安装到 C:/Python/

添加环境变量：C:\Python和C:\Python\scripts

win+r 进入搜索输入cmd打开命令行:

1、更新pip工具

```python
pip install --upgrade pip
```

2、使用pip命令安装setuptools（是一组Python的 distutilsde工具的增强工具，可以更方便的创建和发布 Python包。）

```python
pip install --upgrade setuptools
```

3、安装OpenCV依赖库

```python
pip install numpy
```

***NumPy***系统是Python的一种开源的数值计算扩展。这种工具可用来存储和处理大型矩阵，比Python自身的嵌套列表（nested list structure）结构要高效的多。

参考：[http://www.numpy.org](http://www.numpy.org)

4、安装OpenCv扩展

```python
pip install opencv-python
```

## 2. centos6.7 : 

1、安装环境依赖

```
yum install cmake gcc gcc-c++ gtk+-devel gimp-devel gimp-devel-tools gimp-help-browser zlib-devel libtiff-devel libjpeg-devel libpng-devel gstreamer-devel libavc1394-devel libraw1394-devel libdc1394-devel jasper-devel jasper-utils swig python libtool nasm -y
```

**下载源码:** 

```
wget https://www.python.org/ftp/python/2.7.15/Python-2.7.15.tgz
```
以下操作按顺序执行：

```
//解压文件
gzip –d Python-2.7.15.tgz && cd Python-2.7.15

//建立新的python路径
mkdir /usr/local/python27

//把python2.6改名
mv /usr/bin/python /usr/bin/python2

//安装python2.7.15
./configure --prefix=/usr/local/python27
make && make install
ln -s /usr/local/python27/bin/python2.7 /usr/bin/python
```

修改yum的执行脚本，否则yum命令无法使用:

```shell
vim /usr/bin/yum
#!/usr/bin/python 修改为 #!/usr/bin/python2
```

2、查看PIP命令是否可用，如果不可用则编译安装pip工具

```
wget https://files.pythonhosted.org/packages/ae/e8/2340d46ecadb1692a1e455f13f75e596d4eab3d11a57446f08259dee8f02/pip-10.0.1.tar.gz
```

```
tar zxvf pip-10.0.1.tar.gz && cd pip-10.0.1
python setup.py build 
python setup.py install
```

添加全局变量（否则要添加路径到PATH）

```
ln -s /usr/local/python/bin/pip /usr/bin/pip
ln -s /usr/local/python/bin/pip3 /usr/bin/pip3
```

3、下载numpy依赖包

```
wget https://files.pythonhosted.org/packages/71/90/ca61e203e0080a8cef7ac21eca199829fa8d997f7c4da3e985b49d0a107d/numpy-1.14.3-cp36-cp36m-manylinux1_x86_64.whl
```
添加依赖

```
pip install numpy-1.14.3-cp36-cp36m-manylinux1_x86_64.whl
```

4、安装opencv

```
wget https://github.com/opencv/opencv/archive/2.4.13.6.zip
unzip 2.4.13.6.zip &&  cd opencv-2.4.13.6
mkdir build && cd build
cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local -D INSTALL_PYTHON_EXAMPLES=ON -D BUILD_EXAMPLES=ON -D WITH_GTK_2_X=ON ..
```
**make -j [N] :表示在那个一时间内进行编译的任务数 ，如果-j 后面不跟任何数字，则不限制处理器并行编译的任务数**

```
make -j4
make install
```

##3.功能实现

新建一个文件名称：**RealFaceDetection.py**;

```python
#-*- coding: utf-8 -*-

import os
import time
import base64
import urllib
import urllib2
import cv2
import sys
from PIL import Image
        
#视频来源，可以来自一段已存好的视频，也可以直接来自USB摄像头
#cap = cv2.VideoCapture(sys.argv[1])
cap = cv2.VideoCapture("C:/Users/sherry/Desktop/1.mp4")
#print cap.isOpened()
    
#告诉OpenCV使用人脸识别分类器
classfier = cv2.CascadeClassifier("C:\Python\Lib\site-packages\cv2\data\haarcascade_frontalface_alt2.xml")
		        
#识别出人脸后要画的边框的颜色，RGB格式
color = (0, 255, 0)
i=1
#每25帧检测一次
real = 25 
while cap.isOpened():
    ok, frame = cap.read() #读取一帧数据
    if not ok:            
        break  
    #将当前帧转换成灰度图像
    grey = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    #人脸检测，1.1和3分别为图片缩放比例和需要检测的有效点数
    faceRects = classfier.detectMultiScale(grey, scaleFactor = 1.1, minNeighbors = 3, minSize = (32, 32))
    #大于0则检测到人脸
    if len(faceRects) > 0:
        #单独框出每一张人脸
        if(i%real == 0):
            imgdir = "C:\Users\sherry\images\imgs_"+str(int(time.time()))+".jpg"
            cv2.imwrite(imgdir, frame)
            img = Image.open(imgdir)
            with open(imgdir,"rb") as istr:
                imgbase64 = base64.b64encode(istr.read())
            images = [imgbase64]
            for faceRect in faceRects:
                x, y, w, h = faceRect        
                cv2.rectangle(frame, (x - 50, y - 50), (x + w + 50, y + h + 50), color, 2)
                cv2.waitKey(10)
                headir = "C:\Users\sherry\images\head_"+str(int(time.time()))+".jpg"
                headimage = img.crop([x - 50, y - 50, x + w + 50, y + h + 50])
                #保存图像
                headimage.save(headir)
                with open(headir,"rb") as hstr:
                    headbase64 = base64.b64encode(hstr.read())
                images.append(headbase64)
                #os.remove(imgdir)
            #检测到的人脸发送到测试服务器
            url  = "http://latsu/api/test"
            data = urllib.urlencode({"images":images})
            request = urllib2.Request(url, data)
            response = urllib2.urlopen(request)
            #res = response.read()
            #print res
        i = i+1
    #显示图像
    cv2.imshow("The realtime video stream face detection", frame)
    #每帧间隔时间30ms
    c = cv2.waitKey(30)
    if c & 0xFF == ord('q'):
        break
#释放摄像头并销毁所有窗口
cap.release()
cv2.destroyAllWindows() 

```

## 4. 注意事项

`由于OpenCV的库是开源的，用的是比较老的人脸检测和识别算法，开发者后期并没有对算法进行精度优化，所以`**建议**`不要在生产环境使用此功能，人脸检测的准确率和精度太低。`


