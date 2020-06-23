# Fun data races in CMake


In CMake, the add_custom_command instructions are not thread safe.

Let us consider an example:

```
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp
    COMMAND ${CMAKE_COMMAND} -E env echo "allo" && pwd && ${CMAKE_CURRENT_SOURCE_DIR}/script.sh && echo "file outputted"
)
```

This command just calls a script and tries to create a new file textfile.cpp.

We can create three targets that depend on each other and that depend all on the `add_custom_command`:

```
add_library(joe1 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
add_library(joe2 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
add_library(joe3 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
target_link_libraries(joe2 INTERFACE joe1)
target_link_libraries(joe3 INTERFACE joe2)
```

The result? The add_custom_command gets called three times. That is, even though joe3 appears to depends on joe2 which itself appears depends on joe1, the joe2 and joe3 targets will still call the add_custom_command to generate the file they need.

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


You need to state your dependencies explicitly...
```
add_dependencies(joe2 joe1)
add_dependencies(joe3 joe2)
```
