# Getting Started With Torch7 in DIGITS

Table of Contents
=================
* [Installation](#installation)
* [Enabling support for Torch7 in DIGITS](#enabling-support-for-torch7-in-digits)
* [Selecting Torch7 when creating a model in DIGITS](#selecting-torch7-when-creating-a-model-in-digits)
* [Defining a Torch7 model in DIGITS](#defining-a-torch7-model-in-digits)
    * [External Parameters](#external-parameters)
    * [Internal Parameters](#internal-parameters)
    * [Tensors](#tensors)
* [Examples](#examples)
    * [Adjusting model to inputs dimensions and number of classes](#adjusting-model-to-input-dimensions-and-number-of-classes)
    * [Selecting the NN Backend](#selecting-the-nn-backend)
    * [Supervised Regression Learning](#supervised-regression-learning)
    * [Command Line Inference](#command-line-inference)
    * [Multi-GPU training](#multi-gpu-training)
* [Tutorials](#tutorials)
    * [Training an autoencoder](#training-an-autoencoder)
    * [Training a regression model](#training-a-regression-model)

With v3.0, DIGITS now supports Torch7 as an optional alternative backend to Caffe.

> NOTE: Torch support is still experimental!

## Installation

Follow [these instructions](BuildTorch.md) to install Torch.

## Enabling support for Torch7 in DIGITS

DIGITS should automatically enable support for Torch7 if the `th` executable is in your path. If not, you may explicitly point DIGITS to the appropriate location:

```
$ ./digits-devserver -c

...

==================================== Torch =====================================
Where is torch installed?

	Suggested values:
	(*)  [Previous]       <PATHS>
	(P)  [PATH/TORCHPATH] <PATHS>
	(N)  [none]           <NONE>
>> /home/user/torch/install/bin/th
```

## Selecting Torch7 when creating a model in DIGITS

Select one of the "torch" tabs on the model creation page:

![Home page](images/torch-selection.png)

## Defining a Torch7 model in DIGITS

To define a Torch7 model in DIGITS you need to write a Lua function that takes a table of external network parameters as argument and returns a table of internal network parameters. For example, the following code defines a flavour of LeNet:

```lua
return function(params)
    -- adjust to number of channels in input images - default to 1 channel
    -- during model visualization
    local channels = (params.inputShape and params.inputShape[1]) or 1
    local lenet = nn.Sequential()
    lenet:add(nn.MulConstant(0.00390625))
    lenet:add(nn.SpatialConvolution(channels,20,5,5,1,1,0)) -- channels*28*28 -> 20*24*24
    lenet:add(nn.SpatialMaxPooling(2, 2, 2, 2)) -- 20*24*24 -> 20*12*12
    lenet:add(nn.SpatialConvolution(20,50,5,5,1,1,0)) -- 20*12*12 -> 50*8*8
    lenet:add(nn.SpatialMaxPooling(2,2,2,2)) --  50*8*8 -> 50*4*4
    lenet:add(nn.View(-1):setNumInputDims(3))  -- 50*4*4 -> 800
    lenet:add(nn.Linear(800,500))  -- 800 -> 500
    lenet:add(nn.ReLU())
    lenet:add(nn.Linear(500, 10))  -- 500 -> 10
    lenet:add(nn.LogSoftMax())
    return {
        model = lenet,
        loss = nn.ClassNLLCriterion(),
        trainBatchSize = 64,
        validationBatchSize = 100,
    }
end
```

### External parameters

External parameters are provided by DIGITS:

Parameter name  | Type     | Description
--------------- | -------- | --------
ngpus           | number   | Tells how many GPUs are available (0 means CPU)
nclasses        | number   | Number of classes (for classification datasets). For other datasets this is undefined.
inputShape      | Tensor   | Shape (1D Tensor) of first input Tensor. For image data this is set to {channels, height, width}. Note: this parameter is undefined during model visualization.

### Internal parameters

Those parameters are returned by the user-defined function:

Parameter name        | Type         | Mandatory | Description
-----------------     | ------------ | --------- | -------------
model                 | nn.module    | Yes       | A nn.module container that defines the model to use.
loss                  | nn.criterion | No        | A nn.criterion to use during training. Defaults to nn.ClassNLLCriterion.
croplen               | number       | No        | If specified, inputs images will be cropped randomly to a square of the specified size.
labelHook             | function     | No        | A function(input,dblabel) that returns the intended label(target) for the current batch given the provided input and label in database. By default the database label is used.
trainBatchSize        | number       | No        | If specified, sets train batch size. May be overridden by user in DIGITS UI.
validationBatchSize   | number       | No        | If specified, sets validation batch size. May be overridden by user in DIGITS UI.

### Tensors

Networks are fed with Torch Tensor objects in the NxCxHxW format (index in batch x channels x height x width). If a GPU is available, Tensors are provided as Cuda tensors and the model and criterion are moved to GPUs through a call to their cuda() method. In the absence of GPUs, Tensors are provided as Float tensors.

## Examples

### Adjusting model to input dimensions and number of classes

The following network defines a linear network that takes any 3D-tensor as input and produces one categorical output per class:
```lua
return function(p)
    -- model should adjust to any 3D-input
    local nClasses = p.nclasses or 1
    local nDim = 1
    if p.inputShape then p.inputShape:apply(function(x) nDim=nDim*x end) end
    local model = nn.Sequential()
    model:add(nn.View(-1):setNumInputDims(3)) -- c*h*w -> chw (flattened)
    model:add(nn.Linear(nDim, nclasses)) -- chw -> nClasses
    model:add(nn.LogSoftMax())
    return {
        model = model
    }
end
```

### Selecting the NN backend

Convolution layers are supported by a variety of backends (e.g. `nn`, `cunn`, `cudnn`, ...). The following snippet shows how to select between `nn`, `cunn`, `cudnn` based on their availability in the system:

```lua
if pcall(function() require('cudnn') end) then
   backend = cudnn
   convLayer = cudnn.SpatialConvolution
else
   pcall(function() require('cunn') end)
   backend = nn -- works with cunn or nn
   convLayer = nn.SpatialConvolutionMM
end
local net = nn.Sequential()
lenet:add(backend.SpatialConvolution(1,20,5,5,1,1,0)) -- 1*28*28 -> 20*24*24
lenet:add(backend.SpatialMaxPooling(2, 2, 2, 2)) -- 20*24*24 -> 20*12*12
lenet:add(backend.SpatialConvolution(20,50,5,5,1,1,0)) -- 20*12*12 -> 50*8*8
lenet:add(backend.SpatialMaxPooling(2,2,2,2)) --  50*8*8 -> 50*4*4
lenet:add(nn.View(-1):setNumInputDims(3))  -- 50*4*4 -> 800
lenet:add(nn.Linear(800,500))  -- 800 -> 500
lenet:add(backend.ReLU())
lenet:add(nn.Linear(500, 10))  -- 500 -> 10
lenet:add(nn.LogSoftMax())
```

### Supervised regression learning

In supervised regression learning, labels may not be scalars like in classification learning. To learn a regression model, a generic dataset may be created using one database for input samples and one database for labels (only 1D row label vectors are supported presently). The appropriate loss function must be specified using the `loss` internal parameters. For example the following snippet defines a simple regression model on 1x10x10 images using MSE loss:

```lua
local net = nn.Sequential()
net:add(nn.View(-1):setNumInputDims(3))  -- 1*10*10 -> 100
net:add(nn.Linear(100,2))
return function(params)
    return {
        model = net,
        loss = nn.MSECriterion(),
    }
end
```

### Command Line Inference

DIGITS Lua wrappers may also be used from command line. For example, to classify an image using the snapshot at epoch `10` of a model job `20150921-141321-86c1` using a dataset `20150916-001059-e0cd`:

```
th /fast-scratch/gheinrich/ws/digits/tools/torch/test.lua --image=/path/to/image.png --network=model --networkDirectory=/path/to/jobs/20150921-141321-86c1 --load=/path/to/20150921-141321-86c1 --snapshotPrefix=snapshot --mean=/path/to/jobs/20150916-001059-e0cd/mean.jpg --labels=/path/to/jobs/20150916-001059-e0cd/labels.txt --epoch=10 --crop=no --subtractMean=image
2015-09-22 15:21:55 [INFO ] Loading network definition from /path/to/jobs/20150921-141321-86c1/model
2015-09-22 15:21:55 [INFO ] Loading /path/to/jobs/20150921-141321-86c1/snapshot_10_Weights.t7 file
2015-09-22 15:21:55 [INFO ] For image 1, predicted class 1: 10 (9) 0.99923830445863
2015-09-22 15:21:55 [INFO ] For image 1, predicted class 2: 9 (8) 0.00074051392287852
2015-09-22 15:21:55 [INFO ] For image 1, predicted class 3: 8 (7) 1.6892548943146e-05
2015-09-22 15:21:55 [INFO ] For image 1, predicted class 4: 4 (3) 2.9689886060496e-06
2015-09-22 15:21:55 [INFO ] For image 1, predicted class 5: 5 (4) 9.7695222396362e-07
```

### Multi-GPU training

Data parallelism is supported in Torch7 by cunn through the [DataParallelTable](https://github.com/torch/cunn/blob/master/doc/cunnmodules.md#nn.DataParallelTable)
module. DIGITS provides the number of available GPUs through the `ngpus` external parameter.

Assuming `net` is a container that encapsulates the definition of a network, the following snippet may be used
to enable data parallelism into a container called `model`:

```lua
local model
if ngpus>1 then
   model = nn.DataParallelTable(1)  -- Split along first (batch) dimension
   for i = 1, ngpus do
      cutorch.setDevice(i)
      model:add(net:clone(), i)  -- Use the ith GPU
   end
   cutorch.setDevice(1)  -- This is the 'primary' GPU
else
   model = net
end
```

## Tutorials

### Training an autoencoder

Follow [these instructions](../examples/autoencoder/README.md) to learn how to create an autoencoder using Torch7 in DIGITS.

### Training a regression model

Follow [these instructions](../examples/regression/README.md) to learn how to create a regression model using Caffe or Torch7 in DIGITS.
