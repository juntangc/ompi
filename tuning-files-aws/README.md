This branch was created on 05/16/2024 to make it easier for AWS users to use the recent improvement made to Open MPI Allreduce. There are two sets of instructions for users of Open MPI 5 and Open MPI 4.

Part a. Follow the instructions below to use the improved Open MPI allreduce collective (Open MPI 5)
1. I recommend to compile libfabric from source (until the EFA/libfabric improvement in November 2023 are included in a release); default installation directory is /shared/libfabric/install.

```
export INSTALLDIR=/shared
cd /shared/tools
[ -d libfabric ] || git clone https://github.com/ofiwg/libfabric.git -b main
pushd libfabric
./autogen.sh
./configure CFLAGS='-g -fno-omit-frame-pointer' --prefix=${INSTALLDIR}/libfabric/install --disable-verbs --disable-psm3 --disable-opx --disable-usnic --disable-rstream --enable-efa
make -j install
popd
```

2. compile Open MPI from source
```
cd /shared/tools
git clone --recursive https://github.com/juntangc/ompi.git ./ompi5-improved
cd ompi5-improved
git checkout ompi5-improved
./autogen.pl

INSTALLDIR=/shared
OPENMPI_VERSION=5-improved
EFA_LIB_DIR=/shared/libfabric/install
cd /shared/tools/ompi5-improved
rm -rf build
mkdir build
cd build
../configure --prefix=${INSTALLDIR}/openmpi-${OPENMPI_VERSION}  --with-pmix=internal  --without-munge --with-libfabric=${EFA_LIB_DIR} --with-libevent=internal --with-hwloc=internal
make -j$(nproc) && make install
```

3. check the Open MPI installtion is correct.  You will find the following method with `ompi_info --param coll tuned --level 9`.  The new allreduce algorithm use alg 7 `allgather_reduce` for Allreduce, alg 2 `bruck with radix k` for allgather, and alg 8 `knomial` for reduce.
```
          MCA coll tuned: parameter "coll_tuned_allreduce_algorithm" (current value: "ignore", data source: default, level: 5 tuner/de
tail, type: int)
                          Which allreduce algorithm is used. Can be locked down to any of: 0 ignore, 1 basic linear, 2 nonoverlapping (tuned reduce + tuned bcast), 3 recursive doubling, 4 ring, 5 segmented ring, 6 rabenseifner, 7 allgather_reduce. Only relevant if coll_tuned_use_dynamic_rules is true.

          MCA coll tuned: parameter "coll_tuned_allgather_algorithm" (current value: "ignore", data source: default, level: 5 tuner/detail, type: int)
                          Which allgather algorithm is used. Can be locked down to choice of: 0 ignore, 1 basic linear, 2 bruck with radix k, 3 recursive doubling, 4 ring, 5 neighbor exchange, 6: two proc only, 7: sparbit, 8: direct messaging. Only relevant if coll_tu
ned_use_dynamic_rules is true.

          MCA coll tuned: parameter "coll_tuned_reduce_algorithm" (current value: "ignore", data source: default, level: 5 tuner/detail, type: int)
                          Which reduce algorithm is used. Can be locked down to choice of: 0 ignore, 1 linear, 2 chain, 3 pipeline, 4 binary, 5 binomial, 6 in-order binary, 7 rabenseifner, 8 knomial. Only relevant if coll_tuned_use_dynamic_rules is true.
```


4. specify tuning rules to use the improved allreduce method
```
module purge
export PATH=/shared/openmpi-5-improved/bin:$PATH
export LD_LIBRARY_PATH=/shared/openmpi-5-improved/lib:$LD_LIBRARY_PATH
export PATH=/shared/libfabric/install/bin:$PATH
export LD_LIBRARY_PATH=/shared/libfabric/install/lib:$LD_LIBRARY_PATH

    mpirun  -np ${ntasks} \
            --mca coll_han_use_dynamic_file_rules 1 \
            --mca coll_han_dynamic_rules_filename /shared/tools/ompi5-improved/tuning-files-aws/han_rules_1 \
            --mca coll_tuned_use_dynamic_rules 1 \
            --mca coll_tuned_dynamic_rules_filename /shared/tools/ompi5-improved/tuning-files-aws/tuned_rules_1 \
            <executable>
```

Part b. Follow the instructions below to use the improved Open MPI allreduce collective (Open MPI 4). Please be aware that due to lack of a tuning utility, this patch use node-aware HAN approach for both small and large messages (not the intended use) and the radix/fanout have been hard coded. The improved allreduce method sacrifices bandwidth (large messages) for better latency (small messages).  So you are going to see mixed results unless the majority of allreduce messages are small.

```
cd /shared/tools
git clone --recursive https://github.com/open-mpi/ompi.git ./ompi4-improved
cd ompi4-improved
git checkout v4.1.x
wget https://raw.githubusercontent.com/juntangc/ompi/ompi5-improved/patch-for-ompi4/patch-ompi-v4.1.x.diff
git apply patch-ompi-v4.1.x.diff
./autogen.pl

INSTALLDIR=/shared
OPENMPI_VERSION=4-improved
EFA_LIB_DIR=/shared/libfabric/install
cd /shared/tools/ompi4-improved
rm -rf build
mkdir build
cd build
../configure --prefix=${INSTALLDIR}/openmpi-${OPENMPI_VERSION}  --with-pmix=internal  --without-munge --with-libfabric=${EFA_LIB_DIR} --with-libevent=internal --with-hwloc=internal
make -j$(nproc) && make install

```
