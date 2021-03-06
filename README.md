Primal Dual Operator Splitting for Conic Optimization
=====================================================
Brendan O'Donoghue, Eric Chu, and Stephen Boyd
----------------------------------------------

This code provides a solver for second-order cone problems. It is an
implementation of the algorithm described in `pdos.pdf`. It provides both a
direct and an indirect solver in the form of a static library for inclusion
in other projects.

It simultaneously solves the primal cone program

    minimize     c'*x
    subject to   A*x + s == b
                 s in K 
                 
and its dual

    maximize     -b'*y
    subject to   -A'*y == c
                 y in K^* 

where `K` is a product cone of zero cones, linear cones `{ x | x >= 0 }`, and
second-order cones `{ (t,x) | ||x||_2 <= t }`; `K^*` is its dual cone. The
dual of the zero cone is the free cone; all other cones are self-dual.

A reference Matlab implementation is included under the `matlab` folder; it is called `pdos.m`.

Installing
----------
Typing `make` at the command line should do the trick. It will produce two libaries, `libpdosdir.a` and `libpdosindir.a` found under the `lib` folder. As a byproduct, it will also produce two demo binaries under the `bin` folder called `demo_direct` and `demo_indirect`.

Let us know if the build process fails for you.

### Compiling a Matlab mex file
Running `make_pdos` in Matlab under the `matlab` folder will produce two mex files, `pdos_direct` and `pdos_indirect`.

Remember to include the `matlab` directory in your Matlab path if you wish to use the mex file in your Matlab code. The calling sequence is

    [x,s,y,status] = pdos_direct(data,cones,params)

Type `help pdos_direct` at the Matlab prompt to see its documentation.

### Installing a CVX solver
For users familiar with [CVX](http://cvxr.com), we supply a CVX shim which can be easily installed by invoking the following in the Matlab command line under the `matlab` directory.

    >> cvx_install_pdos
    
You can select the PDOS solver (for SOCPs) with CVX as follows 

    >> cvx_solver 'pdos'
    >> cvx_begin
    >> ...
    >> cvx_end
 
### Compiling a Python extension
If you have no trouble with `make`, then this should be straightforward as well. It requires [CVXOPT](http://cvxopt.org) for use in Python. The relevant files are under the `python` directory. The command

    python setup.py install

should install the two extensions, `pdos_direct` and `pdos_indirect` into your Python directories. You can run python and `import pdos_direct` or `import pdos_indirect` to use the codes. These two modules expose only a single `solve` function.

The `solve` function is called as follows:

    sol = solve(c, A, b, dims)

The matrix "A" is a sparse matrix in column compressed storage. The vectors "c" and "b" are dense column vectors.

* `A` is a CVXOPT (m x n) sparse matrix
* `b` is a (dense) CVXOPT (m x 1) matrix of doubles
* `c` is a (dense) CVXOPT (n x 1) matrix of doubles
* `dims` is a dictionary:
    * `dims['f']` is an integer giving the number of free variables
    * `dims['l']` is an integer giving the number of nonnegative constraints
    * `dims['q']` is a Python list of integers giving the number of cone constraints
 
The code returns a Python dictionary with four keys, 'x', 's', 'y', and
'status'. These report the primal and dual solution (as CVXOPT matrices) and the
solver status (as a string). There are also optional arguments (for both the
direct and indirect solver) which must be supplied in the following order:

    sol = solve(c, A, b, dims, opts, x0, y0, z0)

* `opts` is an optional dictionary with
  *  `opts['MAX_ITERS']` is an integer. Sets the maximum number of ADMM iterations. Defaults to 2000.
  * `opts['EPS_ABS']` is a double. Sets the quitting tolerance for ADMM. Defaults to 5e-3.
  * `opts['ALPHA']` is a double in (0,2) (non-inclusive). Sets the over-relaxation parameter. Defaults to 1.0.
  * `opts['VERBOSE']` is an integer (or Boolean) either 0 or 1. Sets the verbosity of the solver. Defaults to 1 (or True).
  * `opts['NORMALIZE']` is an integer (or Boolean) either 0 or 1. Tells the solver to normalize the data. Defaults to 0 (or False).
  * `opts['CG_MAX_ITS']` is an integer. Sets the maximum number of CG iterations. Defaults to 20.
  * `opts['CG_TOL']` is a double. Sets the tolerance for CG. Defaults to 1e-4.

The last two options are ignored in the direct solver. Additionally, initial ADMM iterates can be optionally supplied:

* `x0` is a (dense) CVXOPT (n x 1) matrix of doubles giving the initial guess for the primal solution `x`
* `y0` is a (dense) CVXOPT (m x 1) matrix of doubles giving the initial guess for the dual solution `y`
* `s0` is a (dense) CVXOPT (m x 1) matrix of doubles giving the initial guess for the primal slack `s`; note that it doesn't necessarily need to satsify `Ax + s = b`
  
Although we wish to support a Numpy interface, we require a sparse matrix C module, which is either unavailable or poorly documented in SciPy.

Usage in C
----------
If `make` completes successfully, it will produce two static library files,
`libpdosdir.a` and `libpdosindir.a` under the `lib` folder. To include the
libraries in your own source code, compile with the linker option with
`-L(PATH_TO_PDOS)\lib` and `-lpdosdir` or `-lpdosindir` (as needed).

These libraries (and `pdos.h`) expose only three API functions:

* Sol \*pdos(const Data \*d, const Cone \*k);
    
    This solves the problem specified in the `Data` and `Cone` structures. 
    The solution is returned in a `Sol` structure.
    
* void freeData(Data \*\*d, Cone \*\*k);
    
    This frees the `Data` and `Cone` structures.
    
* void freeSol(Sol \*\*sol);

    This frees the `Sol` structure.
    
The relevant data structures are:

    typedef struct PROBLEM_DATA {
      idxint n, m; /* problem dimensions */
      /* problem data, A, b, c: */
      double * Ax;
      idxint * Ai, * Ap;
      double * b, * c;
      
      /* initial guesses; can be NULL */
      double *x;
      double *y;
      double *s;
  
      Params * p;
    } Data;
    
    typedef struct PROBLEM_PARAMS {
      idxint MAX_ITERS, CG_MAX_ITS;
      double EPS_ABS, ALPHA, CG_TOL;
      idxint VERBOSE, NORMALIZE;  // boolean
    } Params;

    typedef struct SOL_VARS {
      idxint n, m; /* solution dimensions */
      double *x, *s, *y;
      char status[16];
    } Sol;

    typedef struct CONE {
        idxint f;          /* number of linear equality constraints */
        idxint l;          /* length of LP cone */
        idxint *q;   		   /* array of second-order cone constraints */
        idxint qsize;      /* length of SOC array */
    } Cone;

The data matrix `A` is specified in column-compressed format and the vectors
`b` and `c` are specified as dense arrays. The solutions `x` (primal), `s`
(slack), and `y` (dual) are returned as dense arrays. Cones are specified in
terms of their lengths; the only special one is the second-order cone, where
the lengths are provided as an array of second-order cone lengths (and a
variable `qsize` giving its length).


Scalability
-----------
Note that this code is merely meant as an implementation of the ideas in our
paper. The actual code does not use more than a single CPU. Nevertheless, for
problems that fit in memory on a single computer, this code will (attempt to)
solve them.

To scale this solver, one must either provide a distributed solver for linear
systems (c.f., Jack Poulson's Elemental) or a distributed matrix-vector multiplication, perhaps by way of CUDA. 


Known Issues
------------
* When using older versions of OSX's built-in version of Python, `setup.py` may attempt to build Power PC versions of the Python extensions. If you upgraded to Xcode 4, then support for Power PC has been removed. In that case, the build process may complain about the Power PC architecture. Simply use the following instead:

    ARCHFLAGS='-arch i386 -arch x86_64' python setup.py install
