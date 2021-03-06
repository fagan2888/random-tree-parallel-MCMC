# PART: Parallelizing MCMC with Random Partition Trees

MATLAB implementation of **PART**<sup>[1](#myfootnote1)</sup>: a fast algorithm for aggregating MCMC sub-chain samples

[TOC]

## Introduction

The modern scale of data has brought new challenges to Bayesian inference. In particular, conventional MCMC algorithms are computationally very expensive for large data sets. A promising approach to solve this problem is **embarrassingly parallel MCMC** (EP-MCMC), which first partitions the data into multiple subsets and runs independent sampling algorithms on each subset. The subset posterior draws are then aggregated via some combining rules to obtain the final approximation. 

**PART**<sup>[1](#myfootnote1)</sup> is an EP-MCMC algorithm that applies random partition tree to combine the subset posterior draws, which is distribution-free, easy to resample from and can adapt to multiple scales. This repository maintains a MATLAB implementation of PART. 

## QuickStart

Running parallel MCMC with PART consists of the following steps. 

1. Partition the data into M subsets. 
   
   For example, supposing there are 1,000 data points, one can partition them into M=10 subsets, with each subset containing 100 data points. Partitioning does not have to be equal-sized. 
   
2. Run MCMC on each subset in parallel with your favorite sampler.
   
   For example, you have p=4 parameters in the model and for each subset i (i=1,2,….,10), you draw 5,000 posterior **sub-chain samples**, resulting in 10 matrices of size 5,000 x 4.
   
   >  **Note**: PART currently supports real-valued samples only.
   
3. Call PART to aggregate **sub-chain samples** into **posterior samples**, as if the posterior samples come from running MCMC on the whole dataset. 
   
   With the running example, do the following in MATLAB.
   
   1. Run `init` under the `src/` to add paths. 
      
   2. Then, store all the sub-chain samples into a cell `sub_chain = cell(1, 10)` with `sub_chain{i}` being a 5,000 x 4 matrix corresponding to MCMC samples on subset `i`. 
      
   3. Now, call PART to aggregate samples. The following code draws 10,000 samples from the combined posterior with **PART-KD** algorithm running **pairwise** aggregation. `combined_posterior_kd_pairwise` will be a 10,000 x 4 matrix.
      
      ``` octave
      options = part_options('cut_type', 'kd', 'resample_N', 10000);
      combined_posterior_kd_pairwise = aggregate_PART_pairwise(sub_chain, options);
      ```
      
      > **Before running the code above**: run `src/init.m` to make sure paths are included.

We strongly recommend reading and running the demos under the `examples/` directory.

### Aggregation Schemes

We provide two schemes for aggregation: **pairwise** (recommended) and **one-stage**. 

1. Calling `combined_posterior = aggregate_PART_pairwise(sub_chain, options)` will aggregate subset samples recursively by pairs. For examples, for 4 sub-chains, it will first aggregate 1 and 2 into (1+2) and 3 and 4 into (3+4), then aggregate (1+2) and (3+4) into the final samples. 
   
   Set `options.match=true` (enabled by default) will match subsets into better pairs.
   
2. Calling `combined_posterior = aggregate_PART_onestage(sub_chain, options)` will aggregate all subsets at once. This is not recommended when the number of subsets is large.

### Options

The parameters of PART are set by the `options` structure, which is configured by 

``` 
options = part_options('Option_1', Value_1, 'Option_2', Value_2, ...)
```

in terms of *(Name, Value)* pairs. `part_options(…)` accepts the following configurations.

1. `ntree`: number of trees (default = 16)
2. `resample_N`: number of samples drawn from the combined posterior (default = 10,000)
3. `parallel`: if `true`, building trees in parallel with MATLAB Parallel Computing Toolbox (default = true)
4. `cut_type`: partition rule<sup>[1](#myfootnote1)</sup>, currently supporting `kd` (median cut, default) and `ml` (maximum likelihood cut). `kd` can be significantly faster than `ml` for large sample size.
5. `min_cut_length`: stopping rule, minimum side length of a block, corresponding to $\delta_a$ in the paper<sup>[1](#myfootnote1)</sup> (default = 0.001). It is recommended to adjust the scale of the samples such that `min_cut_length` is applicable to all dimensions. Alternatively, one can manually set `min_cut_length` to a vector of p, corresponding to each dimension of the parameter. 
6. `min_fraction_block`: stopping rule, corresponding to $\delta_{\rho}$ in the paper<sup>[1](#myfootnote1)</sup>, such that the leaf nodes should contain no smaller a fraction of samples than this number. So roughly, this is the minimum probability of a block. A number in (0,1) is required (default = 0.01). 
7. `local_gaussian_smoothing`: if `true`, local Gaussian smoothing<sup>[1](#myfootnote1)</sup> is applied to the block-wise densities. (default = `true`)
8. `match`: if `true`, doing better pairwise matching in `aggregate_PART_pairwise(…)`. (default = `true`)

Please refer to `help part_options` for more options. 

## Evaluation and Visualization Tools

We have implemented the following functions in `src/utils` for evaluation and visualization. 

1. `plot_marginal_compare({chain_1, chain_2, ...}, {name_1, name_2, ...}, ...)` plots the marginals of several samples. 
2. `plot_pairwise_compare({chain_1, chain_2, ...}, {name_1, name_2, ...}, ...)` plots the bivariate distribution for several samples. 
3. `approximate_KL(samples_left, samples_right)` computes the approximate KL divergence KL(left||right) between two samples. The KL is computed from Laplacian approximations fitted to both distributions.
4. `performance_table({chain_1, chain_2, ...}, {'chain_1', chain_2', ...}, full_posterior, theta)` outputs a table comparing the approximation accuracy of each chain to `full_posterior`. `theta` is a point estimator to the parameters sampled.

## Other Algorithms

We also implemented several other popular aggregation algorithms under `src/alternatives`. 

1. `aggregate_average(sub_chain)` aggregates by simple averaging. 
2. `aggregate_weighted_average(sub_chain)` aggregates by weighted averaging, with weights optimal for Gaussian distributions. 
3. `aggregate_uai_parametric(sub_chain, n)` outputs n samples by drawing from multiplied Laplacian approximation to subset posteriors<sup>[2](#myfootnote2)</sup>.
4. `aggregate_uai_nonparametric(sub_chain, n)` draws n samples from multiplied kernel density estimation to subchain posteriors<sup>[2](#myfootnote2)</sup>.
5. `aggregate_uai_semiparametric(sub_chain, n)` draws n samples from multiplied semi-parametric estimation to subchain posteriors<sup>[2](#myfootnote2)</sup>.

## Inner Working

### Files

``` 
├── README.md                           readme      
├── docs
│   └── tree15.pdf                      PART paper
├── examples/                           QuickStart examples
│   ├── dirichrnd.m                     sample from Dirichlet
│   ├── sample_mog.m                    Gibbs sampling for mean of mixture of Gaussian
│   ├── test_bimodal.m                  demo - bimodal
│   ├── test_mixture_of_gaussian.m      demo - mixture of Gaussian
│   └── test_rare_binary.m              demo - rare Bernoulli
└── src
    ├── PART                            implementation of PART
    │   ├── MultiStageMCMC.m            pairwise aggregation
    │   ├── NormalizeForest.m           normalize trees in a forest
    │   ├── NormalizeTree.m             flatten and normalize a tree
    │   ├── OneStageMCMC.m              one-stage aggregation
    │   ├── aggregate_PART_onestage.m   one-stage aggregation, used with part_options
    │   ├── aggregate_PART_pairwise.m   pairwise aggregation, used with part_options
    │   ├── buildForest.m               build a forest
    │   ├── buildHistogram.m            generic 1-D histogram building with a supplied cutting function
    │   ├── buildTree.m                 build a partition tree
    │   ├── cleanTree.m                 cleans a given forest of random partition trees
    │   ├── copyTree.m                  a deepCopy function that copies a forest of random partition 
    │   ├── cut                         various "partition rules"
    │   │   ├── consensusMLCut.m        ML (maximum likelihood)
    │   │   ├── fastMLCut.m             faster ML with stochastic approximation
    │   │   ├── kdCut.m                 KD/median
    │   │   ├── meanCut.m               average
    │   │   └── midPointCut.m           mid-point
    │   ├── empiricalLogLikelihood.m    evaluates the empirlcal log likelihood of x under cuts
    │   ├── part_options.m              configure an option for running PART algorithm
    │   ├── treeDensity.m               evaluate the combined density for a given point
    │   ├── treeNode.m                  tree structure
    │   ├── treeNormalize.m             normalizer
    │   └── treeSampling.m              resample from a tree density estimator
    ├── alternatives/                   a few other EP-MCMC algorithms
    ├── init.m                          add paths
    └── utils/                          a few evaluation and visualization functions
```

### Key Functions

We give a brief introduction to the following key functions in PART MATLAB implementation.

#### Low level

1. `[obj, nodes, log_probs] = buildTree(...)` builds a single tree by first pooling all the data across subsets so that the tree structure is determined. Then unnormalized probability and corresponding density are estimated by block-wise multiplication. `obj` is the root tree node; `nodes` are the leaf nodes that each corresponds to a block in the sample space;`log_probs` are **unnormalized** probabilities of those leaf nodes in logarithm, in correspondence to `nodes`.
2. `[tree, sampler, sampler_prob] = NormalizeTree(...)` performs (1) recomputing or (2) normalization of a tree by summerming over leaf nodes.

#### Medium level

1. `[trees, sampler, sampler_prob] = buildForest(RawMCMC, display_info)` builds a random ensemble of trees by calling `buildTree(...)` in parallel. Each tree is also normalized with `NormalizeTree(...)`.

#### High level

1. `[trees, sampler, sampler_prob, RawMCMC] = OneStageMCMC(MCdraws, option)` is a wrapper of `buildForest(...)`. The resulting ensemble consists of multiple trees, with each tree combines the posterior across subsets with block-wise direct multiplication.
2. `[trees, sampler, sampler_prob] = MultiStageMCMC(MCdraws, option)` does pairwise combine for multiple layers until converging into one posterior. Two subsets A and B are matched with some criteria, `OneStageMCMC(...)` is called to combine them into A+B. Samples are drawn from (A+B) and (C+D) respectively to represent the distributions after 1st stage combination. Then, (A+B) and (C+D) are combined into (A+B+C+D) by again calling`OneStageMCMC(...)` on their samples.
3. `aggregated_samples = aggregate_PART_pairwise(sub_chain, options)` is a high level wrapper of `MultiStageMCMC(...)` that draws samples from pairwise aggregated sub-chains. 
4. `aggregated_samples = aggregate_PART_onestage(sub_chain, options)` is a high level wrapper of `OneStageMCMC(…)` that draws samples from one-stage aggregated sub-chains.

### References

<a name="myfootnote1">1</a>: Xiangyu Wang, Fangjian Guo, Katherine A. Heller and David B. Dunson. Parallelizing MCMC with random partition trees. NIPS 2015. <http://arxiv.org/abs/1506.03164>

<a name="myfootnote2">2</a>: W Neiswanger, C Wang, E Xing. Asymptotically Exact, Embarrassingly Parallel MCMC.  UAI 2014.

## Authors

[Richard Guo](https://github.com/richardkwo) and [Samuel Wang](https://github.com/wwrechard). We would appreciate your feedback under [*issues*](https://github.com/richardkwo/random-tree-parallel-MCMC/issues) of this repository. 

## License

© Contributors, 2015. Released under the [MIT](https://github.com/richardkwo/random-tree-parallel-MCMC/blob/master/LICENSE) license.