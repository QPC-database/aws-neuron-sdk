.. _appnote-performance-tuning:

Application Note: Performance Tuning
====================================

This guide is intended to provide the reader with an in-depth
understanding on how to optimize neural network performance on
Inferentia for both throughput and latency. For simplicity, the guide
uses TensorFlow and ResNet-50 model as a teaching example to learn how
choosing between different compile-time optimizations (e.g. Batching and
NeuronCore Pipeline), as well as model-serving optimizations (e.g.
multi-threading and dynamic-batching) improves inference performance.

The following guides are considered prerequisites for this tutorial:

-  :ref:`tensorflow-resnet50`
-  :ref:`tensorflow-serving-neurocore-group`
-  :ref:`neuron-batching`
-  :ref:`neuroncore-pipeline`

Batching and pipelining (technical background)
----------------------------------------------

Neuron provides developers with various performance optimization knobs.

Two of the most widely used ones are batching and pipelining. Both
techniques aim to keep the data close to the compute engines, but
achieve that in different ways. In batching it is achieved by loading
the data into an on-chip cache and reusing it multiple times for
multiple different model-inputs, while in pipelining this is achieved by
caching all model parameters into the on-chip cache across multiple
NeuronCores and streaming the calculation across them.

As a general rule of thumb, batching is preferred for applications that
aim to optimize throughput and cost at the expense of latency, while
pipelining is preferred for applications with high-throughput
requirement under a strict latency budget.

Compiling for batching optimization
-----------------------------------

To enable the batching optimization, we first need to compile the model
for a target batch-size. This is done by specifying the batch size in
the input tensor's batch dimension during compilation. Users are
encouraged to evaluate multiple batch sizes in order to determine the
optimal latency/throughput deployment-point, which is application
dependent.

For example, the code snippet below enables batching on a ResNet50
model, with a batch-size of 5:

.. code:: python

   import numpy as np
   import tensorflow.neuron as tfn

   # To change the batch size, change the first dimension in example_input
   batch_size = 5
   example_input = np.zeros([batch_size,224,224,3], dtype='float16')

   # Note: Users should temporarily use the following compilation flags when
   # batch size is larger than 1. These flags are only applicable to CNNs
   # (ResNet50 and similar models) and will be deprecated in the future.
   compiler_args = ['--batching_en', '--rematerialization_en', '--spill_dis',
                    '--sb_size', str((batch_size + 6)*10),
                    '--enable-replication', 'True']

   tfn.saved_model.compile("rn50_fp16",
                           "rn50_fp16_compiled/1",
                           model_feed_dict={'input_1:0': example_input },
                           dynamic_batch_size=True,
                           compiler_args=compiler_args)

.. note::

   Users should temporarily use the following compilation flags when
   batch size is larger than 1:
   ``--batching_en --rematerialization_en --spill_dis --sb_size <(batch_size + 6)*10> --enable-replication=True``.
   These flags are only applicable to CNNs (ResNet50 and similar models)
   and will be deprecated in the future.

.. note::

   Depending on the neural network size, Neuron will have a maximum
   batch size that works optimally on Inferentia. Currently, float16
   ResNet50 is supported up to batch 5 only. Additionally, ResNet50 with
   FP32 input is limited to batch 1 only. These limitations are being
   addressed and will be fixed in a future releases of the compiler. If
   a unsupported batch size is used, an internal compiler error message
   will be displayed (see `Known Issues <#known-issues>`__ section
   below).

Compiling for pipeline optimization
-----------------------------------

With NeuronCore Pipeline mode, Neuron stores the model parameters onto
the Inferentias' local caches, and streams the inference requests across
the available NeuronCores, as specified by the
``--neuroncore-pipeline-cores`` compiler argument. For example, to
compile the model to fit pipeline size of four Inferentia devices (16
NeuronCores) avaliable in the inf1.6xlarge instance size:

.. code:: python

   import numpy as np
   import tensorflow.neuron as tfn

   compiler_args = ['--neuroncore-pipeline-cores', '16']
   example_input = np.zeros([1,224,224,3], dtype='float16')
   tfn.saved_model.compile("rn50_fp16",
                           "rn50_fp16_compiled/1",
                           model_feed_dict={'input_1:0': example_input },
                           compiler_args=compiler_args)

The minimum number of NeuronCores needed to run a compiled model can be
found using Neuron Check Model tool. Please see :ref:`neuron_check_model`.

Model-serving inference optimizations
-------------------------------------

In order to fully realize the maximum throughput of the compiled model
(for either batching and pipelining), users need to launch multiple host
CPU threads to feed inputs into the Neuron pipeline. The number of
threads need to be larger than the specified maximum number of
NeuronCores.

Additionally, dynamic batching (framework optimization currently
supported only by TensorFlow-Neuron) can be used to process a larger
client-side inference batch-size and the framework automatically breaks
up the user-batch into smaller batch sizes to match the compiled
batch-size. This technique increases the achievable throughput by hiding
the framework-to-neuron overhead, and amortizing it over a larger batch
size. To use dynamic batching, set the argument
``--dynamic_batch_size=True`` during compilation and send larger
inference batch size (user inference batch size) that is equal to a
multiple of the compiled batch size.

Both of methods can be applied together if that shows improvement in
performance. However, multi-threading is always needed as a first step
to achieve high throughput. You may need to experiment in order to find
the right optimization settings for your application.

By default, the framework sets the number of outstanding inference
requests to the total number of NeuronCores plus three. This can be
changed by setting the NEURON_MAX_NUM_INFERS environment variable. For
example, if the compiled model includes some CPU partitions (as when
Neuron compiler decided some operations are more efficient to execute on
CPU), the number of threads should be increased to account for the
additional compute performed on the CPU. Note that the available
instance host memory size should be taken into consideration to avoid
out-of-memory errors. As above, you need to experiment in order to find
the right optimization settings for your application.

.. note::

   By default the framework allocates NeuronCore Group size to
   match the size of the compiled model. The size of the model is the
   number of NeuronCores limit passed to compiler during compilation
   (``--neuroncore-pipeline-cores`` option). For more information see
   :ref:`tensorflow-serving-neurocore-group`.

Other considerations
--------------------

Mixed Precision
~~~~~~~~~~~~~~~

Reduced precision data-types are typically used to improve performance.
In the example below, we convert all operations to float16. Neuron also
supports conversion to a mixed-precision graph, wherein only the weights
and the data inputs to matrix multiplies and convolutions are converted
to bfloat16, while the rest of the intermediate results are kept at float32.

By default the Neuron compiler automatically converts (also referred to
as auto-cast) from float32 model to bfloat16 for execution on Inferentia.
The automatic conversion preserves the float32 input and output tensors.
This feature, in most cases, produces similar accuracy to float32 and
does not require you to downcast or retrain models.

There are several --fp32-cast modes. To selectively cast only inputs to MatMul
and Conv operators, use option
 ``--fp32-cast=matmult``. This option may be required in certain
networks such as BERT where additional accuracy is desired.

The casts modes provide trade off between dynamic range (matmult-bf16)
and accuracy (matmult-fp16). The accuracy generally increases and
 performance decreases in order
of all (the default), matmult-bf16, matmult (due to more accurate transpose),
matmult-fp16.

The Neuron compiler preserves the input and output tensor types.
For large tensors the float32 inputs/outputs being transferred to/from Inferentia
may add some execution overhead. Therefore using a
pre-trained float16 model is suggested for fastest performance. If not available,
it is also possible to use a pre-casting script to convert float32 model to be used as float16.

Operator support
~~~~~~~~~~~~~~~~

The Neuron Compiler maintains an evolving list of supported operators
for each framework: :ref:`neuron-supported-operators`

AWS Neuron handles unsupported operators by partitioning the graph into
subgraph, and executing them on different targets (e.g. NeuronCore
partition, CPU partition). If the entire model can run on Inferentia
(i.e. all operators are supported), then the model will be compiled into
a single subgraph which will be executed by a NeuronCore Group.

Debug
~~~~~

You can examine the post-compiled model to view the compilation results
using the Neuron plugin for TensorBoard.
See :ref:`tensorboard-plugin-view-graph`.

ResNet-50 optimization example
------------------------------

For an example demonstrating the concepts described here, see
:ref:`tensorflow-keras-resnet50`
