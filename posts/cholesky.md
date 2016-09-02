# Cholesky decomposition

Hi! Currently I'm preparing for [High Performance Matrix Computations](http://hpac.rwth-aachen.de/teaching/hpmc-16/) course exam. As usual, just reading never helps, so, I decided to write some code and sort the things out. This is the post about Cholesky decomposition and how to compute it. The accompanying jupyter notebook can be found [here](https://github.com/yobibyte/rwth-hpmc/blob/master/cholesky.ipynb).

Let's start from the definition. According to [Wikipedia](https://en.wikipedia.org/wiki/Cholesky_decomposition), 'Cholesky decomposition is a decomposition of a [Hermitian](https://en.wikipedia.org/wiki/Hermitian_matrix), [positive-definite matrix](https://en.wikipedia.org/wiki/Positive-definite_matrix) into the product of a [lower triangular matrix](https://en.wikipedia.org/wiki/Lower_triangular_matrix) and its [conjugate transpose](https://en.wikipedia.org/wiki/Conjugate_transpose).' In this tutorial I will focus only on real numbers, so, conjugate transpose is just transpose and a hermitian matrix is just a symmetric matrix.

The most important question we should have now, why the hell do we need such a thing? It may seem utterly stupid, but not everybody ask this question. And I'm also quite late for the party. In short, we need the decomposition to solve systems of linear equations: **A**x=b. So, we see the system **A**x=b:

* first we get the decomposition **A**=**L****L**'
* solve new system **L**y=b (it's much [easier](https://en.wikipedia.org/wiki/Triangular_matrix#Forward_and_back_substitution) as **L** is lower-triangular)
* solve another one **L**x=y (again, it's easy as **L** is lower-triangular). 
* x is known, we are happy

But why, you can say, cannot we just compute inverse of **A** and be happy with that? I did [this](https://en.wikipedia.org/wiki/Invertible_matrix#Analytic_solution) at school you will say.

<img class='center' src="pics/cholesky/board.jpg"/>

Fist, it will take you for ages, second, it has so many operations that while you compute it, π will turn into 4 (and not only π) because of the round off error. Yeah, application of pure linear algebra in real life has many interesting issues and this is only one of them. 

So, we want to find **L** such that **A**=**L****L**' and **L** is lower triangular. Lets write **A** and **L** as a [block matrix](https://en.wikipedia.org/wiki/Block_matrix) (*tl* is top left, *tr* is top right, *bl* is bottom left, *br* is bottom right. 

<img class='center' src="pics/cholesky/block_matrices.png"/>

Using the properties of symmetry, positive-definiteness and triangularity we get the following:

<img class='center' src="pics/cholesky/cholesky_blocks.png"/>

So, the idea of the decomposition goes directly from the picture above:

<img class='center' src="pics/cholesky/cholesky_alg.png"/>

Some things here:
  * there is recursive call of CHOLESKY here. The base case is matrix of one number, where decomposition is the following [a] = [sqrt(a)][sqrt(a)].
  * we use TRSM to solve the system of linear equations here, TRSM is a blas-3 level routine, I will say about it later. Funny enough: we are doing decomposition to solve a system and solve system inside using some BLAS routine. Actually, I do not yet know how TRSM works, so, will not say anyting about that now.

"Talk is cheap, show me the code." Let's write a naive algorithm that will do stuff for us. It's naive as we do not care about performance here.

Generate data for testing the code:

```python
import numpy as np

DIM = 10

# Cholesky decomposition is unique if the main diagonal is positive
L = np.tril(np.random.rand(DIM,DIM))
A = np.dot(L,L.T)
# check that LL' is a Cholesky decomposition of A
np.testing.assert_almost_equal(np.linalg.cholesky(A), L)
```

```python
def cholesky_non_blocked(A):
    ''' Returns L such that A = LL'
    '''
    if A.shape[0] == 1:
        return np.sqrt(A[0,0])
    
    A_tl = A[0,0]
    A_bl = A[1:,0]
    A_br = A[1:,1:]
    
    L_tl = np.sqrt(A_tl)
    L_bl = (A_bl/np.sqrt(A_tl))
    # Use reshape to transpose in a linear algebra way but not to deal with np.matrix
    L_br = cholesky_non_blocked(A_br-np.dot(L_bl.reshape(-1,1),L_bl.reshape((1,-1))))
    
    L = np.eye(len(A))
    L[0, 0]  = L_tl
    L[1:,0]  = L_bl
    L[1:,1:] = L_br
    return L

np.testing.assert_array_almost_equal(cholesky_non_blocked(A), L)
```
And now let's think about the performance. Of couse, I do not talk about writing speed of light code in python, but I will talk a little bit about general matrix matrix product (GEMM), BLAS and block algorithms and show the code of a blocked algorithm using python instead of pseudocode.

```python
TL_CONSTANT = 3

def cholesky_blocked(A, split=TL_CONSTANT):
    
    ''' Returns L such that A = LL'
        for small top right we use unblocked version
        then we proceed with blocked algorithm
    '''

    if A.shape[0] <= split:
        return cholesky_non_blocked(A)
        
    A_tl = A[:TL_CONSTANT,:TL_CONSTANT]
    A_bl = A[TL_CONSTANT:,:TL_CONSTANT]
    A_br = A[TL_CONSTANT:,TL_CONSTANT:]
    
    L_tl = cholesky_non_blocked(A_tl)
    L_bl = np.linalg.solve(L_tl, A_bl.T).T
    L_br = cholesky_blocked(A_br-np.dot(L_bl,L_bl.T))

    L = np.eye(len(A))
    L[:TL_CONSTANT,:TL_CONSTANT]  = L_tl    
    L[TL_CONSTANT:,:TL_CONSTANT]  = L_bl
    L[TL_CONSTANT:,TL_CONSTANT:]  = L_br
 
    return L
np.testing.assert_array_almost_equal(cholesky_blocked(A), L)
```

As matrix multiplication is of paramount importance in computation, there are very efficient algorithms for that. The **speed of light** algorithm for matrix multiplication is called GEMM ( (General Matrix Matrix product). It does the following **C** <-- α**A**+β**B**. When you hear that some SKYNET supercomputer has a performance of 42 yobiflops per second, it's 99% sure was tested on GEMM. GEMM is also a part of [BLAS](https://en.wikipedia.org/wiki/Basic_Linear_Algebra_Subprograms) specification that has hierarchical design. BLAS have 3 levels and when we program stuff, the higher level the better it is for us (as memory is slow, we want to utilise it as much as we can: e.g. instead of loading matrix element by element, load the whole matrix and do operations on it). 

As we can see from our pictures with Cholesky decomposition algorithm, we have one system of linear equations to solve (BLAS routine for that is called TRSM) and one SYRK routine: **A**_br - **L**_bl * **L**_bl'. So, we need to use it and solve them efficiently. To get profit from 3rd level of BLAS we want to do operation with matrices, not on scalars or vectors. The general idea of the block algorithm is to find the decomposition for small **A**_tl in a naive way as I shown before and then utilise power of BLAS-3 level with SYRK and TRSM. So, as it can be seen on the picture below, each iteration we update the column of width k in a final decomposition, where k is the one dimension of **A**_bl. 

<img class='center' src="pics/cholesky/cholesky_schema.png"/>

If you think, that the performance optimisation is solved so far, you are wrong. There is also a problem of paralellizing the code, for instance, and it's not easy at all. But that's all for this post. If you are still sceptical and say: 'Pffff, that's only for symmetric positive semi-definite matrices. My matrices are usually trickier!'. There are other [decompositions](https://en.wikipedia.org/wiki/Matrix_decomposition) for you. For further information read an amazing [book](https://www.amazon.com/Computations-Hopkins-Studies-Mathematical-Sciences/dp/B00BD2DVIC/) by Golub and Van Loan).

P.S. If you see a mistake or just want to say something, please send me a mail to <a href="mailto:vitaliykurin@gmail.com">vitaliykurin@gmail.com</a>.
