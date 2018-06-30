# MNIST with TensorFlow Lite on Android

This project demonstrates how to use [TensorFlow Lite](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/lite) on Android for handwritten digits classification from MNIST.

<div align="center">
  <img src="image/demo.gif" heigit="500"/>
</div>

## How to build from scratch

### Requirement

- Python 3.6, TensorFlow 1.8.0
- Android Studio 3.0, Gradle 4.1
- Linux or macOS if you want to convert the model to .tflite as described in the [Step 2](#step-2-model-conversion) below


### Step 1. Training

```
python train.py --model_dir ./saved_model --iterations 10000
```

After training, a collection of checkpoint files and a frozen GraphDef file `mnist.pb` will be generated in `./saved_model`.

You can test the model on test set using the command below.

```
python test.py --model_dir ./saved_model
```

### Step 2. Model conversion

The standard TensorFlow model obtained in Step 1 cannot be used directly in TensorFlow Lite. We need to freeze the graph and convert the model to flatbuffer format (.tflite). There are two ways to convert the model: use TOCO command-line or python API (which uses TOCO under the hood). 

You will need Linux or macOS for this step as TOCO is not available and cannot be bulit on Windows at the moment.

#### Option 1. Use TOCO command-line

TOCO is a Tensorflow optimizing converter that can convert a TensorFlow GraphDef to TensorFlow Lite flatbuffer. We need to build TOCO with Bazel from Tensorflow repository and use it to convert the `mnist.pb` to a `mnist.tflite`. 

1. Install Bazel

Install Bazel by following the [instructions](https://docs.bazel.build/versions/master/install.html).

2. Clone TensorFlow repository.

```
git clone https://github.com/tensorflow/tensorflow
```

3. Build TOCO

Navigate to the TensorFlow repository directory, run the following command to build TOCO.

```
bazel build tensorflow/contrib/lite/toco:toco
```

4. Convert model

Stay at the TensorFlow repository directory, run the following command to convert the model.

```
/bazel-bin/tensorflow/contrib/lite/toco/toco  \
  --input_file=./saved_model/mnist.pb \
  --input_format=TENSORFLOW_GRAPHDEF  --output_format=TFLITE \
  --output_file=./mnist.tflite --inference_type=FLOAT \
  --input_type=FLOAT --input_arrays=x \
  --output_arrays=output --input_shapes=1,28,28,1
```

The `input_file` argument should point to the TensorFlow GraphDef file (`mnist.pb`) trained in Step 1. The `output_file` argument specifies the location for the converted model.

Notice that the `mnist.pb` generated by [mnist.py] is already frozen, so we can skip the "freeze the graph" step and use it directly for the conversion.

More TOCO examples can be found [here](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/toco/g3doc/cmdline_examples.md).

#### Option 2. Use Python API

Instead of using TOCO command line, we can also convert the model by Python API.

```
python convert.py --model_dir ./saved_model --output_file ./mnist.tflite
```

The `model_dir` argument should point to the checkpoint files directory generated in Step 1. The `output_file` argement specifies the location for the converted model. 

The [convert.py] restores the lastest checkpoint from Step 1, freezes the graph and invokes [`tf.contrib.lite.toco_convert`](https://www.tensorflow.org/versions/master/api_docs/python/tf/contrib/lite/toco_convert) to convert the model.

### Step 3. Build Android app

Copy the `mnist.tflite` generated in Step 2 to `/android/app/src/main/assets`, then build and run the app.
The [Classifer] reads the `mnist.tflite` from `assets` directory and loads it into an [Interpreter](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/java/src/main/java/org/tensorflow/lite/Interpreter.java) for inference. The Interpreter provides an interface between TensorFlow Lite model and Java code, which is included in the following library.

```
implementation 'org.tensorflow:tensorflow-lite:0.1.1'
```

If you are building your own app, remember to add the following code to [build.gradle] to prevent compression for model files.

```
aaptOptions {
    noCompress "tflite"
    noCompress "lite"
}
```

## Credits

- The basic model architecture comes from [tensorflow-mnist-tutorial](https://github.com/martin-gorner/tensorflow-mnist-tutorial).
- The official TensorFlow Lite [Android demo](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/lite/java/demo/app).
- The [FingerPaint](https://android.googlesource.com/platform/development/+/master/samples/ApiDemos/src/com/example/android/apis/graphics/FingerPaint.java) from Android API demo.
