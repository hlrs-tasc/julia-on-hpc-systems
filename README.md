# Julia on HPC systems

The purpose of this repository is to document best practices for running Julia
on HPC systems (i.e., "supercomputers"). At the moment, both information
relevant for supercomputer operators as well as users is collected here.
There is no guarantee for permanence or that information here is up-to-date,
neither for a useful ordering and/or categorization of issues.

## For operators


### Official Julia binaries vs. building from source
According to 
[this Discourse post](https://discourse.julialang.org/t/compiling-julia-using-lto-pgo/39168/5),
the difference between compiling Julia from source with architecture-specific
optimization and using the official Julia binaries is negligible. Since
installing from source using, e.g., Spack, can sometimes be cumbersome, the
general recommendation is to go with the pre-built binaries unless benchmarked
and found to be different.

*Last update: April 2022*


### Ensure correct libraries are loaded
When using Julia on a system that uses an environment-variable based module
system (such as [Lmod](https://github.com/TACC/Lmod)), the `LD_LIBRARY_PATH`
variable might be filled with entries pointing to different packages and
libraries. To avoid issues from Julia loading another library instead of the
ones packaged with Julia, make sure that Julia's `lib` directory is always the
*first* directory in `LD_LIBRARY_PATH`.

One possibility to achieve this is to create a wrapper shell script that
modifies `LD_LIBRARY_PATH` before calling the Julia executable. Inspired by a
[script](https://github.com/UCL-RITS/rcps-buildscripts/blob/04b2e2ccfe7e195fd0396b572e9f8ff426b37f0e/files/julia/julia.sh)
from UCL's [Owain Kenway](https://github.com/owainkenwayucl):
```shell
#!/usr/bin/env bash

# This wrapper makes sure the julia binary distributions picks up the GCC
# libraries provided with it correctly meaning that it does not rely on
# the gcc-libs version.

# Dr Owain Kenway, 20th of July, 2021
# Source: https://github.com/UCL-RITS/rcps-buildscripts/blob/04b2e2ccfe7e195fd0396b572e9f8ff426b37f0e/files/julia/julia.sh

location=$(readlink -f $0)
directory=$(readlink -f $(dirname ${location})/..)

export LD_LIBRARY_PATH=${directory}/lib/julia:${LD_LIBRARY_PATH}
exec ${directory}/bin/julia "$@"
```

Note that using `readlink` might not be optimal from a performance perspective
if used in a massively parallel environment. Alternatively, hard-code the Julia
path or set an environment variable accordingly.

*Last update: April 2022*


### Julia depot path
The Julia folder (typically `~/.julia`) may very well reside in the user's home
directory. In case multiple platforms share a single home directory, it might
make sense to make the depot path platform dependend by setting the
`JULIA_DEPOT_PATH` environment variable appropriately, e.g.,
```tcl
prepend-path JULIA_DEPOT_PATH $env(HOME)/.julia/$platform
```
where `$platform` contains the current system name
([source](https://gitlab.blaschke.science/nersc/julia/-/blob/e63483ff9d356128814571b181760acaaf16ecbe/modulefiles/templates/julia_cray_module.tcl#L45)).


## For users
tbd.

## Authors

* [Michael Schlottke-Lakemper](https://www.hlrs.de/about-us/organization/divisions-departments/av/tasc/) (University of Stuttgart, Germany)

## Acknowledgments

These people have provided input to this repository via private communication:
* Mosè Giordano ([@giordano](https://github.com/giordano))
* Johannes Blaschke ([@jblaschke](https://github.com/jblaschke))

## Disclaimer

Everything is provided as is and without warranty. Use at your own risk!
