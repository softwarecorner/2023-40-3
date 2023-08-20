# Futureverse - Worry-Free Parallelization in R
  
Henrik Bengtsson (Department of Epidemiology and Biostatistics, 
University of California San Francisco (UCSF), United States)

_Editor note: This article was originally published in the Biometric Bulletin (2023) Volume 40 Issue 3._

# Introduction

Slow code is never fun. How often haven't we said "I'll just run a
quick analysis in R" just to find ourselves waiting for hours or even
days for it to complete? There can be many reasons for our code not
running as fast as we want. Inefficient programming is one reason for
slow code performance, but we can often improve it with minor changes,
with some experience.  For example, R users quickly discover that
iterating element by element is slow, but vectorized function calls
are much faster. If you want to learn how to write more efficient R
code, I recommend Gillespie & Lovelace (2016), and Wickham (2019).
However, it is not always possible for us to speed up the code we
run. A common reason is that it is not our code that is slow.
Instead, it may be an R package maintained by someone else that is
slow, and it may not be worth our time, or it might be beyond our
skill levels, to improve on that code. Another reason could be that
the underlying algorithm is computationally expensive and its
implementation is already optimized.  Even after considering options
and rewriting code, we may still have slow-running code. This is when
we could turn to parallelization to shorten the processing time.

This article gives a brief overview of _The Futureverse_ (Bengtsson,
2021), which is a framework making simple to parallelize R code. The
learning curve is low and most users are up and running within
minutes.  As we will see below, sometimes it is just a matter of
renaming a function call to make it run in parallel.  The Futureverse
makes it straightforward to write robust and statistically sound R
code that can run in parallel on a variety of compute resources
available to the end-user.


# Background

Where does the Futureverse position itself in the R ecosystem and
where can it be used?  To answer those questions, it is helpful to
know what solutions for parallel processing already exist in R.  For
starters, all R installations come with the **parallel** package,
which provides some rudimentary, built-in functions for running R code
in parallel. You might have heard of the `mclapply()` function.  We
can use it to turn a sequential `lapply()` call:

```r
y <- lapply(X, slow_fcn)
```

into a parallel version:

```r
library(parallel)
y <- mclapply(X, slow_fcn)
```

that calls `slow_fcn()` on each `X` elements in parallel.  The
`mclapply()` function works only on Linux and macOS, because it relies
on a concept called _forked processing_, which R does not support on
MS Windows.  If used on MS Windows, it falls back to running
sequentially just like `lapply()`. Importantly, `mclapply()` is not
stable in all environments, or in combination with certain R packages,
resulting in abrupt crashing of R. The R documentation -
`help("mclapply")` - warns about this, and the RStudio GUI engineers
[recommend against using it](https://github.com/rstudio/rstudio/issues/2597#issuecomment-482187011) when running R in the RStudio software.

Another famous function of the **parallel** package is `parLapply()`.
We can use it somewhat similarly to `mclapply()`, with the advantage
that it works on all operating systems. The downside is that it
requires more low-level orchestration, e.g. we need to set up a
cluster of background workers using `makeCluster()`, manually load
required packages on the workers using `clusterEvalQ()`, and export
all variables needed by the workers using `clusterExport()`. Only then
can we call `parLapply()`. When we are done, we must remember to shut
down the workers using `stopCluster()`. This makes it tedious and
error prone to use, and it clutters up our R code with a lot of
verbatim code, which blurs the algorithm of interest.

There are also solutions outside of base-R and the **parallel**
package.  Almost everyone trying to run R in parallel ends up
discovering the popular **foreach** package (Kane et al., 2013), which
is available on the Comprehensive R Archive Network (CRAN).  This
package has a beautiful design that simplifies how we write parallel
code in R.  When it was first released in 2009, it introduced the idea
of separating the code to be parallelized from the code that controls
how and where parallelization is performed.  For example,

```r
library(foreach)
library(parallel)

## Create a cluster of 4 parallel R workers running in the background
cl <- makeCluster(4)

## Configure foreach to process elements in parallel on these workers
doParallel::registerDoParallel(cl)

## For each element in 'X' call slow_fcn().
## Multiple elements are processed concurrently.
y <- foreach(x = X) %dopar% {
  slow_fcn(x)
}

## Stop workers, when done
stopCluster(cl)
```

This separation allows the developer to focus on the parts that should
run in parallel (i.e. the `foreach()` call) without having to worry to
much about how and where the code is run. Instead, it is the end-user
that have the control over how to parallelize,
e.g. `registerDoParallel()`.  In this example, we used four workers on
the local machine, but we could equally well have set up four workers
on four external computers - all without having to rewrite the
`foreach()` statement.  This design philosophy of **foreach** is
powerful.  Unfortunately, there are some pitfalls that prevent us from
writing code that is guaranteed to work with all parallel backend. For
example, `foreach()` takes backend-specific arguments breaking the
developer-user separation.  That said, the separation of
responsibilities between the developer and the end-user is one of the
main reason **foreach** being popular.


# What is the Futureverse?

The Futureverse was created in order to remove most of the friction
and hurdles that we are experiencing with the traditional
parallel-programming techniques in R.  It is based on a programming
construct referred to as _futures_ (Hibbard, 1976; Hewitt & Baker,
1977; Friedman & Wise, 1978), which is particularly well suited for
functional programming languages such as R.  The Futureverse is
designed to make it as simple as possible to parallelize existing code
letting the developer focus on the algorithm without being distracted
by technicalities related to parallelization.  For instance, it
automatically identifies and exports objects that the parallel workers
need.  It will also protect against common mistakes that may occur
when running code in parallel.  As an example, Futureverse detects and
warns us when we use random numbers that are not statistically sound
because they were generated in parallel.  You can easily install all
Futureverse packages from CRAN, such as **future** with the command
`install.packages("future")`.

The Futureverse follows the lead of **foreach** by strictly separating
what to parallelize from how to parallelize it.  The developer can
also stay with their favorite coding style, e.g. base-R, Tidyverse, or
foreach.  For instance, if you prefer base-R apply functions such as
`lapply()`, `mapply()`, and `by()`, you can use the plug-and-play
replaceable `future_lapply()`, `future_mapply()`, and `future_by()`
functions available in the **future.apply** package.  For example,

```r
library(future.apply)
plan(multisession)  ## parallelize on the local machine

y <- future_lapply(X, slow_fcn)
```

works just like `lapply()`, except that it runs in parallel via the
`multisession` backend.  The `multisession` backend uses background R
workers running on the same machine similar to the cluster workers
created by `parallel::makeCluster()`.

If you prefer the Tidyverse style, the pipe operator (`|>`), and
functions like `map()` of the **purrr** package, there are parallel
versions in the **furrr** package.  For example,

```r
library(furrr)  
plan(multisession)

y <- X |> future_map(slow_fcn)
```


Alternatively, if you like the **foreach** style of iterating over
elements, you can use the **doFuture** package and its `%dofuture%`
operator.  For example,

```r
library(doFuture)
plan(multisession)

y <- foreach(x = X) %dofuture% {
  slow_fcn(x)
}
```

The above examples produce identical results to the corresponding
`lapply()` call.  Note how `plan()` controls which parallel backend is
used - the end-user can easily switch to an alternative backend with a
single change of settings.  For example, a user with SSH access to
additional local and remote machines can harness them by using:

```r
plan(cluster, workers = c("mydesktop", "anotherdesktop", "server.myuni.edu"))
```

There are other parallel backends available in the Futureverse,
e.g. **future.batchtools** can be used to parallelize via job
schedulers on high-performance compute (HPC) clusters, and
**future.callr** provides an alternative to `multisession`.  One of
the advantages of the future ecosystem is that all types of backends,
including those that will be developed in the future, will
automatically work without having to modify any code.

There are a few things to be aware of when running algorithms in
parallel. For example, variables and functions that exist on the
current machine have to be available on the parallel workers --
workers that may run on a remote machine across the world.  With
traditional techniques such as `parLapply()` of the **parallel**
package, it is the developer's responsibility to identify such objects
and make sure they are exported to each parallel worker upfront.  This
is not necessary when using futures. The future framework identifies
and exports required objects automatically, removing this often
tedious and error-prone task and helps keep the code tidy.

Another important thing to keep in mind when running code in parallel
is random number generation.  Some techniques we use in statistical
analysis require high-quality RNG, e.g. permutation tests and
bootstrapping. Without it, there is a risk that the results become
biased. R uses the Mersenne-Twister RNG (Matsumoto & Nishimura; 1998)
by default, which has good random properties when running
sequentially.  However, when running parallelly there is a risk that
such random numbers generated by one parallel worker are not
independent of those generated in another parallel worker. To address
this problem, special RNGs have been designed that works well also in
parallel processing, e.g. the L'Ecuyer-CMRG method (L'Ecuyer, 1999).
The Futureverse has built-in support for L'Ecuyer-CMRG RNG. When used,
any random numbers generated, and therefore also the results, are
numerically reproducible, regardless of which parallel backend and the
number of parallel workers being used. If the developer forgets to use
it, the future framework will detect the mistake and produce a warning
detailing that the results may not be valid and how to solve it.

Another advantage with futures is that output, messages, warnings and
errors work the same as when running sequentially. For more details,
and additional arguments for using futures to parallelize R code, see
Bengtsson (2021).


# Who are using Futureverse?

The uptake of the future ecosystem by the R community has grown
steadily since it was first released in 2015.  The **future** package
is now among the top-1% most downloaded packages on CRAN. Nearly 300
packages on CRAN and Bioconductor rely on it directly, and many more
rely on it indirectly -- often without the end-user even being aware.

Prominent examples where the future framework is used for parallel and
asynchronous processing are **Seurat** (Large-Scale Single-Cell
Genomics), **shiny** (Scalable, Asynchronous UX), **plumber** (An API
Generator for R), **mlr3** (Next-Generation Machine Learning),
**targets** (Pipeline Toolkit for Reproducible Computation at Scale),
and **EpiNow2** (Estimate Real-Time Case Counts and Time-Varying
Epidemiological Parameters).


# Summary

The goal of the Futureverse is that you should not have to spend much
time and mental efforts on getting your existing R code to run in
parallel.  In contrast to the more traditional approaches, you only
have to do minor rewrites to make code run in parallel and you can
stay with your favorite coding style.  The future framework aims at
making parallelization something that "just works", allowing you to
keep a focus on the analysis.  As a developer, you rarely know who
your end-users are and less so what their compute resources are.  One
user might want to parallelize on a laptop, whereas another one wants
to scale out a cloud service.  When using the Futureverse, you give
the end-user full control on how and where to parallelize.  Strict
requirements and validation of the future backends guarantee that your
code will work anywhere.

If you wish to learn more about the Futureverse, please visit
<https://www.futureverse.org>, which provides documentation, examples,
tutorials, presentations, and blog posts on how to use futures for
running R in parallel.


# Acknowledgments

The work on Futureverse was sponsored by the R Consortium
Infrastructure Steering Committee (ISC) and the Chan Zuckerberg
Initiative (CZI) Essential Open Source-Software (EOSS) program.


# References

* Bengtsson H. (2021). _A Unifying Framework for Parallel and
  Distributed Processing in R using Futures_, The R Journal, 13:2, 
  208-227. <https://doi.org/10.32614/RJ-2021-048>

* Friedman D. P. & Wise D. S. (1978). _Aspects of applicative programming for
  parallel processing_. IEEE Transactions on Computers,
  C-27(4), 289–296. <https://doi.org/10.1109/tc.1978.1675100>

* Gillespie C. and Lovelace R. (2016). _Efficient R programming_,
  O'Reilly Media, Inc. <https://bookdown.org/csgillespie/efficientR/>
  
* Hewitt C. & Baker H. G. (1977). _Laws for communicating parallel
  processes_. In IFIP Congress, 987–992.

* Hibbard P. (1976). _Parallel processing facilities_. In S. A. Schuman,
  editor, New Directions in Algorithmic Languages. IRIA.

* Kane M., Emerson J. W., & Weston S. (2013). _Scalable Strategies
  for Computing with Massive Data. Journal of Statistical Software_,
  55(14), 1–19. <https://doi.org/10.18637/jss.v055.i14>

* L'Ecuyer P. (1999). _Good parameters and implementations for combined
  multiple recursive random number generators_. Operations Research,
  47(1), 159–164. <https://doi.org/10.1287/opre.47.1.159>

* Matsumoto M. & Nishimura T. (1998). _Mersenne Twister: A
  623-dimensionally equidistributed uniform pseudo-random number
  generator_, ACM Transactions on Modeling and Computer Simulation, 8,
  3–30.

* Wickham H. (2019). _Advanced R_. 2nd ed. CRC
  Press. <https://adv-r.hadley.nz/>
