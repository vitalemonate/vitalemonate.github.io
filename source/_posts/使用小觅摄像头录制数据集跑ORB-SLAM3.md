---
title: 使用小觅摄像头录制数据集跑ORB-SLAM3
date: 2021-06-10 17:55:38
tags: SLAM
---
#  概述
本文档主要记录使用小觅摄像头录制数据集跑ORB-SLAM3的流程，其中录制的数据集需要经过处理，打包成ORB-SLAM3要求的格式才可以，之前都是手动用excel来调整格式，最近写了两个Python脚本对数据进行处理，提高了工作效率，特此记录
<!--more-->

# 使用小觅摄像头录制数据集
  运行小觅SDK目录下samples/_output/bin/dataset目录下的 record即可进行录制

* 可以在samples/dataset/record.cc中的main函数中cam.Open(params)后添加3行代码来控制曝光值
  <div align="center">
  <img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig0.png">
  </div>

* 输出为一个dataset目录，其中包括left目录以及一个motion.txt，其中left目录中存放的是图像以及图像的索引文件stream.txt，motion.txt中存放的是IMU数据
  <div align="center">
  <img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig1.png">
  </div>

* 仿照EuRoC数据集的目录格式将dataset下的目录结构进行更改，并分别在cam0以及imu0目录下存入对应的脚本, 其中cam0目录下的data.csv为原steam.txt，imu0目录下的data.csv为原motion.txt
  <div align="center">
  <img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig2.png">
  </div>

# 处理图像数据
## camera_script.py
```python
# 参考
# https://cloud.tencent.com/developer/article/1687784
# https://blog.csdn.net/weixin_43245453/article/details/90084328
# https://blog.csdn.net/sinat_34615726/article/details/62891393

import os
import pandas as pd
import csv
from itertools import islice

# 对原始时间戳进行处理并按照euroc的格式构建data.csv
# 注意这里的列句为" timestamp"而不是"timestamp"(无语...)
original_file=pd.read_csv('./data.csv',usecols=[' timestamp'])

# 处理后的时间戳：将单位从10us转为1ns
timestamp_list=original_file[' timestamp'].values.tolist()
timestamp_list=[int(time*1e4)  for time in timestamp_list]

# 新的图像名："时间戳.png"
filename_list=[str(time)+".png"  for time in timestamp_list]

processed_dict={"#timestamp": timestamp_list, "filename": filename_list}
dataframe=pd.DataFrame(processed_dict)
dataframe.to_csv('./data.csv', columns=["#timestamp", "filename"], index=False)
print("-"*8 + "data.csv处理完成" + "-"*8)

# 读取data目录下的图像的名称
old_name_list=os.listdir("./data")
old_name_list.sort()

# 读取处理后的data.csv中的filename列为图像的新名称
new_name_file=pd.read_csv('./data.csv',usecols=['filename'])
new_name_list=new_name_file['filename'].values.tolist()

# 构建一个用于批量重命名的文件rename.csv，用于存放图像对应的新旧名称
rename_dict={"old_name": old_name_list, "new_name": new_name_list}
dataframe=pd.DataFrame(rename_dict)
dataframe.to_csv("./rename.csv", columns=["old_name", "new_name"], index=False)

# 根据rename.csv文件对data下的文件进行批量重命名
with open('./rename.csv') as f:
    lines = csv.reader(f)
    # 使用islice是为了跳过rename.csv中第一行标题
    for line in islice(lines, 1, None):
        os.rename("./data/"+line[0], "./data/"+line[1])

print("-"*8 + "图像批量重命名完成" + "-"*8)
```

## 运行脚本处理图像数据
在cam0目录下运行python3 camera_script.py, 这时data目录下的图像根据rename.csv的内容重命名
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig3.png">
</div>

## 脚本主要步骤
### 处理时间戳
根据相邻的两个图像的时间戳的差可以确定小觅摄像头录制的数据集的时间戳单位为**10 us**(很神奇)，而EuRoC的数据时间戳为ns，所以要手动将时间戳单位调整为以ns。因此要将时间戳的值乘以1e4

此外，按照ORB-SLAM3读取图像的要求，图像的命名规则为"时间戳.png"，因此要将timestamp列的内容加上".png"后作为新的图像名
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig4.png">
</div>

### 批量重命名
根据rename.csv的内容对data下的图像进行批量重命名，重命名后rename.csv文件就没有用了

# 处理IMU数据
## imu_script.py
```python
import os
import pandas as pd
import math
import numpy as np

imu_file = "./data.csv"
data = pd.read_csv(imu_file, usecols=[" timestamp", " accel_x", " accel_y", " accel_z", " gyro_x", " gyro_y", " gyro_z"])
timestamp_list = data[" timestamp"].values.tolist()

accel_x_list = data[" accel_x"].values.tolist()
accel_y_list = data[" accel_y"].values.tolist()
accel_z_list = data[" accel_z"].values.tolist()

gyro_x_list = data[" gyro_x"].values.tolist()
gyro_y_list = data[" gyro_y"].values.tolist()
gyro_z_list = data[" gyro_z"].values.tolist()

# 加速度计的各个分量乘9.8
old_acc_list = [accel_x_list, accel_y_list, accel_z_list]
new_acc_list = [[value * 9.8 for value in demision] for demision in old_acc_list]

new_acc_x_list, new_acc_y_list, new_acc_z_list = new_acc_list[0],new_acc_list[1],new_acc_list[2]

print("-"*16+"加速度计的各个分量乘9.8"+"-"*16)

# 陀螺仪的各个分量乘以PI()除以180
old_gyro_list = [gyro_x_list,gyro_y_list,gyro_z_list]
new_gyro_list = [[value*math.pi/180  for value in demision] for demision in old_gyro_list]

new_gyro_x_list, new_gyro_y_list, new_gyro_z_list = new_gyro_list[0],new_gyro_list[1],new_gyro_list[2]
print("-"*16+"陀螺仪的各个分量乘以PI()除以180"+"-"*16)

# 对陀螺仪数据进行插值并舍弃第一个IMU数据
zipped_data = zip(timestamp_list, new_acc_x_list, new_acc_y_list, new_acc_z_list, new_gyro_x_list, new_gyro_y_list, new_gyro_z_list)
zipped_data_list = list(zipped_data)
data_length = len(zipped_data_list)

# 存放插值后的陀螺仪
new_gyro=[]
for index, single_data in enumerate(zipped_data_list):
    if index == 0:
        continue
    elif index == data_length - 1:
        break

    single_data=list(single_data)    

    cur_timestamp = zipped_data_list[index][0]
    prev_timestamp = zipped_data_list[index - 1][0]
    next_timestamp = zipped_data_list[index + 1][0]

    prev_gyro = np.array(zipped_data_list[index - 1][4:7])
    next_gyro = np.array(zipped_data_list[index + 1][4:7])

    interpolation_factor = (next_timestamp - cur_timestamp)/(next_timestamp - prev_timestamp)
    interpolated_gyro = interpolation_factor * prev_gyro + (1 - interpolation_factor) * next_gyro

    new_gyro.append(list(interpolated_gyro))

# zipped_data_list中的各组数据为元组，不可修改，要修改得先转为list
zipped_data_list = [list(item) for item in zipped_data_list]

for index, single_data in enumerate(zipped_data_list):
    if index == 0:
        continue
    elif index == data_length - 1:
        break

    single_data[4:7]=new_gyro[index - 1]

# 删除第一个、最后一个IMU数据以及加速度计陀螺仪值均为0的数据
del zipped_data_list[0], zipped_data_list[-1]
for index, single_data in enumerate(zipped_data_list):
    if not any(single_data[1:7]):
        del zipped_data_list[index]

print("-"*16+"完成陀螺仪的插值"+"-"*16)

# 得到经过处理后的数据
timestamp_list, new_acc_x_list, new_acc_y_list, new_acc_z_list, new_gyro_x_list, new_gyro_y_list, new_gyro_z_list = [list(item) for item in zip(*zipped_data_list)]

# 调整时间戳的单位
timestamp_list = [int(time*1e4) for time in timestamp_list]

# 将调整后的数据写入文件
data_dict={"#timestamp": timestamp_list, "gyro_x": new_gyro_x_list, "gyro_y": new_gyro_y_list, "gyro_z": new_gyro_z_list,
	 "accel_x": new_acc_x_list, "accel_y": new_acc_y_list, "accel_z": new_acc_z_list}
dataframe=pd.DataFrame(data_dict)
dataframe.to_csv(imu_file, columns=["#timestamp", "gyro_x", "gyro_y", "gyro_z", "accel_x", "accel_y", "accel_z"], index=False)

print("-"*16+"完成IMU数据的全部处理工作"+"-"*16)
```
## 运行脚本处理IMU数据
在imu0目录下运行python3 imu_script.py, data.csv中的IMU数据就转为ORB-SLAM3所需的格式

## 脚本主要步骤
### 处理加速度计和陀螺仪数据
* 对加速度计的各个分量乘以9.8，并将陀螺仪的各个分量乘以PI()除以180

* 将陀螺仪的值根据时间戳进行线性插值 interpolated_gyro = (next_time - cur_time)/(next_time - prev_time)*prev_gyro + (cur_time - prev_time)/(next_time - prev_time)*next_gyro，舍弃第一个以及最后一个IMU数据

* 这样数据的结果是隔一行全为0，这些数据需要剔除掉，也可以使用excel来剔除这些数据
    * CTRL + H将所有的 0（单元格匹配） 替代为空

    * 选中所有数据的范围，在“查找和选择” 一栏选择定位 “空值”

    * 右键删除，点击删除整行

* 按照处理图像时间戳的方法处理IMU时间戳，即乘以1e4

* 调换加速度计和陀螺仪的位置，最终IMU数据的第一行为 #timestamp(按照和euroc数据集一样的格式加#) gyro_x  gyro_y gyro_z accel_x accel_y accel_z

# Run ORB-SLAM3
## 使用小觅SDK获取相机以及IMU的相关参数
### IMU相关参数
在小觅SDK目录下运行./samples/_output/bin/get_imu_params会在当前目录下生成一个名为imu_params.params的文件存放相机和IMU的外参（单位mm）以及IMU内参（但是不知道为什么输出的IMU noise和bias的随机游走都是0）
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig5.png">
</div>

#### IMU内参说明
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig6.png">
</div>

由于上一小节用SDK得到的IMU的随机游走和噪声项为0, 在这种配置下跑ORB-SLAM3会失败，因此IMU的内参采用了之前的工程文件中保留的参数，具体的值为
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig7.png">
</div>

#### IMU和左目相机的外参
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig8.png">
</div>

### 相机相关参数
在小觅SDK目录下运行./samples/output/bin/get_img_params会在当前文件夹下生成一个新的文件image_params.params（这里显示的内参是自己使用小觅的标定工具标定的结果，不是原来的内参值，由于不确定标定是否精准，所以在跑ORB-SLAM3时采用旧的内参值）
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig9.png">
</div>

旧的相机内参
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig10.png">
</div>

## 配置ORB-SLAM3的相关文件
* 在Examples/Monocular-Inertial目录下新建小觅摄像头的配置文件MYNT.yaml并将上一节得到的IMU以及相机参数填入MYNT.yaml
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig11.png">
</div>

* 在Examples/Monocular-Inertial/下新建目录MYNT——Timestamps存放存储图像的时间戳的文件

* 根据mono_inertial_euroc.cc的main函数开头处的提示修改词袋路径，数据集路径，配置路径，地图路径以及图像时间戳文件的路径即可用小觅录制的数据集跑ORB-SLAM3
<div align="center">
<img src="/all_images/使用小觅摄像头录制数据集跑ORB-SLAM3/fig12.png">
</div>
