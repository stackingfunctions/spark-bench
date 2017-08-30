
# Understanding `spark-bench`

`spark-bench` is a flexible system for benchmarking and simulating Spark jobs. 

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Summary](#summary)
- [Data Generation](#data-generation)
- [Workloads](#workloads)
  - [Types of Workloads](#types-of-workloads)
  - [Custom Workloads](#custom-workloads)
  - [Parameters](#parameters)
  - [Output vs. WorkloadResultsOutput](#output-vs-workloadresultsoutput)
- [Suites](#suites)
- [Spark-Submit-Configs](#spark-submit-configs)
- [Levels and Combinations of Parallelism](#levels-and-combinations-of-parallelism)
  - [Minimal spark-bench Config File](#minimal-spark-bench-config-file)
  - [Classic Benchmarking](#classic-benchmarking)
  - [Classic Benchmarking Across Systems](#classic-benchmarking-across-systems)
  - [Same Algorithm, Different Spark Settings](#same-algorithm-different-spark-settings)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

`spark-bench` can do two main things:
1. Run workloads in a highly configurable fashion
2. Generate data

Users can interact with spark-bench through a CLI (with limited functionality) or a configuration (with full functionality). 

Users configure the way their jobs run by defining `spark-submits`, `suites`, and `workloads` in a nested structure.

## Data Generation

`spark-bench` has the capability to generate data according to many different configurable generators. 
Generated data can be written to any storage addressable by Spark, including local files, hdfs, S3, etc.

Data generators are run just like workloads in spark-bench. Users should exercise caution to ensure that data generation happens before the workloads that need that input run.
This is fairly simple to ensure in most cases.
However, if in doubt, a bullet-proof way to do this is to create two different configuration files, one for your data generation and one with your workloads, and run them each through spark-bench.

 
## Workloads

The atomic unit of organization in `spark-bench` is the workload. Workloads are standalone Spark jobs that read their input data, if any,
from disk, and write their output, if the user wants it, out to disk.

Some workloads are designed to exercise a particular algorithm implementation or a particular method. Others are designed to 
simulate Spark use cases such as multiple notebook users hitting a single Spark cluster.

### Types of Workloads
Some existing categories of workloads include:
- ML workloads: Logistic Regression, KMeans, etc.
- "Exercise" workloads: designed to examine one particular portion of the Spark pipeline. A good example is SparkPi, a very compute-heavy workload with no need to for disk IO.
- Simulation workloads: designed to simulate Spark usage cases such as notebooks

### Custom Workloads
Users can easily create and run their own workloads by implementing the `Workload` trait and placing the resulting JAR in the classpath.  For details, see the [developer guide](developer-guide.md).

### Parameters
| Name        | Required (y/n)| Description  |
| ------------- |-------------| -----|
| name       | yes | The name/type of workload. For example, "kmeans", "sparkpi", "logisticregression"... | 
| input      | yes, for some | The path (local, hdfs, S3, etc.) to the input dataset for the workload. Some workloads (ex: SparkPi) do not require input. |
| workloadresultsoutput | no | If the user wishes to keep the output of the workload (ex: the value of pi generated by SparkPi), they can specify a path here. |
| arguments specific to the workload | depends | Most workloads provide configuration arguments. Ex: Value of K for KMeans. |

### Output vs. WorkloadResultsOutput

***Output:*** The timing results, workload config, system config, and other benchmarking info. One row per workload.

***WorkloadResultsOutput:*** The output generated by the workload itself. Examples: the data returned by a SQL query. 
The clusters generated by running KMeans. The model generated by running Logistic Regression.

In the configuration file, paths for ***output*** are set in the Suite (see more below). 
Paths for ***workloadresultsoutput*** paths are set in the workloads.

## Suites

Suites are collections of one or more workloads. The workloads in a suite can be run serially or in parallel.

Suites serve the following functions:
- collect the benchmark output info from each run of each workload and write in one common file (or in the console)
- control whether workloads are run serially or kicked off in parallel from a thread pool
- control how many times each workload is run. Many times it is advantageous to run a workload multiple times, particularly for benchmarking. Suites let you do that all in one place.

Suites themselves can be run serially or in parallel.

## Spark-Submit-Configs

`spark-bench` allows you to launch multiple spark-submit commands by creating and launching multiple spark-submit scripts.
This can be advantageous in a number of situations. To name just a few:

- Comparing benchmark times of the same workloads with different Spark settings
- Simulating multiple batch applications hitting the same cluster at once.
- Comparing benchmark times against two different Spark clusters!

Just like suites and workloads, spark-submit-config can be launched serially or in parallel.

## Levels and Combinations of Parallelism

There are many, many different ways of controlling parallelism in `spark-bench`.

You can control parallelism at the level of the spark-submit-config, and/or the suite, and/or the workload.

This is a not the easiest thing to understand, so let's try to understand by example. Here, we'll highlight a few key use cases as a way of illustrating different options.

### Minimal spark-bench Config File

A spark-bench config file only needs one workload defined to work, but it must also have the other structures as well.

```hocon
spark-bench = {
  spark-submit-config = [{
    workload-suites = [
      {
        descr = "One run of SparkPi and that's it!"
        benchmark-output = "console"
        workloads = [
          {
            name = "sparkpi"
            slices = 10
          }
        ]
      }
    ]
  }]
}
```

When I run `./bin/spark-bench.sh examples/from-docs/minimal-example.conf` from my spark-bench distribution file, I get the following output in my terminal: 

```
One run of SparkPi and that's it!                                               
+-------+-------------+-------------+-----------------+-----+------------------------+------+---+-----------------+-----------------+----------------------------+--------------------+--------------------+-----------------+-----------------------+------------+-------------------+--------------------+
|   name|    timestamp|total_runtime|   pi_approximate|input|workloadResultsOutputDir|slices|run|spark.driver.host|spark.driver.port|hive.metastore.warehouse.dir|          spark.jars|      spark.app.name|spark.executor.id|spark.submit.deployMode|spark.master|       spark.app.id|         description|
+-------+-------------+-------------+-----------------+-----+------------------------+------+---+-----------------+-----------------+----------------------------+--------------------+--------------------+-----------------+-----------------------+------------+-------------------+--------------------+
|sparkpi|1498683099328|   1032871662|3.141851141851142|     |                        |    10|  0|     10.200.22.54|            61657|        file:/Users/ecurt...|file:/Users/ecurt...|com.ibm.sparktc.s...|           driver|                 client|    local[2]|local-1498683099078|One run of SparkP...|
+-------+-------------+-------------+-----------------+-----+------------------------+------+---+-----------------+-----------------+----------------------------+--------------------+--------------------+-----------------+-----------------------+------------+-------------------+--------------------+
```

Where did all these output fields come from? Let's break them down.

- **`name`, `input`, `workloadResultsOutputDir`, `slices`, `description`:** These are parameters from our configuration 
file. `input` and `workloadResultsOutputDir` came from the defaults define for SparkPi, which are None and None as SparkPi doesn't require any input and only has one number as a result.
- **`timestamp`, `total_runtime`, `pi_approximate`:** These are output fields from SparkPi. The timestamp is the start 
time when actual workload began. The total runtime is how long the workload took. In some workloads, the total runtime 
will be a composite of other runtime numbers; in the case of SparkPi, it's just one.
- **`run`:** The index of the run. In this case, we only ran SparkPi once, so this one-run is run #0, because we're good
computer scientists and like things to be 0-indexed.
- **`spark.master`** Because I didn't define a master in my configuration file, this came from the environment variable SPARK_MASTER_HOST.
- **Other Spark Settings:** Because I didn't specify any options in the config, these came from my default settings in my Spark installation. 
If I had specified, say, a different driver port in the config, that would have overridden the default.

I am not specifying any parallelism settings at any level of my config, not that it would matter if I was because there's only one way to run one workload one time.
The default setting for all parallelism options is `false`, meaning that it will not be parallel.

### Classic Benchmarking

Classic benchmarking involves running of collection of methods in order to get timing numbers on each. To have statistically significant results, 
it's best to run each method several times and analyze the results.

For classic benchmarking, users will probably want one spark context containing one suite that runs single instances of different workloads serially. 

```hocon
spark-bench = {

  spark-submit-parallel = false
  spark-submit-config = [{
    spark-args = {
      master = "yarn"
    }
    suites-parallel = false
    workload-suites = [
      {
        descr = "Generating data for the benchmarks to use"
        parallel = false
        repeat = 1 // generate once and done!
        benchmark-output = "console"
        workloads = [
          {
            name = "data-generation-kmeans"
            output = "/tmp/spark-bench-test/kmeans-data.parquet"
            rows = 1000000
            cols = 14
          },
          {
            name = "data-generation-logistic-regression"
            output = "/tmp/spark-bench-test/logistic-regression.parquet"
            rows = 1000000
            cols = 14
          }
        ]
      },
      {
        descr = "Classic benchmarking"
        parallel = false
        repeat = 10 // lots of repeating here because we want statistically valid results 
        benchmark-output = "console"
        workloads = [
          {
            name = "kmeans"
            input = "/tmp/spark-bench-test/kmeans-data.parquet"
            // not specifying any kmeans arguments as we want the defaults for benchmarking
          },
          {
            name = "logisticregression"
            input = "/tmp/spark-bench-test/logistic-regression.parquet"
            // again, not specifying arguments
          },
          // ...more workloads
        ]
      }
    ]
  }]
}
```

Running workloads in parallel here would compromise the integrity of the timings. 
Similarly, running multiple suites in parallel, even if the workloads were serial, would result in two sets of serial
workloads being run in parallel. 

### Classic Benchmarking Across Systems

There are infinite variations on classic benchmarking. A common one might be running the same benchmarks against two different clusters.

```hocon
spark-bench = {

  spark-submit-parallel = true //since this is going against two different clusters, may as well run them in parallel!
  spark-submit-config = [{
    spark-args = {
      master = "spark://10.0.0.1:7077"
    }
    suites-parallel = false
    workload-suites = [
      {
        descr = "Classic benchmarking across systems"
        parallel = false
        repeat = 10 
        benchmark-output = "console"
        workloads = [
          // workloads...
        ]
      }
    ]
  },
  {
    spark-args = {
      master = "spark://10.0.0.2:7077" // different cluster!
    }
    suites-parallel = false
    workload-suites = [
    {
      descr = "Classic benchmarking across systems"
      parallel = false
      repeat = 10 
      benchmark-output = "console"
      workloads = [
        // workloads...
      ]
    }]
  }]
}
```

### Same Algorithm, Different Spark Settings

```hocon
spark-bench = {

  spark-submit-parallel = false
  spark-submit-config = [{
    spark-args = {
      master = "spark://10.0.0.1:7077"
      executor-mem = "128M"
    }
    suites-parallel = false
    workload-suites = [
    {
      descr = "Spark Pi at 128M executor-mem"
      parallel = false
      repeat = 10 // lots of repeating here because we want statistically valid results 
      benchmark-output = "console"
      workloads = [
        {
          name = "sparkpi"
          slices = [10, 100, 1000]
        }
      ]
    }]
  },
  {
    spark-args = {
      master = "spark://10.0.0.1:7077"
      executor-mem = "8G"
    }
    suites-parallel = false
    workload-suites = [
    {
      descr = "SparkPi at 8G executor-mem"
      parallel = false
      repeat = 10 // lots of repeating here because we want statistically valid results 
      benchmark-output = "console"
      workloads = [
        {
          name = "sparkpi"
          slices = [10, 100, 1000]
        }
      ]
    }]
  }]
}
```