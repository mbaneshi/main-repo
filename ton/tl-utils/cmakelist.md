<details>
  <summary>
    source code
  </summary>

```txt
cmake_minimum_required(VERSION 3.0.2 FATAL_ERROR)

set(TL_UTILS_SOURCE
  common-utils.hpp
  tl-utils.hpp
  tl-utils.cpp
)

add_library(tl-utils STATIC ${TL_UTILS_SOURCE})

target_include_directories(tl-utils PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/..>)
target_link_libraries(tl-utils PUBLIC tl_api ton_crypto )

set(TL_LITE_UTILS_SOURCE
  common-utils.hpp
  lite-utils.hpp
  lite-utils.cpp
)

add_library(tl-lite-utils STATIC ${TL_LITE_UTILS_SOURCE})

target_include_directories(tl-lite-utils PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/..>)
target_link_libraries(tl-lite-utils PUBLIC tl_lite_api ton_crypto )
```
It looks like you've provided a CMakeLists.txt snippet for a project related to TON (The Open Network) blockchain. Let's break down what's happening in this code:

1. **Minimum CMake Version:**
   ```cmake
   cmake_minimum_required(VERSION 3.0.2 FATAL_ERROR)
   ```
   This sets the minimum required version of CMake for building the project. If a version lower than 3.0.2 is used, it will generate a fatal error.

2. **TON Utility Library:**
   ```cmake
   set(TL_UTILS_SOURCE
     common-utils.hpp
     tl-utils.hpp
     tl-utils.cpp
   )

   add_library(tl-utils STATIC ${TL_UTILS_SOURCE})

   target_include_directories(tl-utils PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/..>)
   target_link_libraries(tl-utils PUBLIC tl_api ton_crypto )
   ```
   </details>
   This block defines a static library called `tl-utils` that includes source files `common-utils.hpp`, `tl-utils.hpp`, and `tl-utils.cpp`. It specifies include directories and links the library to other dependencies, such as `tl_api` and `ton_crypto`.

3. **TON Lite Utility Library:**
   ```cmake
   set(TL_LITE_UTILS_SOURCE
     common-utils.hpp
     lite-utils.hpp
     lite-utils.cpp
   )

   add_library(tl-lite-utils STATIC ${TL_LITE_UTILS_SOURCE})

   target_include_directories(tl-lite-utils PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/..>)
   target_link_libraries(tl-lite-utils PUBLIC tl_lite_api ton_crypto )
   ```
   Similar to the previous block, this defines another static library named `tl-lite-utils` with its set of source files. It includes the same `common-utils.hpp` file and links to `tl_lite_api` and `ton_crypto`.

In summary, this CMake configuration is setting up two static libraries (`tl-utils` and `tl-lite-utils`) for TON blockchain code. The libraries have specific source files and dependencies, and the include directories are configured accordingly. The use of CMake in this context indicates a cross-platform build system for your project. If you have specific questions about any part of this code or if you'd like a more detailed analysis, feel free to ask!
