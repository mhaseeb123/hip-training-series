cmake_minimum_required(VERSION 3.21 FATAL_ERROR)
project(dgemm LANGUAGES CXX)

# ----------------------------------------------------------------SET CXX STANDARD--
set(CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS OFF)

if (NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif(NOT CMAKE_BUILD_TYPE)

string(REPLACE -O2 -O3 CMAKE_CXX_FLAGS_RELWITHDEBINFO ${CMAKE_CXX_FLAGS_RELWITHDEBINFO})

# ----------------------------------------------------------------SET GPU RUNTIME for Perlmutter --

if (NOT CMAKE_GPU_RUNTIME)
   set(GPU_RUNTIME "ROCM" CACHE STRING "Switches between ROCM and CUDA")
else (NOT CMAKE_GPU_RUNTIME)
   set(GPU_RUNTIME "${CMAKE_GPU_RUNTIME}" CACHE STRING "Switches between ROCM and CUDA")
endif (NOT CMAKE_GPU_RUNTIME)
# Really should only be ROCM or CUDA, but allowing HIP because it is the currently built-in option
set(GPU_RUNTIMES "ROCM" "CUDA" "HIP")
if(NOT "${GPU_RUNTIME}" IN_LIST GPU_RUNTIMES)
    set(ERROR_MESSAGE "GPU_RUNTIME is set to \"${GPU_RUNTIME}\".\nGPU_RUNTIME must be either HIP, ROCM, or CUDA.")
    message(FATAL_ERROR ${ERROR_MESSAGE})
endif()

# GPU_RUNTIME for AMD GPUs should really be ROCM, if selecting AMD GPUs
# so manually resetting to HIP if ROCM is selected
if (${GPU_RUNTIME} MATCHES "ROCM")
   set(GPU_RUNTIME "HIP")
endif (${GPU_RUNTIME} MATCHES "ROCM")
set_property(CACHE GPU_RUNTIME PROPERTY STRINGS ${GPU_RUNTIMES})

enable_language(${GPU_RUNTIME})
set(CMAKE_${GPU_RUNTIME}_EXTENSIONS OFF)
set(CMAKE_${GPU_RUNTIME}_STANDARD_REQUIRED ON)

set(CMAKE_${GPU_RUNTIME}_FLAGS_DEBUG "-ggdb")

# ----------------------------------------------------------------Get HIP PATH and setup hipblas --
if (DEFINED ENV{HIP_PATH})
   set(HIP_PATH $ENV{HIP_PATH})
   set(hipblas_INCLUDE ${HIP_PATH}/hipblas/include)
   set(hipblas_LIBRARIES ${HIP_PATH}/hipblas/lib/libhipblas.so)
else (DEFINED ENV{HIP_PATH})
   execute_process(COMMAND hipconfig --path OUTPUT_VARIABLE HIP_PATH ERROR_QUIET)
endif (DEFINED ENV{HIP_PATH})

# Make example runnable using ctest
#add_test(NAME Stream COMMAND stream )
#set_property(TEST Stream
#             PROPERTY PASS_REGULAR_EXPRESSION "PASSED!")

set(ROCMCC_FLAGS "${ROCMCC_FLAGS} -fno-gpu-rdc -munsafe-fp-atomics -fPIC -w")
set(CUDACC_FLAGS "${CUDACC_FLAGS} ")

if (${GPU_RUNTIME} MATCHES "HIP")
   set(HIPCC_FLAGS "${ROCMCC_FLAGS}")
else (${GPU_RUNTIME} MATCHES "CUDA")
   set(HIPCC_FLAGS "${CUDACC_FLAGS} -I/${HIP_PATH}/include")
endif (${GPU_RUNTIME} MATCHES "HIP")


# ------------------------------------------------------------------------------SRC--
add_subdirectory("src")

# EOF
