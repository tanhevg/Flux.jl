# Training

To actually train a model we need four things:

* A *objective function*, that evaluates how well a model is doing given some input data.
* The trainable parameters of the model.
* A collection of data points that will be provided to the objective function.
* An [optimiser](optimisers.md) that will update the model parameters appropriately.

With these we can call `Flux.train!`:

```julia
Flux.train!(objective, params, data, opt)
```

There are plenty of examples in the [model zoo](https://github.com/FluxML/model-zoo).

## Loss Functions

The objective function must return a number representing how far the model is from its target – the *loss* of the model. The `loss` function that we defined in [basics](../models/basics.md) will work as an objective. We can also define an objective in terms of some model:

```julia
m = Chain(
  Dense(784, 32, σ),
  Dense(32, 10), softmax)

loss(x, y) = Flux.mse(m(x), y)
ps = Flux.params(m)

# later
Flux.train!(loss, ps, data, opt)
```

The objective will almost always be defined in terms of some *cost function* that measures the distance of the prediction `m(x)` from the target `y`. Flux has several of these built in, like `mse` for mean squared error or `crossentropy` for cross entropy loss, but you can calculate it however you want.

At first glance it may seem strange that the model that we want to train is not part of the input arguments of `Flux.train!` too. However the target of the optimizer is not the model itself, but the objective function that represents the departure between modelled and observed data. In other words, the model is implicitly defined in the objective function, and there is no need to give it explicitly. Passing the objective function instead of the model and a cost function separately provides more flexibility, and the possibility of optimizing the calculations.

## Model parameters

The model to be trained must have a set of tracked parameters that are used to calculate the gradients of the objective function. In the [basics](../models/basics.md) section it is explained how to create models with such parameters. The second argument of the function `Flux.train!` must be an object containing those parameters, which can be obtained from a model `m` as `params(m)`.

Such an object contains a reference to the model's parameters, not a copy, such that after their training, the model behaves according to their updated values.

## Datasets

The `data` argument provides a collection of data to train with (usually a set of inputs `x` and target outputs `y`). For example, here's a dummy data set with only one data point:

```julia
x = rand(784)
y = rand(10)
data = [(x, y)]
```

`Flux.train!` will call `loss(x, y)`, calculate gradients, update the weights and then move on to the next data point if there is one. We can train the model on the same data three times:

```julia
data = [(x, y), (x, y), (x, y)]
# Or equivalently
data = Iterators.repeated((x, y), 3)
```

It's common to load the `x`s and `y`s separately. In this case you can use `zip`:

```julia
xs = [rand(784), rand(784), rand(784)]
ys = [rand( 10), rand( 10), rand( 10)]
data = zip(xs, ys)
```

Note that, by default, `train!` only loops over the data once (a single "epoch").
A convenient way to run multiple epochs from the REPL is provided by `@epochs`.

```julia
julia> using Flux: @epochs

julia> @epochs 2 println("hello")
INFO: Epoch 1
hello
INFO: Epoch 2
hello

julia> @epochs 2 Flux.train!(...)
# Train for two epochs
```

## Callbacks

`train!` takes an additional argument, `cb`, that's used for callbacks so that you can observe the training process. For example:

```julia
train!(objective, ps, data, opt, cb = () -> println("training"))
```

Callbacks are called for every batch of training data. You can slow this down using `Flux.throttle(f, timeout)` which prevents `f` from being called more than once every `timeout` seconds.

A more typical callback might look like this:

```julia
test_x, test_y = # ... create single batch of test data ...
evalcb() = @show(loss(test_x, test_y))

Flux.train!(objective, ps, data, opt,
            cb = throttle(evalcb, 5))
```

Calling `Flux.stop()` in a callback will exit the training loop early.

```julia
cb = function ()
  accuracy() > 0.9 && Flux.stop()
end
```
