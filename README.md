This fork is a fork of mutability fork with changes to supports windows from tpainter.

# dump1090-flightairmap GNU/Linux & Windows package

This is a fork of tpainter windows version of the fork of Oliver Jowett's version of dump1090 (dump1090-mutability).

This version is licensed under the GPL (v3 or later).
See the file COPYING for details.
The source license has been "upgraded" to GPLv3 using the "or later" clause.
This has been done because of the inclusion of required libraries being under
licenses that are not compatible with the GPLv2; namely the Apache License 2.0.

# Features

* 2.4MHz "oversampling" support
* supports FlightAware-TSV-format connections directly (same as the FlightAware version - no faup1090 needed)
* map view uses receiver lat/long given to dump1090 automatically
* somewhat cleaned-up network code
* tries to do things "the debian way" when it comes to config, package structure, etc
* probably a bunch of other things I've forgotten..

# Install for Windows Users

**TODO**

## Logging

No logging in Windows is currently configured.

# Bug reports, feedback etc

Please use the [github issues page](https://github.com/ysurac/dump1090/issues) to report any problems.

# Future plans

It is anticipated that each release of dump1090-mutability will be closely followed by a release of dump1090-flightairmap.

# Building from source

## For Linux
    mkdir build
    cd build
    cmake ./..

## For Windows
    mkdir build
    cd build
    cmake ./.. -DCMAKE_TOOLCHAIN_FILE=../cmake/Modules/Toolchain-mingw32.cmake

