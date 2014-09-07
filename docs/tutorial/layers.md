---
title: Layer Catalogue
---
# Layers

To create a Caffe model you need to define the model architecture in a protocol buffer definition file (prototxt).

Caffe layers and their parameters are defined in the protocol buffer definitions for the project in [caffe.proto](https://github.com/BVLC/caffe/blob/master/src/caffe/proto/caffe.proto). The latest definitions are in the [dev caffe.proto](https://github.com/BVLC/caffe/blob/dev/src/caffe/proto/caffe.proto).

TODO complete list of layers linking to headings

### Vision Layers

* Header: `./include/caffe/vision_layers.hpp`

Vision layers usually take *images* as input and produce other *images* as output.
A typical "image" in the real-world may have one color channel ($$c = 1$$), as in a grayscale image, or three color channels ($$c = 3$$) as in an RGB (red, green, blue) image.
But in this context, the distinguishing characteristic of an image is its spatial structure: usually an image has some non-trivial height $$h > 1$$ and width $$w > 1$$.
This 2D geometry naturally lends itself to certain decisions about how to process the input.
In particular, most of the vision layers work by applying a particular operation to some region of the input to produce a corresponding region of the output.
In contrast, other layers (with few exceptions) ignore the spatial structure of the input, effectively treating it as "one big vector" with dimension $$chw$$.


#### Convolution

* LayerType: `CONVOLUTION`
* CPU implementation: `./src/caffe/layers/convolution_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/convolution_layer.cu`
* Parameters (`ConvolutionParameter convolution_param`)
    - Required
        - `num_output` (`c_o`): the number of filters
        - `kernel_size` (or `kernel_h` and `kernel_w`): specifies height and width of each filter
    - Strongly Recommended
        - `weight_filler` [default `type: 'constant' value: 0`]
    - Optional
        - `bias_term` [default `true`]: specifies whether to learn and apply a set of additive biases to the filter outputs
        - `pad` (or `pad_h` and `pad_w`) [default 0]: specifies the number of pixels to (implicitly) add to each side of the input
        - `stride` (or `stride_h` and `stride_w`) [default 1]: specifies the intervals at which to apply the filters to the input
        - `group` (g) [default 1]: If g > 1, we restrict the connectivity of each filter to a subset of the input. Specifically, the input and output channels are separated into g groups, and the $$i$$th output group channels will be only connected to the $$i$$th input group channels.
* Input
    - `n * c_i * h_i * w_i`
* Output
    - `n * c_o * h_o * w_o`, where `h_o = (h_i + 2 * pad_h - kernel_h) / stride_h + 1` and `w_o` likewise.
* Sample (as seen in `./examples/imagenet/imagenet_train_val.prototxt`)

      layers {
        name: "conv1"
        type: CONVOLUTION
        bottom: "data"
        top: "conv1"
        blobs_lr: 1          # learning rate multiplier for the filters
        blobs_lr: 2          # learning rate multiplier for the biases
        weight_decay: 1      # weight decay multiplier for the filters
        weight_decay: 0      # weight decay multiplier for the biases
        convolution_param {
          num_output: 96     # learn 96 filters
          kernel_size: 11    # each filter is 11x11
          stride: 4          # step 4 pixels between each filter application
          weight_filler {
            type: "gaussian" # initialize the filters from a Gaussian
            std: 0.01        # distribution with stdev 0.01 (default mean: 0)
          }
          bias_filler {
            type: "constant" # initialize the biases to zero (0)
            value: 0
          }
        }
      }

The `CONVOLUTION` layer convolves the input image with a set of learnable filters, each producing one feature map in the output image.

#### Pooling

* LayerType: `POOLING`
* CPU implementation: `./src/caffe/layers/pooling_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/pooling_layer.cu`
* Parameters (`PoolingParameter pooling_param`)
    - Required
        - `kernel_size` (or `kernel_h` and `kernel_w`): specifies height and width of each filter
    - Optional
        - `pool` [default MAX]: the pooling method. Currently MAX, AVE, or STOCHASTIC
        - `pad` (or `pad_h` and `pad_w`) [default 0]: specifies the number of pixels to (implicitly) add to each side of the input
        - `stride` (or `stride_h` and `stride_w`) [default 1]: specifies the intervals at which to apply the filters to the input
* Input
    - `n * c * h_i * w_i`
* Output
    - `n * c * h_o * w_o`, where h_o and w_o are computed in the same way as convolution.
* Sample (as seen in `./examples/imagenet/imagenet_train_val.prototxt`)

      layers {
        name: "pool1"
        type: POOLING
        bottom: "conv1"
        top: "pool1"
        pooling_param {
          pool: MAX
          kernel_size: 3 # pool over a 3x3 region
          stride: 2      # step two pixels (in the bottom blob) between pooling regions
        }
      }

#### Local Response Normalization (LRN)

* LayerType: `LRN`
* CPU Implementation: `./src/caffe/layers/lrn_layer.cpp`
* CUDA GPU Implementation: `./src/caffe/layers/lrn_layer.cu`
* Parameters (`LRNParameter lrn_param`)
    - Optional
        - `local_size` [default 5]: the number of channels to sum over (for cross channel LRN) or the side length of the square region to sum over (for within channel LRN)
        - `alpha` [default 1]: the scaling parameter (see below)
        - `beta` [default 5]: the exponent (see below)
        - `norm_region` [default `ACROSS_CHANNELS`]: whether to sum over adjacent channels (`ACROSS_CHANNELS`) or nearby spatial locaitons (`WITHIN_CHANNEL`)

The local response normalization layer performs a kind of "lateral inhibition" by normalizing over local input regions. In `ACROSS_CHANNELS` mode, the local regions extend across nearby channels, but have no spatial extent (i.e., they have shape `local_size x 1 x 1`). In `WITHIN_CHANNEL` mode, the local regions extend spatially, but are in separate channels (i.e., they have shape `1 x local_size x local_size`). Each input value is divided by $$(1 + (\alpha/n) \sum_i x_i)^\beta$$, where $$n$$ is the size of each local region, and the sum is taken over the region centered at that value (zero padding is added where necessary).

#### im2col

`IM2COL` is a helper for doing the image-to-column transformation that you most likely do not need to know about. This is used in Caffe's original convolution to do matrix multiplication by laying out all patches into a matrix.

### Loss Layers

Loss drives learning by comparing an output to a target and assigning cost to minimize. The loss itself is computed by the forward pass and the gradient w.r.t. to the loss is computed by the backward pass.

#### Softmax

`SOFTMAX_LOSS`

#### Sum-of-Squares / Euclidean

`EUCLIDEAN_LOSS`

#### Hinge / Margin

* LayerType: `HINGE_LOSS`
* CPU implementation: `./src/caffe/layers/hinge_loss_layer.cpp`
* CUDA GPU implementation: `NOT_AVAILABLE`
* Parameters (`HingeLossParameter hinge_loss_param`)
    - Optional
        - `norm` [default L1]: the norm used. Currently L1, L2
* Inputs
    - `n * c * h * w` Predictions
    - `n * 1 * 1 * 1` Labels
* Output
    - `1 * 1 * 1 * 1` Computed Loss
* Samples

      # L1 Norm
      layers {
        name: "loss"
        type: HINGE_LOSS
        bottom: "pred"
        bottom: "label"
      }

      # L2 Norm
      layers {
        name: "loss"
        type: HINGE_LOSS
        bottom: "pred"
        bottom: "label"
        top: "loss"
        hinge_loss_param {
          norm: L2
        }
      }

#### Sigmoid Cross-Entropy

`SIGMOID_CROSS_ENTROPY_LOSS`

#### Infogain

`INFOGAIN_LOSS`

#### Accuracy and Top-k

`ACCURACY` scores the output as the accuracy of output with respect to target -- it is not actually a loss and has no backward step.

### Activation / Neuron Layers

In general, activation / Neuron layers are element-wise operators, taking one bottom blob and producing one top blob of the same size. In the layers below, we will ignore the input and out sizes as they are identical:

* Input
    - n * c * h * w
* Output
    - n * c * h * w

#### ReLU / Rectified-Linear and Leaky-ReLU

* LayerType: `RELU`
* CPU implementation: `./src/caffe/layers/relu_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/relu_layer.cu`
* Parameters (`ReLUParameter relu_param`)
    - Optional
        - `negative_slope` [default 0]: specifies whether to leak the negative part by multiplying it with the slope value rather than setting it to 0.
* Sample (as seen in `./examples/imagenet/imagenet_train_val.prototxt`)

        layers {
          name: "relu1"
          type: RELU
          bottom: "conv1"
          top: "conv1"
        }

Given an input value x, The `RELU` layer computes the output as x if x > 0 and negative_slope * x if x <= 0. When the negative slope parameter is not set, it is equivalent to the standard ReLU function of taking max(x, 0). It also supports in-place computation, meaning that the bottom and the top blob could be the same to preserve memory consumption.

#### Sigmoid

* LayerType: `SIGMOID`
* CPU implementation: `./src/caffe/layers/sigmoid_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/sigmoid_layer.cu`
* Sample (as seen in `./examples/imagenet/mnist_autoencoder.prototxt`)

        layers {
          name: "encode1neuron"
          bottom: "encode1"
          top: "encode1neuron"
          type: SIGMOID
        }

The `SIGMOID` layer computes the output as sigmoid(x) for each input element x.

#### TanH / Hyperbolic Tangent

* LayerType: `TANH`
* CPU implementation: `./src/caffe/layers/tanh_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/tanh_layer.cu`
* Sample

        layers {
          name: "layer"
          bottom: "in"
          top: "out"
          type: TANH
        }

The `TANH` layer computes the output as tanh(x) for each input element x.

#### Absolute Value

* LayerType: `ABSVAL`
* CPU implementation: `./src/caffe/layers/absval_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/absval_layer.cu`
* Sample

        layers {
          name: "layer"
          bottom: "in"
          top: "out"
          type: ABSVAL
        }

The `ABSVAL` layer computes the output as abs(x) for each input element x.

#### Power

* LayerType: `POWER`
* CPU implementation: `./src/caffe/layers/power_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/power_layer.cu`
* Parameters (`PowerParameter power_param`)
    - Optional
        - `power` [default 1]
        - `scale` [default 1]
        - `shift` [default 0]
* Sample

        layers {
          name: "layer"
          bottom: "in"
          top: "out"
          type: POWER
          power_param {
            power: 1
            scale: 1
            shift: 0
          }
        }

The `POWER` layer computes the output as (shift + scale * x) ^ power for each input element x.

#### BNLL

* LayerType: `BNLL`
* CPU implementation: `./src/caffe/layers/bnll_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/bnll_layer.cu`
* Sample

        layers {
          name: "layer"
          bottom: "in"
          top: "out"
          type: BNLL
        }

The `BNLL` (binomial normal log likelihood) layer computes the output as log(1 + exp(x)) for each input element x.


### Data Layers

#### Database

`DATA`

#### In-Memory

`MEMORY_DATA`

#### HDF5 Input

`HDF5_DATA`

#### HDF5 Output

`HDF5_OUTPUT`

#### Images

`IMAGE_DATA`

#### Windows

`WINDOW_DATA`

#### Dummy

`DUMMY_DATA` is for development and debugging. See `DummyDataParameter`.

### Common Layers

#### Inner Product

* LayerType: `INNER_PRODUCT`
* CPU implementation: `./src/caffe/layers/inner_product_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/inner_product_layer.cu`
* Parameters (`InnerProductParameter inner_product_param`)
    - Required
        - `num_output` (`c_o`): the number of filters
    - Strongly recommended
        - `weight_filler` [default `type: 'constant' value: 0`]
    - Optional
        - `bias_filler` [default `type: 'constant' value: 0`]
        - `bias_term` [default `true`]: specifies whether to learn and apply a set of additive biases to the filter outputs
* Input
    - `n * c_i * h_i * w_i`
* Output
    - `n * c_o * 1 * 1`
* Sample

    layers {
      name: "fc8"
      type: INNER_PRODUCT
      blobs_lr: 1          # learning rate multiplier for the filters
      blobs_lr: 2          # learning rate multiplier for the biases
      weight_decay: 1      # weight decay multiplier for the filters
      weight_decay: 0      # weight decay multiplier for the biases
      inner_product_param {
        num_output: 1000
        weight_filler {
          type: "gaussian"
          std: 0.01
        }
        bias_filler {
          type: "constant"
          value: 0
        }
      }
      bottom: "fc7"
      top: "fc8"
    }

The `INNER_PRODUCT` layer (also usually referred to as the fully connected layer) treats the input as a simple vector and produces an output in the form of a single vector (with the blob's height and width set to 1).

#### Splitting

The `SPLIT` layer is a utility layer that splits an input blob to multiple output blobs. This is used when a blob is fed into multiple output layers.

#### Flattening

The `FLATTEN` layer is a utility layer that flattens an input of shape `n * c * h * w` to a simple vector output of shape `n * (c*h*w) * 1 * 1`.

#### Concatenation

* LayerType: `CONCAT`
* CPU implementation: `./src/caffe/layers/concat_layer.cpp`
* CUDA GPU implementation: `./src/caffe/layers/concat_layer.cu`
* Parameters (`ConcatParameter concat_param`)
    - Optional
        - `concat_dim` [default 1]: 0 for concatenation along num and 1 for channels.
* Input
    - `n_i * c_i * h * w` for each input blob i from 1 to K.
* Output
    - if `concat_dim = 0`: `(n_1 + n_2 + ... + n_K) * c_1 * h * w`, and all input `c_i` should be the same.
    - if `concat_dim = 1`: `n_1 * (c_1 + c_2 + ... + c_K) * h * w`, and all input `n_i` should be the same.
* Sample

        layers {
          name: "concat"
          bottom: "in1"
          bottom: "in2"
          top: "out"
          type: CONCAT
          concat_param {
            concat_dim: 1
          }
        }

The `CONCAT` layer is a utility layer that concatenates its multiple input blobs to one single output blob. Currently, the layer supports concatenation along num or channels only.

#### Slicing

The `SLICE` layer is a utility layer that slices an input layer to multiple output layers along a given dimension (currently num or channel only) with given slice indices.

#### Elementwise Operations

`ELTWISE`

#### Argmax

`ARGMAX`

#### Softmax

`SOFTMAX`

#### Mean-Variance Normalization

`MVN`