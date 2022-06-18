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
optimization and using the official Julia binaries is negligible.
This has been confirmed by [Ludovic Räss](#acknowledgments) for an Nvidia DGX-1
system at CSCS, where also no performance differences between a Spack-installed
version and the official binaries were found (April 2022).

Since installing from source using, e.g., Spack, can sometimes be cumbersome,
the general recommendation is to go with the pre-built binaries unless
benchmarked and found to be different. This is also the current approach taken at NERSC, CSCS, and PC2.

*Last update: April 2022*


### Ensure correct libraries are loaded
When using Julia on a system that uses an environment-variable based module
system (such as [modules](https://github.com/cea-hpc/modules) or
[Lmod](https://github.com/TACC/Lmod)), the `LD_LIBRARY_PATH` variable might
be filled with entries pointing to different packages and libraries. To avoid
issues from Julia loading another library instead of the ones packaged with
Julia, make sure that Julia's `lib` directory is always the *first* directory in
`LD_LIBRARY_PATH`.

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

Also note that fixing the `LD_LIBRARY_PATH` variable does not seem to be a hard
requirement, since it is not used universally (e.g., it is not necessary on NERSC's systems).

*Last update: April 2022*


### Julia depot path
Since the available file systems can differ significantly between HPC centers, it is hard to make a general statement about where the Julia depot folder (by default on
Unix-like systems: `~/.julia`) should be placed (via [`JULIA_DEPOT_PATH`](https://docs.julialang.org/en/v1/manual/environment-variables/#JULIA_DEPOT_PATH)).
Generally speaking, the file system hosting the Julia depot should have
* good (parallel) I/O
* no tight quotas
* read and write access
* no mechanism for the automatic deletion of unused files (or the depot should be excluded as an exception)

On some systems, it resides in the user's home directory (e.g. at NERSC). On other systems, it is put on a parallel scratch file system (e.g. CSCS and PC2). At the time
of writing (April 2022), there does not seem to be reliable performance data
available that could help to make a data-based decision.

If multiple platforms, e.g., systems with different architecture, would access the same Julia depot, for example because the file system is shared, it might
make sense to create platform-dependend Julia depots by setting the
`JULIA_DEPOT_PATH` environment variable appropriately, e.g.,
```
prepend-path JULIA_DEPOT_PATH $env(HOME)/.julia/$platform
```
where `$platform` contains the current system name
([source](https://gitlab.blaschke.science/nersc/julia/-/blob/e63483ff9d356128814571b181760acaaf16ecbe/modulefiles/templates/julia_cray_module.tcl#L45)).


### MPI.jl
It is generally recommended to set
```
JULIA_MPI_BINARY=system
```
such that MPI.jl will always use a system MPI instead of the Julia artifact (i.e. [MPI_jll.jl](https://github.com/JuliaParallel/MPI.jl)). For more configuration options see [this part](https://juliaparallel.org/MPI.jl/latest/configuration/#environment_variables) of the MPI.jl documentation.

Additionally, on the NERSC systems, there is a pre-built MPI.jl for each programming
environment, which is loaded through a settings module. More information on the
NERSC module file setup can be found [here](#modules-file-setup).


### CUDA.jl
It seems to be generally advisable to set the environment variables
```bash
JULIA_CUDA_USE_BINARYBUILDER=false
JULIA_CUDA_USE_MEMORY_POOL=none
```
in the module files when loading Julia on a system with GPUs. Otherwise, Julia
will try to download its own BinaryBuilder.jl-provided CUDA stack, which is
typically *not* what you want on a production HPC system. Instead, you should
make sure that Julia finds the local CUDA installation by setting relevant
environment variables (see also the
[CUDA.jl docs](https://cuda.juliagpu.org/stable/installation/troubleshooting/#Could-not-find-a-suitable-CUDA-installation)).
Disabling the memory pool is advisable to make CUDA-aware MPI work on multi-GPU
nodes (see also the [MPI.jl docs](https://juliaparallel.org/MPI.jl/latest/knownissues/#Hints-to-ensure-CUDA-aware-MPI-to-be-functional)).


### Modules file setup
[Johannes Blaschke](https://github.com/jblaschke) provides scripts and
templates to set up modules file for Julia on some of NERSC's systems:<br>
https://gitlab.blaschke.science/nersc/julia/-/tree/main/modulefiles

There are a number of environment variables that should be considered to be set
through the module mechanism:
* [`JULIA_DEPOT_PATH`](#julia-depot-path): Ensure depot path is on the correct file
                                           system
* [`JULIA_MPI_BINARY`](#mpijl): Use system-provided MPI backend
* [`JULIA_CUDA_USE_BINARYBUILDER`](#cudajl): Use system-provided CUDA stack
* [`JULIA_CUDA_USE_MEMORY_POOL`](#cudajl): Make CUDA-aware MPI work


### Easybuild resources
[Samuel Omlin](https://github.com/omlins) and colleagues from CSCS provide their
Easybuild configuration files used for Piz Daint online at
https://github.com/eth-cscs/production/tree/master/easybuild/easyconfigs/j/Julia. For example, there are configurations available for
[Julia 1.7.2](https://github.com/eth-cscs/production/blob/d6d0802746db0dcdea1cf1711ae1d21858058fcf/easybuild/easyconfigs/j/Julia/Julia-1.7.2-CrayGNU-21.09.eb)
and for
[Julia 1.7.2 with CUDA support](https://github.com/eth-cscs/production/blob/d6d0802746db0dcdea1cf1711ae1d21858058fcf/easybuild/easyconfigs/j/Julia/Julia-1.7.2-CrayGNU-21.09-cuda.eb).
Looking at these files also helps to decide which kind of environment variables
are useful to set.


### Further resources
* There is a lengthy discussion on the Julia Discourse about how to set up a
  centralized Julia installation. Some of it is already dated (probably), but
  it gives a good overview of some best practices and about approaches that work
  (and some which do not). In particular, the summary from CSCS is very helpful:<br>
  https://discourse.julialang.org/t/how-does-one-set-up-a-centralized-julia-installation/13922/32
* NERSC's [Johannes Blaschke](https://github.com/jblaschke) has a nice repository set up with lots
  of scripts and helpful information on setting up Julia on Cori and
  Perlmutter:<br>
  https://gitlab.blaschke.science/nersc/julia/-/tree/main


## For users
### HPC systems with Julia support
The following is an (incomplete) list of HPC systems that provide a Julia
installation and/or support for using Julia to its users:

**Center** | **System** | **Installation** | **Support** | **Interactive** | **Architecture** | **Accelerators** | **Documentation**
-----|-----|-----|-----|-----|-----|-----|-----
[CSCS](https://www.cscs.ch) | [Piz Daint](https://www.cscs.ch/computers/piz-daint/) | ✅ | ✅ | ✅ | [Intel Xeon Broadwell + Haswell](https://www.cscs.ch/computers/piz-daint/) | [Nvidia Tesla P100](https://www.cscs.ch/computers/piz-daint/) | [1](https://user.cscs.ch/tools/interactive/julia/)
[HLRS](https://www.hlrs.de) | [Hawk](https://www.hlrs.de/systems/hpe-apollo-hawk/) | ✅ | ✅ | ❌ | [AMD EPYC Rome](https://www.hlrs.de/systems/hpe-apollo-hawk/) | [Nvidia Tesla A100](https://www.hlrs.de/systems/hpe-apollo-hawk/) | [1](https://kb.hlrs.de/platforms/index.php/Julia)
[NERSC](https://www.nersc.gov) | [Cori](https://www.nersc.gov/systems/cori/) | ✅ | ? | ? | [Intel Xeon Haswell](https://docs.nersc.gov/systems/cori/#system-specification) | [Intel Xeon Phi](https://docs.nersc.gov/systems/cori/#system-specification) | [1](https://docs.nersc.gov/development/languages/julia/)
[NERSC](https://www.nersc.gov) | [Perlmutter](https://www.nersc.gov/systems/perlmutter/) | ✅ | ✅ | ? | [AMD EPYC Milan](https://docs.nersc.gov/systems/perlmutter/system_details/#cpus) | [Nvidia Ampere A100](https://docs.nersc.gov/systems/perlmutter/system_details/#gpus) | [1](https://docs.nersc.gov/development/languages/julia/), [2](https://docs.nersc.gov/performance/readiness/#julia)
[PC2, U Paderborn](https://pc2.uni-paderborn.de/) | [Noctua 1](https://pc2.uni-paderborn.de/hpc-services/available-systems/noctua1) | ✅ | ✅ | ✅ | [Intel Xeon Skylake](https://pc2.uni-paderborn.de/hpc-services/available-systems/noctua1) | [Intel Stratix 10](https://pc2.uni-paderborn.de/hpc-services/available-systems/noctua1) + consumer GPUs | [1](https://uni-paderborn.atlassian.net/wiki/spaces/PC2DOK/pages/12878307/Julia)
[PC2, U Paderborn](https://pc2.uni-paderborn.de/) | [Noctua 2](https://pc2.uni-paderborn.de/hpc-services/available-systems/noctua2) | ✅ | ✅ | ✅ | [AMD EPYC Milan](https://pc2.uni-paderborn.de/hpc-services/available-systems/noctua2) | [Nvidia Ampere A100](https://pc2.uni-paderborn.de/hpc-services/available-systems/noctua2), [Xilinx Alveo U280](https://pc2.uni-paderborn.de/hpc-services/available-systems/noctua2) | [1](https://uni-paderborn.atlassian.net/wiki/spaces/PC2DOK/pages/12878307/Julia)
[ARC, UCL](https://www.ucl.ac.uk/arc) | [Myriad](https://www.rc.ucl.ac.uk/docs/Clusters/Myriad/), [Kathleen](https://www.rc.ucl.ac.uk/docs/Clusters/Kathleen/), [Michael](https://www.rc.ucl.ac.uk/docs/Clusters/Michael/), [Young](https://www.rc.ucl.ac.uk/docs/Clusters/Young/) | ✅ | ✅ | ? | various Intel Xeon | various GPUs | [1](https://www.rc.ucl.ac.uk/docs/)
[ZDV, U Mainz](https://hpc-en.uni-mainz.de/) | [MOGON II](https://hpc-en.uni-mainz.de/high-performance-computing/systeme/) | ✅ | ? | ? | [Intel Xeon Broadwell + Skylake](https://hpc-en.uni-mainz.de/high-performance-computing/systeme/) | no | [1](https://mogonwiki.zdv.uni-mainz.de/dokuwiki/start:development:scripting_languages:julia)
[HPC2N, Umeå U](https://www.hpc2n.umu.se/) | [Kebnekaise](https://www.hpc2n.umu.se/resources/hardware/kebnekaise) | ✅ | ✅ | ? | [Intel Xeon Broadwell + Skylake](https://www.hpc2n.umu.se/resources/hardware/kebnekaise) | [Nvidia Tesla K80](https://www.hpc2n.umu.se/resources/hardware/kebnekaise), [Nvidia Tesla V100](https://www.hpc2n.umu.se/resources/hardware/kebnekaise) | [1](https://www.hpc2n.umu.se/resources/software/julia)
[NeSI](https://www.nesi.org.nz/) | [Mahuika, Māui](https://www.nesi.org.nz/services/high-performance-computing-and-analytics/platforms) | ✅ | ✅ | ✅ | [Intel Xeon Broadwell/Cascade Lake + AMD EPYC Milan](https://www.nesi.org.nz/services/high-performance-computing-and-analytics/platforms) | [Nvidia Tesla P100, A100](https://support.nesi.org.nz/hc/en-gb/articles/360001471955) | [1](https://support.nesi.org.nz/hc/en-gb/articles/360001175895-Julia)


**Nomenclature:**
* *Center:* The HPC center's name
* *System:* The compute system's "marketing" name
* *Installation:* Is there a pre-installed Julia configuration available?
* *Support:* Is Julia "officially" supported on the system, i.e., will Julia
users be supported by HPC center staff if they have questions/problems?
* *Interactive:* Is interactive computing with Julia supported, i.e., can you run parallel jobs on the system interactively via, e.g., Jupyter notebooks?
* *Architecture:* The main CPU used in the system
* *Accelerators:* The main accelerator (if anything) in the system
* *Documentation:* Links to documentation for Julia users


## License and contributing

The contents of this repository are published under the MIT license (see
[LICENSE](LICENSE)). Our main goal is to publicly curate information on
using Julia on HPC systems, as a service *from* the community and *for* the
community. Therefore, we are very happy to accept contributions from everyone,
preferably in the form of a PR.


## Authors

This repository is maintained by
[Michael Schlottke-Lakemper](https://www.hlrs.de/about-us/organization/divisions-departments/av/tasc/)
(University of Stuttgart, Germany).

The following people have provided valuable contributions, either in the form of
PRs or via private communication:
* Johannes Blaschke ([@jblaschke](https://github.com/jblaschke))
* Carsten Bauer ([@carstenbauer](https://github.com/carstenbauer))
* Valentin Churavy ([@vchuravy](https://github.com/vchuravy))
* Mosè Giordano ([@giordano](https://github.com/giordano))
* Ludovic Räss ([@luraess](https://github.com/luraess))
* Pedro Ojeda ([@pojeda](https://github.com/pojeda))
* Samuel Omlin ([@omlins](https://github.com/omlins))


## Disclaimer

Everything is provided as is and without warranty. Use at your own risk!
