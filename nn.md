# Create a neural network

Now let's look how to create neural networks in Gluon. We first import the neural network `nn` package from `gluon`.

```{.python .input  n=1}
from mxnet import nd
from mxnet.gluon import nn
```

## Create a layer

Now create a dense layer with 2 output units.

```{.python .input  n=2}
layer = nn.Dense(2)
layer
```

Then initialize its weights with the default initialization method, which draws random values uniformly from $[-0.7, -0.7]$

```{.python .input  n=3}
layer.initialize()
```

In order to compute the output, or called forward, we create a $(3,4)$ shape random input `x` and feed into the layer.

```{.python .input  n=4}
x = nd.random.uniform(-1,1,(3,4))
layer(x)
```

As can be seen, we got a $(3,2)$ shape output. Note that we didn't specify the input size of `layer` before (though we can specify it with the argument `in_units=4` here), the system will automatically infer it during the first time we feed in data, create and initialize the weights. So we can access the weight after the first forward:

```{.python .input  n=5}
layer.weight.data()
```

## Create a neural network flexibly with `nn.Block`

To create a network, we can create a subclass of `nn.Block` and implement two methods:

- `__init__` create the layers
- `forward` define the forward function.

```{.python .input  n=6}
class LeNet(nn.Block):
    def __init__(self, **kwargs):
        # invoke nn.Block's __init__ method
        super(LeNet, self).__init__(**kwargs)
        # In order to save/load parameters to/from disks, layers 
        # from the nn package needs to be created within a name 
        # scope. 
        with self.name_scope():
            # Simliar to Dense, no necessary to specify 
            # the input channels by the argument `in_channels`, 
            # which will be automatically inferred in the first time 
            # invoke `forward`
            #
            # In additional, one can use a tuple to specify a 
            # non-square kernel size, such as `kernel_size=(2,4)`
            self.conv1 = nn.Conv2D(channels=6, kernel_size=5)
            # One can also use a tuple to specify non-symmetric 
            # pool and stide sizes
            self.pool1 = nn.MaxPool2D(pool_size=2, strides=2)
            self.conv2 = nn.Conv2D(channels=16, kernel_size=3)
            self.pool2 = nn.MaxPool2D(pool_size=2, strides=2)
            self.dense1 = nn.Dense(120)
            self.dense2 = nn.Dense(84)
            self.dense3 = nn.Dense(10)
            
    # x is an array with NDArray type
    def forward(self, x):
        # Note we used the relu operator in the nd package. Any other 
        # function in this package can be used as well.
        y = self.pool1(nd.relu(self.conv1(x)))
        # Since it is purely imperative, we can get 
        # results immediatly. Such as inserting `print(y)` will print 
        # the intermediate results.  
        y = self.pool2(nd.relu(self.conv2(y)))
        # flattern the 4-D input into 2-D with shape 
        # `(y.shape[0], y.size/y.shape[0])` 
        y = nd.flatten(y)
        y = nd.relu(self.dense1(y))
        y = nd.relu(self.dense2(y))
        y = self.dense3(y)
        return y
    
net = LeNet()
net    
```

The usage of `LeNet` is similar to `nn.Dense`. In fact, `nn.Dense` is a subclass of `nn.Block` as well. The following codes show how to create an instance, initialize the weights and run the forward method.

```{.python .input  n=8}
net.initialize()
# Input shape is (batch_size, RGB_channels, height, width)
x = nd.random.uniform(-1, 1, (4,1,28,28))
# Equal to call `net.forward(x)`
y = net(x)
y.shape
```

Access the first convolution layer's weight and first dense layer's bias.

```{.python .input  n=11}
(net.conv1.weight.data().shape, 
 net.dense1.bias.data().shape)
```

## A shorter way with `nn.Sequential`

We have the freedom to write a complex forward function with the `nn.Block` approach, including using arbitrary functions in the `nd` package, printing intermediate results, and using Python's control flow such as `if-else` and `for`.

If the network forward function is simply invoking the layers one-by-one, we can use `nn.Sequential` to make the codes shorter. Let's reimplement the above `LeNet`.

```{.python .input  n=13}
net = nn.Sequential()
# Creating layers in a name scope.
with net.name_scope():
    # Chain a sequence of layers. Note that we can only add 
    # subclass of `nn.Block`, which includes all layers in the `nn` package.
    net.add(
        # We absorb the activation layer into Conv2D
        nn.Conv2D(channels=6, kernel_size=5, activation='relu'),
        nn.MaxPool2D(pool_size=2, strides=2),
        nn.Conv2D(channels=16, kernel_size=3, activation='relu'),
        nn.MaxPool2D(pool_size=2, strides=2),
        # Use the Fattern function in the `nn` package  
        nn.Flatten(),        
        nn.Dense(120, activation="relu"),
        nn.Dense(84, activation="relu"),
        nn.Dense(10)
    )
net
```

`nn.Sequential` will automatically generate a `forward` method that calls the added layers sequentially.

```{.python .input  n=14}
net.initialize()
y = net(x)
y.shape
```

We can use `[]` to index a particular layer

```{.python .input  n=15}
(net[0], net[-1])
```

## A mixed approach

In Gluon, no matter it is a single layer such as `nn.Dense`, or a complete neural network such as `net`, it shares the same base class `nn.Block`. We can flexibly glue them together. For example, the following example shows that how we can construct a neural work with a mixed usage of `nn.Block` and `nn.Sequential`.

```{.python .input}
class MixMLP(nn.Block):
    def __init__(self, **kwargs):
        super(MixMLP, self).__init__(**kwargs)
        with self.name_scope():
            # `nn.Sequential` is a subclass of `nn.Block`, it will 
            # be reconginzed by Gluon
            self.blk = nn.Sequential()
            # Already within a name scope, no need to create
            # another scope.
            self.blk.add(
                nn.Dense(10, activation='relu'),
                nn.Dense(20, activation='relu')
            )
            self.dense = nn.Dense(30)
    def forward(self, x):
        return self.dense(nd.relu(self.blk(x)))

net = nn.Sequential()
with net.name_scope():
    net.add(
        nn.Dense(5),
        # `add` supports any subclass of `nn.Block`
        MixMLP(),
        nn.Dense(8)
    )
net
```
