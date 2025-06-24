---
title: tensorflow中tfrecord数据格式
date: 2019-07-03 19:47:54
tags: 技术 深度学习
---
## JPEG to Tfrecord
tensorflow 在resnet50等模型上都用了imagnet的数据集tfrecord格式
这个tfrecord格式的数据集是通过脚本从jpeg格式转换过来的:
https://github.com/tensorflow/models/blob/master/research/inception/inception/data/build_imagenet_data.py
关键有一行代码：
```
self._decode_jpeg = tf.image.decode_jpeg(self._decode_jpeg_data, channels=3)
```

## Tfrecord to Jpeg
脚本在运行data_sess读到图片之后
```
#从tfrecord里面读取图片
np_images, np_labels = data_sess.run([images, labels])
#保存第一张图片
img = tf.image.encode_jpeg(np_images[0], format='rgb')
writeOp = tf.write_file("out_new.jpeg",img)
sess = tf.Session()
with sess.as_default():
  sess.run(writeOp)
```
np_images的shape应该是n,224,224,3
n就是batchsize

### 类别对应的具体类型查询
比如np_labels的值在脚本里面打印出来是998
在这个地方去搜：
https://blog.csdn.net/weixin_41770169/article/details/80482942
但是数字要减一(998-1=997) 对应的类别就是bolete
