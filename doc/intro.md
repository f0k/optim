<a name='optim.overview'></a>
# Overview

Most optimization algorithms have the following interface:

```lua
x*, {f}, ... = optim.method(opfunc, x[, config][, state])
```

where:

* `opfunc`: a user-defined closure that respects this API: `f, df/dx = func(x)`
* `x`: the current parameter vector (a 1D `Tensor`)
* `config`: a table of parameters, dependent upon the algorithm
* `state`: a table of state variables, if `nil`, `config` will contain the state
* `x*`: the new parameter vector that minimizes `f, x* = argmin_x f(x)`
* `{f}`: a table of all `f` values, in the order they've been evaluated (for some simple algorithms, like SGD, `#f == 1`)


<a name='optim.example'></a>
## Example

The state table is used to hold the state of the algorithm.
It's usually initialized once, by the user, and then passed to the optim function as a black box.
Example:

```lua
config = {
   learningRate = 1e-3,
   momentum = 0.5
}

for i, sample in ipairs(training_samples) do
    local func = function(x)
       -- define eval function
       return f, df_dx
    end
    optim.sgd(func, x, config)
end
```

<a name="optim.training"></a>
## Training using optim ##

`optim` is a quite general optimizer, for minimizing any function with respect to a set
of parameters.  In our case, our
function will be the loss of our network, given an input, and a set of weights.  The goal of training 
a neural net is to
optimize the weights to give the lowest loss over our training set of input data.  So, we are going to use optim
to minimize the loss with respect to the weights, over our training set.  We will feed the data to 
`optim` in minibatches.  For this particular example, we will use just one minibatch, but in your own training
you will almost certainly want to break your training set into minibatches, and feed each minibatch to `optim`,
one by one.

We need to give `optim` a function that will output the loss and the derivative of the loss with respect to the
weights, given the current weights, as a function parameter.  The function will have access to our training minibatch, and use this
to calculate the loss, for this minibatch.  Typically, the function would be defined inside our loop over
batches, and therefore have access to the current minibatch data.

Here's how this looks:

__Neural Network__

We create a simple neural network with one hidden layer.
```lua
require 'nn'

local model = nn.Sequential();  -- make a multi-layer perceptron
local inputs = 2; local outputs = 1; local HUs = 20; -- parameters
model:add(nn.Linear(inputs, HUs))
model:add(nn.Tanh())
model:add(nn.Linear(HUs, outputs))
```

__Criterion__

We choose the Mean Squared Error loss criterion:
```lua
local criterion = nn.MSECriterion()
```

We are using an `nn.MSECriterion` because we are training on a regression task, predicting float target values.
For a classification task, we would add an `nn.LogSoftMax()` layer to the end of our
network, and use a `nn.ClassNLLCriterion` loss criterion.

__Dataset__

We will just create one minibatch of 128 examples.  In your own networks, you'd want to break down your
rather larger dataset into multiple minibatches, of around 32-512 examples each.

```lua
local batchSize = 128
local batchInputs = torch.Tensor(batchSize, inputs)
local batchLabels = torch.DoubleTensor(batchSize)

for i=1,batchSize do
  local input = torch.randn(2)     -- normally distributed example in 2d
  local label = 1
  if input[1]*input[2]>0 then     -- calculate label for XOR function
    label = -1;
  end
  batchInputs[i]:copy(input)
  batchLabels[i] = label
end
```

__Flatten Parameters__

`optim` expects the parameters that are to be optimized, and their gradients, to be one-dimensional tensors.
But, our network model contains probably multiple modules, typically multiple convolutional layers, and each
of these layers has their own weight and bias tensors.  How to handle this?

It is simple: we can call a standard method `:getParameters()`, that is defined for any network module.  When
we call this method, the following magic will happen:
- a new tensor will be created, large enough to hold all the weights and biases of the entire network model
- the model weight and bias tensors are replaced with views onto the new contiguous parameter tensor
- and the exact same thing will happen for all the gradient tensors: replaced with views onto one single
contiguous gradient tensor

We can call this method as follows:
```lua
local params, gradParams = model:getParameters()
```

These flattened tensors have the following characteristics:
- to `optim`, the parameters it needs to optimize are all contained in one single one-dimensional tensor
- when `optim` optimizes the parameters in this large one-dimensional tensor, it is implicitly optimizing
the weights and biases in our network model, since those are now simply views onto this large one-dimensional
parameter tensor.

It will look something like this:

![Parameter Flattening](image/parameterflattening.png?raw=true "Parameter Flattening")

Note that flattening the parameters redefines the weight and bias tensors for all the network modules
in our network model.  Therefore, any pre-existing references to the original model layer weight and bias tensors
will no longer point to the model weight and bias tensors, after flattening.

__Training__

Now that we have created our model, our training set, and prepared the flattened network parameters,
we can run training, using `optim`.  `optim` provides [various training algorithms](https://github.com/torch/optim/blob/master/doc/index.md).  We
will use the stochastic gradient descent algorithm [sgd](https://github.com/torch/optim/blob/master/doc/index.md#x-sgdopfunc-x-state).  We
need to provide the learning rate, via an optimization state table:

```lua
local optimState = {learningRate=0.01}
```

We define an evaluation function, inside our training loop, and use `optim.sgd` to run training:
```lua
require 'optim'

for epoch=1,50 do
  -- local function we give to optim
  -- it takes current weights as input, and outputs the loss
  -- and the gradient of the loss with respect to the weights
  -- gradParams is calculated implicitly by calling 'backward',
  -- because the model's weight and bias gradient tensors
  -- are simply views onto gradParams
  local function feval(params)
    gradParams:zero()

    local outputs = model:forward(batchInputs)
    local loss = criterion:forward(outputs, batchLabels)
    local dloss_doutput = criterion:backward(outputs, batchLabels)
    model:backward(batchInputs, dloss_doutput)

    return loss,gradParams
  end
  optim.sgd(feval, params, optimState)
end
```
__Test the network__

For the prediction task, we will also typically use minibatches, although we can run prediction sample by
sample too.  In this example, we will predict sample by sample.  To run prediction on a minibatch, simply
pass in a tensor with one additional dimension, which represents the sample index.

```lua
x = torch.Tensor(2)
x[1] =  0.5; x[2] =  0.5; print(model:forward(x))
x[1] =  0.5; x[2] = -0.5; print(model:forward(x))
x[1] = -0.5; x[2] =  0.5; print(model:forward(x))
x[1] = -0.5; x[2] = -0.5; print(model:forward(x))
```

You should see something like:
```lua
> x = torch.Tensor(2)
> x[1] =  0.5; x[2] =  0.5; print(model:forward(x))

-0.3490
[torch.Tensor of dimension 1]

> x[1] =  0.5; x[2] = -0.5; print(model:forward(x))

 1.0561
[torch.Tensor of dimension 1]

> x[1] = -0.5; x[2] =  0.5; print(model:forward(x))

 0.8640
[torch.Tensor of dimension 1]

> x[1] = -0.5; x[2] = -0.5; print(model:forward(x))

-0.2941
[torch.Tensor of dimension 1]
```

If we were running on a GPU, we would probably want to predict using minibatches, because this will
hide the latencies involved in transferring data from main memory to the GPU.  To predict
on a minbatch, we could do something like:

```lua
local x = torch.Tensor({
  {0.5, 0.5},
  {0.5, -0.5},
  {-0.5, 0.5},
  {-0.5, -0.5}
})
print(model:forward(x))
```
You should see something like:
```lua
> print(model:forward(x))
 -0.3490
 1.0561
 0.8640
 -0.2941
[torch.Tensor of size 4]
```

That's it! For minibatched prediction, the output tensor contains one value for each of our input data samples.
