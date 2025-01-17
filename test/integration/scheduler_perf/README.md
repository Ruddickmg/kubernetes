# Scheduler Performance Test

This package contains the scheduler performance tests, often called scheduler_perf.
We use it for benchmarking the scheduler with in-tree plugins, which is visible at [perf-dash](https://perf-dash.k8s.io/#/?jobname=scheduler-perf-benchmark&metriccategoryname=Scheduler&metricname=BenchmarkPerfResults&Metric=SchedulingThroughput&Name=SchedulingBasic%2F5000Nodes%2Fnamespace-2&extension_point=not%20applicable&result=not%20applicable).
Also you can use it outside the Kubernetes repository with out-of-tree plugins by making use of `RunBenchmarkPerfScheduling`.

## Motivation

We already have a performance testing system -- Kubemark. However, Kubemark requires setting up and bootstrapping a whole cluster, which takes a lot of time.

We want to have a standard way to reproduce scheduling latency metrics result and benchmark scheduler as simple and fast as possible. We have the following goals:

- Save time on testing
  - The test and benchmark can be run in a single box.
    We only set up components necessary to scheduling without booting up a cluster.
- Profiling runtime metrics to find out bottleneck
  - Write scheduler integration test but focus on performance measurement.
    Take advantage of go profiling tools and collect fine-grained metrics,
    like cpu-profiling, memory-profiling and block-profiling.
- Reproduce test result easily
  - We want to have a known place to do the performance related test for scheduler.
    Developers should just run one script to collect all the information they need.

Currently the test suite has the following:

- benchmark
  - make use of `go test -bench` and report nanosecond/op.
  - schedule b.N pods when the cluster has N nodes and P scheduled pods. Since it takes relatively long time to finish one round, b.N is small: 10 - 100.

## How To Run

### Benchmark tests

```shell
# In Kubernetes root path
make test-integration WHAT=./test/integration/scheduler_perf ETCD_LOGLEVEL=warn KUBE_TEST_VMODULE="''" KUBE_TEST_ARGS="-run=^$$ -benchtime=1ns -bench=BenchmarkPerfScheduling"
```

The benchmark suite runs all the tests specified under config/performance-config.yaml.
By default, it runs all workloads that have the "performance" label. In the configuration,
labels can be added to a test case and/or individual workloads. Each workload also has
all labels of its test case. The `perf-scheduling-label-filter` command line flag can
be used to select workloads. It works like GitHub label filtering: the flag accepts
a comma-separated list of label names. Each label may have a `+` or `-` as prefix. Labels with
`+` or no prefix must be set for a workload for it to be run. `-` means that the label must not
be set. For example, this runs all performance benchmarks except those that are labeled
as "fast":
```shell
make test-integration WHAT=./test/integration/scheduler_perf ETCD_LOGLEVEL=warn KUBE_TEST_VMODULE="''" KUBE_TEST_ARGS="-run=^$$ -benchtime=1ns -bench=BenchmarkPerfScheduling -perf-scheduling-label-filter=performance,-fast"
```

Once the benchmark is finished, JSON file with metrics is available in the current directory (test/integration/scheduler_perf). Look for `BenchmarkPerfScheduling_benchmark_YYYY-MM-DDTHH:MM:SSZ.json`.
You can use `-data-items-dir` to generate the metrics file elsewhere.

In case you want to run a specific test in the suite, you can specify the test through `-bench` flag:

Also, bench time is explicitly set to 1ns (`-benchtime=1ns` flag) so each test is run only once.
Otherwise, the golang benchmark framework will try to run a test more than once in case it ran for less than 1s.

```shell
# In Kubernetes root path
make test-integration WHAT=./test/integration/scheduler_perf ETCD_LOGLEVEL=warn KUBE_TEST_VMODULE="''" KUBE_TEST_ARGS="-run=^$$ -benchtime=1ns -bench=BenchmarkPerfScheduling/SchedulingBasic/5000Nodes/5000InitPods/1000PodsToSchedule"
```

To produce a cpu profile:

```shell
# In Kubernetes root path
make test-integration WHAT=./test/integration/scheduler_perf KUBE_TIMEOUT="-timeout=3600s" ETCD_LOGLEVEL=warn KUBE_TEST_VMODULE="''" KUBE_TEST_ARGS="-run=^$$ -benchtime=1ns -bench=BenchmarkPerfScheduling -cpuprofile ~/cpu-profile.out"
```

### How to configure benchmark tests

Configuration file located under `config/performance-config.yaml` contains a list of templates.
Each template allows to set:
- node manifest
- manifests for initial and testing pod
- number of nodes, number of initial and testing pods
- templates for PVs and PVCs
- feature gates

See `op` data type implementation in [scheduler_perf_test.go](scheduler_perf_test.go) 
for available operations to build `WorkloadTemplate`.

Initial pods create a state of a cluster before the scheduler performance measurement can begin.
Testing pods are then subject to performance measurement.

The configuration file under `config/performance-config.yaml` contains a default list of templates to cover
various scenarios. In case you want to add your own, you can extend the list with new templates.
It's also possible to extend `op` data type, respectively its underlying data types
to extend configuration of possible test cases.

### Logging

The default verbosity is 2 (the recommended value for production). -v can be
used to change this. The log format can be changed with
-logging-format=text|json. The default is to write into a log file (when using
the text format) or stderr (when using JSON). Together these options allow
simulating different real production configurations and to compare their
performance.

During interactive debugging sessions it is possible to enable per-test output
via -use-testing-log.

### Integration tests

To run integration tests, use:
```
make test-integration WHAT=./test/integration/scheduler_perf KUBE_TEST_ARGS=-use-testing-log
```

Integration testing uses the same `config/performance-config.yaml` as
benchmarking. By default, workloads labeled as `integration-test`
are executed as part of integration testing (in ci-kubernetes-integration-master job).
`-test-scheduling-label-filter` can be used to change that.
All test cases should have at least one workload labeled as `integration-test`.

Running the integration tests with command above will only execute the workloads labeled as `short`.
`SHORT=--short=false` variable added to the command can be used to disable this filtering.

We should make each test case with `short` label very small,
so that all tests with the label should take less than 5 min to complete.
The test cases labeled as `short` are executed in pull-kubernetes-integration job.

### Labels used by CI jobs

| CI Job                           | Labels                 |
|----------------------------------|------------------------|
| ci-kubernetes-integration-master | integration-test       |
| pull-kubernetes-integration      | integration-test,short |
| ci-benchmark-scheduler-perf      | performance            |

## Scheduling throughput thresholds

Thresholds are used to capture scheduler performance regressions in a periodic ci-benchmark-scheduler-perf job. 
Most test cases have a threshold set for the largest `performance` workloads. 
By default, these are defined for the `Average` statistic of the `SchedulingThroughput` metric. 
It is possible to use other metric by configuring `thresholdMetricSelector` per test case or workload. 

### How to calculate the threshold

The initial values for scheduling throughput thresholds were calculated through an analysis of historical data, 
specifically focusing on the minimum, average, and standard deviation values for each workload 
(see [#126871](https://github.com/kubernetes/kubernetes/pull/126871)). 
Our goal is to set the thresholds somewhat pessimistically to minimize flakiness, 
so it's recommended to set the threshold slightly below the observed historical minimum. 
Depending on variability of data, the threshold can be lowered more. 

Thresholds should be adjusted based on the flakiness level and minima observed in the future. 
Remember to set the value for newly added test cases as well, 
but after collecting some data on workload characteristics.

### How to determine the failed workload

When the workload's scheduling throughput doesn't exceed the threshold, 
the ci-benchmark-scheduler-perf periodic job will fail with an error log such as:

```
--- FAIL: BenchmarkPerfScheduling/SchedulingBasic/5000Nodes_10000Pods
    ...
    scheduler_perf.go:1098: ERROR: op 2: SchedulingBasic/5000Nodes_10000Pods/namespace-2: expected SchedulingThroughput Average to be higher: got 256.12, want 270
```

This allows to analyze which workload failed. Make sure that the failure is not an outlier 
by checking multiple runs of the job. If the failures are not related to any regression, 
but to an incorrect threshold setting, it is reasonable to decrease it.
