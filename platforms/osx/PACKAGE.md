# Packaging TileMill.app standalone

Thes are the steps to setup tilemill to be portable within an .app bundle.

This is only necessary for developers that wish to build a fully
distributable tilemill.app without requiring any other installation steps.


## Build node

We build node with two cpu architectures, aka universal/fat to support older macs:

    wget http://nodejs.org/dist/node-v0.4.12.tar.gz
    tar xvf node-v0.4.12.tar.gz
    cd node-v0.4.12
    # ia32 == i386
    ./configure --without-snapshot --jobs=`sysctl -n hw.ncpu` --blddir=node-32 --dest-cpu=ia32
    make install # will install headers
    # x64 == x86_64
    ./configure --without-snapshot --jobs=`sysctl -n hw.ncpu` --blddir=node-64 --dest-cpu=x64
    make
    lipo -create node-32/default/node node-64/default/node -output node
    
    # then copy that node overwriting the previously installed uni-arch node
    # so it is default on your PATH
    cp node /usr/local/bin/node
    chmod +x /usr/local/bin/node
    
    # confirm it comes first
    which node | grep /usr/local/bin
    
    # should report two archs
    file `which node`


## Install latest npm

    curl http://npmjs.org/install.sh | sudo sh


## Build testing tools globally

This will keep these out of tilemill's local node_modules, avoid having to strip them
from the final package, and most importantly avoid any compile failures due to
custom flags we set later on.

    npm install -g jshint expresso
    
Note: jshint installation will fail with clang compiler so do:

    export CC=gcc
    export CXX=g++


## Build tilemill

Clear out any previous builds:

    cd tilemill
    rm -rf node_modules
    
Also ensure that you have no globally installed node modules (other than `jshint` and `expresso`).
You may need to check various node_modules depending on your $NODE_PATH.

TODO: we should be able to avoid having to clear out node_modules by telling/tricking
npm to avoid finding them (just needs testing).

Now build tilemill with a few custom flags:

    export CORE_CXXFLAGS="-O3 -arch x86_64 -arch i386 -mmacosx-version-min=10.6 -isysroot /Developer/SDKs/MacOSX10.6.sdk"
    export CORE_LINKFLAGS="-arch x86_64 -arch i386 -Wl,-syslibroot,/Developer/SDKs/MacOSX10.6.sdk"
    export CXXFLAGS=$CORE_LINKFLAGS
    export LINKFLAGS=$CORE_LINKFLAGS
    export JOBS=`sysctl -n hw.ncpu`
    npm install . --verbose

As long as you don't have any globally installed modules that tilemill uses, this should deposit all 
tilemill dependencies in node_modules/. Now the task is to check on a few and rebuild a few.


## Check node-zlib

We need to make sure node-zlib is linked against the system zlib. The output of
the command below should be similar to https://gist.github.com/1125399.

    otool -L node_modules/mbtiles/node_modules/zlib/lib/zlib_bindings.node

We also need to make sure that node-zlib and other C++ addons were compiled universal (both 32/64 bit).
The command output should look like https://gist.github.com/1132771.

    file node_modules/mbtiles/node_modules/zlib/lib/zlib_bindings.node


## Check, node-srs, node-zipfile, and node-eio

We need to do the same linking and architecture test for 3 more modules:

    # eio
    otool -L node_modules/tilelive-mapnik/node_modules/eio/build/default/eio.node
    file node_modules/tilelive-mapnik/node_modules/eio/build/default/eio.node
    
    # srs
    otool -L ./node_modules/millstone/node_modules/srs/lib/_srs.node
    file ./node_modules/millstone/node_modules/srs/lib/_srs.node
    
    # zipfile
    otool -L ./node_modules/millstone/node_modules/zipfile/lib/_zipfile.node
    file ./node_modules/millstone/node_modules/zipfile/lib/_zipfile.node


## Uninstall any globally installed libmapnik2.dylib

    # move to where your mapnik sources are
    cd src/mapnik-trunk
    sudo make uninstall


## Set up Mapnik SDK

Mapnik needs to be compiled such that all dependencies are either statically linked
or are linked using @rpath/@loader_path (and then all those dylib deps are included).

An experimental SDK includes all mapnik dependencies such that you can set up
mapnik to be compiled statically against them.

To set up the SDK and build mapnik do:

    git clone git://github.com/mapnik/mapnik.git -b macbinary-tilemill
    cd mapnik
    mkdir osx
    cd osx/
    curl -o sources.tar.bz2 http://dbsgeo.com/tmp/mapnik-static-sdk-2.1.0-dev_r1.tar.bz2
    tar xvf sources.tar.bz2
    cd ../
    ./configure JOBS=`sysctl -n hw.ncpu` (If you have problems with Clang, add "CC=gcc CXX=g++" as well.)
    make
    make install

    # set critical shell env settings
    export MAPNIK_ROOT=`pwd`/osx/sources
    export PATH=$MAPNIK_ROOT/usr/local/bin:$PATH

Confirm the SDK is working by checking mapnik-config presence at that path:

    # this should produce a line of output pointing to valid mapnik-config
    which mapnik-config | grep $MAPNIK_ROOT


Note: the sdk was created using https://github.com/mapnik/mapnik-packaging/blob/master/osx/scripts/static-universal.sh

## Edit the mapnik-config script

A major flaw in the portability of the above system we've yet to take time to tackle is the hardcoded
absolute paths from the SDK creation. So, before continuing we need to manually edit the mapnik-config
script.

Open the script:

    vim `which mapnik-config`

Edit the file and change all paths that include:

    /Users/dane/projects/mapnik-dev/trunk-build-static-universal/osx/sources

To be equal to your $MAPNIK_ROOT value.


## Change into tilemill dir

Now, in the same shell that you set the above environment settings
navigate to your tilemill development directory.

   cd ~/tilemill # or wherever you git cloned tilemill


## Rebuild node-mapnik


Move to the node-mapnik directory:

    cd node_modules/mapnik/


Then rebuild like:

    make clean
    export CXXFLAGS="-I$MAPNIK_ROOT/include -I$MAPNIK_ROOT/usr/local/include $CORE_CXXFLAGS"
    export LINKFLAGS="-L$MAPNIK_ROOT/lib -L$MAPNIK_ROOT/usr/local/lib -Wl,-search_paths_first $CORE_LINKFLAGS"
    export MAPNIK_INPUT_PLUGINS="path.join(__dirname, 'input')"
    export MAPNIK_FONTS="path.join(__dirname, 'fonts')"
    ./configure
    node-waf -v build
    SONAME=2
    cp $MAPNIK_ROOT/usr/local/lib/libmapnik2.dylib lib/libmapnik$SONAME.dylib
    install_name_tool -id libmapnik$SONAME.dylib lib/libmapnik2.dylib
    install_name_tool -change /usr/local/lib/libmapnik2.dylib @loader_path/libmapnik$SONAME.dylib lib/_mapnik.node

    mkdir -p lib/fonts
    rm lib/fonts/*
    cp -R $MAPNIK_ROOT/usr/local/lib/mapnik2/fonts lib/
    
    mkdir -p lib/input
    rm lib/input/*.input
    cp $MAPNIK_ROOT/usr/local/lib/mapnik2/input/*.input lib/input/
    for i in $(ls lib/input/*input);
    do install_name_tool -change /usr/local/lib/libmapnik2.dylib @loader_path/../libmapnik$SONAME.dylib $i;
    done;


Run the tests:

    make test


Check a few things:

    otool -L lib/_mapnik.node
    file lib/_mapnik.node

    # check plugins: should return nothing
    otool -L lib/input/*input | grep /usr/local


Now go back to the main tilemill directory:

    cd ../../


And check builds overall. These commands can help find possible errors but are not necessary to run:

    # for reference see all the C++ module dependencies
    for i in $(find . -name '*.node'); do otool -L $i; done;

    # should return nothing
    for i in $(find . -name '*.node'); do otool -L $i | grep local; done;

    # dump out file to inspect if problems
    for i in $(find . -name '*.node'); do otool -L $i >>t.txt; done;


Test that the app still works

    ./index.js


Now go build and package the tilemill app:

    cd platforms/osx
    make clean
    make run # test and check version
    make zip # package

Then rename the TileMill.zip to TileMill-$VER.zip. For example:

   mv TileMill.zip TileMill-0.6.0.zip

Upload to https://github.com/mapbox/tilemill/downloads