# Echo Noise for Exact Mutual Information Calculation

Tensorflow/Keras code replicating the experiments in:  https://arxiv.org/abs/1904.07199
   
```
@article{brekelmans2019exact,
  title={Exact Rate-Distortion in Autoencoders via Echo Noise},
  author={Brekelmans, Rob and Moyer, Daniel and Galstyan, Aram and Ver Steeg, Greg},
  journal={arXiv preprint arXiv:1904.07199},
  year={2019}
}
```

Echo noise is flexible, data-driven alternative to Gaussian noise that admits an simple, exact expression for mutual information by construction.  Applied in the autoencoder setting, we show that regularizing with I(X:Z) corresponds to the optimal choice of prior in the Evidence Lower Bound and leads to significant improvements over VAEs.  

## Echo Noise

For easy inclusion in other projects, the echo noise functions are included
in one all-in-one file, ```echo_noise.py```, which can be copied to a project
and included directly, e.g.:
```python
import echo_noise
```

There are two basic functions
implemented, the noise function itself (```echo_sample```)
and the MI calculation (```echo_loss```), both of which are included in
```echo_noise.py```. Except for libaries, ```echo_noise.py``` has no other
file dependencies.

Echo noise is meant to be used similarly to the Gaussian noise in VAEs, and
was implemented with VAE implementations in mind. Assuming the decoder
provides ```z_mean``` and ```z_log_scale```, a Gaussian Encoder would look
something like:
```python
z = z_mean + tf.exp(z_log_scale) * tf.random.normal( tf.shape(z_mean) )
```
The Echo noise equivalent implemented here is:
```python
z = echo_noise.echo_sample( [z_mean, z_log_scale] )
```
Similarly, VAEs often calculate a KL divergence penalty based on
```z_mean``` and ```z_log_scale```. The Echo noise penalty, which is the
mutual information `I(x,z)`, can be computed
using:
```python
loss = ... + echo_noise.echo_loss([z_log_scale])
```
A Keras version of this might look like the following:
```python
z_mean = Dense(latent_dim, activation = model_utils.activations.tanh64)(h)
z_log_scale = Dense(latent_dim, activation = tf.math.log_sigmoid)(h)
z_activation = Lambda(echo_noise.echo_sample)([z_mean, z_log_scale])
echo_loss = Lambda(echo_noise.echo_loss)([z_log_scale])
```

These functions are also found in the experiments code, ```model_utils/layers.py``` and ```model_utils/losses.py```.

We can choose to sample training examples with or without replacement from within the batch for constructing Echo noise.  A quirk of the current implementation is that sampling with replacement sets the batch dimension != None. This means the model cannot accommodate different batch sizes (e.g. if ```data.shape[0] % batch > 0```) and should be trained using, e.g. ```fit_generator```.   Sampling without replacement does not have this issue.


## Instructions:  
```
python run.py --config 'echo.json' --beta 1.0 --filename 'echo_example' --dataset 'binary_mnist'
```
Experiments are specifed using the config files, which specify the network architecture and loss functions.  ```run.py``` calls ```model.py``` to parse these ```configs/``` and create / train a model.  You can also modify the tradeoff parameter ```beta```, which is multiplied by the rate term, or specify the dataset using ```'binary_mnist'```, ```'omniglot'```, or ```'fmnist'.``` . Analysis tools are mostly omitted for now, but the model loss training history is saved in a pickle file.



## Comparison Methods
We compare diagonal Gaussian noise encoders ('VAE') and IAF encoders, alongside several marginal approximations : standard Gaussian prior, standard Gaussian with MMD penalty (```info_vae.json``` or ```iaf_prior_mmd.json```), Masked Autoregressive Flow (MAF), and VampPrior.  All combinations can be found in the ```configs/``` folder. 
