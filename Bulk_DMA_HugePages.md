# Bulk DMA with hugepages recognition

Instructions for building and testing the *dma-bulk-hugepage* branch of veos, ve_drv, vp. libved.

## Clone repositories, checkout proper branch

```
mkdir BLD
cd BLD
GITH="https://github.com/efocht"

for repo in build-veos veos libved ve_drv-kmod vp-kmod vhcall-memtransfer-bm;
    git clone $GITH/$repo.git
done

ln -s build-veos/x .

for repo in veos libved ve_drv-kmod vp-kmod;
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
../x/vemods_unload

# replace installed RPMs by the newly built ones
RPMREPLACE=1

cd libved
. ../x/bld.libved
cd ..

cd vp-kmod
. ../x/bld.vp
cd ..

cd ve_drv-kmod
. ../x/bld.ve_drv
cd ..

cd veos
. ../x/bld.veos
cd ..

# load VE modules, start VEOS and VE related services
../x/vemods_load
```

## Test performance
