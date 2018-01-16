---
title: "Elastic Embedding"
date: "January 12, 2018"
output:
  html_document:
    theme: cosmo
    toc: true
    toc_float:
      collapsed: false
---

[Elastic Embedding (PDF)](http://faculty.ucmerced.edu/mcarreira-perpinan/papers/icml10.pdf)
is related to symmetric SNE, but isn't normalized. Apart from the method itself,
the paper is interesting for being the earliest one I'm aware of that makes
the connection between the attractive part of the cost function of SNE with 
[spectral methods](https://jlmelville.github.io/smallvis/spectral.html) 
(e.g. Laplacian eigenmaps). It also manages to produce some good results (on 
an admittedly small number of datasets) without using normalization of the 
affinities. This makes the cost function much more amenable to stochastic
gradient descent techniques that would allow even better scaling to large
datasets than the Barnes-Hut approach used with t-SNE. And indeed, LargeVis
(and UMAP), which uses SGD for its scaling, had a very similar cost function
to EE.

The elastic embedding cost function is:

$$
C = 
\sum_{ij} v_{ij}^{+} d_{ij}^{2} + 
\lambda \sum_{ij} v_{ij}^{-} \exp\left(-d_{ij}^{2}\right)
$$

This differs slightly from t-SNE and its variants in using un-normalized weights
(like LargeVis and UMAP) and introducing separate input weights, $v_{ij}^{+}$ and
$v_{ij}^{-}$ for the attractive and repulsive part of the cost function, respectively.
Additionally, a weighting parameter, $\lambda$, is introduced to weight the 
attractive and repulsive contributions to the cost function. Note that this
is unrelated to the $\lambda$ used in the NeRV cost function.

### Positive Weights

The positive weights are defined as:

$$
v_{ij}^{+} = \exp\left(-\beta_i r_{ij}^{2}\right)
$$
i.e. the usual Gaussian input weights as used in SNE, although note we are using
un-normalized affinities rather than the probabilities $p_{ij}$.

The original EE paper uses a global $\beta$ for all $i$ so the input 
weights are symmetric. The later 
[spectral embedding](https://arxiv.org/abs/1206.4646) paper uses
the SNE affinities by perplexity calibration and symmetrizes (but does not
normalize) the resulting affinities. For the implementation in `smallvis` we use
the perplexity calibration approach, so we will proceed with the assumption that
the weights are always symmetric.

### Negative weights

The negative weights are the raw Euclidean input distances:

$$
v_{ij}^{-} = r_{ij}^{2}
$$

In one example using the Swiss Roll dataset, the original EE paper sets a 
uniform negative weight, $v_{ij}^{-} = 1$. This is also used by 
[Yang and co-workers](http://jmlr.org/proceedings/papers/v32/yange14.html) when
analyzing the connection between Elastic Embedding and SSNE.

### Homotopy parameter, $\lambda$

The weighting of attractive to repulsive forces is a free parameter, $\lambda$ 
in EE (a similar parameter is used in LargeVis also). The topological term 
"homotopy" is applied to this parameter. Values of `1`, `10` and 
`100` have been used. The spectral directions paper looked at both the COIL-20
and a 20,000-digit sample of MNIST and used $\lambda = 100$ for both, so we'll
try that here too.

A "homotopy method" is also described which means starting the optimization at a
small (effectively 0) value of $\lambda$, and increasing it to the target value
over the course of the optimization. In the spectral embedding paper, the
initial value is `1e-4`, with fifty log-spaced increments. As up to 10,000
iterations were allowed at each value of $\lambda$, this seems like a major
time investment. This approach seems similar in spirit to the approach advocated
in the original [NeRV](http://www.jmlr.org/papers/v11/venna10a.html) paper,
and the [multi-scale JSE](https://dx.doi.org/10.1016/j.neucom.2014.12.095) paper,
which both suggest scaling the input probabilities over the course of the
optimization to get better results.

`smallvis` does not currently support the homotopy method.

### Gradient

If we define output weights as we would in SSNE:

$$
w_{ij} = \exp\left(-d_{ij}^2\right)
$$

then we can rewrite the EE cost function in terms of the weights as:

$$
C = 
-\sum_{ij} v_{ij}^{+} \log w_{ij} + 
\lambda \sum_{ij} v_{ij}^{-} w_{ij}
$$

and the gradient is:

$$
\frac{\partial C_{EE}}{\partial \mathbf{y_i}} = 
4\sum_j^N \left( v_{ij}^{+} - \lambda v_{ij}^{-}{w_{ij}} \right) 
\left(\mathbf{y_i - y_j}\right) 
$$


## Datasets

See the [Datasets](https://jlmelville.github.io/smallvis/datasets.html) page.

The Swiss Roll dataset, last seen giving the 
[distance-based methods](https://jlmelville.github.io/smallvis/mmds.html) a
workout, also makes its return here, having been used in the EE paper. To be
like the EE paper, we'll use 2000 points in the example here:

```
sr2k <- snedata::swiss_roll(n = 2000)
```

## Settings

Typical settings used for the results here are:

```
iris_ee_100 <- smallvis(iris, perplexity = 40, scale = FALSE, Y_init = "spca", method = list("ee", lambda = 100), ret_extra = c("dx", "dy"), eta = 100)
```

For `lambda = 1000`, the learning rate had to be reduced to `eta = 10` for 
`iris` and `oli`. This didn't have much effect on the results for the other
datasets.

For the Swiss Roll, it's not possible to get exactly the same results
as with the EE paper, because that uses a k-nearest neighbor Gaussian kernel
with $k = 12$ and a fixed bandwidth $\sigma = 15$. For our experiments, we
calibrate the affinities to give a perplexity of `12`, which necessitates 
varying bandwidths. The use of $k = 12$ with GMMDS unrolls this dataset without
disconnections, so the equivalent perplexity should be in the right ballpark.

```
sr2k_ee_100 <- smallvis(sr2k, perplexity = 12, scale = FALSE, Y_init = "spca", method = list("ee", lambda = 100), ret_extra = c("dx", "dy"), eta = 100)
```

### iris

|                             |                           |
:----------------------------:|:--------------------------:
![iris EE (1)](../img/ee/iris_1.png)|![iris EE (10)](../img/ee/iris_10.png)
![iris EE (100)](../img/ee/iris_100.png)|![iris EE (1000)](../img/ee/iris_1000.png)


### s1k

|                             |                           |
:----------------------------:|:--------------------------:
![s1k EE (1)](../img/ee/s1k_1.png)|![s1k EE (10)](../img/ee/s1k_10.png)
![s1k EE (100)](../img/ee/s1k_100.png)|![s1k EE (1000)](../img/ee/s1k_1000.png)

### oli

|                             |                           |
:----------------------------:|:--------------------------:
![oli EE (1)](../img/ee/oli_1.png)|![oli EE (10)](../img/ee/oli_10.png)
![oli EE (100)](../img/ee/oli_100.png)|![oli EE (1000)](../img/ee/oli_1000.png)

### frey

|                             |                           |
:----------------------------:|:--------------------------:
![frey EE (1)](../img/ee/frey_1.png)|![frey EE (10)](../img/ee/frey_10.png)
![frey EE (100)](../img/ee/frey_100.png)|![frey EE (1000)](../img/ee/frey_1000.png)

### coil20

|                             |                           |
:----------------------------:|:--------------------------:
![coil20 EE (1)](../img/ee/coil20_1.png)|![coil20 EE (10)](../img/ee/coil20_10.png)
![coil20 EE (100)](../img/ee/coil20_100.png)|![coil20 EE (1000)](../img/ee/coil20_1000.png)

### mnist

|                             |                           |
:----------------------------:|:--------------------------:
![mnist EE (1)](../img/ee/mnist_1.png)|![mnist EE (10)](../img/ee/mnist_10.png)
![mnist EE (100)](../img/ee/mnist_100.png)|![mnist EE (1000)](../img/ee/mnist_1000.png)

### fashion

|                             |                           |
:----------------------------:|:--------------------------:
![fashion EE (1)](../img/ee/fashion_1.png)|![fashion EE (10)](../img/ee/fashion_10.png)
![fashion EE (100)](../img/ee/fashion_100.png)|![fashion EE (1000)](../img/ee/fashion_1000.png)

### swiss roll

|                             |                           |
:----------------------------:|:--------------------------:
![sr2k EE (1)](../img/ee/sr2k_1.png)|![sr2k EE (10)](../img/ee/sr2k_10.png)
![sr2k EE (100)](../img/ee/sr2k_100.png)|![sr2k EE (1000)](../img/ee/sr2k_1000.png)

## Conclusions

As noted by the EE paper itself, depending on the choice of `lambda`, EE gives
results that seem to be equivalent to SSNE, without requiring any normalization.
The default `lambda = 100` seems like a good choice. You can also use the 
default delta-bar-delta optimization settings with this setting.

It would be nice to get results as good as t-SNE, but as far as I can tell, 
no-one has reported any results with a t-distributed version of EE. The advent
of methods like LargeVis and UMAP may have filled that niche now.

The Swiss Roll results look a bit disappointing. Probably allowing for a (much)
longer optimization would help unroll it a bit more. SSNE on the Swiss Roll also 
shows a similar slow unrolling. I suspect there is an effect due to the 
calibrated bandwidths not being used in the output kernel.

The results in the EE paper *do* show full unrolling, but do so by not including
negative input weights in the repulsion and by using a fixed bandwidth for all
positive weights (*and* by setting the weights to zero outside of the 12 nearest
neighbors). I suspect the largest effect is by dropping the negative weights,
but this will also have an effect on a good value of `lambda` and initial 
learning rate. In my initial experiments, I was unable to find a satisfactory
combination of settings.

This document will be updated if either the "polytope method" or the removal
of negative weights are implemented.