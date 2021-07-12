将voxblox ++中的mask rcnn更改至tf2环境总结：
保留build文件夹！！！！！！！！！

1.安装tf2.6-nightly,cuda11.2,cudnn8.1.0,使得3090可以工作
在安装TensorFlow2.6，cuda11.2，cudnn8.1.0,python3.8,配置3090 工作环境时，使用tf.test.is_gpu_available()测试tf时出现错误：
tensorflow.python.framework.errors_impl.InternalError: CUDA runtime implicit initialization on GPU:0 failed. Status: all CUDA-capable devices are busy or unavailable
错误原因：显卡驱动问题，之前讲显卡驱动安装为专用驱动460，更改为开源驱动460后，成功！！！

2.安装模组，以及修改tf2中已经不存在的函数，改个名或者替换掉

3.出现问题ValueError: Tried to convert 'shape' to a tensor and failed. Error: None values not supported.
在GitHub上找了很长时间，发现很多人也有这个问题，最终找到https://github.com/matterport/Mask_RCNN/issues/1070
原文：The following steps worked for me:
1. Upgrade the scripts by using the following line on the root folder:
tf_upgrade_v2 --intree Mask_RCNN --inplace --reportfile report.txt
This will automatically update the existing code to TF2. You will also get a list of changes made in report.txt
2. Replace the following line:
mrcnn_bbox = layers.Reshape((-1, num_classes, 4), name="mrcnn_bbox")(x)
with this this if-else code block:
if s[1]==None:
  mrcnn_bbox = layers.Reshape((-1, num_classes, 4), name="mrcnn_bbox")(x)
 else:
  mrcnn_bbox = layers.Reshape((s[1], num_classes, 4), name="mrcnn_bbox")(x)
3. Change the following line:
indices = tf.stack([tf.range(probs.shape[0]), class_ids], axis=1)
with this line:
indices = tf.stack([tf.range(tf.shape(probs)[0]), class_ids], axis = 1)
4. Now, you need to replace:
from keras import saving
with:
from tensorflow.python.keras import saving
then you will also want to replace the lines in both if and else block:
saving.load_weights_from_hdf5_group(f, layers)
and so on with the follwoing lines, inside if and else block respectively:
saving.hdf5_format.load_weights_from_hdf5_group_by_name(f, layers)
saving.hdf5_format.load_weights_from_hdf5_group(f, layers)
实测有效。

5.下载了最新的maskrcnn源码，在此基础上修改，保留了coco.py,还有部分函数可以从旧代码中移植进来

voxblox编译过程：
更改了maskrcnn后，在工作空间下catkin build mask_rcnn_ros gsm......,可能出现的错误：
*.fatal error: boostdesc_bgm.i: 没有那个文件或目录 #include "boostdesc_bgm.i"
将成功的工程下/build/opencv3_build/downloads/xfeatures2d下的文件拷到对应位置，以及报错的那个include文件夹下，如/build/opencv3_catkin/opencv3_contrib_src/modules/xfeatures2d/include下
*.fatal error: opencv2/xfeatures2d/cuda.hpp: 没有那个文件或目录#  include "opencv2/xfeatures2d/cuda.hpp"
将成功的工程下/build/opencv3_catkin/opencv3_contrib_src/modules/xfeatures2d/include/opencv2下的文件拷到报错的那个include文件夹下，如build/opencv3_catkin/opencv3_src/modules/stitching/include/opencv2

与工程中depth_segmentation冲突报错，图片尺寸不对
修改mask_rcnn_ros_node中的_build_result_msg函数中的mask.data = (result['masks'][:, :, i] * 255).tobytes() 的*255删掉即可，可能是新版本的maskrcnn使用的bool编码，旧的用的uint8
