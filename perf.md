### CPU Sampling with `perf` and Flame Graphs

In this lab, you will experiment with CPU profiling of a simple application. The techniques you learn will be applicable to application profiling as well as profiling the entire system, will apply to user-mode and kernel-mode code, and will apply to any language or framework that has `perf` symbol support through the `/tmp/perf-$PID.map` convention.

- - -

#### Task 1: Compile the Profiled Application

The application you're about to profile is a very simple command-line tool that counts prime numbers in a given range, and uses [OpenMP](http://www.openmp.org) to parallelize the computation. But first, let's build it and make sure everything works. Navigate to the directory that contains [primes.c](primes.c) and build it using the following command:

```
$ gcc -g -fopenmp primes.c -o primes
```

Now, experiment with multiple executions of this application with varying numbers of threads:

```
$ for i in `seq 1 2 16`; do
>   OMP_NUM_THREADS=$i time ./primes
> done
```

Elapsed time should stabilize when you create enough threads to keep all the cores saturated throughout the entire run. Because the distribution of work between threads is not ideal, on an N-core system you will need more than N threads to keep them all busy.

- - -

#### Task 2: Profile with `perf`

It's time to find out what's taking so long to count our primes. Use the following command to get `perf`'s opinion. You will need a root shell for this.

```
# export OMP_NUM_THREADS=16
# perf record -ag -F 997 -- ./primes
```

> In the preceding command, the -a switch indicates that you want to capture samples on all CPUs; the -g switch indicates that you want the call stack to be captured; the -F switch indicates that your event of interest is CPU samples; and the number 997 is the rate of sampling -- 997 times per second.

To figure out where the bottleneck is, inspect `perf`'s report:

```
# perf report --stdio
```

It seems that we're spending a whole lot of time in the `is_divisible` function. If you want to know where exactly, you can use the following command:

```
# perf annotate
```

> Note that, as discussed in class, there may be skid in the displayed results. Notably, the specific instruction that seems to be taking most of the time might actually be masquerading for the previous or following instruction (or couple of instructions).

- - -

#### Task 3: Extract a Flame Graph

Now that you have the basic picture of what's happening in the application, it would be interesting to inspect everything else. The perf report is quite long, though:

```
# perf report --stdio | wc -l
1286
```

So instead of reading long reports, let's take flame graphs for a spin. If you haven't already, clone the [FlameGraph](https://github.com/BrendanGregg/FlameGraph) repository, and then run the following commands to post-process the perf report:

```
# perf script > primes.stacks
# FlameGraph/stackcollapse-perf.pl primes.stacks > primes.collapsed
# FlameGraph/flamegraph.pl primes.collapsed > primes.svg
```

Open the resulting flame graph and enjoy the results. It is now immediately evident that there are no significant stack tree branches other than the `is_prime` calling `is_divisible` branch that we have already seen.

- - -

#### Task 4: Flame Graphs for Java Code

TODO

- - -

#### Bonus: Some More Flame Graphs

For some more fun, experiment with collecting CPU samples and then generating flame graphs while the following commands are running:

1. `sha1sum /dev/zero`
1. `dd if=/dev/zero of=/dev/null bs=100K count=50K`

You might get a lot of `[unknown]` frames because of missing debuginfo. Try to obtain debuginfo for the missing components and then try again. If there's no choice, you can always build from source with the `-g` option to make sure debuginfo will be available.
