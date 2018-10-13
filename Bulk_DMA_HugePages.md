# Bulk DMA with hugepages recognition

Instructions for building and testing the *dma-bulk-hugepage* branch of veos, ve_drv, vp. libved.

## Clone repositories, checkout proper branch

```
mkdir BLD
cd BLD
GITH="https://github.com/efocht"

for repo in build-veos veos libved ve_drv-kmod vp-kmod vhcall-memtransfer-bm; do
    git clone $GITH/$repo.git
done

ln -s build-veos/x .

for repo in veos libved ve_drv-kmod vp-kmod; do
    cd $repo
    git checkout dma-bulk-hugepage
    cd ..
done
```

## Test performance

```
# add some (more) huge pages
sudo sysctl -w vm.nr_hugepages=8192

# switch cpufreq governor to *performance*
sudo cpupower frequency-set --governor performance

# build memtransfer benchmark
cd vhcall-memtransfer-bm
make

./scan_ve2vh.sh

HUGE=1 ./scan_ve2vh.sh

./scan_vh2ve.sh

HUGE=1 ./scan_vh2ve.sh

cd ..
```


## Build RPMs and replace old ones by new ones

```
# stop VEOS and VE related services, unload VE modules
x/vemods_unload

# replace installed RPMs by the newly built ones
RPMREPLACE=1

cd libved
../x/bld.libved
cd ..

cd vp-kmod
../x/bld.vp
cd ..

cd ve_drv-kmod
../x/bld.ve_drv
cd ..

cd veos
../x/bld.veos
cd ..

# load VE modules, start VEOS and VE related services
x/vemods_load
```

## Test performance

If done in the section above, then the benchmarks for ve_send/recv_data() are already built, therefore:

```
# build memtransfer benchmark
cd vhcall-memtransfer-bm

./scan_ve2vh.sh

HUGE=1 ./scan_ve2vh.sh

./scan_vh2ve.sh

HUGE=1 ./scan_vh2ve.sh

cd ..
```

----------

----------

## Why try this?

On one hand, it is an example of how to build and test VEOS when
you've done some changes. I hope I can encourage developers to start
experimenting with Aurora VEOS developments.


On the other hand, this concrete example contains a tuned version of
the system/privileged DMA handling. If you care about IO performance
or have workloads using VEO or VHcall and transfer large buffer
between VE and VH, this patched version could help. It comes with no
guarantee for fitness for any purpose. **THESE PATCHES ARE
EXPERIMENTAL! Use on your own risk! This version is pretty fresh and
not (hopefully...yet) in the official VEOS tree.**

Results of the core VE to VH send functions with this modified version are:

**small pages (4k), unpinned buffer**

`# ./scan_ve2vh.sh`

```
|    buff | veos 1.2.2 | v1_64_96 |
|      kb |  BW MiB/s  | BW MiB/s |
| ------- | ---------- | -------- |
|      32 |      131   |      106 |
|      64 |      249   |      263 |
|     128 |      447   |      477 |
|     256 |      698   |      839 |
|     512 |      924   |     1714 |
|    1024 |     1190   |     1755 |
|    2048 |     1374   |     2504 |
|    4096 |     1487   |     3071 |
|    8192 |     1599   |     4320 |
|   16384 |     1633   |     5378 |
|   32768 |     1720   |     5670 |
|   65536 |     1683   |     5328 |
|  131072 |     1655   |     5817 |
|  262144 |     1644   |     5748 |
|  524288 |     1734   |     5557 |
| 1048576 |     1649   |     5527 |
```

**huge pages (2M), unpinned buffer**

`# HUGE=1 ./scan_ve2vh.sh`

```
|   buff  | veos 1.2.2 | v1_64_96 |
|     kb  |  BW MiB/s  | BW MiB/s |
| ------- | ---------- | -------- |
|      32 |         79 |     110  |
|      64 |        248 |     291  |
|     128 |        490 |     550  |
|     256 |        718 |     928  |
|     512 |       1008 |    1566  |
|    1024 |       1296 |    2949  |
|    2048 |       1447 |    4421  |
|    4096 |       1566 |    6905  |
|    8192 |       1649 |    8272  |
|   16384 |       1704 |    9570  |
|   32768 |       1725 |   10129  |
|   65536 |       1676 |   10545  |
|  131072 |       1681 |   10690  |
|  262144 |       1698 |   10669  |
|  524288 |       1772 |   10802  |
| 1048576 |       1692 |   10872  |
```

The original version of the DMA processing from VEOS is a correct and
a straight forward way to deal with unpinned buffers by walking and
pinning them page by page and issuing the DMA requests once they are
all prepared. The *dma-bulk-hugepage* introduces bulk address
translation and more fine granular asyncronicity in DMA request
submission.
