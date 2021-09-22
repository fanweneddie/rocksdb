# RocksDB

[toc]

## Install

### Environment

* CPU: 8 * Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz 
* CPU cache: 6144 KB
* OS: ubuntu 20.04
* kernel: 5.4.0-84-generic
* gcc: 9.3.0
* RocksDB: 6.24

To be more specific, they are 

```
RocksDB:    version 6.24
Date:       Wed Sep 22 23:26:32 2021
CPU:        8 * Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz
CPUCache:   6144 KB
Keys:       16 bytes each (+ 0 bytes user-defined timestamp)
Values:     100 bytes each (50 bytes after compression)
Entries:    1000000
Prefix:    0 bytes
Keys per prefix:    0
RawSize:    110.6 MB (estimated)
FileSize:   62.9 MB (estimated)
Write rate: 0 bytes/second
Read rate: 0 ops/second
Compression: Snappy
Compression sampling rate: 0
Memtablerep: SkipListFactory
Perf Level: 1
```

### Step

1. Get the repository

   ```
   $ git clone https://github.com/facebook/rocksdb.git
   ```

2. Go to the directory and compile

   ```
   $ cd rocksdb
   $ make -jNCPU db_bench DEBUG_LEVEL=0
   ```

   

**Plus**: Don't use `cmake` because it only supports win64, or it will have the errors below:

1. If we meet this error when we are running `cmake`,

```
-- Could NOT find gflags (missing: gflags_DIR)
CMake Error at /usr/local/share/cmake-3.21/Modules/FindPackageHandleStandardArgs.cmake:230 (message):
  Could NOT find gflags (missing: GFLAGS_LIBRARIES GFLAGS_INCLUDE_DIR)
Call Stack (most recent call first):
  /usr/local/share/cmake-3.21/Modules/FindPackageHandleStandardArgs.cmake:594 (_FPHSA_FAILURE_MESSAGE)
  cmake/modules/Findgflags.cmake:15 (find_package_handle_standard_args)
  CMakeLists.txt:136 (find_package)
```

probably it is because we haven't installed `gflags`. 

So the solution is to simply run

```
$ sudo apt-get install libgflags-dev
```

2.  When I run

```
$ make install
```

I encounter this bug:

```
[100%] Built target db_stress
Install the project...
-- Install configuration: "Debug"
-- Installing: /usr/local/rocksdb/include/rocksdb
CMake Error at cmake_install.cmake:46 (file):
  file INSTALL cannot make directory "/usr/local/rocksdb/include/rocksdb": No
  such file or directory.
```

That's because we have to install the head files into `/usr/local/rocksdb/` and it requires the superuser privilege.

3. when I run 

   ```
   ./db_bench --benchmarks="fillseq"
   ```

   in directory `build`, it will have an error

   ```
   Open error: Invalid argument: Compression type Snappy is not linked with the binary
   ```

   I try everything searched in github, but I still can only use `--compression_type=none` as a compromise. No other way to fix this error!

   That is some of my solution, but in vain :-(

   1. I did not install snappy for compression. So just run

      ```
      $ sudo apt-get install libsnappy-dev
      ```

   2. I should turn on the option of `build with SNAPPY` in `CMakeLists.txt`) and cmake again. That is, we should change

      ```
      option(WITH_SNAPPY "build with SNAPPY" OFF)
      ```

      to

      ```
      option(WITH_SNAPPY "build with SNAPPY" ON)
      ```


## Test

I use `db_bench` to test the performance of `rocksdb`.

### fill

#### fillseq

##### default

Sequential, Asynchronous

I run the command of writing N values in **sequential** key order in **async** mode

```
$ ./db_bench --benchmarks="fillseq,stats" --statistics
```

and the brief result is

```
------------------------------------------------
Initializing RocksDB Options from the specified file
Initializing RocksDB Options from command-line flags
DB path: [/tmp/rocksdbtest-1000/dbbench]
fillseq      :       2.759 micros/op 362390 ops/sec;   40.1 MB/s
```

The size of key is 16 bytes and the size of value is 100 bytes.

There are 1000000 entries. The total time is 2.8 s.

And for the info of compaction stats,  only level `L0` is involved, with the statistics

* file

  * file number: 1

  * file size: 29.65 MB

* write

  * W-Amp: 1
  * Wr: 103.5 MB/s

* compare

  * Comp: 0.29 s
  * CompMergeCPU: 0.23 s
  * Comp(cnt): 1

I am not sure why does it need to compare and merge since they are in level `L0` **:(**

Also, the percentiles of range is like

```
Percentiles: P50: 2.00 P75: 4.00 P99: 42.00 P99.9: 42.00 P99.99: 42.00
------------------------------------------------------
[       0,       1 ]        1  25.000%  25.000% #####
(       1,       2 ]        1  25.000%  50.000% #####
(       3,       4 ]        1  25.000%  75.000% #####
(      34,      51 ]        1  25.000% 100.000% #####
```

**Is there only 4 files in total???**

##### more data

I add the number of KV entries and run

```
$ ./db_bench --benchmarks="fillseq,stats" --statistics --num=10000000
```

Then, it takes 26.9 s to finish, almost 10 times.

The brief result is that

```
fillseq      :       2.686 micros/op 372298 ops/sec;   41.2 MB/s
```

Compared with less data inserted, **the throughput is almost similar**.

Level `L0`, `L1`, `L2` are entailed. The important info is

* `L0`: 3 files, total file size is 88.97 MB, W-Amp is 1, Wr speed is 102.4 MB/s. Comp takes 6.08s where CompMergeCPU takes 5.21 s.
* `L1`: 8 files, total file size is 237.26 MB, Moved data is 0.5 GB.
* `L2`: 10 files, total file size is 622.79 MB, Moved data is 0.3 GB 

This is level0 read latency histogram:

```
Count: 84 Average: 11.1071  StdDev: 14.26
Min: 1  Median: 4.1176  Max: 47
Percentiles: P50: 4.12 P75: 15.00 P99: 47.00 P99.9: 47.00 P99.99: 47.00
------------------------------------------------------
[       0,       1 ]       29  34.524%  34.524% #######
(       1,       2 ]       12  14.286%  48.810% ###
(       4,       6 ]       17  20.238%  69.048% ####
(       6,      10 ]        4   4.762%  73.810% #
(      10,      15 ]        1   1.190%  75.000% 
(      22,      34 ]       12  14.286%  89.286% ###
(      34,      51 ]        9  10.714% 100.000% ##
```

#### fillrandom

##### default

Random, Asynchronous

I run the command of writing N values in **random** key order in **async** mode

```
$ ./db_bench --benchmarks="fillrandom,stats" --statistics
```

The brief result is

```
fillrandom   :       4.620 micros/op 216428 ops/sec;   23.9 MB/s
```

And it takes 4.6 s

Only level `L0` is entailed, with the statistics:

* file

  * file number: 1

  * file size: 24.13 MB

* write

  * W-Amp: 1
  * Wr: 77.2 MB/s

* compare

  * Comp: 0.63 s
  * CompMergeCPU: 0.53 s
  * Comp(cnt): 2

So in random mode, Not only does writing slow down, but also it requires more time for comparing and merging.

This is level `L0` read latency histogram

```
** Level 0 read latency histogram (micros):
Count: 8 Average: 9.6250  StdDev: 11.13
Min: 1  Median: 2.0000  Max: 29
Percentiles: P50: 2.00 P75: 10.00 P99: 29.00 P99.9: 29.00 P99.99: 29.00
------------------------------------------------------
[       0,       1 ]        2  25.000%  25.000% #####
(       1,       2 ]        2  25.000%  50.000% #####
(       6,      10 ]        2  25.000%  75.000% #####
(      22,      34 ]        2  25.000% 100.000% #####
```

##### more data

I run the command(with more entries)

```
$ ./db_bench --benchmarks="fillrandom,stats" --statistics --num=10000000
```

The brief result is

```
fillrandom   :       4.613 micros/op 216769 ops/sec;   24.0 MB/s
```

And it takes 46.1 s.

Also, level `L0`, `L1` and `L2` are entailed.

* `L0`: 5 files, total file size is 147.39 MB; W-Amp is 1,  Wnew is 0.6 GB, Write is 0.6 GB, Wr speed is 82.2 MB/s; Comp takes 7.53s where CompMergeCPU takes 6.42 s, and Comp count is 21.
* `L1`: 4 files, total file size is 212.41 MB; Read 0.9 GB; Write 0.8 GB and Wnew 0.3 GB, with W-Amp 1.8; Rd speed 82.8 MB/s and Wr speed 71.5 MB/s; Comp takes 11.61 s where CompMergeCPU takes 10.60 s, and Comp count is 4
* `L2`: 2 files, total file size is 128.74MB. The size of moved data is 0.1 GB.

The time spent is about 10 times as default.

The total size is 488.54 MB, which is about 20 times as default.

### read

#### seq

Firstly, fill randomly; Then, **read sequentially**.(level by level**???**)

```
$ ./db_bench --benchmarks="fillrandom,readseq,stats" --statistics --num=10000000
```

And the brief result is

```
DB path: [/tmp/rocksdbtest-1000/dbbench]
fillrandom   :       4.305 micros/op 232310 ops/sec;   25.7 MB/s
DB path: [/tmp/rocksdbtest-1000/dbbench]
readseq      :       0.459 micros/op 2177142 ops/sec;  240.8 MB/s
```

This is the read histogram.  In fact level `L2` stores 193.10 MB, but there is no read data on it...  **??**

```
** File Read Latency Histogram By Level [default] **
** Level 0 read latency histogram (micros):
Count: 341242 Average: 1.5019  StdDev: 0.77
Min: 0  Median: 0.9239  Max: 83
Percentiles: P50: 0.92 P75: 1.48 P99: 2.81 P99.9: 6.38 P99.99: 23.50
------------------------------------------------------
[       0,       1 ]   184666  54.116%  54.116% ###########
(       1,       2 ]   147563  43.243%  97.359% #########
(       2,       3 ]     6942   2.034%  99.393% 
(       3,       4 ]     1093   0.320%  99.713% 
(       4,       6 ]      612   0.179%  99.893% 
(       6,      10 ]      259   0.076%  99.969% 
(      10,      15 ]       61   0.018%  99.987% 
(      15,      22 ]       10   0.003%  99.989% 
(      22,      34 ]       15   0.004%  99.994% 
(      34,      51 ]       18   0.005%  99.999% 
(      51,      76 ]        4   0.001% 100.000% 
(      76,     110 ]        1   0.000% 100.001% 

** Level 1 read latency histogram (micros):
Count: 505839 Average: 1.9462  StdDev: 58.18
Min: 0  Median: 0.8840  Max: 27244
Percentiles: P50: 0.88 P75: 1.45 P99: 2.78 P99.9: 8.61 P99.99: 741.71
------------------------------------------------------
[       0,       1 ]   286096  56.559%  56.559% ###########
(       1,       2 ]   207364  40.994%  97.553% ########
(       2,       3 ]     9350   1.848%  99.401% 
(       3,       4 ]     1400   0.277%  99.678% 
(       4,       6 ]      871   0.172%  99.850% 
(       6,      10 ]      386   0.076%  99.926% 
(      10,      15 ]       74   0.015%  99.941% 
(      15,      22 ]       11   0.002%  99.943% 
(      22,      34 ]       12   0.002%  99.946% 
(      34,      51 ]        3   0.001%  99.946% 
(      51,      76 ]        9   0.002%  99.948% 
(      76,     110 ]        7   0.001%  99.949% 
(     110,     170 ]        4   0.001%  99.950% 
(     250,     380 ]        5   0.001%  99.951% 
(     380,     580 ]      144   0.028%  99.980% 
(     580,     870 ]       94   0.019%  99.998% 
(     870,    1300 ]        5   0.001%  99.999% 
(    1900,    2900 ]        1   0.000%  99.999% 
(    2900,    4400 ]        3   0.001% 100.000% 
(    6600,    9900 ]        2   0.000% 100.000% 
(   14000,   22000 ]        2   0.000% 100.001% 
(   22000,   33000 ]        1   0.000% 100.001% 
```

#### random

Firstly, fill randomly; Then, **read randomly**.

```
$ ./db_bench --benchmarks="fillrandom,readrandom,stats" --statistics --num=10000000
```

And the brief result is

```
DB path: [/tmp/rocksdbtest-1000/dbbench]
fillrandom   :       4.494 micros/op 222539 ops/sec;   24.6 MB/s
DB path: [/tmp/rocksdbtest-1000/dbbench]
readrandom   :      13.618 micros/op 73431 ops/sec;    8.1 MB/s (10000000 of 10000000 found)
```

**So random read is much slower than sequential read!!!**

### Summary

The Speed:

* fill seq:	about 41 MB/s
* fill random: about 23 MB/s
* read seq: about 240 MB/s
* read random: about 8.1 MB/s
