# All You Need is a Good Init

[ICLR 2016](http://www.iclr.cc/doku.php?id=iclr2016:main#accepted_papers_conference_track) has a really interesting [paper](http://arxiv.org/abs/1511.06422) "All You Need is a Good Init". In this post I will try to repeat the results of the authors and will do that in Torch.

## General idea

So, what is the motivation? [Batch Normalization](http://arxiv.org/abs/1502.03167) helps, but it slows down the training process (the authors claim it's 30%). Can we do better without additional overhead?

Yes, we can spend more time for smart weights initialization (not much more), but get benefits in training speed, ability to use bigger learning rates and better results.

```pseudo
Pre-initialize network with orthonormal matrices as in Saxe et al.(2014)
for each layer L do
    while |Var(B_L) - 1.0| >= Tol_var and (T_i) < T_max) do
        do Forward pass with a mini-batch
        calculate Var(B_L)
        W_L = W_L /sqrt(Var(B_L))
    end while
end for
```

What I found important for the implementation here:

* L is a Convolutional or Fully Connected layer
* For each layer we use a new minibatch.
* We compute variance for the whole data in minibatch: Var(B_L)! (at first I thought that we compute variance feature-wise)

## Torch implementation

```lua
require 'nn'

nn.Sequential.lsuvInit = function (self, get_batch, tol_var, t_max)
   local tol_var = tol_var or 0.1
   local t_max   = t_max or 10

   for _,m in ipairs(self:listModules()) do
      if m.weight ~= nil then
         local t_i = 1
         while true do
            local input = get_batch()
            self:forward(input)
            local var = torch.var(m.output)
            if torch.abs(var - 1.0) < tol_var or t_i > t_max then
               break
            end
            m.weight:div(math.sqrt(var))
            t_i = t_i + 1
         end
      end
   end
end
```

Usage (from MNIST example):

```lua
require 'nn'

-- add nninit.orthogonal to all convolutional and fully connected layers
model:add(nn.SpatialConvolutionMM(1, 32, 5, 5):init('weight', nninit.orthogonal, {gain = 'relu'}))
model:add(nn.ReLU())
...
model:add(nn.Linear(200, #classes):init('weight', nninit.orthogonal, {gain = 'relu'}))

--do LSUV after orthogonal init above
if opt.lsuv then
  model:lsuvInit(get_batch)
end
```
## MNIST example

I used the following bash command to run the experiment (-f for full mnist dataset: 60 000 for training and 10 000 for testing):

```bash
th mnist-example.lua --lsuv -r lr
```

epoch |with lsuv (lr=0.1)| with lsuv (lr=0.05) | without lsuv (lr=0.001) | with lsuv (lr=0.001)
------------ | ------------ | ------------- | ------------- | -------------
1|97.77%|96.69%|83.39%|78.28%
2| 98.45%| 97.94%|89.25%|87.75%
3| 98.63%|98.37%|91.23%|91.19%
4| 98.74%|98.57|92.46%|92.82%
5| 98.88%|98.72%|93.23%|93.81%
6| 98.97%|98.75%| 93.88%|94.53%
7| 99.03%|98.86%| 94.44%|95.06%
8| 99.01%|98.86%|94.81%|95.4%
9| 99.01%|98.9%|95.03%|95.87%
10|98.96%|98.91|95.29%|96.15%

I did not wait for 100 epochs as the authors of the original paper did. At first, I thought that we can use bigger learning rates when we use LSUV, but then I realised that MNIST nolsuv case does not use BN, so, this is not true. And MNIST results just show us that training works and the accuracy rates are pretty comparable. Let's have a look at CIFAR-10 experiment.

## CIFAR example

I did not check the limit of the accuracy we can achieve, but just checked if the training is comparable in general. And it is. Test dataset accuracy is on the pic.

<img class='center' src="https://github.com/yobibyte/yobiblog/blob/master/pics/cifar_test_cost.png"/>


## References
* "All You Need is a Good Init" [paper](http://arxiv.org/abs/1511.06422).
* [Original implementation](https://github.com/ducha-aiki/LSUVinit) in Caffe.
* [Orthogonal init](http://arxiv.org/abs/1312.6120) also used in paper
* Super convenient torch [module](https://github.com/Kaixhin/nninit) for initialization

Thanks for the debugging and help to [@ikostrikov](https://github.com/ikostrikov)

If you want to ask me a question, you can find me [here](https://github.com/yobibyte/yobiblog/blob/master/pages/about.md)
