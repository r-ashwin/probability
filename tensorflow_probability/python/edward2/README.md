# Upgrading from Edward to Edward2

This guide outlines how to port code from the
[Edward](http://edwardlib.org/)
probabilistic programming system to
[Edward2](https://github.com/tensorflow/probability/tree/master/tensorflow_probability/python/edward2),
a probabilistic programming language available in
[TensorFlow Probability](https://github.com/tensorflow/probability).
We recommend Edward users use Edward2 for specifying models and other TensorFlow
Probability primitives for performing downstream computation.

Edward2 is a distillation of Edward. It is a low-level language for specifying
probabilistic models as programs and manipulating their computation.
Probabilistic inference, criticism, and any other part of the scientific process
(Box, 1976) use arbitrary TensorFlow ops. Their associated abstractions live in
the TensorFlow ecosystem such as in TensorFlow Probability, and do not strictly
require Edward2.

We're in the process of porting all Edward features, examples, and tutorials
to TensorFlow Probability. For current examples:

+ Probabilistic PCA
  ([Edward](https://github.com/blei-lab/edward/blob/master/notebooks/probabilistic_pca.ipynb),
  [TensorFlow Probability](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/examples/jupyter_notebooks/Probabilistic_PCA.ipynb))
+ Eight schools
  ([Edward](https://github.com/blei-lab/edward/blob/master/notebooks/eight_schools.ipynb),
  [TensorFlow Probability](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/examples/jupyter_notebooks/Eight_Schools.ipynb))
+ Linear mixed effects models
  ([Edward](https://github.com/blei-lab/edward/blob/master/notebooks/linear_mixed_effects_models.ipynb),
  [TensorFlow Probability](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/examples/jupyter_notebooks/Linear_Mixed_Effects_Models.ipynb))
+ Variational Autoencoder
  ([Edward](https://github.com/blei-lab/edward/blob/master/examples/vae_convolutional.py),
  [TensorFlow Probability](https://github.com/tensorflow/probability/tree/master/tensorflow_probability/examples/vae.py))
+ Mixture of Gaussians
  ([Edward](https://github.com/blei-lab/edward/blob/master/notebooks/unsupervised.ipynb),
  [TensorFlow Probability](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/examples/jupyter_notebooks/Bayesian_Gaussian_Mixture_Model.ipynb))
+ Logistic regression
  ([Edward](https://github.com/blei-lab/edward/blob/master/examples/bayesian_logistic_regression.py),
  [TensorFlow Probability](https://github.com/tensorflow/probability/tree/master/tensorflow_probability/examples/logistic_regression.py))

Are you having difficulties upgrading to Edward2? Raise a
[GitHub issue](https://github.com/tensorflow/probability/issues)
and we're happy to help. Alternatively, if you have tips, feel free to send a
pull request to improve this guide.

<!--
TODO(trandustin): Provide links when available.

See the
[TensorFlow blog post](TODO(trandustin))
for an outline of new features in Edward2. To learn more about TensorFlow
Probability overall, see
[TensorFlow Probability's guides](TODO(trandustin)).
-->

## Namespaces

__Edward__.

```python
import edward as ed
from edward.models import Empirical, Gamma, Poisson

dir(ed)
## ['criticisms',
##  'inferences',
##  'models',
##  'util',
##   ...,  # criticisms in global namespace for convenience
##   ...,  # inference algorithms in global namespace for convenience
##   ...]  # utility functions in global namespace for convenience
```

__Edward2 / TensorFlow Probability__.

```python
import tensorflow_probability as tfp
from tensorflow_probability import edward2 as ed

dir(ed)
## [...,  # random variables
##  'as_random_variable',  # various tools for manipulating program execution
##  'get_interceptor',
##  'interception',
##  'make_log_joint_fn']
```

## Probabilistic Models

__Edward__. You write models inline with any other code, composing
random variables. As illustration, consider a
[deep exponential family](https://github.com/blei-lab/edward/blob/master/examples/deep_exponential_family.py)
(Ranganath et al., 2015).

```python
bag_of_words = np.random.poisson(5., size=[256, 32000])  # training data as matrix of counts
data_size = bag_of_words.shape[0]  # number of documents
feature_size = bag_of_words.shape[1]  # number of words (vocabulary)
units = [100, 30, 15]  # number of stochastic units per layer
shape = 0.1  # Gamma shape parameter

W2 = Gamma(0.1, 0.3, sample_shape=[units[2], units[1]])
W1 = Gamma(0.1, 0.3, sample_shape=[units[1], units[0]])
W0 = Gamma(0.1, 0.3, sample_shape=[units[0], feature_size])

z2 = Gamma(0.1, 0.1, sample_shape=[data_size, units[2]])
z1 = Gamma(shape, shape / tf.matmul(z2, W2))
z0 = Gamma(shape, shape / tf.matmul(z1, W1))
x = Poisson(tf.matmul(z1, W0))
```

__Edward2 / TensorFlow Probability__. You write models as functions, where
random variables operate with the same behavior as Edward's.

```python
def deep_exponential_family(data_size, feature_size, units, shape):
  """A multi-layered topic model over a documents-by-terms matrix."""
  W2 = ed.Gamma(0.1, 0.3, sample_shape=[units[2], units[1]], name="W2")
  W1 = ed.Gamma(0.1, 0.3, sample_shape=[units[1], units[0]], name="W1")
  W0 = ed.Gamma(0.1, 0.3, sample_shape=[units[0], feature_size], name="W0")

  z2 = ed.Gamma(0.1, 0.1, sample_shape=[data_size, units[2]], name="z2")
  z1 = ed.Gamma(shape, shape / tf.matmul(z2, W2), name="z1")
  z0 = ed.Gamma(shape, shape / tf.matmul(z1, W1), name="z0")
  x = ed.Poisson(tf.matmul(z0, W0), name="x")
  return x
```

Broadly, the function's outputs capture what the probabilistic program is over
(the `y` in `p(y | x)`), and the function's inputs capture what the
probabilistic program conditions on (the `x` in `p(y | x)`). Note it's best
practice to write names to all random variables: this is useful for cleaner
TensorFlow name scopes as well as for manipulating model computation.

## TensorFlow Sessions

__Edward__. In graph mode, you fetch values from the TensorFlow graph using a
built-in Edward session. Eager mode is not available.

```python
# Generate from model: return np.ndarray of shape (data_size, feature_size).
with ed.get_session() as sess:
  sess.run(x)
```

__Edward2 / TensorFlow Probability__. In graph mode, you fetch values from the
TensorFlow graph using a TensorFlow session.

```python
# Generate from model: return np.ndarray of shape (data_size, feature_size).
x = deep_exponential_family(data_size, feature_size, units, shape)

with tf.Session() as sess:  # or, e.g., tf.train.MonitoredSession()
  sess.run(x)
```

You can also use Edward2 in eager mode (`tf.enable_eager_execution()`), where
`x` already fetches the sampled NumPy array (obtainable as `x.value`).

## Probabilistic Inference

In Edward, there is a taxonomy of inference algorithms, with many built-in from
the abstract classes of `ed.MonteCarlo` (sampling) and `ed.VariationalInference`
(optimization). In TensorFlow Probability, inference algorithms are modularized
so that they can depend on arbitrary TensorFlow ops; any associated abstractions
do not strictly require Edward2. Below we outline variational inference, Markov
chain Monte Carlo, and how to schedule training.

### Variational Inference

__Edward__. You construct random variables with free parameters, representing
the model's posterior approximation. You align these random variables together
with the model's and construct an inference class.

```python
def trainable_pointmass(shape, name=None):
  """Learnable point mass distribution with softplus unconstraints."""
  with tf.variable_scope(name, default_name="trainable_pointmass"):
    return PointMass(tf.nn.softplus(tf.get_variable("mean", shape)))

def trainable_gamma(shape, name=None):
  """Learnable Gamma via shape and scale, with softplus unconstraints."""
  with tf.variable_scope(name, default_name="trainable_gamma"):
    return Gamma(tf.nn.softplus(tf.get_variable("shape", shape)),
                 1.0 / tf.nn.softplus(tf.get_variable("scale", shape)))

qW2 = trainable_pointmass(W2.shape)
qW1 = trainable_pointmass(W1.shape)
qW0 = trainable_pointmass(W0.shape)
qz2 = trainable_gamma(z2.shape)
qz1 = trainable_gamma(z1.shape)
qz0 = trainable_gamma(z0.shape)

inference = ed.KLqp({W0: qW0, W1: qW1, W2: qW2, z0: qz0, z1: qz1, z2: qz2},
                    data={x: bag_of_words})
```

__Edward2 / TensorFlow Probability__. We're in the process of making
variational inference easier. For now, you set up variational inference manually
(possibly using `tfp.losses`) and/or build your own abstractions.

Below we use Edward2's
[`interceptors`](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/python/edward2/interceptor.py)
in order to manipulate model computation. We define the variational
approximation—another Edward2 program—and apply interceptors to write the
evidence lower bound (Hinton & Camp, 1993; Jordan, Ghahramani, Jaakkola, & Saul,
1999; Waterhouse, MacKay, & Robinson, 1996).

```python
def def_variational():
  """Posterior approx. for deep exponential family p(W{0,1,2}, z{1,2,3} | x)."""
  qW2 = trainable_pointmass(W2.shape)  # same func as above but with ed2 rv's
  qW1 = trainable_pointmass(W1.shape)
  qW0 = trainable_pointmass(W0.shape)
  qz2 = trainable_gamma(z2.shape)  # same func as above but with ed2 rv's
  qz1 = trainable_gamma(z1.shape)
  qz0 = trainable_gamma(z0.shape)
  return qW2, qW1, qW0, qz2, qz1, qz0

def make_recording(trace):
  def record(f, *args, **kwargs):
    """Records execution to a trace."""
    output = ed.interceptable(f)(*args, **kwargs)
    trace[kwargs.get("name")] = output
    return output
  return record

def make_value_settings(trace):
  alignment = {"z0": "qz0", "z1": "qz1", "z2": "qz2",
               "W0": "qW0", "W1": "qW1", "W2": "qW2"}
  def set_values(f, *args, **kwargs):
    """Sets random variable values to the trace's aligned value."""
    name = kwargs.get("name")
    if name in alignment:
      kwargs["value"] = trace.get(alignment[name])
    return ed.interceptable(f)(*args, **kwargs)
  return set_values

# Compute expected log-likelihood. First, sample from the variational
# distribution; second, compute the log-likelihood given the sample.
variational_trace = collections.OrderedDict({})
with ed.interception(make_recording(variational_trace)):
  _ = def_variational()

model_trace = collections.OrderedDict({})
with ed.interception(make_recording(model_trace)):
  with ed.interception(make_value_settings(variational_trace)):
    posterior_predictive = deep_exponential_family(data_size, feature_size, units, shape)

log_likelihood = posterior_predictive.log_prob(bag_of_words)

# Compute analytic KL-divergence between variational and prior distributions.
kl = variational_trace["qz0"].kl_divergence(model_trace["z0"])
kl += variational_trace["qz1"].kl_divergence(model_trace["z1"])
kl += variational_trace["qz2"].kl_divergence(model_trace["z2"])
kl += variational_trace["qW0"].kl_divergence(model_trace["W0"])
kl += variational_trace["qW1"].kl_divergence(model_trace["W1"])
kl += variational_trace["qW2"].kl_divergence(model_trace["W2"])

elbo = tf.reduce_mean(log_likelihood - kl)
tf.summary.scalar("elbo", elbo)
train_op = optimizer.minimize(-elbo)
```

### Markov chain Monte Carlo

__Edward__. Similar to variational inference, you construct random variables
with free parameters, representing the model's posterior approximation. You
align these random variables together with the model's and construct an
inference class.

```python
num_samples = 10000  # number of events to approximate posterior

qW2 = Empirical(tf.get_variable("qW2/params", [num_samples, units[2], units[1]]))
qW1 = Empirical(tf.get_variable("qW1/params", [num_samples, units[1], units[0]]))
qW0 = Empirical(tf.get_variable("qW0/params", [num_samples, units[0], feature_size]))
qz2 = Empirical(tf.get_variable("qz2/params", [num_samples, data_size, units[2]]))
qz1 = Empirical(tf.get_variable("qz1/params", [num_samples, data_size, units[1]]))
qz0 = Empirical(tf.get_variable("qz0/params", [num_samples, data_size, units[0]]))

inference = ed.HMC({W0: qW0, W1: qW1, W2: qW2, z0: qz0, z1: qz1, z2: qz2},
                   data={x: bag_of_words})
```

__Edward2 / TensorFlow Probability__. Use the `tfp.mcmc` module. Operating with
`tfp.mcmc` comprises two stages: set up a transition kernel which determines how
one state propagates to the next; and apply the transition kernel over multiple
iterations until convergence.

Below we first rewrite the Edward2 model in terms of its target log-probability
as a function of latent variables. Namely, it is the model's log-joint
probability function with fixed hyperparameters and observations anchored at the
data. We then apply the higher-level `tfp.mcmc.sample_chain` which applies a
Hamiltonian Monte Carlo transition kernel to return a collection of state
transitions.

```python
num_samples = 10000  # number of events to approximate posterior
qW2 = tf.nn.softplus(tf.random_normal([units[2], units[1]]))  # initial state
qW1 = tf.nn.softplus(tf.random_normal([units[1], units[0]]))
qW0 = tf.nn.softplus(tf.random_normal([units[0], feature_size]))
qz2 = tf.nn.softplus(tf.random_normal([data_size, units[2]]))
qz1 = tf.nn.softplus(tf.random_normal([data_size, units[1]]))
qz0 = tf.nn.softplus(tf.random_normal([data_size, units[0]]))

log_joint = ed.make_log_joint_fn(deep_exponential_family)

def target_log_prob_fn(W2, W1, W0, z2, z1, z0):
  """Target log-probability as a function of states."""
  return log_joint(data_size, feature_size, units, shape,
                   W2=W2, W1=W1, W0=W0, z2=z2, z1=z1, z0=z0, x=bag_of_words)

hmc_kernel = tfp.mcmc.HamiltonianMonteCarlo(
    target_log_prob_fn=target_log_prob_fn,
    step_size=0.01,
    num_leapfrog_steps=5)
states, kernels_results = tfp.mcmc.sample_chain(
    num_results=num_samples,
    current_state=[qW2, qW1, qW0, qz2, qz1, qz0],
    kernel=hmc_kernel,
    num_burnin_steps=1000)
```

### The Training Loop

__Edward__. To schedule training, you call `inference.run()` which automatically
handles the schedule. Alternatively, you manually schedule training with
`inference`'s class methods.

```python
inference.initialize(n_iter=10000)

tf.global_variables_initializer().run()
for _ in range(inference.n_iter):
  info_dict = inference.update()
  inference.print_progress(info_dict)

inference.finalize()
```

__Edward2 / TensorFlow Probability__. To schedule training, you use TensorFlow
ops. For an equivalent `inference.run()`-like API, see
[TensorFlow Estimator](https://www.tensorflow.org/guide/estimators)
([example](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/examples/latent_dirichlet_allocation.py)).
For finetuning variational inference, below is one example.

```python
max_steps = 10000  # number of training iterations
model_dir = None  # directory for model checkpoints

sess = tf.Session()
summary = tf.summary.merge_all()
summary_writer = tf.summary.FileWriter(model_dir, sess.graph)
start_time = time.time()

tf.global_variables_initializer().run()
for step in range(max_steps):
  start_time = time.time()
  _, elbo_value = sess.run([train_op, elbo])
  if step % 500 == 0:
    duration = time.time() - start_time
    tf.logging.info("Step: {:>3d} Loss: {:.3f} ({:.3f} sec)".format(
        step, elbo_value, duration))
    summary_str = sess.run(summary)
    summary_writer.add_summary(summary_str, step)
    summary_writer.flush()
```

Finetuning Markov chain Monte Carlo is similar. Instead of tracking a loss
function, one uses, for example, a counter for the number of accepted samples.
This lets us monitor a running statistic of MCMC's acceptance rate by
accumulating `kernel_results.is_accepted` over session runs.

## Model & Inference Criticism

__Edward__. You typically use two functions: `ed.evaluate` for assessing how
model predictions match the true data; and `ed.ppc` for
assessing how data generated from the model matches the true data.

```python
# Build posterior predictive: it is parameterized by a variational posterior sample.
posterior_predictive = ed.copy(
    x, {W0: qW0, W1: qW1, W2: qW2, z0: qz0, z1: qz1, z2: qz2})

# Evaluate average log-likelihood of data.
ed.evaluate('log_likelihood', data={posterior_predictive: bag_of_words})
## np.ndarray of shape ()

# Compare TF-IDF on real vs generated data.
def tfidf(bag_of_words):
  """Computes term-frequency inverse-document-frequency."""
  num_documents = bag_of_words.shape[0]
  idf = tf.log(num_documents) - tf.log(tf.count_nonzero(bag_of_words, axis=0))
  return bag_of_words * idf

observed_statistics, replicated_statistics = ed.ppc(
    lambda data, latent_vars: tf_idx(data[posterior_predictive]),
    {posterior_predictive: bag_of_words},
    n_samples=100)
```

__Edward2 / TensorFlow Probability__. Build the metric manually or use TensorFlow
abstractions such as `tf.metrics`.

```python
# See posterior_predictive built in Variational Inference section.
log_likelihood = tf.reduce_mean(posterior_predictive.log_prob(bag_of_words))
## tf.Tensor of shape ()

# Simple version: Compare statistics by sampling from model in a for loop.
observed_statistic = sess.run(tfidf(bag_of_words))
replicated_statistic = tfidf(posterior_predictive)
replicated_statistics = [sess.run(replicated_statistic) for _ in range(100)]
```

See
[Bayesian neural networks](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/examples/bayesian_neural_network.py)
for training with `tf.metrics.accuracy` and
[Eight Schools](https://github.com/tensorflow/probability/blob/master/tensorflow_probability/examples/jupyter_notebooks/Eight_Schools.ipynb)
for visualizing manually written predictive checks.

## References

1. George Edward Pelham Box. Science and statistics. _Journal of the American Statistical Association_, 71(356), 791–799, 1976.
2. Hinton, G. E., & Camp, D. van. (1993). Keeping the neural networks simple by minimizing the description length of the weights. In Conference on learning theory. ACM.
3. Jordan, M. I., Ghahramani, Z., Jaakkola, T. S., & Saul, L. K. (1999). An introduction to variational methods for graphical models. Machine Learning, 37(2), 183–233.
4. Rajesh Ranganath, Linpeng Tang, Laurent Charlin, David M. Blei. Deep exponential families. In _Artificial Intelligence and Statistics_, 2015.
5. Waterhouse, S., MacKay, D., & Robinson, T. (1996). Bayesian methods for mixtures of experts. Advances in Neural Information Processing Systems, 351–357.