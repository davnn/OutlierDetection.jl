# OutlierDetection.jl

[![Chat](https://img.shields.io/badge/slack-julialang/outlierdetection-blue.svg?logo=slack)](https://julialang.slack.com/archives/C02EXTD7WGG)
[![Documentation (dev)](https://img.shields.io/badge/docs-dev-blue.svg)](https://OutlierDetectionJL.github.io/OutlierDetection.jl/dev)
[![Build Status](https://github.com/OutlierDetectionJL/OutlierDetection.jl/workflows/CI/badge.svg)](https://github.com/OutlierDetectionJL/OutlierDetection.jl/actions)
[![Coverage](https://codecov.io/gh/OutlierDetectionJL/OutlierDetection.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/OutlierDetectionJL/OutlierDetection.jl) <!-- ALL-CONTRIBUTORS-BADGE:START - Do not remove or modify this section -->
[![All Contributors](https://img.shields.io/badge/all_contributors-6-orange.svg?style=flat-square)](#contributors-)
<!-- ALL-CONTRIBUTORS-BADGE:END -->

*OutlierDetection.jl* is a Julia toolkit for detecting outlying objects, also known as *anomalies*. This package is an effort to make Julia a first-class citizen in the Outlier- and Anomaly-Detection community. *Why should you use this package?*

- Provides a unified API for outlier detection in Julia
- Provides access to state-of-the-art outlier detection algorithms
- Seamlessly integrates with Julia's existing machine learning ecosystem

**Citing**

If you use *OutlierDetection.jl* in a scientific publication, we appreciate citations to:

```
@article{muhr2022outlierdetection,
  title={OutlierDetection.jl: A modular outlier detection ecosystem for the Julia programming language},
  author={Muhr, David and Affenzeller, Michael and Blaom, Anthony D},
  journal={arXiv preprint arXiv:2211.04550},
  year={2022}
}
```

or

```
Muhr, David, Michael Affenzeller, and Anthony D. Blaom. "OutlierDetection.jl: A modular outlier detection ecosystem for the Julia programming language." arXiv preprint arXiv:2211.04550 (2022).
```

## Installation

It is recommended to use [Pkg.jl](https://julialang.github.io/Pkg.jl) for installation. Follow the command below to install the latest official release or use `] add OutlierDetection` in the Julia REPL.

```julia
import Pkg
Pkg.add("OutlierDetection")
```

If you would like to modify the package locally, you can use `Pkg.develop("OutlierDetection")` or `] dev OutlierDetection` in the Julia REPL. This fetches a full clone of the package to `~/.julia/dev/` (the path can be changed by setting the environment variable `JULIA_PKG_DEVDIR`).

## Usage

 *OutlierDetection.jl* is built on top of [MLJ](https://github.com/alan-turing-institute/MLJ.jl) and provides many `Detector` implementations for MLJ. A `Detector` simply assigns a real-valued score to each sample, which is defined to be increasing with increasing outlierness. The detectors live in sub-packages of [OutlierDetectionJL](https://github.com/OutlierDetectionJL/), e.g. [OutlierDetectionNeighbors](https://github.com/OutlierDetectionJL/OutlierDetectionNeighbors.jl),and can be loaded directly with MLJ, as shown below.

```julia
using MLJ
using OutlierDetection
using OutlierDetectionData: ODDS

# download and open the thyroid benchmark dataset
X, y = ODDS.load("thyroid")

# use 50% of the data for training
train, test = partition(eachindex(y), 0.5, shuffle=true)

# load the detector
KNN = @iload KNNDetector pkg=OutlierDetectionNeighbors

# instantiate a detector with default parameters, returning scores
knn = KNN()

# bind the detector to data and learn a model with all data
knn_raw = machine(knn, X) |> fit!

# transform data to raw outlier scores based on the test data; note that there
# is no `predict` defined for raw detectors
transform(knn_raw, rows=test)

# OutlierDetection.jl provides helper functions to normalize the scores,
# for example using min-max scaling based on the training scores
knn_probas = machine(ProbabilisticDetector(knn), X) |> fit!

# predict outlier probabilities based on the test data
predict(knn_probas, rows=test)

# OutlierDetection.jl also provides helper functions to turn scores into classes,
# for example by imposing a threshold based on the training data percentiles
knn_classifier = machine(DeterministicDetector(knn), X) |> fit!

# predict outlier classes based on the test data
predict(knn_classifier, rows=test)
```

It is also possible to use *OutlierDetection.jl* without MLJ, however, note that more explicit steps are necessary.

```julia
using OutlierDetection: fit, transform, scale_minmax, classify_quantile, outlier_fraction
using OutlierDetectionNeighbors: KNNDetector # explicitly import detector
using OutlierDetectionData: ODDS

X, y = ODDS.load("thyroid")
knn = KNNDetector()

# explicit conversion to a native array is necessary
# note that we are using the transposed data, because column-major data is expected
Xmatrix = Matrix(X)'

# explicit fit result and training scores
model, scores_train = fit(knn, Xmatrix[:, 11:end]; verbosity = 0)

# transform the first 10 points to scores (not used for training)
scores_test = transform(knn, model, Xmatrix[:, 1:10])

# explicitly normalize train and test scores
proba_train, proba_test = scale_minmax((scores_train, scores_test))

# explicitly convert scores to labels (> 95th percentile would be an outlier)
labels_train, labels_test = classify_quantile(0.95)((scores_train, scores_test))
```

## Algorithms (also known as Detectors)

Algorithms marked with '✓' are implemented in Julia. Algorithms marked with '✓ (py)' are implemented in Python (thanks to the wonderful [PyOD library](https://github.com/yzhao062/pyod)) with an existing Julia interface through [PyCall](https://github.com/JuliaPy/PyCall.jl). If you would like to know more, open the [detector reference](https://OutlierDetectionJL.github.io/OutlierDetection.jl/dev/API/detectors/).

| Name    | Description                                  | Year  | Status | Authors                |
| ------- | -------------------------------------------- | :---: | :----: | ---------------------- |
| CD      | Cook's distance                              | 1977  | ✓ (py) | Dennis R. Cook         |
| LMDD    | Linear deviation-based outlier detection     | 1996  | ✓ (py) | Arning et al.          |
| KNN     | Distance-based outliers                      | 1997  |   ✓    | Knorr and Ng           |
| MCD     | Minimum covariance determinant               | 1999  | ✓ (py) | Rousseeuw and Driessen |
| KNN     | Distance to the k-th nearest neighbor        | 2000  |   ✓    | Ramaswamy              |
| LOF     | Local outlier factor                         | 2000  |   ✓    | Breunig et al.         |
| OCSVM   | One-Class support vector machine             | 2001  | ✓ (py) | Schölkopf et al.       |
| KNN     | Sum of distances to the k-nearest neighbors  | 2002  |   ✓    | Angiulli and Pizzuti   |
| COF     | Connectivity-based outlier factor            | 2002  |   ✓    | Tang et al.            |
| LOCI    | Local correlation integral                   | 2003  | ✓ (py) | Papadimitirou et al.   |
| CBLOF   | Cluster-based local outliers                 | 2003  | ✓ (py) | He et al.              |
| PCA     | Principal component analysis                 | 2003  | ✓ (py) | Shyu et al.            |
| KDE     | Kernel Density Estimation                    | 2007  | ✓ (py) | Latecki et al.         |
| IForest | Isolation forest                             | 2008  | ✓ (py) | Liu et al.             |
| ABOD    | Angle-based outlier detection                | 2009  |   ✓    | Kriegel et al.         |
| SOD     | Subspace outlier detection                   | 2009  | ✓ (py) | Kriegel et al.         |
| HBOS    | Histogram-based outlier score                | 2012  | ✓ (py) | Goldstein and Dengel   |
| SOS     | Stochastic outlier selection                 | 2012  | ✓ (py) | Janssens et al.        |
| AE      | Auto-encoder reconstruction loss outliers    | 2015  |   ✓    | Aggarwal               |
| ABOD    | Stable angle-based outlier detection         | 2015  |   ✓    | Li et al.              |
| GMM     | Gaussian Mixture Model                       | 2015  | ✓ (py) | Aggarwal and Sathe     |
| LODA    | Lightweight on-line detector of anomalies    | 2016  | ✓ (py) | Pevný                  |
| INNE    | Isolation-based nearest neighbors            | 2018  | ✓ (py) | Bandaragoda et al.     |
| DeepSAD | Deep semi-supervised anomaly detection       | 2019  |   ✓    | Ruff et al.            |
| COPOD   | Copula-based outlier detection               | 2020  | ✓ (py) | Li et al.              |
| ROD     | Rotation-based outlier detection             | 2020  | ✓ (py) | Almardeny et al.       |
| ESAD    | End-to-end semi-supervised anomaly detection | 2020  |   ✓    | Huang et al.           |
| ECOD    | Empirical Cumulative Distribution Functions  | 2022  | ✓ (py) | Li et al.              |

If there are already so many algorithms available in Python - *why Julia, you might ask?* Let's have some fun!

```julia
using OutlierDetection, MLJ
using BenchmarkTools: @benchmark
X = rand(10, 100000)
LOF =  @iload LOFDetector pkg=OutlierDetectionNeighbors
PyLOF =  @iload LOFDetector pkg=OutlierDetectionPython
lof = machine(LOF(k=5, algorithm=:kdtree, leafsize=30, parallel=true), X) |> fit!
pylof = machine(PyLOF(n_neighbors=5, algorithm="kd_tree", leaf_size=30, n_jobs=-1), X) |> fit!
```

Julia enables you to implement your favorite algorithm in no time, and it will be fast, *blazingly fast*.

```julia
@benchmark transform(lof, X)
> median time:      341.464 ms (0.00% GC)
```

Interoperating with Python is easy!

```julia
@benchmark transform(pylof, X)
> median time:      7.934 s (0.00% GC)
```

## Contributing

OutlierDetection.jl is a community effort and your help is extremely welcome! See our [contribution guide](https://OutlierDetectionJL.github.io/OutlierDetection.jl/dev/documentation/contributing/) for more information how to contribute to the project.

## Contributors ✨

Thanks go to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="http://fastpaced.com"><img src="https://avatars.githubusercontent.com/u/1233304?v=4?s=100" width="100px;" alt="David Muhr"/><br /><sub><b>David Muhr</b></sub></a><br /><a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=davnn" title="Code">💻</a> <a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=davnn" title="Tests">⚠️</a> <a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=davnn" title="Documentation">📖</a> <a href="#maintenance-davnn" title="Maintenance">🚧</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/PallHaraldsson"><img src="https://avatars.githubusercontent.com/u/8005416?v=4?s=100" width="100px;" alt="Páll Haraldsson"/><br /><sub><b>Páll Haraldsson</b></sub></a><br /><a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=PallHaraldsson" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://ablaom.github.io"><img src="https://avatars.githubusercontent.com/u/30517088?v=4?s=100" width="100px;" alt="Anthony Blaom, PhD"/><br /><sub><b>Anthony Blaom, PhD</b></sub></a><br /><a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=ablaom" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/pitmonticone"><img src="https://avatars.githubusercontent.com/u/38562595?v=4?s=100" width="100px;" alt="Pietro Monticone"/><br /><sub><b>Pietro Monticone</b></sub></a><br /><a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=pitmonticone" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/rolling-robot"><img src="https://avatars.githubusercontent.com/u/18036097?v=4?s=100" width="100px;" alt="Petr Mukhachev"/><br /><sub><b>Petr Mukhachev</b></sub></a><br /><a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=rolling-robot" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/tylerjthomas9"><img src="https://avatars.githubusercontent.com/u/36181311?v=4?s=100" width="100px;" alt="Tyler Thomas"/><br /><sub><b>Tyler Thomas</b></sub></a><br /><a href="https://github.com/OutlierDetectionJL/OutlierDetection.jl/commits?author=tylerjthomas9" title="Code">💻</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind welcome!
