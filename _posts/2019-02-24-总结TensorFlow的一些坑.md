---
layout:     post
title:      总结TensorFlow的一些坑与Tips
date:       2019-02-24
author:     sg
catalog: true
tags:
    - Android
    - TensorFlow
---

### 

​	最近在做一个根据传感器特征来进行人机是别的项目，总结了一些TensorFlow的一些坑

### 1.TensorFlow与TensorFlowLite

​	移动端一定要用TensorFlowLite，我最开始就是弄错了，做了很多无用功😭。两种使用的模型是不同的，客户端的sdk也是不同的。训练模型的时候倒是可以使用TensorFlow，生成的pb文件可以使用flite_convert来转换成lite文件。命令如下:

```bash
tflite_convert \
  --graph_def_file=model.pb \
  --output_file=optimized_graph.lite \
  --input_format=TENSORFLOW_GRAPHDEF \
  --output_format=TFLITE \
  --input_shape=1,128,9 \
  --input_array=inputs \
  --output_array=softmax_tensor_output \
  --inference_type=FLOAT \
  --input_data_type=FLOAT
```

### 2.编译客户端sdk

​	TensorFlow是发布了AAR的，但是为什么要编译呢，主要就是TensorFlow在AAR中提供的so不包含armeabi的版本，然而国内的App由于一些历史原因apk内只有armeabi，因此我们要独立的编译一份jar和so,然后将armeabi-v7a的so直接放到armeabi文件夹下面，当然前提是你是min sdk足够高。否则捕获一个UnsatisfiedLinkError也行。

​	TensorFlow使用bazel作为编译系统，TF的代码也在快速的更新，我觉得直接使用master分支的代码进行编译反而比较好。编译TF最重要的一件事就是，新建一个Ubuntu虚拟机！别用mac！clone下载运行./configure，不用像网上说的那样修改配置文件，那样反而容易出错，然后直接编译就好了。

### 3.模型转换

​	转换pb到lite/tflite有一个坑就是tensorflow的一些算子不被tflite所支持，遇到这种情况直接将不支持的算子替换一种实现就好了

### 4.Code

下面是主要的代码，run的返回值返回值是一组对应标签的概率。

```java
    public TFClassifier(Context c){
        try {
            tflite = new Interpreter(loadModelFile(c));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public Ret getActivityProb(float[] input_signal){
        float[][] fs = new float[1][8];
        tflite.run(input_signal, fs);
        Ret ret = new Ret();
        System.arraycopy(fs[0], 0, ret.probs, 0, 8);
        ret.state = ret.getMax();
        return ret;
    }

    private MappedByteBuffer loadModelFile(Context context) throws IOException {
        AssetFileDescriptor fileDescriptor = context.getAssets().openFd("model.lite");
        FileInputStream inputStream = new FileInputStream(fileDescriptor.getFileDescriptor());
        FileChannel fileChannel = inputStream.getChannel();
        long startOffset = fileDescriptor.getStartOffset();
        long declaredLength = fileDescriptor.getDeclaredLength();
        return fileChannel.map(FileChannel.MapMode.READ_ONLY, startOffset, declaredLength);
    }

```

