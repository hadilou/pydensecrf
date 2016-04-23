PyDenseCRF
==========

This is a (Cython-based) Python wrapper for [Philipp Krähenbühl's Fully-Connected CRFs](http://www.philkr.net/home/densecrf) (version 2).

If you use this code for your reasearch, please cite:

```
Efficient Inference in Fully Connected CRFs with Gaussian Edge Potentials
Philipp Krähenbühl and Vladlen Koltun
NIPS 2011
```

and provide a link to this repository as a footnote or a citation.

Installation
============

You can install this using `pip` by executing:

```
pip install git+https://github.com/lucasb-eyer/pydensecrf.git
```

and ignoring all the warnings coming from Eigen.

Note that you need a relatively recent version of Cython (at least version 0.22) for this wrapper,
the one shipped with Ubuntu 14.04 is too old. (Thanks to Scott Wehrwein for pointing this out.)
I suggest you use a [virtual environment](https://virtualenv.readthedocs.org/en/latest/) and install
the newest version of Cython there (`pip install cython`), but you may update the system version by

```
sudo apt-get remove cython
sudo pip install -U cython
```

Usage
=====

For images, the easiest way to use this library is using the `DenseCRF2D` class:

```python
import numpy as np
import pydensecrf.densecrf as dcrf

d = dcrf.DenseCRF2D(640, 480, 5)  # width, height, nlabels
```

Unary potential
---------------

You can then set a fixed unary potential in the following way:

```python
U = np.array(...)     # Get the unary in some way.
print(U.shape)        # -> (5, 640, 480)
print(U.dtype)        # -> dtype('float32')
U = U.reshape((5,-1)) # Needs to be flat.
d.setUnaryEnergy(U)

# Or alternatively: d.setUnary(ConstUnary(U))
```

Remember that `U` should be negative log-probabilities, so if you're using
probabilities `py`, don't forget to `U = -np.log(py)` them.

Requiring the `reshape` on the unary is an API wart that I'd like to fix, but
don't know how to without introducing an explicit dependency on numpy.

Pairwise potentials
-------------------

The two-dimensional case has two utility methods for adding the most-common pairwise potentials:

```python
# This adds the color-independent term, features are the locations only.
d.addPairwiseGaussian(sxy=(3,3), compat=3, kernel=dcrf.DIAG_KERNEL, normalization=dcrf.NORMALIZE_SYMMETRIC)

# This adds the color-dependent term, i.e. features are (x,y,r,g,b).
# im is an image-array, e.g. im.dtype == np.uint8 and im.shape == (640,480,3)
d.addPairwiseBilateral(sxy=(80,80), srgb=(13,13,13), rgbim=im, compat=10, kernel=dcrf.DIAG_KERNEL, normalization=dcrf.NORMALIZE_SYMMETRIC)
```

Both of these methods have shortcuts and default-arguments such that the most
common use-case can be simplified to:

```python
d.addPairwiseGaussian(sxy=3, compat=3)
d.addPairwiseBilateral(sxy=80, srgb=13, rgbim=im, compat=10)
```

### Compatibilities

The `compat` argument can be any of the following:

- A number, then a `PottsCompatibility` is being used.
- A 1D array, then a `DiagonalCompatibility` is being used.
- A 2D array, then a `MatrixCompatibility` is being used.

### Kernels

Possible values for the `kernel` argument are:

- `CONST_KERNEL`
- `DIAG_KERNEL` (the default)
- `FULL_KERNEL`

### Normalizations

Possible values for the `normalization` argument are:

- `NO_NORMALIZATION`
- `NORMALIZE_BEFORE`
- `NORMALIZE_AFTER`
- `NORMALIZE_SYMMETRIC` (the default)

Inference
---------

The easiest way to do inference is to simply call:

```python
Q = d.inference(n_iterations=5)
```

And the MAP prediction is then:

```python
map = np.argmax(Q, axis=0).reshape((640,480))
```

Step-by-step inference
----------------------

If for some reason you want to run the inference loop manually, you can do so:

```python
Q, tmp1, tmp2 = d.startInference()
for i in range(5):
    print("KL-divergence at {}: {}".format(i, d.klDivergence(Q)))
    d.stepInference(Q, tmp1, tmp2)
```

Generic non-2D
--------------

The `DenseCRF` class can be used for generic (non-2D) dense CRFs.
Its usage is exactly the same as above, except that the 2D-specific pairwise
potentials `addPairwiseGaussian` and `addPairwiseBilateral` are missing.

Instead, you need to use the generic `addPairwiseEnergy` method like this:

```python
d = dcrf.DenseCRF(100, 5)  # npoints, nlabels

feats = np.array(...)  # Get the pairwise features from somewhere.
print(feats.shape)     # -> (7, 100) = (feature dimensionality, npoints)
print(feats.dtype)     # -> dtype('float32')

dcrf.addPairwiseEnergy(feats)
```

In addition, you can pass `compatibility`, `kernel` and `normalization`
arguments just like in the 2D gaussian and bilateral cases.

The potential will be computed as `w*exp(-0.5 * |f_i - f_j|^2)`.

Learning
--------

The learning has not been fully wrapped. If you need it, get in touch or better
yet, wrap it and submit a pull-request!
