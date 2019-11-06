| Linux/OSX                                       | Windows                                             |
|-------------------------------------------------|-----------------------------------------------------|
| [![Build Status][travis_status]][travis_builds] | [![Build Status][appveyor_status]][appveyor_builds] |

[travis_status]: https://travis-ci.org/hunter-packages/gate.png?branch=master
[travis_builds]: https://travis-ci.org/hunter-packages/gate

[appveyor_status]: https://ci.appveyor.com/api/projects/status/nmkbcuoxwre7w2jp/branch/master?svg=true
[appveyor_builds]: https://ci.appveyor.com/project/ruslo/gate/branch/master

This is a gate file to [Hunter](https://github.com/cpp-pm/hunter) package manager.

The gate project will download the latest [packages archive release](https://github.com/cpp-pm/hunter/releases/), unless `HUNTER_URL` and `HUNTER_SHA1` are configured.

The gate project will issue `hunter_add_package` to all packages specified in `HUNTER_PACKAGES` list. For every package it will create a `hunter_config`
in which `VERSION` and `CMAKE_ARGS` variables will be set.

For manual configuration of `hunter_config` file you can set the `HUNTER_LOCAL_CONFIG`, `HUNTER_GLOBAL_CONFIG`, or `HUNTER_FILEPATH_CONFIG` variables.

The gate project will set `CMAKE_BUILD_TYPE` as `HUNTER_CONFIGURATION_TYPES`, so that if you're building `Release` you won't be getting a `Debug` build of the packages.

The gate project will create a `HunterToolchain.cmake` file which will set up the `CMAKE_PREFIX_PATH` and `CMAKE_FIND_ROOT_PATH` variables with the installed path of the Hunter packages.
If a `CMAKE_TOOLCHAIN_FILE` was used during compilation, it will be included in the `HunterToolchain.cmake` file.

## Usage

Setting up Hunter, and adding a package list is as easy as:

```cmake
# FetchContent_MakeAvailable is available since CMake version 3.14
cmake_minimum_required(VERSION 3.14)

# Setting up dlib as an external package
set(HUNTER_PACKAGES dlib)

include(FetchContent)
FetchContent_Declare(SetupHunter GIT_REPOSITORY https://github.com/cpp-pm/gate)
FetchContent_MakeAvailable(SetupHunter)

# Using the dlib package
find_package(dlib REQUIRED)
target_link_libraries(myapp PRIVATE dlib::dlib)
```

## Usage with custom CMake variables per package
```cmake
set(HUNTER_pcre2_CMAKE_ARGS
  PCRE2_BUILD_PCRE2_8=OFF
  PCRE2_BUILD_PCRE2_16=ON
  PCRE2_BUILD_PCRE2_32=OFF
  PCRE2_SUPPORT_JIT=ON)

set(HUNTER_PACKAGES pcre2)
```

## Usage with package components
```cmake
set(HUNTER_Boost_COMPONENTS Filesystem Python)
set(HUNTER_PACKAGES Boost)
```

## Usage with specific package version
```cmake
set(HUNTER_fmt_VERSION 5.3.0)
set(HUNTER_PACKAGES fmt)
```

## Usage with specific Hunter Gate
```cmake
set(HUNTER_URL "https://github.com/cpp-pm/hunter/archive/v0.23.224.tar.gz")
set(HUNTER_SHA1 "18e57a43efc435f2e1dae1291e82e42afbf940be")

set(HUNTER_PACKAGES fmt ZLIB)

include(FetchContent)
FetchContent_Declare(SetupHunter GIT_REPOSITORY https://github.com/cpp-pm/gate)
FetchContent_MakeAvailable(SetupHunter)
```

## Usage without modifying existing CMake code

With CMake 3.15 it is possible to inject CMake code before or after the `project` function call,
this way we can have a `SetupHunter.cmake` which will configure the 3rd party packages for the
project.

Give the following CMake code:
```cmake
cmake_minimum_required(VERSION 3.15)

project(ZLIBTest)

find_package(ZLIB REQUIRED)

add_executable(main main.c)
target_link_libraries(main PRIVATE ZLIB::ZLIB)
```

Notice that we do not have the `CONFIG` or `NO_MODULE` argument to `find_package`.
We can have a `SetupHunter.cmake` file like this:

```cmake
set(CMAKE_FIND_PACKAGE_PREFER_CONFIG TRUE)

set(HUNTER_PACKAGES ZLIB)

include(FetchContent)
FetchContent_Declare(SetupHunter GIT_REPOSITORY https://github.com/cpp-pm/gate)
FetchContent_MakeAvailable(SetupHunter)
```

And then configuring the `ZLIBTest` project like:
```
cmake -GNinja -DCMAKE_PROJECT_INCLUDE_BEFORE=[%cd%|$PWD]/SetupHunter.cmake -S . -B build
```

By using CMake 3.15's [CMAKE_FIND_PACKAGE_PREFER_CONFIG](https://cmake.org/cmake/help/v3.15/variable/CMAKE_FIND_PACKAGE_PREFER_CONFIG.html),
and [CMAKE_PROJECT_INCLUDE_BEFORE](https://cmake.org/cmake/help/v3.15/variable/CMAKE_PROJECT_INCLUDE_BEFORE.html) we can make
sure that the Hunter 3rd party packages are built, and used in a non-intrusive manner.

## Usage with HunterToolchain.cmake

This allows building the 3rd party packages separate than the user projects, and simply use CMake's
`find_package` with no usage of Hunter at all.

```cmake
cmake_minimum_required(VERSION 3.14)

set(HUNTER_PACKAGES freetype ZLIB PNG double-conversion pcre2)

set(HUNTER_pcre2_CMAKE_ARGS
    PCRE2_BUILD_PCRE2_8=OFF
    PCRE2_BUILD_PCRE2_16=ON
    PCRE2_BUILD_PCRE2_32=OFF
    PCRE2_SUPPORT_JIT=ON)

include(FetchContent)
FetchContent_Declare(SetupHunter GIT_REPOSITORY https://github.com/cpp-pm/gate)
FetchContent_MakeAvailable(SetupHunter)

project(3rdparty)
```

Then compile your project with:
```
cmake -DCMAKE_TOOLCHAIN_FILE=/path/to/your/3rdparty/build/HunterToolchain.cmake
```

## Usage with explicit config
```cmake
file(WRITE ${CMAKE_BUILD_DIR}/HunterConfig.cmake [=[
hunter_config(zlib VERSION 1.2.8)

hunter_config(pcre2
  VERSION ${HUNTER_pcre2_VERSION}
  CMAKE_ARGS
    PCRE2_BUILD_PCRE2_8=OFF
    PCRE2_BUILD_PCRE2_16=ON
    PCRE2_BUILD_PCRE2_32=OFF
    PCRE2_SUPPORT_JIT=ON
)
]=])

set(HUNTER_URL "https://github.com/cpp-pm/hunter/archive/v0.23.224.tar.gz")
set(HUNTER_SHA1 "18e57a43efc435f2e1dae1291e82e42afbf940be")
set(HUNTER_FILEPATH_CONFIG ${CMAKE_BUILD_DIR}/HunterConfig.cmake)

include(FetchContent)
FetchContent_Declare(SetupHunter GIT_REPOSITORY https://github.com/cpp-pm/gate)
FetchContent_MakeAvailable(SetupHunter)

hunter_add_package(ZLIB)
hunter_add_package(pcre2)

find_package(ZLIB CONFIG REQUIRED)
find_package(pcre2 CONFIG REQUIRED)

add_executable(boo main.c)
target_link_libraries(boo PRIVATE ZLIB::zlib PCRE2::PCRE2)
```

## Usage (manual)

* Copy file `HunterGate.cmake` to project
* Include gate file: `include("cmake/HunterGate.cmake")`
* Put any valid [Hunter](https://github.com/cpp-pm/hunter/releases) archive with `SHA1` hash:
```cmake
HunterGate(
    URL "https://github.com/cpp-pm/hunter/archive/v0.23.224.tar.gz"
    SHA1 "18e57a43efc435f2e1dae1291e82e42afbf940be"
)
```

## Usage (custom config)

Optionally custom [config.cmake][1] file can be specified. File may has different locations:

* `GLOBAL`. The one from Hunter archive:
```cmake
HunterGate(
    URL "https://github.com/cpp-pm/hunter/archive/v0.23.224.tar.gz"
    SHA1 "18e57a43efc435f2e1dae1291e82e42afbf940be"
    GLOBAL myconfig
        # load `${HUNTER_SELF}/cmake/configs/myconfig.cmake` instead of
        # default `${HUNTER_SELF}/cmake/configs/default.cmake`
)
```
* `LOCAL`. Default local config.
```cmake
HunterGate(
    URL "https://github.com/cpp-pm/hunter/archive/v0.23.224.tar.gz"
    SHA1 "18e57a43efc435f2e1dae1291e82e42afbf940be"
    LOCAL # load `${CMAKE_CURRENT_LIST_DIR}/cmake/Hunter/config.cmake`
)
```
* `FILEPATH`. Any location.
```cmake
HunterGate(
    URL "https://github.com/cpp-pm/hunter/archive/v0.23.224.tar.gz"
    SHA1 "18e57a43efc435f2e1dae1291e82e42afbf940be"
    FILEPATH "/any/path/to/config.cmake"
)
```

* [Example](https://github.com/ruslo/hunter/wiki/example.custom.config.id)

### Notes

* If you're in process of patching Hunter and have a HUNTER_ROOT pointed to git repository location then HunterGate will not use `URL` and `SHA1` values. It means when you update `SHA1` of Hunter archive new commits/fixes will not be applied at all. In this case you have to update your git repo manually (i.e. do `git pull`)
* You don't need to specify [hunter_config][2] command for all projects. Set version of the package you're interested in - others will be used from default `config.cmake`.
* If you want to get full control of what Hunter-SHA1 root directories you want to auto-install you can set [HUNTER_DISABLE_AUTOINSTALL](https://github.com/ruslo/hunter/wiki/CMake-Variables-%28User%29#hunter_disable_autoinstall-environment-variable) environment variable and use [HUNTER_RUN_INSTALL=YES](https://github.com/ruslo/hunter/wiki/CMake-Variables-%28User%29#hunter_run_install) CMake variable to allow installations explicitly.

## Effects
* Try to detect `Hunter`:
 * test CMake variable `HUNTER_ROOT` (control, shared downloads and builds)
 * test environment variable `HUNTER_ROOT` (**recommended**: control, shared downloads and builds)
 * test directory `${HOME}/.hunter` (shared downloads and builds)
 * test directory `${SYSTEMDRIVE}/.hunter` (shared downloads and builds, windows only)
 * test directory `${USERPROFILE}/.hunter` (shared downloads and builds, windows only)
* Set `HUNTER_GATE_*` variables
* Include Hunter master file `include("${HUNTER_SELF}/cmake/Hunter")`

## Flowchart (for developers)
![flowchart](https://raw.githubusercontent.com/hunter-packages/gate/master/wiki/flowchart.png)

## Examples
* [This](https://github.com/hunter-packages/gate/blob/master/CMakeLists.txt)
* [Simple](https://github.com/forexample/hunter-simple)
* [Weather](https://github.com/ruslo/weather)

## Links
* [Hunter](https://github.com/cpp-pm/hunter)
* [Some packages](https://github.com/ruslo/hunter/wiki/Packages)

[1]: https://github.com/ruslo/hunter/blob/master/cmake/configs/default.cmake
[2]: https://github.com/ruslo/hunter/wiki/Hunter-modules#hunter_config
