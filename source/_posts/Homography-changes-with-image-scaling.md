---
title: Homography changes with image scaling
date: 2021-11-01 17:05:46
tags:
categories: SLAM
---
# 概述
本篇文章主要记录最近在评估特征点性能时遇到的一个问题：已知两幅图像和它们之间对应的单应矩阵，当两幅图像分别以不同的比例进行缩放后，它们之间新的单应矩阵与原来的单应矩阵存在什么样的关联？

这是一个常见但是容易被忽略的问题，下文将对两个单应矩阵的关系进行推导，并做实验进行验证。
<!--more-->
# 推导
<div align="center">
<img src="/all_images/Homography changes with image scaling/1.png">
</div>

# 验证
实验中使用的图像以及图像间的单应矩阵来自Hpatches数据集的v_bird序列，[代码github链接](https://github.com/vitalemonate/test_homography_with_img_scaling)
```python
import cv2
import os
import numpy as np


def test_homography_with_img_scaling():
    # 1.读取用于实验的两张图像，分辨率分别为1068×973和986×653
    image_path = "./v_bird"
    img1 = cv2.imread(os.path.join(image_path, "1.ppm"), cv2.IMREAD_COLOR)
    img2 = cv2.imread(os.path.join(image_path, "2.ppm"), cv2.IMREAD_COLOR)

    # 2.将两张图像都resize到640×480分辨率
    img1_resized = cv2.resize(img1, [640, 480])
    img2_resized = cv2.resize(img2, [640, 480])
    homography_path = os.path.join(image_path, "H_1_2")
    H = np.loadtxt(homography_path)

    # 3.构造两个scale矩阵
    s1x, s1y = img1_resized.shape[1]/img1.shape[1], img1_resized.shape[0]/img1.shape[0]
    s2x, s2y = img2_resized.shape[1]/img2.shape[1], img2_resized.shape[0]/img2.shape[0]
    S1 = np.diag([s1x, s1y, 1])
    S2 = np.diag([s2x, s2y, 1])
    # 4.计算image resize后的单应矩阵
    H_resized = S2 @ H @ np.linalg.inv(S1)

    # 5.将img1的四个顶点warp到img2中并画出边框
    pts1 = np.float32([[0, 0], [0, img1.shape[0] - 1],
                       [img1.shape[1] - 1, img1.shape[0] - 1], [img1.shape[1] - 1, 0]]).reshape(-1, 1, 2)
    dst1 = cv2.perspectiveTransform(pts1, H)  # 计算变换后的四个顶点坐标位置
    img2_with_broder = cv2.polylines(img2, [np.int32(dst1)], True, (255, 0, 0), 3, cv2.LINE_AA)
    # 6.将resize后的img1的四个顶点warp到resize后的img2中并画出边框
    pts1_resized = np.float32([[0, 0], [0, img1_resized.shape[0] - 1],
                       [img1_resized.shape[1] - 1, img1_resized.shape[0] - 1], [img1_resized.shape[1] - 1, 0]]).reshape(-1, 1, 2)
    dst1_resized = cv2.perspectiveTransform(pts1_resized, H_resized)  # 计算变换后的四个顶点坐标位置
    img2_resized_with_broder = cv2.polylines(img2_resized, [np.int32(dst1_resized)], True, (255, 0, 0), 3, cv2.LINE_AA)

    # 7.显示并保存实验结果
    cv2.imshow("img2_with_broder", img2_with_broder)
    cv2.imshow("img2_resized_with_broder", img2_resized_with_broder)
    cv2.imwrite("img2_with_broder.png", img2_with_broder)
    cv2.imwrite("img2_resized_with_broder.png", img2_resized_with_broder)
    cv2.waitKey(0)


if __name__ == '__main__':
    test_homography_with_img_scaling()
```
# 实验结果
img2_with_broder：
<div align="center">
<img src="/all_images/Homography changes with image scaling/2.png">
</div>

img2_resized_with_broder：
<div align="center">
<img src="/all_images/Homography changes with image scaling/3.png">
</div>

可以看出在图像resize前后，img1的四个顶点通过单应矩阵warp后，都位于img2的同一个位置，所以上述计算单应矩阵的方法是正确的

# 参考
* [How to change the homography with the scale of the image?](https://stackoverflow.com/questions/21019338/how-to-change-the-homography-with-the-scale-of-the-image)
