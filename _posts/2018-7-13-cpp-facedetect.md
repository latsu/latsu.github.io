---
layout: post
category: ['C++', 'OpenCV']
title: 基于C++实现的实时视频人脸检测
---
# 环境配置
## 1.工作平台

**说明:** 

之前用Python和OpenCV写的人脸检测无法到达生产环境的要求，Libfacedectect扩展的人脸检测识别率要高于Dlib和OpenCV自带的人脸检测功能，然而Libfacedetect人脸检测库只支持C++，所以这次试用C++来实现。

- Windows10 64位操作系统
- Visual Studio 2017 vc15

## 2.配置OpenCV库

1、**项目->属性->VC++目录**

**包含目录**添加:

	PATH\opencv\include\opencv2
	PATH\opencv\include\opencv
	PATH\opencv\include

**库目录**添加:

	PATH\opencv\x64\vc15\lib

2、**项目->属性->连接器->输入->附加依赖项**

	opencv_calib3d341.lib
	opencv_core341.lib
	opencv_dnn341.lib
	opencv_features2d341.lib
	opencv_flann341.lib
	opencv_highgui341.lib
	opencv_imgcodecs341.lib
	opencv_imgproc341.lib
	opencv_ml341.lib
	opencv_objdetect341.lib
	opencv_photo341.lib
	opencv_shape341.lib
	opencv_stitching341.lib
	opencv_superres341.lib
	opencv_video341.lib
	opencv_videoio341.lib
	opencv_videostab341.lib

*参考：* [【Windows 7 x64】OpenCV 3.4.1 下载与安装详细教程](https://blog.csdn.net/JM_1013/article/details/80465705)

## 3.配置Libfacedetect扩展库

1、**项目->属性->VC++目录**

**包含目录**添加:

	PATH\libfacedetect\include

**库目录**添加:

	PATH\libfacedetect\lib

2、**项目->属性->连接器->输入->附加依赖项**

	olibfacedetect-x64.lib


*参考：* 

- [如何快糙好猛的使用Shiqi.Yu老师的公开人脸检测库（附源码）](https://blog.csdn.net/mr_curry/article/details/51804072
)
- [OpenCV学习笔记（11）：libfacedetection人脸检测的配置与使用](https://blog.csdn.net/cv_jason/article/details/78819088)

## 4. 配置Libcurl扩展库

1、**项目->属性->VC++目录**

**包含目录**添加:

	PATH\libcurl\include

**库目录**添加:

	PATH\libcurl\lib

2、**项目->属性->连接器->输入->附加依赖项**

	libcurl_a.lib
	Ws2_32.lib
	Wldap32.lib
	winmm.lib
	Crypt32.lib
	Normaliz.lib


*参考：*

- [The Easy interface](https://curl.haxx.se/libcurl/c/)
- [C/C++使用libcurl库实现post图片的两种方式](https://blog.csdn.net/LeeKitch/article/details/80194011)

## 5.全局配置
1、**配置环境变量**

	PATH\libfacedetect\bin
	PATH\opencv\x64\vc15\bin
2、**项目->属性->C/C++->预处理器**

预处理器定义:

	CURL_STATICLIB
	HTTP_ONLY
	_CRT_SECURE_NO_WARNINGS

3、**说明：**

- opencv和libcurl的库需要编译。

- opencv使用CMake编译，libcurl使用VS自带的命令行工具。


# 实现代码

```c++
#include"facedetect-dll.h"
#include<iostream>
#include<opencv2\opencv.hpp>
#include <string>
#include "curl/curl.h"
#define DETECT_BUFFER_SIZE 0x20000

using namespace std;
using namespace cv;
/**
 * author: Latsu
 * params:  string url
 *          string array header
 *          int size
 * return 服务器返回结果
 * POST人像图片到服务器
 */
CURLcode curl_post_request(const string &url, string header[], int size)
{
	CURL *curl = curl_easy_init();
	CURLcode result;
	struct curl_httppost *formpost = NULL;
	struct curl_httppost *lastptr = NULL;
	for (int i = 0; i < size; i++) {
		curl_formadd(&formpost, &lastptr, 
			CURLFORM_PTRNAME, "image[]", 
			CURLFORM_FILE, header[i].c_str(),
			CURLFORM_CONTENTTYPE, "image/jpeg", 
			CURLFORM_END);
	}
	if (curl) {
		curl_easy_setopt(curl, CURLOPT_POST, 1);
		curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
		curl_easy_setopt(curl, CURLOPT_HTTPPOST, formpost);
		curl_easy_setopt(curl, CURLOPT_NOSIGNAL, 1);
		curl_easy_setopt(curl, CURLOPT_HEADER, 1);
		curl_easy_setopt(curl, CURLOPT_CONNECTTIMEOUT, 30);
		curl_easy_setopt(curl, CURLOPT_TIMEOUT, 30);
		result = curl_easy_perform(curl);
	}
	curl_easy_cleanup(curl);
	curl_formfree(formpost);
	return result;
}

void faceDetection(const Mat&image) 
{
	Mat gray;
	cvtColor(image, gray, CV_BGR2GRAY);
	int * pResults = NULL;
	unsigned char * pBuffer = (unsigned char *)malloc(DETECT_BUFFER_SIZE);
	int doLandmark = 1;
	pResults = facedetect_multiview_reinforce(pBuffer,
	(unsigned char*)(gray.ptr(0)),
		gray.cols,
		gray.rows,
		(int)gray.step,
		1.1f, 3, 48, 0,
		doLandmark);
	if (int res = pResults ? *pResults : 0) {
		string imagepath = "D:\\image\\header\\";
		string *header = new string[res];
		Mat result_multiview = image.clone();
		for (int i = 0; i < res; i++) {
			short * p = ((short*)(pResults + 1)) + 142 * i;
			int x = p[0] - 20;
			int y = p[1] - 60;
			int w = p[2] + 30;
			int h = p[3] + 60;
			int neighbors = p[4];
			int angle = p[5];
			if (x < 0) { x = 0; }
			if (y < 0) { y = 0; }
			if (w < 0) { w = 0; }
			if (h < 0) { h = 0; }
			if (neighbors < 0) { neighbors = 0; }
			if (angle < 0) { angle = 0; }
			header[i] = imagepath + "headimg_" + std::to_string(i) + ".jpg";
			Mat image_cut = Mat(result_multiview, Rect(x, y, w, h));
			imwrite(header[i], image_cut);
		}
		//把剪切出来的头像post给php服务器
		curl_global_init(CURL_GLOBAL_ALL);
		string postUrlStr = "https://dev.api.gmall.gaopeng.com//api/camera/GP_SH_004";
		//string postUrlStr = "121.41.44.70/api/test";
		auto result = curl_post_request(postUrlStr, header, res);
		//system("pause");
		if (result != CURLE_OK) {
			cerr << "curl_easy_perform() failed: " + string(curl_easy_strerror(result)) << endl;
		} else {
			//发送完成以后删除人像
			for (int i = 0; i < res; i++) {
				remove(header[i].c_str());
			}
		}
		curl_global_cleanup();
		waitKey(0);
		free(pResults);
	}
	
}

int main() {
	//读取视频或摄像头
	VideoCapture capture("rtsp://admin:admin@123@10.98.0.13:554/?tcp");
	if (!capture.isOpened()) {
		cout << "Couldn't open the video file." << endl;
		return 1;
	}
	string imagepath= "D:\\image\\original\\";
	string imagename;
	Mat frame;
	int i = 1;
	int real = 60;
	capture >> frame;
	while (true) {
		Sleep(1);
		if (frame.empty())
			break;
		if (i%real == 0) {
			imagename = imagepath + "originalImage_" + std::to_string(10000 + i) + ".jpg";
			imwrite(imagename, frame);
			Mat src = imread(imagename.c_str(), IMREAD_COLOR);
			faceDetection(src);
			remove(imagename.c_str());
		}
		i++;
		waitKey(30);
	}
	return 0;
}
```
### **`需要优化的地方还是很多的，比如多个摄像头采用的方式可能是多线程或写配置文件等等，以后有时间在弄。`**

## 参考文章
- [从视频中提取图片，对图片做人脸检测并截取人脸区域](https://www.cnblogs.com/yamin/p/7338070.html)
