+++
title = 'Mathematics for Machine Learning'
date = 2025-05-23T11:17:02+07:00
draft = true
tags = ['math']
summary = ''
math = true
ShowToc = true
authors = ['Marc Peter Deisenroth', 'A. Aldo Faisal', 'Cheng Soon Ong']
+++

{{<details title="Table of Symbols">}}

- $a, b, c, \alpha, \beta, \gamma$ : Scalars are lowercase
- $\bold{x}, \bold{y}, \bold{z}$ : Vectors are bold lowercase
- $\bold{A}, \bold{B}, \bold{C}$ : Matrices are bold uppercase
- $\bold{x}^\top, \bold{A}^\top$ : Transpose of a vector or matrix
- $\bold{A}^{-1}$ : Inverse of a matrix
- $\braket{\bold{x}, \bold{y}}$ : Inner product of $\bold{x}$ and $\bold{y}$
- $\bold{x}^\top\bold{y}$ : Dot product of $\bold{x}$ and $\bold{y}$
- $B=(\bold{b_1}, \bold{b_2}, \bold{b_3})$ : (Ordered) tuple
- $\bold{B}=[\bold{b_1}, \bold{b_2}, \bold{b_3}]$ : Matrix of column vectors stacked horizontally
- $\mathcal{B}={\bold{b_1}, \bold{b_2}, \bold{b_3}}$ : Set of vectors (unordered)
- $\mathbb{Z}, \mathbb{N}$ : Integers and natural numbers, respectively
- $\mathbb{R}, \mathbb{C}$ : Real and complex numbers, respectively
- $\mathbb{R}^n$ : $n$-dimensional vector space of real numbers
- ---
- $\forall x$ : Universal quantifier: for all $x$
- $\exist x$ : Existential quantifier: there exists $x$
- $a := b$ : a is defined as b
- $a =: b$ : b is defined as a
- $a \propto b$ : $a$ is proportional to $b$, i.e., $a = \text{constant} \cdot b$
- $g \circ f$ : Function composition: "$g$ after $f$"
- $\iff$ : If and only if
- $\implies$ : Implies
- $\mathcal{A, C}$ : Sets
- $a \in \mathcal{A}$ : $a$ is an element of set $\mathcal{A}$
- $\empty$ : Empty set
- $\mathcal{A} \setminus \mathcal{B}$ : A without B: the set of elements in A but not in B
- $D$ : Number of dimensions; indexed by $d = 1, . . . , D$
- $N$ : Number of data points; indexed by $n = 1,...,N$
- ---
- $\bold{I}_m$ : Identity matrix of size ${m \times m}$
- $\bold{0}_{m \times n}$ : Matrix of zeros of size ${m \times n}$
- $\bold{1}_{m \times n}$ : Matrix of ones of size ${m \times n}$
- $\bold{e}_i$ : Standard/canonical vector (where $i$ is the component that is $1$)
- $dim$ : Dimensionality of vector space
- $rk(\bold{A})$ : Rank of matrix A
- $Im(\Phi)$ : Image of linear mapping $\Phi$
- $ker(\Phi)$ : Kernel (null space) of a linear mapping $\Phi$
- $span[\bold{b}_1]$ : Span (generating set) of $\bold{b}_1$
- $tr(\bold{A})$ : Trace of A
- $det(\bold{A})$ : Determinant of A
- $| \cdot |$ : Absolute value or determinant (depending on context)
- $\Vert \cdot \Vert$ : Norm; Euclidean, unless specified
- $\lambda$ : Eigenvalue or Lagrange multiplier
- $E_{\lambda}$ : Eigenspace corresponding to eigenvalue $\lambda$
- ---
- $\bold{x} \perp \bold{y}$ : Vectors x and y are orthogonal
- $V$ : Vector space
- $V^\top$ : Orthogonal complement of vector space V
- $\sum_{n=1}^N x_n$ : Sum of the $xn:x_1 +...+x_N$
- $\prod_{n=1}^N x_n$ : Product of the $x_n : x_1 \cdot...\cdot x_N$
- $\bold{\theta}$ : Parameter vector
- $\frac{\partial f}{\partial x}$ : Partial derivative of f with respect to x
- $\frac{df}{dx}$ : Total derivative of f with respect to x
- $\nabla$ : Gradient
- $f_* = min_x f(x)$ : The smallest function value of f
- $x_* \in arg min_x f(x)$ : The value $x_*$ that minimizes f (note: arg min returns a set of values)
- $\frak{L}$ : Lagrangian
- $\mathcal{L}$ : Negative log-likelihood
- $\binom{n}{k}$ : Binomial coefficient, $n$ choose $k$
- $\mathbb{V}_X[\bold{x}]$ : Variance of $\bold{x}$ with respect to the random variable $X$
- $\mathbb{E}_X[\bold{x}]$ : Expectation of $\bold{x}$ with respect to the random variable $X$
- $Cov_{X,Y}[\bold{x}, \bold{y}]$ : Covariance between x and y.
- $X \Vert Y | Z$ : $X$ is conditionally independent of $Y$ given $Z$
- $X \text{\textasciitilde} p$ : Random variable $X$ is distributed according to $p$
- $\mathcal{N}(\bold{\mu}, \bold{\Sigma})$ : Gaussian distribution with mean $\bold{\mu}$ and covariance $\bold{\Sigma}$
- $Ber(\mu)$ : Bernoulli distribution with parameter $\mu$
- $Bin(N, \mu)$ : Binomial distribution with parameters $N$, $\mu$
- $Beta(\alpha, \beta)$ : Beta distribution with parameters $\alpha$, $\beta$
- --- 

Table of Abbreviations and Acronyms

- e.g. : Exempli gratia (Latin: for example)
- GMM : Gaussian mixture model
- i.e. : Id est (Latin: this means)
- i.i.d. : Independent, identically distributed
- MAP : Maximum a posteriori
- MLE : Maximum likelihood estimation/estimator
- ONB : Orthonormal basis
- PCA : Principal component analysis
- PPCA : Probabilistic principal component analysis
- REF : Row-echelon form
- SPD : Symmetric, positive definite
- SVM : Support vector machine

{{</details>}}


## Part 1 : Mathematical Foundations

### 1. Introduction and Motivation

Core concepts of machine learning:
1. data
2. a model
3. learning

The goal of machine learning is to extract patterns from *data*.

*A model* is said to learn from data if its performance on a given task improves after the data is taken into account.

*Learning* is a way to automatically find patterns and structure in data by optimizing the parameters of the model.


#### 1.1 Finding Words for Intuitions

Machine learning algorithm to mean 
- a system that makes predictions based on input data refered to *predictor*.
- adapts some internal parameters of the predictor so that it performs well on future unseen input data refered to *training*


*A model* is used to described a process for generating data.

*Training* means to use the data to optimize some parameters of the model.

summary
1. data represented as vectors
2. choose an appropriate model, 
3. learn from data


#### 1.2 Two Ways to Read This Book

4 pillars of machine learning
1. regression
2. dimensionality reduction
3. density estimation
4. classification

Part 1 is about Mathematics.

Linear algebra studies about vectors and matrices.

*Analytic geometry*, the study of similarity between 2 or more vectors.

Part 2 is about machine learning

how mathematical concepts in part 1 are the foundation.



### 2. Linear Algebra
### 3. Analytic Geometry
### 4. Matrix Decompositions
### 5. Vector Calculus
### 6. Probability and Distributions
### 7. Continuous Optimization

## Part 2 : Central Machine Learning Problems
