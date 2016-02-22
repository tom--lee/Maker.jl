# Make.jl: A tool like make for Julia

This repository is an **experimental** approach to process data using
dependencies in a similar fashion to
[make](https://en.wikipedia.org/wiki/Makefile) or
[Rake](http://docs.seattlerb.org/rake/). Like make, Make.jl evaluates file and
code dependencies to update processing. The focus here is on data analysis, not
build automation for code. The features to support data analysis include:

- *Detection of code changes*--If part of the code changes, downstream 
  dependencies are updated during processing.
- *Variable targets*--Most make-like tools rely mainly on files for 
  dependency analysis. In Make.jl, Julia variables can also be used
  in this role. That allows more fine-grained control of code and dependencies. 
 
Make.jl was derived from [Jake.jl](https://github.com/nolta/Jake.jl) by Mike
Nolta. [Juke.jl](https://github.com/kshramt/Juke.jl) is another Make-like tool
written in Julia. The API here is most similar to Ruby's
[Rake](http://docs.seattlerb.org/rake/).

Most build tools like make use a Makefile or other build-instruction file. In
Make.jl, a separate build-script can be used, but the API right now focuses on
use within Julia script files.

## Example

The following shows a brief example of how to define file and variable 
dependencies for data processing.

```julia
Make.file("in1.csv")    # These are the input files.
Make.file("in2.csv")

# Two input files are processed to produce the file df.csv.
# "df.csv" is the name of the target and the name of the file
# associated with the target. "in1.csv" and "in2.csv" are the 
# dependencies for "df.csv". If either of the inputs are 
# changed (newer timestamp), this target will run.
Make.file("df.csv", ["in1.csv", "in2.csv"]) do 
    df = readtable("in1.csv")
    df = vcat(df, readtable("in2.csv"))
    writetable("df.csv", df) 
end

# This is a variable target. It creates a global variable
# `df` based on the result of this calculation. This target
# takes its timestamp from the timestamp of the "df.csv" 
# dependency.
Make.variable("df", "df.csv") do 
    println("Reading `df`.")
    readtable("df.csv")
end

# This simple target defines which dependencies to check
# when `make()` is run.
Make.task("default", "df.csv")

make()  # Run the "default" target.
```

See [here](https://github.com/tshort/Make.jl/blob/master/test/runtests.jl) for a
longer version of this example. In the example above, do-syntax is used. These
use anonymous functions to define actions. Normal, generic functions can also be 
used.

## API 

The main API here is rather basic:

* `make(task="default")` -- Process the target `task` by evaluating dependencies
  that need to be updated. "default" is the name of the default task.

* `Make.file`, `Make.task`, and `Make.variable` -- These methods all define
  tasks or targets to be updated. Each task has a name, zero or more
  dependencies, and an action as follows:
    
```julia 
Make.task(action::Function, name::AbstractString, dependencies::Vector{AbstractString})
```
Normally with the `file` target, the `action` will perform an operation to create
or update the file of the given name. With the `variable` target, the result
of the action is assigned to a variable with the specified name. The `task`
target is a generic target that can be used for general processing and to 
connect dependencies (like the PHONY target in a Makefile). All dependencies
(also called prerequisites) must be satisfied before running the action.

## Discussions

Here are some miscellaneous questions and open issues on this approach:

- As implemented, each function applied to a target takes no arguments. Rake
  and Juke.jl pass the target as an argument at least for some types of targets.
  Zero arguments is easier and looks cleaner with do syntax, but it might be
  useful in some cases to have the target as an argument.
  
- Right now, there's no way to enter "rules". Given the focus on code for data
  analytics, I'm not sure how critical this is.
  
- I'm not sure what to provide for iteration over tasks and targets. This 
  might include wildcards for files. Filename extension utilities might also
  help.

- It'd be nice not to have to explicitly specify files used as dependencies. 
  Maybe if the file by that name exists and the string isn't currently a target,
  we could auto-add it as a target.
  
- Right now, all targets are named with strings. It might be nice to allow
  symbols for variable and task targets. Rake and Juke.jl do this.

- Parallel operation may be tricky. One may have to be careful using 
  `@everywhere` and friends with Make.jl. Each process might try to update 
  dependencies at the same time, leading to race conditions. Another topic to
  think about is how to take advantage of parallel operations (or threads when 
  they come to Julia). 
  
- API to help with debugging might help.
    

  
