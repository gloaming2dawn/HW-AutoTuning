
## Using CLI to setup proxy on DSS server
We use [brook](https://github.com/txthinking/brook.git) as the proxy provider.
Download and install brook on DSS server, then start the brook server and client:
```shell
nohup brook server -l :9999 -p password &
nohup brook client -s 127.0.0.1:9999 -p password --http 0.0.0.0:1081 &
```
Note we need set the client ip address to `0.0.0.0` to allow access from other machines.
Test on DSS to see whether we can access internet through the brook proxy:

```
curl -x 127.0.0.1:1081 www.google.com
```

### Set proxy for apt
Then we setup  proxy for `apt` on the jetson TX2 board:
```
sudo vim /etc/apt/apt.conf.d/proxy.conf
```
and add the following lines to the `proxy.conf`:
```shell
Acquire::http::Proxy "http://169.254.25.45:1081/";
Acquire::https::Proxy "http://169.254.25.45:1081/";
```
Note that  `169.254.25.45` is the ip address of the ethernet which connects with jetson boards.
Then we can use apt to install software on jetson TX2 board.

### Set proxy for git
```shell
git config --global http.proxy  "http://169.254.25.45:1081/"
git config --global https.proxy  "http://169.254.25.45:1081/"
```

### Set global proxy
add the following lines to `~/.bashrc`
```
export http_proxy=169.254.25.45:1081/
export https_proxy=169.254.25.45:1081/
```
## Fix freq
```shell
sudo apt install cpufrequtils 
sudo cpufreq-set -r -g userspace
sudo cpufreq-set -r --freq 2.04G
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_min_freq
cat /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq
sudo cpufreq-set -r --min 2035200
sudo nvpmodel -q
sudo nvpmodel -m 3
```

## Install `perf` from source
We tried to install perf by apt but their isn't perf for jetson TX2's linux version (We can get the linux version with cmd `uname -r`).
First install necessary libraries.
```shell
sudo apt install libdw-dev libelf-dev libperl-dev
```

We need to build from linux source code. If /usr/src/linux-headers-4.4.38-tegra/tools/perf already exists
```shell
cd /usr/src/linux-headers-4.4.38-tegra/tools/perf
sudo apt install libdw-dev libelf-dev libperl-dev
sudo make
sudo make -j4
```

Else, we need to download the nvidia released linux source code.
```shell
wget https://developer.download.nvidia.com/embedded/L4T/r28_Release_v2.1/public_sources.tbz2
tar -xvf public_sources.tbz2
tar -xvf public_release/kernel_src.tbz2
cd kernel/kernel-4.4/tools/perf
sudo make -j4
```

Optional
```shell
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.4.38.tar.gz
tar -xvf linux-4.4.38.tar.gz
cd linux-4.4.38/tools/perf
sudo make -j4
```



For newer Jetpack, first check the version
```shell
cat /etc/nv_tegra_release
```
```shell
sudo apt install libdw-dev libelf-dev libperl-dev
wget https://developer.nvidia.com/embedded/L4T/r32_Release_v7.4/Sources/T186/public_sources.tbz2
tar -xvf public_sources.tbz2
tar -xvf Linux_for_Tegra/source/public/kernel_src.tbz2
cd kernel/kernel-4.9/tools/perf
sudo make -j4
```

If everthing is OK, then we can see `perf` in the folder.
copy it the the system binary folder:

```shell
cp ./perf /usr/bin/
sudo sh -c 'echo 1 >/proc/sys/kernel/perf_event_paranoid'
```

Then we can test whether `perf` works and list all the software and hardware events:

```shell
perf list
```
For more perf usage, go to [this](https://www.brendangregg.com/perf.html) website.


## collect profiling data for cBench
First exec `perf list` to see avaliable events:
```shell
List of pre-defined events (to be used in -e):

  branch-misses                                      [Hardware event]
  cache-misses                                       [Hardware event]
  cache-references                                   [Hardware event]
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]

  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
  cpu-migrations OR migrations                       [Software event]
  dummy                                              [Software event]
  emulation-faults                                   [Software event]
  major-faults                                       [Software event]
  minor-faults                                       [Software event]
  page-faults OR faults                              [Software event]
  task-clock                                         [Software event]

  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-loads                                    [Hardware cache event]
  L1-dcache-store-misses                             [Hardware cache event]
  L1-dcache-stores                                   [Hardware cache event]
  branch-load-misses                                 [Hardware cache event]
  branch-loads                                       [Hardware cache event]

  rNNN                                               [Raw hardware event descriptor]
  cpu/t1=v1[,t2=v2,t3 ...]/modifier                  [Raw hardware event descriptor]
   (see 'man perf-list' on how to encode it)

  mem:<addr>[/len][:access]                          [Hardware breakpoint]
```
These are the avaliable performance events on jetson TX2 board.

Use perf to collect PMU:
```
perf stat -e branch-misses,cache-misses,cache-references,cpu-cycles,instructions,cpu-clock,L1-dcache-load-misses,L1-dcache-loads,L1-dcache-store-misses,L1-dcache-stores,branch-load-misses,branch-loads cmd
```

Use perf to collect profiling data:
```shell
perf record -e branch-misses,cache-misses,cache-references,cpu-cycles,instructions,cpu-clock,L1-dcache-load-misses,L1-dcache-loads,L1-dcache-store-misses,branch-load-misses,branch-loads -o out_pofile_name-perf.data cmd
```
Note replace the `out_pofile_name-perf.data` and `cmd` to your own output file name and execute command.
We can also record the call-graph to inspect which set of functions consumes the most latency.
