
This project consists of several bugfinding tools that look for memory
protection errors in the C source code of the [GNU
R](http://www.r-project.org/) core and packages. **The documentation is
behind the code, the tools do much more than currently described here.**

As input the tools need the LLVM bitcode of the R binary. To check an R
package, the tools also need the bitcode of the shared library of that
package. To get these bitcode files one needs to build R with the CLANG
compiler, with LTO (link time optimizations) enabled but all other compiler
optimizations disabled (-O0), and telling CLANG to emit the LLVM bitcode in
addition to executable files. Detailed instructions (tested on Ubuntu 14.04 and
Fedora Core 20) are available [here](todo). For convenience, the bitcode of
the R binary from R-devel version 67741 generated by CLANG/LLVM 4.2.0 is
included in the `examples` directory.

The tools need [LLVM 3.5.0](http://llvm.org/releases/download.html) to build
(one can download one of the Clang [pre-built
binaries](http://llvm.org/releases/download.html#3.5.0)).  To build the
tools, one just needs to modify the LLVM installation path in `src/Makefile`
and run `make`.

## Detecting Allocating Functions

The key to the correct use of the `PROTECT` macros is knowing which
functions may allocate and thus trigger garbage collection.  Except for few
obvious cases this is hard to find out.

The `csfpcheck` tool does the job automatically:

`csfpcheck ./src/main/R.bin.bc >lines`

produces a list of source files with line numbers where allocation may
happen. This script puts a comment at the end of each such line:

`cat lines | while read F L ; do echo "sed -i '$L,$L"'s/$/ \/* GC *\//g'\'" $F" ; done | bash`

The tool errs on the safe side, which is saying that a function may
allocate.  It may be that in fact the function won't allocate for the given
inputs.  The tool, however, does take some function arguments into account:
e.g.  it can detect that `getAttrib(,R_ClassSymbol)` will not allocate, but
`getAttrib(,R_NamesSymbol)` might.  In the following snippet from `eval.c`,

```
SEXP attribute_hidden do_enablejit(SEXP call, SEXP op, SEXP args, SEXP rho)
{
    int old = R_jit_enabled, new;
    checkArity(op, args);
    new = asInteger(CAR(args)); /* GC */
    if (new > 0)
        loadCompilerNamespace(); /* GC */
    R_jit_enabled = new;
    return ScalarInteger(old); /* GC */
}
```

the allocation in `ScalarInteger` is correct and obvious to any developer,
the allocation in `loadCompilerNamespace` is correct but perhaps less obvious,
and the allocation in `asInteger` is surprising.  But it is correct!  The
function will allocate when displaying a warning, e.g.  when the conversion
loses accuracy or produces NAs.

A patch for the R-devel version 67741 with the generated "GC" comments by
the tool is available in the `examples` directory for convenience. To get
the annotated sources, one just needs to

```
cd examples
svn checkout -r 67741 http://svn.r-project.org/R/trunk
cd trunk
zcat ../csfpcheck.67741.patch.gz | patch -p0
```
