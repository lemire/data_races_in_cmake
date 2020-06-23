# Fun data races in CMake

Executive summary
----------------

Custom commands in CMake are NOT targets, they are just that: commands that trigger when you ask for the files in them. They may run multiple times, and are intended to do so, recalculating whatever they do each time. If it isn't thread safe, you must synchronize it using actual targets. And interface libraries are not actual targets.



Usage:
-----

```
git clone https://github.com/lemire/data_races_in_cmake.git
cd data_races_in_cmake/
mkdir build
cd build/
cmake ..
make -j4
```

Analysis:
-----

In CMake, the add_custom_command instructions are not thread safe. If more than one target depends on the result of the command, you may get serious problems.

Let us consider an example:

```
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp
    COMMAND ${CMAKE_COMMAND} -E env echo "allo" && pwd && ${CMAKE_CURRENT_SOURCE_DIR}/script.sh && echo "file outputted"
)
```

This command just calls a script and tries to create a new file textfile.cpp.

We can create three targets that depend all on the `add_custom_command`:

```
add_library(joe1 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
add_library(joe2 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
add_library(joe3 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
```

The result? The add_custom_command gets called three times. 

```
 cmake .. && make -j4
-- The CXX compiler identification is AppleClang 11.0.3.11030032
-- Check for working CXX compiler: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/c++
-- Check for working CXX compiler: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/c++ - works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /Users/lemire/CVS/github/data_races_in_cmake/build
[ 11%] Generating textfile.cpp
[ 22%] Generating textfile.cpp
[ 33%] Generating textfile.cpp
allo
allo
/Users/lemire/CVS/github/data_races_in_cmake/build
allo
/Users/lemire/CVS/github/data_races_in_cmake/build
/Users/lemire/CVS/github/data_races_in_cmake/build
file outputted
file outputted
file outputted
```




You may think that you can just patch it up by introducing an earlier custom target...

```

add_custom_target(fake DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp)
add_dependencies(joe1 fake)
```

But it will not help.


## Proper fix

You need to state your dependencies explicitly...
```
add_dependencies(joe2 joe1)
add_dependencies(joe3 joe2)
```

Then you know for sure that joe2 won't start to build before joe1 is done. And so forth.
