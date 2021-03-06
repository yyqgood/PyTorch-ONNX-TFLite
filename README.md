# TFLite Conversion

<h3 style="color:#ac5353;"> PyTorch -> ONNX -> TF -> TFLite </h3>

Convert PyTorch Models to TFLite and run inference in TFLite Python API.

## PyTorch to ONNX

Load the PyTorch Model:

```python
device = torch.device('cpu')
model = Model()
model.load_state_dict(torch.load(pt_model_path, map_location=device)).eval()
```

Prepare the Input:

```python
img = torch.zeros((1, 3, height, width))
```

Export to ONNX format:

```python
torch.onnx.export(
    model,                  # PyTorch Model
    img,                    # Input tensor
    onnx_model_path,        # Output file (eg. 'output_model.onnx')
    opset_version=12,       # Operator support version
    input_names=['image']   # Input tensor name (arbitary)
    output_names=['output'] # Output tensor name (arbitary)
)
```

> **_opset-version_**: `opset_version` is very important. Some PyTorch operators are still not supported in ONNX even if `opset_version=12`. Default `opset_version` in PyTorch is 12. Please check official ONNX repo for supported PyTorch operators. If your model includes unsupported operators, convert to supported operators. For example, `torch.repeat_interleave()` is not supported, it can be converted into supported `torch.repeat() + torch.view()` to achieve the same function.

> **_output-names_**: If your model returns more than 1 output, provide exact length of arbitary names. For example, if your model returns 3 outputs, then `output_names` should be `['output0', 'output1', 'output3']`. If you don't provide exact length, although PT-ONNX conversion is successful, ONNX-TFLite conversion will not.

## ONNX to TF

You cannot convert ONNX model directly into TFLite model. You must first convert to TensorFlow model.

Use [onnx-tensorflow](https://github.com/onnx/onnx-tensorflow) to convert models from ONNX to Tensorflow.

For Tensorflow 1.x, install as follows:

```bash
pip install onnx-tf
```

For Tensorflow 2.x, install as follows:

```bash
git clone https://github.com/onnx/onnx-tensorflow.git && cd onnx-tensorflow
pip install -e .
```

> **_Note_**: TensorFlow version means the version of TensorFlow installed in your system. It effects in TFLite conversion.

Load the ONNX model:

```python
import onnx

onnx_model = onnx.load(onnx_model_path)
```

Convert with onnx-tf:

```python
from onnx_tf.backend import prepare

tf_rep = prepare(onnx_model)
```

Export TF model:

```python
tf_rep.export_graph(tf_model_path)
```

You will get a Tensorflow `.pb` model.


## TF to TFLite

To convert TF models into TFLite models, you can use official `tf.lite.TFLiteConverter` class.

If you installed TensorFlow 1.x, you can directly convert `.pb` model into `.tflite` model.

To do this:

```python
import tensorflow as tf

converter = tf.compat.v1.lite.TFLiteConverter.from_frozen_graph(
    graph_def_file=tf_model_path,
    input_arrys=['image'],
    input_shapes={'image': [1, height, width, 3]},
    output_arrays=['output']
)
tflite_model = converter.convert()

with open(tflite_model_path, 'wb') as f:
    f.write(tflite_model)
```

For TensorFlow 2.x, you cannot convert `.pb` model directly into `.tflite` model because `TFLiteConverter` doesn't support `.pb` only model anymore. You must first convert `.pb` model into Frozen graph and convert with `from_concrete_functions()`.

```python
def wrap_frozen_graph(graph_def, inputs, outputs):
    def _import_graph_def():
        tf.compat.v1.import_graph_def(graph_def, name="")

    wrapped_import = tf.compat.v1.wrap_function(_import_graph_def, [])
    import_graph = wrapped_import.graph

    return wrapped_import.prune(
        tf.nest.map_structure(import_graph.as_graph_element, inputs),
        tf.nest.map_structure(import_graph.as_graph_element, outputs)
    )

with tf.io.gfile.GFile(tf_model_path, 'rb') as f:
    graph_def = tf.compat.v1.GraphDef()
    loaded = graph_def.ParseFromString(f.read())

frozen_func = wrap_frozen_graph(
    graph_def,
    inputs=["images"],  
    outputs=["output"]  
)

converter = tf.lite.TFLiteConverter.from_concrete_functions([frozen_func])
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS, tf.lite.OpsSet.SELECT_TF_OPS]
converter.optimizations = [tf.lite.Optimize.DEFAULT]

tf_lite_model = converter.convert()

with open(tflite_model_path, 'wb') as f:
    f.write(tflite_model)
```

> **_Note_**: If possible, use TensorFlow 2.x method because inference codes and optimizations are better in TensorFlow 2.x.


## Load and Run TFLite Model 

The following sections are only for TensorFlow 2.x.

```python
import numpy as np
import tensorflow as tf

# Load the TFLite model and allocate tensors
interpreter = tf.lite.Interpreter(model_path="converted_model.tflite")
interpreter.allocate_tensors()

# Get input and output tensors
input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

# Test the model on random input data
input_shape = input_details[0]['shape']
input_data = np.array(np.random.random_sample(input_shape), dtype=np.float32)
interpreter.set_tensor(input_details[0]['index'], input_data)

interpreter.invoke()

# get_tensor() returns a copy of the tensor data
# use tensor() in order to get a pointer to the tensor
output_data = interpreter.get_tensor(output_details[0]['index'])
print(output_data)
```

## Supported Ops and Limitations

TFlite supports a subset of TF operations with some limitations. For full list of operations and limitations see [TF Lite Ops page](https://www.tensorflow.org/mlir/tfl_ops).

Most TFLite ops target float32 and quantized uint8 or int8 inference, but many ops don't support other types like float16 and strings.

## Running TFLite with TF ops

Since TFLite builtin ops only supports a limited number of TF operators, not every model is convertible. 

To allow conversion, usage of certain TF ops can be enabled in TFLite model.

However, running TFLite models with TF Ops requires pulling in the core TF runtime, which increases TFLite interpreter binary size.

[TF Ops that can be enabled in TFLite](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/delegates/flex/allowlisted_flex_ops.cc)

### Convert a Model

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir)
converter.target_spec.supported_ops = [
    tf.lite.OpsSet.TFLITE_BUILTINS,  # enable TFLite ops
    tf.lite.OpsSet.SELECT_TF_OPS  # enable TF ops
]
tflite_model = converter.convert()

open("converted_model.tflite", "wb").write(tflite_model)
```

### Run Inference

When using a TFLite model that has been converted with support for select TF ops, the client must also use a TFLite runtime that includes the necessary library of TF ops.

You don't need to do extra steps to use this select TF ops in Python. TFLite is automatically installed with that support.

### Performance

The following table runs inference on MobileNet with Pixel 2.

Build | Time (ms) | APK Size
--- | --- | ---
Builtin ops | 260.7 | 561KB
Builtin ops + TF ops | 264.5 | 8MB

## Model Optimization

TFLite supports optimization via quantization, pruning and clustering.

### Quantization

Quantization works by reducing the precision of the numbers used to represent a model's parameters (default, float32). This results in a smaller model size and faster computation.

Technique | Data Requirements | Size Reduction | Accuracy 
--- | --- | --- | ---
Post-training float16 quantization | No data | Up tp 50% | Insignificant accuracy loss
Post-training dynamic range quantization | No data | Up to 75% | Accuracy loss 
Post-training integer quantization | Unlabelled data | Up to 75% | Smaller accuracy loss
Quantization-aware training | Labelled training data | Up to 75% | Smallest accuracy loss

#### Post-training float16 quantization

Use this when you are deploying to float16-enabled GPU.

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.target_spec.supported_types = [tf.float16]
tflite_quant_model = converter.convert()
```

#### Post-training dynamic range quantization

Don't use this. Use integer quantization.

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_quant_model = converter.convert()
```

#### Post-training integer quantization

##### Integer with float fallback (using default float input/output)

The model is in integer but use float operators when they don't have an integer implementation.

A common use case for ARM CPU.

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir)
converter.optimizations = [tf.lite.Optimize.DEFAULT]

def representative_dataset_gen():
    for _ in range(num_calibration_steps):
        # get sample input data as numpy array 
        yield [input]

converter.representative_dataset = representative_dataset_gen
tflite_quant_model = converter.convert()
```

##### Integer only

A common use case for 8-bit MCU and Coral Edge TPU.

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir)
converter.optimizations = [tf.lite.Optimize.DEFAULT]

def representative_dataset_gen():
    for _ in range(num_calibration_steps):
        # get sample input data as numpy array 
        yield [input]

converter.representative_dataset = representative_dataset_gen
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8
tflite_quant_model = converter.convert()
```

### Pruning

Pruning works by removing parameters within a model that have only a minor impact on its predictions. 

Pruned models are the same size on disk, and have the same runtime latency, but can be compressed more effectively. This makes pruning a useful technique for reducing model download size.

### Clustering 

Clustering works by grouping the weights of each layer in a model into a predefined number of clusters, then sharing the centroid values for the weights belonging to each individual cluster. This reduces the number of unique weight values in a model, thus reducing its complexity.

As a result, clustered models can be compressed more effecitvely, providing deployment benefits similar to pruning.


## References

* [TFLite Documentation](https://www.tensorflow.org/lite/guide)
* [TFLiteConverter TF 1.x version](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/g3doc/r1/convert/python_api.md)
* [ONNX-TensorFlow](https://github.com/onnx/onnx-tensorflow)
