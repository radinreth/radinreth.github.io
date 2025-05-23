+++
title = 'Linear Algebra Done Right, 4e by Sheldon Axler'
date = 2025-05-13T15:56:14+07:00
tags = ['math']
draft = true
math = true
summary = 'Now available in Open Access, this best-selling textbook for a second course in linear algebra is aimed at undergraduate math majors and graduate students. The fourth edition gives an expanded treatment of the singular value decomposition and its consequences. It includes a new chapter on multilinear algebra, treating bilinear forms, quadratic forms, tensor products, and an approach to determinants via alternating multilinear forms. This new edition also increases the use of the minimal polynomial to provide cleaner proofs of multiple results. Also, over 250 new exercises have been added. The novel approach taken here banishes determinants to the end of the book. The text focuses on the central goal of linear algebra: understanding the structure of linear operators on finite-dimensional vector spaces. The author has taken unusual care to motivate concepts and simplify proofs. A variety of interesting exercises in each chapter helps students understand and manipulate the objects of linear algebra. Beautiful formatting creates pages with an unusually student-friendly appearance in both print and electronic versions. No prerequisites are assumed other than the usual demand for suitable mathematical maturity. The text starts by discussing vector spaces, linear independence, span, basis, and dimension. The book then deals with linear maps, eigenvalues, and eigenvectors. Inner-product spaces are introduced, leading to the finite-dimensional spectral theorem and its consequences. Generalized eigenvectors are then used to provide insight into the structure of a linear operator. From the reviews of previous editions: Altogether, the text is a didactic masterpiece. — zbMATH The determinant-free proofs are elegant and intuitive. — American Mathematical Monthly The most original linear algebra book to appear in years, it certainly belongs in every undergraduate library — CHOICE'
+++

# Chapter 1: Vector Spaces
> Vector spaces are defined in this chapter, and their basic properties are developed.

### 1A. $R^n$ and $C^n$

### *Complex Numbers*

It was invented so that we can take a square roots of negative numbers.

A square root of $-1$, denoted by $i$
or $i^2=-1$.

The set of all complex numbers is:
$$
  C = \\{ a+bi : a, b \in \R \\}
$$

Addition and multiplication on $C$
$$
(a + bi) + (c + di) = (a + b) + (b + d)i \\\
(a + bi)(c + di) = (ac - bd) + (ad + bc)i \\\
a, b, c, d \in \R
$$

instead of memorize the formula for the product of 2 complex numbers, use normal distributive and commutative properties.

example:
$$
  (2+3i)(4+5i) \\\
  = 2(4+5i)+3i(4+5i) \\\
  = 8+10i+12i+15i^2 \\\
  = -7+22i
$$

Properties of complex arithmetic

1. Commutativity

$\alpha + \beta = \beta + \alpha$ and 
$\alpha \beta = \beta \alpha$

for all $\alpha,\beta \in \mathbb{C}$

1. Associativity

$(\alpha + \beta) + \lambda = \alpha + (\beta + \lambda)$ and $(\alpha \beta) \lambda = \alpha (\beta \lambda)$

for all $\alpha,\beta, \lambda \in \mathbb{C}$

2. Identities

$\lambda + 0 = \lambda$ and $\lambda 1 = \lambda$

for all $\lambda \in \mathbb{C}$

3. Additive inverse

for every $\alpha \in C$, there exists a unique $\beta \in C$ such that $\alpha + \beta =0$.

4. Multiplicative inverse

for every $\alpha \in C$ with $\alpha \ne 0$, there exists a unique $\beta \in C$ such that $\alpha \beta = 1$.

5. Distributive property

$\lambda(\alpha + \beta) = \lambda\alpha + \lambda\beta$ for all $\lambda, \alpha, \beta \in C$.

$(\alpha^m)^n = \alpha^{mn}$
$(\alpha\beta)^m = \alpha^m \beta^m$



# Chapter 2: Finite-Dimensional Vector Spaces
> Linear independence, span, basis, and dimension are defined in this chapter, which presents the basic theory of finite-dimensional vector spaces.

# Chapter 3: Linear Maps
> The fundamental theorem of linear maps,
> quotient spaces and duality.

# Chapter 4: Polynomials
> The part of the theory of polynomials that will be needed to understand linear operators.

# Chapter 5: Eigenvalues and Eigenvectors
# Chapter 6: Inner Product Spaces
# Chapter 7: Operators on Inner Product Spaces
# Chapter 8: Operators on Complex Vector Spaces
# Chapter 9: Multilinear Algebra and Determinants