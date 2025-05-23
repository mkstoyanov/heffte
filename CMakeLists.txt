cmake_minimum_required(VERSION 3.19)

project("Heffte" VERSION 2.4.1 LANGUAGES CXX)

option(Heffte_ENABLE_FFTW     "Enable the FFTW backend"            OFF)
option(Heffte_ENABLE_CUDA     "Enable the CUDA and cuFFT backend"  OFF)
option(Heffte_ENABLE_ROCM     "Enable the HIP and rocFFT backend"  OFF)
option(Heffte_ENABLE_ONEAPI   "Enable the oneAPI/DPC++ and oneMKL backend"  OFF)
option(Heffte_ENABLE_MKL      "Enable the Intel MKL backend"       OFF)
option(Heffte_ENABLE_DOXYGEN  "Build the Doxygen documentation"    OFF)

option(Heffte_ENABLE_AVX      "Enable the use of AVX registers in the stock backend, adds flags: -mfma -mavx"       OFF)
option(Heffte_ENABLE_AVX512   "Enable the use of AVX512 registers in the stock backend, adds AVX flags plus: -mavx512f -mavx512dq"    OFF)

option(Heffte_ENABLE_MAGMA    "Enable some helper functions from UTK MAGMA for GPU backends"   OFF)

option(Heffte_ENABLE_PYTHON   "Configure the Python scripts"     OFF)
option(Heffte_ENABLE_FORTRAN  "Build the Fortran modules for the selected backends." OFF)
option(Heffte_ENABLE_SWIG     "Rebuild the SWIG bindings." OFF)

option(Heffte_ENABLE_TRACING  "Enable the tracing capabilities"  OFF)

option(BUILD_SHARED_LIBS      "Builds shared libraries using CMake conventions"  ON)

if (Heffte_ENABLE_CUDA OR Heffte_ENABLE_ROCM OR Heffte_ENABLE_ONEAPI)
    if (Heffte_DISABLE_GPU_AWARE_MPI)
        option(Heffte_ENABLE_GPU_AWARE_MPI "GPU to GPU direct MPI calls" OFF)
    else()
        option(Heffte_ENABLE_GPU_AWARE_MPI "GPU to GPU direct MPI calls" ON)
    endif()
else()
    set(Heffte_ENABLE_GPU_AWARE_MPI OFF CACHE BOOL "GPU to GPU direct MPI calls" FORCE)
endif()

if (${CMAKE_PROJECT_NAME} STREQUAL ${PROJECT_NAME})
    option(Heffte_ENABLE_TESTING "Enable the internal unit tests" ON)
else()
    option(Heffte_ENABLE_TESTING "Enable the internal unit tests" OFF)
endif()

option(Heffte_SEQUENTIAL_TESTING  "Force all tests to run sequentially to avoid oversubscribing resources."  OFF)

if (Heffte_ENABLE_FFTW)
    set(Heffte_FFTW_LIBOMP "" CACHE STRING "The OpenMP library used for FFTW if different from the default for the CXX compiler, e.g., if using clang and fftw uses gnu use -DHeffte_FFTW_LIBOMP=-lgomp")
endif()

# add the file with common CMake helper macros
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/heffte_macros.cmake")

set(Heffte_langs "CXX")
if (CMAKE_C_COMPILER)
    enable_language(C)
    set(Heffte_langs "${Heffte_langs} C")
endif()
if (Heffte_ENABLE_FORTRAN)
    enable_language(Fortran)
    set(Heffte_langs "${Heffte_langs} Fortran")
endif()

if (Heffte_ENABLE_CUDA AND NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
    if (CMAKE_VERSION VERSION_LESS 3.24)
        set(CMAKE_CUDA_ARCHITECTURES "OFF" CACHE STRING "CUDA architectures to compile, e.g., -DCMAKE_CUDA_ARCHITECTURES=70;72")
    else()
        set(CMAKE_CUDA_ARCHITECTURES "native" CACHE STRING "CUDA architectures to compile, e.g., -DCMAKE_CUDA_ARCHITECTURES=70;72")
    endif()
endif()
if (Heffte_ENABLE_ROCM AND NOT DEFINED CMAKE_HIP_ARCHITECTURES)
    set(CMAKE_HIP_ARCHITECTURES "OFF" CACHE STRING "HIP architectures to compile, e.g., -DCMAKE_HIP_ARCHITECTURES=gfx803;gfx1100")
endif()

# oneAPI requires regular MKL as a CPU backend
if (Heffte_ENABLE_ONEAPI)
    set(Heffte_ENABLE_MKL ON CACHE BOOL "oneAPI requires Intel MKL backend to be enabled" FORCE)
endif()

if (Heffte_ENABLE_AVX512)
    set(Heffte_ENABLE_AVX ON CACHE BOOL "AVX with 256-bits requires AVX512" FORCE)
endif()

# guard against in-source builds (may be tricked by sym-links, but it is as good as it gets)
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    message(FATAL_ERROR "In-source builds are not allowed, please perform an out-of-source or out-of-place build, see https://cmake.org/runningcmake/ for details.")
endif()

# Set default install path to build
if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set (CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}" CACHE PATH "default install path" FORCE )
endif()

if (Heffte_ENABLE_TESTING)
    enable_testing()
endif()

# try to find git and if available, set the hit-hash into the header
find_package(Git)
# do not set the hash if git is missing or
# if we are generating files for simple GNU Make compatibility
if (Git_FOUND)
    execute_process(COMMAND ${GIT_EXECUTABLE} log --pretty=format:%H -n 1
                    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
                    OUTPUT_VARIABLE   Heffte_GIT_HASH)
else()
    set(Heffte_GIT_HASH "Unknown, CMake could not find git.")
endif()

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/include/heffte_config.cmake.h"
               "${CMAKE_CURRENT_BINARY_DIR}/include/heffte_config.h")

# find common dependencies
find_package(MPI REQUIRED) # always a dependency

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

# define and populate CMAKE_INSTALL_*DIR variables
include(GNUInstallDirs)

# libheffte source files list (does not include the cuda kernels)
add_library(Heffte
    include/heffte.h
    include/heffte_c.h
    include/heffte_utils.h
    include/heffte_trace.h
    include/heffte_geometry.h
    include/heffte_common.h
    include/heffte_backend_vector.h
    include/heffte_magma_helpers.h
    include/heffte_plan_logic.h
    include/heffte_pack3d.h
    include/heffte_reshape3d.h
    include/heffte_compute_transform.h
    include/heffte_fft3d.h
    include/heffte_fft3d_r2c.h
    include/heffte_r2r_executor.h
    include/stock_fft/heffte_stock_algos.h
    include/stock_fft/heffte_stock_allocator.h
    include/stock_fft/heffte_stock_complex.h
    include/stock_fft/heffte_stock_tree.h
    include/stock_fft/heffte_stock_vec_types.h
    src/heffte_c.cpp
    src/heffte_plan_logic.cpp
    src/heffte_magma_helpers.cpp
    src/heffte_reshape3d.cpp
    src/heffte_compute_transform.cpp
    $<$<BOOL:${Heffte_ENABLE_CUDA}>:include/heffte_backend_cuda.h>
    $<$<BOOL:${Heffte_ENABLE_CUDA}>:src/heffte_backend_cuda.cu>
    $<$<BOOL:${Heffte_ENABLE_ROCM}>:include/heffte_backend_rocm.h>
    $<$<BOOL:${Heffte_ENABLE_ROCM}>:src/heffte_backend_rocm.hip.cpp>
    $<$<BOOL:${Heffte_ENABLE_ONEAPI}>:include/heffte_backend_oneapi.h>
    $<$<BOOL:${Heffte_ENABLE_ONEAPI}>:src/heffte_backend_oneapi.cpp>
    )

# handle other dependencies
target_link_libraries(Heffte MPI::MPI_CXX)

if (Heffte_ENABLE_FFTW)
    find_package(HeffteFFTW REQUIRED)
    target_link_libraries(Heffte Heffte::FFTW)
endif()

if (Heffte_ENABLE_MKL)
    find_package(HeffteMKL REQUIRED)
    target_link_libraries(Heffte Heffte::MKL)
endif()

# build CPU libheffte, the CUDA test comes first to use cuda_add_library() vs add_library()
if (Heffte_ENABLE_CUDA)
    enable_language(CUDA)
    find_package(CUDAToolkit REQUIRED)
    set_source_files_properties(include/heffte_backend_cuda.h src/heffte_backend_cuda.cu PROPERTIES LANGUAGE CUDA)
    target_link_libraries(Heffte CUDA::cufft CUDA::cudart)
endif()

if (Heffte_ENABLE_ROCM)
    if (CMAKE_VERSION VERSION_LESS 3.21)
        message(FATAL_ERROR "heFFTe HIP/ROCm GPU backend requires CMake version 3.21 or newer")
    endif()
    if (ROCM_PATH)
        list (APPEND CMAKE_PREFIX_PATH ${ROCM_PATH}/hip ${ROCM_PATH})
    elseif (DEFINED ENV{ROCM_PATH})
        list (APPEND CMAKE_PREFIX_PATH ENV{ROCM_PATH}/hip ENV{ROCM_PATH})
    elseif (IS_DIRECTORY /opt/rocm)
        list (APPEND CMAKE_PREFIX_PATH /opt/rocm/hip /opt/rocm)
    endif()
    enable_language(HIP)
    find_package(rocfft REQUIRED)
    set_source_files_properties(include/heffte_backend_rocm.h src/heffte_backend_rocm.hip.cpp PROPERTIES LANGUAGE HIP)
    target_link_libraries(Heffte roc::rocfft hip::host)
endif()

if (Heffte_ENABLE_ONEAPI)
    find_package(HeffteOneApi REQUIRED)
    target_link_libraries(Heffte Heffte::OneMKL)
    target_compile_options(Heffte PUBLIC $<$<COMPILE_LANGUAGE:CXX>:-fsycl>)
    target_link_options(Heffte PUBLIC $<$<COMPILE_LANGUAGE:CXX>:-fsycl>)
endif()

if (Heffte_ENABLE_MAGMA)
    find_package(HeffteMAGMA REQUIRED)
    target_link_libraries(Heffte Heffte::MAGMA)
endif()

# other target properties
if (Heffte_ENABLE_AVX)
    target_compile_options(Heffte PUBLIC -mfma -mavx)
endif()
if (Heffte_ENABLE_AVX512)
    target_compile_options(Heffte PUBLIC -mavx512f -mavx512dq)
endif()

target_compile_features(Heffte PUBLIC cxx_std_11)
set_target_properties(Heffte PROPERTIES OUTPUT_NAME "heffte"
                                        CXX_EXTENSIONS OFF
                                        SOVERSION ${Heffte_VERSION_MAJOR}
                                        VERSION   ${PROJECT_VERSION})

# include folders
target_include_directories(Heffte PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/>)
target_include_directories(Heffte PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include/>)
target_include_directories(Heffte PUBLIC $<INSTALL_INTERFACE:include>)

if (Heffte_ENABLE_FORTRAN)
    add_subdirectory(fortran)
endif()
if (Heffte_ENABLE_PYTHON)
    add_subdirectory(python)
endif()


###########################
# GNU Make dependencies
###########################
set(heffte_dep_libs_list "")
set(heffte_dep_includes_list "")
if (TARGET Heffte::FFTW)
    list(APPEND heffte_dep_libs_list ${FFTW_LIBRARIES})
    list(APPEND heffte_dep_includes_list ${FFTW_INCLUDES})
endif()
if (TARGET Heffte::MKL)
    list(APPEND heffte_dep_libs_list ${Heffte_MKL_LIBRARIES})
    list(APPEND heffte_dep_includes_list ${Heffte_MKL_INCLUDES})
endif()
if (TARGET CUDA::cufft)
    get_target_property(_heffte_list CUDA::cufft IMPORTED_LOCATION)
    list(APPEND heffte_dep_libs_list ${_heffte_list})
    get_target_property(_heffte_list CUDA::cudart IMPORTED_LOCATION)
    list(APPEND heffte_dep_libs_list ${_heffte_list})
    list(APPEND heffte_dep_includes_list ${CUDAToolkit_INCLUDE_DIRS})
endif()
if (TARGET roc::rocfft)
    list(APPEND heffte_dep_libs_list -L${rocfft_INCLUDE_DIR}/../lib/ -lrocfft )
    list(APPEND heffte_dep_includes_list ${HIP_INCLUDE_DIR})
    list(APPEND heffte_dep_includes_list ${rocfft_INCLUDE_DIR})
endif()
set(heffte_dep_libs "")
foreach(_heffte_var ${heffte_dep_libs_list})
    set(heffte_dep_libs "${heffte_dep_libs} ${_heffte_var}")
endforeach()
set(heffte_dep_includes "")
foreach(_heffte_var ${heffte_dep_includes_list})
    set(heffte_dep_includes "${heffte_dep_includes} -I${_heffte_var}")
endforeach()

if (Heffte_ENABLE_AVX)
    set(heffte_dep_includes "${heffte_dep_includes} -mfma -mavx")
endif()
if (Heffte_ENABLE_AVX512)
    set(heffte_dep_includes "${heffte_dep_includes} -mavx512f -mavx512dq")
endif()

if (Heffte_ENABLE_FORTRAN)
    set(heffte_fortran_backends "")
    foreach(heffte_backend stock fftw mkl cufft rocfft onemkl)
        if (TARGET "heffte_${heffte_backend}")
            set(heffte_fortran_backends "${heffte_fortran_backends} -lheffte${heffte_backend}fortran")
        endif()
    endforeach()
endif()
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/HeffteMakefile.in"
               "${CMAKE_CURRENT_BINARY_DIR}/configured/HeffteMakefile.in")


###########################
# Documentation
###########################
if (Heffte_ENABLE_DOXYGEN)
    # must come after add_library(Heffte ...)
    add_subdirectory(doxygen)
endif()


###########################
# install
###########################
install(TARGETS Heffte EXPORT Heffte_Targets DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(EXPORT Heffte_Targets FILE HeffteTargets.cmake NAMESPACE Heffte:: DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Heffte)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/HeffteConfig.cmake" "${CMAKE_CURRENT_BINARY_DIR}/HeffteConfigVersion.cmake" DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Heffte)
install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR} FILES_MATCHING PATTERN "*.h"
                                                              PATTERN "**~" EXCLUDE
                                                              PATTERN "*_gpu*" EXCLUDE
                                                              PATTERN "*.cu*" EXCLUDE
                                                              )
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/heffte_config.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/configured/HeffteMakefile.in DESTINATION ${CMAKE_INSTALL_DATADIR}/heffte/)

# package-config
include(CMakePackageConfigHelpers)
write_basic_package_version_file("HeffteConfigVersion.cmake" VERSION ${PROJECT_VERSION} COMPATIBILITY SameMajorVersion)
configure_package_config_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/HeffteConfig.cmake"
                              "${CMAKE_CURRENT_BINARY_DIR}/HeffteConfig.cmake"
                              INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/Heffte/")


######################
# EXAMPLES and TESTS #
######################
if (Heffte_ENABLE_TESTING)
    add_subdirectory(benchmarks)
    add_subdirectory(examples)
    add_subdirectory(test)
endif()
if (${CMAKE_PROJECT_NAME} STREQUAL ${PROJECT_NAME})
    if (CUDA_INCLUDE_DIRS)
        # cuda_add_library() apparently adds the includes as private
        target_include_directories(Heffte PUBLIC ${CUDA_INCLUDE_DIRS})
    endif()
    add_library(Heffte::Heffte INTERFACE IMPORTED GLOBAL)
    target_link_libraries(Heffte::Heffte INTERFACE Heffte)
endif()


###########################
# Post Install Test
###########################
# The REGEX helps accept both list and regular set of flags.
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/MakeTest.cmake" "${CMAKE_CURRENT_BINARY_DIR}/configured/CMakeLists.txt" @ONLY)
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/test/post_install_test.cmake.sh" "${CMAKE_CURRENT_BINARY_DIR}/post_install_test.sh" @ONLY)
add_custom_target(test_install COMMAND bash "${CMAKE_CURRENT_BINARY_DIR}/post_install_test.sh")
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/configured/CMakeLists.txt"
        DESTINATION "share/heffte/testing/"
        PERMISSIONS OWNER_WRITE OWNER_READ GROUP_READ WORLD_READ)


##############################
# Examples for post install
##############################
if (Heffte_ENABLE_TESTING)
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/ExampleCMakeLists.cmake" "${CMAKE_CURRENT_BINARY_DIR}/examples/CMakeLists.txt" @ONLY)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/examples/CMakeLists.txt" DESTINATION ${CMAKE_INSTALL_DATADIR}/heffte/examples)
    install(DIRECTORY examples/ DESTINATION ${CMAKE_INSTALL_DATADIR}/heffte/examples FILES_MATCHING PATTERN "*.cpp" PATTERN "*.c" PATTERN "*.f90")
endif()

# print summary of the CMake options, skip if using add_subdirectory(heffte)
if (${CMAKE_PROJECT_NAME} STREQUAL ${PROJECT_NAME})
    include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/print_summary.cmake)
endif()


##############################
# Handles the rpath
##############################
include(cmake/set_rpath.cmake)
