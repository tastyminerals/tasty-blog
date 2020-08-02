---
layout: post
title: "D vs NumPy"
author: tastyminerals
categories: dlang
---

single precision, double precision, complex double precision
write a function the generates Mir Slice of specified precision and sizes.

# Data setup

Matrix sizes:
    10 x 10
    20 x 20
    60 x 60
    300 x 300
    2400 x 2400
    31200 x 31200

Floating precision: double

# Basic benchmarks
element-wise sum
element-wise mul
matrix sum
matrix argmin
matrix argmax
matrix std - NumPy uses two-pass implementation with float64 for more accuracy
matrix transpose
matrix sort
matrix (random) insert
matrix concatenation
matrix dot
matrix product

# Advanced benchmarks
Laplace's equation Python: http://technicaldiscovery.blogspot.com/2011/06/speeding-up-python-numpy-cython-and.html
SVD
np.linalg.cholesky(F)
pca
np.linalg.eig(G) // bug

harrisCorners (3x3)
harrisCorners (5x5)
shiTomasiCorners (3x3)
shiTomasiCorners (5x5)
extractCorners
gray2rgb
hsv2rgb
rgb2gray
rgb2hsv
convolution 1D (3)
convolution 1D (5)
convolution 1D (7)
convolution 2D (3x3)
convolution 2D (5x5)
convolution 2D (7x7)
convolution 3D (3x3)
convolution 3D (5x5)
bilateralFilter (3x3)
bilateralFilter (5x5)
calcGradients
calcPartialDerivatives
canny
filterNonMaximum
nonMaximaSupression
remap
warp