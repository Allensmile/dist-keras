[![DOI](https://zenodo.org/badge/64122944.svg)](https://zenodo.org/badge/latestdoi/64122944)

# Distributed Keras

Distributed Deep Learning with Apache Spark and Keras.

## Introduction

Distributed Keras is a distributed deep learning framework built on top of Apache Spark and Keras. We designed the framework in such a way that a developer could implement a new distributed optimizer with ease, thus enabling a person to focus on research. Several distributed methods are supported, such as, but not restricted to, the training of **ensemble models**, and **data parallel** models.

As discussed above, most methods are implemented as data parallel models. Data parallel models, as described in [[3]](http://papers.nips.cc/paper/4687-large-scale-distributed-deep-networks.pdf), is a learning paradigm where multiple replicas of a model are used to optimize a single objective. Using data parallelism, we are able to significantly increase the training time of the model. Depending on the parametrization, we are able to achieve better model performance compared to a more common training approach (e.g., like the [SingleTrainer](#single-trainer) implementation), and yet, spending less time on the training of the model.

**Attention**: We recommend reading the [workflow](https://github.com/JoeriHermans/dist-keras/blob/master/examples/workflow.ipynb) notebook. This includes a complete description of the problem, how to use it, preprocess your data with Apache Spark, and a performance evaluation of all included distributed optimizers.

## Installation

We will guide you how to install Distributed Keras. However, we will assume that an Apache Spark installation is available.

### pip

When you only require the framework, just use `pip` to install dist-keras.

```bash
pip install git+https://github.com/JoeriHermans/dist-keras.git
```
### git

Using this approach, you will be able to easily execute the examples.

```bash
git clone https://github.com/JoeriHermans/dist-keras
```

In order to install possible missing dependencies, and to compile the dist-keras modules, we need to run `pip`.

```bash
cd dist-keras
pip install -e .
```

### General notes

#### .bashrc

Make sure the following variables are set in your `.bashrc`. It is possible, depending on your system configuration, that the following configuration **doesn't have to be applied**.

```bash
# Example of a .bashrc configuration.
export SPARK_HOME=/usr/lib/spark
export PYTHONPATH="$SPARK_HOME/python/:$SPARK_HOME/python/lib/py4j-0.9-src.zip:$PYTHONPATH"
```

## Running an example

We would like to refer the reader to the `workflow.ipynb` notebook in the examples folder. This will you a complete introduction to the problem of distributed deep learning, and will guide you through the steps that have to be executed.

### Spark 2.0

If you want to run the examples using Apache Spark 2.0.0 and higher. You will need to remove the line containing `sqlContext = SQLContext(sc)`. We need to do this because in Spark 2.0+, the SQLContext, and Hive context are now merged in the Spark session.

## Algorithms

### Single Trainer

A single trainer is in all simplicity a trainer which will use a single Spark thread to train a model. This trainer is usually used as a baseline metrics for new distributed optimizers. However, one could also use this trainer if the dataset is too big to fit in memory.

```python
SingleTrainer(keras_model, worker_optimizer, loss, features_col="features",
              label_col="label", num_epoch=1, batch_size=32)
```

### Asynchronous EASGD (currently recommended)

The distinctive idea of EASGD is to allow the local workers to perform more exploration (small rho) and the master to perform exploitation. This approach differs from other settings explored in the literature, and focus on how fast the center variable converges [[1]](https://arxiv.org/pdf/1412.6651.pdf) .

In this section we propose the asynchronous version of [EASGD](#easgd). Instead of waiting on the synchronization of other trainers, this method communicates the elastic difference (as described in the paper), with the parameter server. The only synchronization mechanism that has been implemented, is to ensure no race-conditions occur when updating the center variable.

```python
AEASGD(keras_model, worker_optimizer, loss, num_workers=2, batch_size=32,
       features_col="features", label_col="label", num_epoch=1, communication_window=32,
       rho=5.0, learning_rate=0.01)
```

### Asynchronous EAMSGD

Asynchronous EAMSGD is a variant of asynchronous EASGD. It is based on the Nesterov's momentum scheme, where the update of the local worker is modified to incorepare a momentum term.

```python
EAMSGD(keras_model, worker_optimizer, loss, num_workers=2, batch_size=32,
       features_col="features", label_col="label", num_epoch=1,  communication_window=32,
       rho=5.0, learning_rate=0.01, momentum=0.9)
```

### DOWNPOUR

An asynchronous stochastic gradient descent procedure supporting a large number of model replicas and leverages adaptive learning rates. This implementation is based on the pseudocode provided by [1] .

```python
DOWNPOUR(keras_model, worker_optimizer, loss, num_workers=2, batch_size=32,
         features_col="features", label_col="label", num_epoch=1, learning_rate=0.01,
         communication_window=3)
```

## Utility classes

### Transformers

A transformer is a utility class which takes a collection of columns (or just a single column), and produces an additional column which is added to the resulting DataFrame. An example of such a Transformer is our `LabelIndexTransformer`.

### Predictors

Predictors are utility classes which addsa prediction column to the DataFrame given a specified Keras model and its input features.

### Evaluators

As the name suggest, an evaluator will evaluate the statistical performance of a Keras model on a dataset. Of course, one could use the MLlib evaluation metrics, but the availability of some metrics in different Spark versions might be limited. For this reason, we implemented our own evaluation functions.

## Known issues

### Synchronous algorithms

1. It is possible, depending on your `batch_size` and the size of your dataset, that there will be an unequal amount of batches distributed over the different worker partitions. Imagine having `n - 1` workers with `k` batches, and a worker with `k - 1` batches. In this case, the `n - 1` workers all wish to process batch `k`, however, the other worker (which has only `k - 1` batches) has already finished its process. This results in endless "waiting" behaviour, i.e., the `n - 1` workers are waiting for the last worker to publish its gradient. But this worker can't send its gradient of batch `k` to the parameter server since it does not have a batch `k`.

    **Possible solutions:**

    a. Make sure that every partition has an equal amount of batches. For example, using a custom partitioner.

    b. Modify the optimizer in such a way that it does allow for timeouts from a worker.

## TODO's

This list below is of all the features that still have to be implemented in order to have a production-ready environment.

- Spark SparseVector and Numpy compatibility.
- Improve efficiency of EASGD synchronization.
- Compression / decompression of network transmissions.
- Monitoring of loss and training.
- Stop on target loss.
- Multiple parameter servers for large Deep Networks.

## Citing

If you use this software in your work, please cite us.

```
@misc{distributed_keras,
  author       = {Joeri Hermans},
  title        = {Distributed Keras: Distributed Deep Learning with Apache Spark and Keras},
  month        = nov,
  year         = 2016,
  doi          = {10.5281/zenodo.167573},
  url          = {https://doi.org/10.5281/zenodo.167573}
}
```

## References

* Zhang, S., Choromanska, A. E., & LeCun, Y. (2015). Deep learning with elastic averaging SGD. In Advances in Neural Information Processing Systems (pp. 685-693). [1]

* Moritz, P., Nishihara, R., Stoica, I., & Jordan, M. I. (2015). SparkNet: Training Deep Networks in Spark. arXiv preprint arXiv:1511.06051. [2]

* Dean, J., Corrado, G., Monga, R., Chen, K., Devin, M., Mao, M., ... & Ng, A. Y. (2012). Large scale distributed deep networks. In Advances in neural information processing systems (pp. 1223-1231). [3]

<!-- @misc{pumperla2015, -->
<!-- author = {Max Pumperla}, -->
<!-- title = {elephas}, -->
<!-- year = {2015}, -->
<!-- publisher = {GitHub}, -->
<!-- journal = {GitHub repository}, -->
<!-- howpublished = {\url{https://github.com/maxpumperla/elephas}} -->
<!-- } -->
* Pumperla, M. (2015). Elephas. Github Repository https://github.com/maxpumperla/elephas/. [4]

## Licensing

![GPLv3](resources/gpl_v3.png) ![CERN](resources/cern_logo.jpg)
