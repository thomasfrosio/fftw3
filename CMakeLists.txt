cmake_minimum_required (VERSION 3.20)

if (NOT DEFINED CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type")
endif ()

project(fftw VERSION 3.3.10)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (POLICY CMP0042)
  cmake_policy(SET CMP0042 NEW)
endif ()

option(FFTW3_BUILD_TESTS "Build tests" OFF)
option(FFTW3_ENABLE_OPENMP "Use OpenMP for multithreading" OFF)
option(FFTW3_ENABLE_THREADS "Use pthread for multithreading" OFF)
option(FFTW3_ENABLE_SSE "Compile with SSE instruction set support" ON)
option(FFTW3_ENABLE_SSE2 "Compile with SSE2 instruction set support" ON)
option(FFTW3_ENABLE_AVX "Compile with AVX instruction set support" ON)
option(FFTW3_ENABLE_AVX2 "Compile with AVX2 instruction set support" ON)

include(GNUInstallDirs)
include(CheckIncludeFile)
check_include_file(alloca.h         HAVE_ALLOCA_H)
check_include_file(altivec.h        HAVE_ALTIVEC_H)
check_include_file(c_asm.h          HAVE_C_ASM_H)
check_include_file(dlfcn.h          HAVE_DLFCN_H)
check_include_file(intrinsics.h     HAVE_INTRINSICS_H)
check_include_file(inttypes.h       HAVE_INTTYPES_H)
check_include_file(libintl.h        HAVE_LIBINTL_H)
check_include_file(limits.h         HAVE_LIMITS_H)
check_include_file(mach/mach_time.h HAVE_MACH_MACH_TIME_H)
check_include_file(malloc.h         HAVE_MALLOC_H)
check_include_file(memory.h         HAVE_MEMORY_H)
check_include_file(stddef.h         HAVE_STDDEF_H)
check_include_file(stdint.h         HAVE_STDINT_H)
check_include_file(stdlib.h         HAVE_STDLIB_H)
check_include_file(string.h         HAVE_STRING_H)
check_include_file(strings.h        HAVE_STRINGS_H)
check_include_file(sys/types.h      HAVE_SYS_TYPES_H)
check_include_file(sys/time.h       HAVE_SYS_TIME_H)
check_include_file(sys/stat.h       HAVE_SYS_STAT_H)
check_include_file(sys/sysctl.h     HAVE_SYS_SYSCTL_H)
check_include_file(time.h           HAVE_TIME_H)
check_include_file(uintptr.h        HAVE_UINTPTR_H)
check_include_file(unistd.h         HAVE_UNISTD_H)
if (HAVE_TIME_H AND HAVE_SYS_TIME_H)
  set(TIME_WITH_SYS_TIME TRUE)
endif ()

include(CheckPrototypeDefinition)
check_prototype_definition(drand48 "double drand48 (void)" "0" stdlib.h HAVE_DECL_DRAND48)
check_prototype_definition(srand48 "void srand48(long int seedval)" "0" stdlib.h HAVE_DECL_SRAND48)
check_prototype_definition(cosl "long double cosl( long double arg )" "0" math.h HAVE_DECL_COSL)
check_prototype_definition(sinl "long double sinl( long double arg )" "0" math.h HAVE_DECL_SINL)
check_prototype_definition(memalign "void *memalign(size_t alignment, size_t size)" "0" malloc.h HAVE_DECL_MEMALIGN)
check_prototype_definition(posix_memalign "int posix_memalign(void **memptr, size_t alignment, size_t size)" "0" stdlib.h HAVE_DECL_POSIX_MEMALIGN)

include(CheckSymbolExists)
check_symbol_exists(clock_gettime time.h HAVE_CLOCK_GETTIME)
check_symbol_exists(gettimeofday sys/time.h HAVE_GETTIMEOFDAY)
check_symbol_exists(getpagesize unistd.h HAVE_GETPAGESIZE)
check_symbol_exists(drand48 stdlib.h HAVE_DRAND48)
check_symbol_exists(srand48 stdlib.h HAVE_SRAND48)
check_symbol_exists(memalign malloc.h HAVE_MEMALIGN)
check_symbol_exists(posix_memalign stdlib.h HAVE_POSIX_MEMALIGN)
check_symbol_exists(mach_absolute_time mach/mach_time.h HAVE_MACH_ABSOLUTE_TIME)
check_symbol_exists(alloca alloca.h HAVE_ALLOCA)
if (NOT HAVE_ALLOCA)
  unset(HAVE_ALLOCA CACHE)
  check_symbol_exists(alloca malloc.h HAVE_ALLOCA)
endif ()
check_symbol_exists(isnan math.h HAVE_ISNAN)
check_symbol_exists(snprintf stdio.h HAVE_SNPRINTF)
check_symbol_exists(strchr string.h HAVE_STRCHR)
check_symbol_exists(sysctl unistd.h HAVE_SYSCTL)

if (UNIX)
  set(CMAKE_REQUIRED_LIBRARIES m)
endif ()
check_symbol_exists(cosl math.h HAVE_COSL)
check_symbol_exists(sinl math.h HAVE_SINL)

include(CheckTypeSize)
check_type_size("float" SIZEOF_FLOAT)
check_type_size("double" SIZEOF_DOUBLE)
check_type_size("int" SIZEOF_INT)
check_type_size("long" SIZEOF_LONG)
check_type_size("long long" SIZEOF_LONG_LONG)
check_type_size("unsigned int" SIZEOF_UNSIGNED_INT)
check_type_size("unsigned long" SIZEOF_UNSIGNED_LONG)
check_type_size("unsigned long long" SIZEOF_UNSIGNED_LONG_LONG)
check_type_size("size_t" SIZEOF_SIZE_T)
check_type_size("ptrdiff_t" SIZEOF_PTRDIFF_T)
math(EXPR SIZEOF_INT_BITS "8 * ${SIZEOF_INT}")
set(C_FFTW_R2R_KIND "C_INT${SIZEOF_INT_BITS}_T")

find_library(LIBM_LIBRARY NAMES m)
if (LIBM_LIBRARY)
  set(HAVE_LIBM TRUE)
endif ()

# Multithreading:
if (FFTW3_ENABLE_THREADS OR FFTW3_ENABLE_OPENMP)
  find_package(Threads)
  if (FFTW3_ENABLE_OPENMP)
    find_package(OpenMP)
  endif ()
endif ()
if (Threads_FOUND)
  if (CMAKE_USE_PTHREADS_INIT)
    set(USING_POSIX_THREADS 1)
  endif ()
endif ()

if (FFTW3_ENABLE_THREADS)
  set(HAVE_THREADS TRUE)
endif ()
if (FFTW3_OPENMP_FOUND)
  set(HAVE_OPENMP TRUE)
endif ()

# SIMD:
include(CheckCCompilerFlag)
if (FFTW3_ENABLE_SSE)
  foreach (FLAG "-msse" "/arch:SSE")
    unset(HAVE_SSE CACHE)
    unset(HAVE_SSE)
    check_c_compiler_flag(${FLAG} HAVE_SSE)
    if (HAVE_SSE)
      set(SSE_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (FFTW3_ENABLE_SSE2)
  foreach (FLAG "-msse2" "/arch:SSE2")
    unset(HAVE_SSE2 CACHE)
    unset(HAVE_SSE2)
    check_c_compiler_flag(${FLAG} HAVE_SSE2)
    if (HAVE_SSE2)
      set(SSE2_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (FFTW3_ENABLE_AVX)
  foreach (FLAG "-mavx" "/arch:AVX")
    unset(HAVE_AVX CACHE)
    unset(HAVE_AVX)
    check_c_compiler_flag(${FLAG} HAVE_AVX)
    if (HAVE_AVX)
      set(AVX_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (FFTW3_ENABLE_AVX2)
  foreach (FLAG "-mavx2" "/arch:AVX2")
    unset(HAVE_AVX2 CACHE)
    unset(HAVE_AVX2)
    check_c_compiler_flag(${FLAG} HAVE_AVX2)
    if (HAVE_AVX2)
      set(AVX2_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

# AVX2 codelets require FMA support as well
if (FFTW3_ENABLE_AVX2)
  foreach (FLAG "-mfma" "/arch:FMA")
    unset(HAVE_FMA CACHE)
    unset(HAVE_FMA)
    check_c_compiler_flag(${FLAG} HAVE_FMA)
    if (HAVE_FMA)
      set(FMA_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (HAVE_SSE2 OR HAVE_AVX)
  set(HAVE_SIMD TRUE)
endif ()

# Set the source files:
file(GLOB fftw_api_SOURCE                 api/*.c                   api/*.h)
file(GLOB fftw_dft_SOURCE                 dft/*.c                   dft/*.h)
file(GLOB fftw_dft_scalar_SOURCE          dft/scalar/*.c            dft/scalar/*.h)
file(GLOB fftw_dft_scalar_codelets_SOURCE dft/scalar/codelets/*.c   dft/scalar/codelets/*.h)
file(GLOB fftw_dft_simd_SOURCE            dft/simd/*.c              dft/simd/*.h)

file(GLOB fftw_dft_simd_sse2_SOURCE       dft/simd/sse2/*.c   dft/simd/sse2/*.h)
file(GLOB fftw_dft_simd_avx_SOURCE        dft/simd/avx/*.c    dft/simd/avx/*.h)
file(GLOB fftw_dft_simd_avx2_SOURCE       dft/simd/avx2/*.c   dft/simd/avx2/*.h dft/simd/avx2-128/*.c dft/simd/avx2-128/*.h)
file(GLOB fftw_kernel_SOURCE              kernel/*.c          kernel/*.h)
file(GLOB fftw_rdft_SOURCE                rdft/*.c            rdft/*.h)
file(GLOB fftw_rdft_scalar_SOURCE         rdft/scalar/*.c     rdft/scalar/*.h)

file(GLOB fftw_rdft_scalar_r2cb_SOURCE    rdft/scalar/r2cb/*.c rdft/scalar/r2cb/*.h)
file(GLOB fftw_rdft_scalar_r2cf_SOURCE    rdft/scalar/r2cf/*.c rdft/scalar/r2cf/*.h)
file(GLOB fftw_rdft_scalar_r2r_SOURCE     rdft/scalar/r2r/*.c  rdft/scalar/r2r/*.h)

file(GLOB fftw_rdft_simd_SOURCE           rdft/simd/*.c       rdft/simd/*.h)
file(GLOB fftw_rdft_simd_sse2_SOURCE      rdft/simd/sse2/*.c  rdft/simd/sse2/*.h)
file(GLOB fftw_rdft_simd_avx_SOURCE       rdft/simd/avx/*.c   rdft/simd/avx/*.h)
file(GLOB fftw_rdft_simd_avx2_SOURCE      rdft/simd/avx2/*.c  rdft/simd/avx2/*.h rdft/simd/avx2-128/*.c rdft/simd/avx2-128/*.h)

file(GLOB fftw_reodft_SOURCE              reodft/*.c        reodft/*.h)
file(GLOB fftw_simd_support_SOURCE        simd-support/*.c  simd-support/*.h)
file(GLOB fftw_libbench2_SOURCE           libbench2/*.c     libbench2/*.h)
list(REMOVE_ITEM fftw_libbench2_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/libbench2/useropt.c)

set(SOURCEFILES
    ${fftw_api_SOURCE}
    ${fftw_dft_SOURCE}
    ${fftw_dft_scalar_SOURCE}
    ${fftw_dft_scalar_codelets_SOURCE}
    ${fftw_dft_simd_SOURCE}
    ${fftw_kernel_SOURCE}
    ${fftw_rdft_SOURCE}
    ${fftw_rdft_scalar_SOURCE}

    ${fftw_rdft_scalar_r2cb_SOURCE}
    ${fftw_rdft_scalar_r2cf_SOURCE}
    ${fftw_rdft_scalar_r2r_SOURCE}

    ${fftw_rdft_simd_SOURCE}
    ${fftw_reodft_SOURCE}
    ${fftw_simd_support_SOURCE}
    ${fftw_threads_SOURCE}
)

# Add multithreading source files:
set(fftw_par_SOURCE
    threads/api.c
    threads/conf.c
    threads/ct.c
    threads/dft-vrank-geq1.c
    threads/f77api.c
    threads/hc2hc.c
    threads/rdft-vrank-geq1.c
    threads/vrank-geq1-rdft2.c)

if (Threads_FOUND)
  list (APPEND SOURCEFILES ${fftw_par_SOURCE} threads/threads.c)
elseif (OPENMP_FOUND)
  list (APPEND SOURCEFILES ${fftw_par_SOURCE} threads/openmp.c)
endif ()

# Add SIMD source files:
if (HAVE_SSE2)
  list(APPEND SOURCEFILES ${fftw_dft_simd_sse2_SOURCE} ${fftw_rdft_simd_sse2_SOURCE})
endif ()
if (HAVE_AVX)
  list(APPEND SOURCEFILES ${fftw_dft_simd_avx_SOURCE} ${fftw_rdft_simd_avx_SOURCE})
endif ()
if (HAVE_AVX2)
  list(APPEND SOURCEFILES ${fftw_dft_simd_avx2_SOURCE} ${fftw_rdft_simd_avx2_SOURCE})
endif ()

# Get the config files, for double and single precision:
set(FFTW_VERSION ${PROJECT_VERSION})
set(FFTW_SINGLE FALSE)
set(BENCHFFT_SINGLE FALSE)
set(PREC_SUFFIX)
configure_file(cmake.config.h.in ${CMAKE_CURRENT_BINARY_DIR}/double/config.h @ONLY)

set(FFTW_SINGLE TRUE)
set(BENCHFFT_SINGLE TRUE)
set(PREC_SUFFIX f)
configure_file(cmake.config.h.in ${CMAKE_CURRENT_BINARY_DIR}/float/config.h @ONLY)

# Common options:
add_library(fftw3_public_options INTERFACE)
add_library(fftw3_private_options INTERFACE)

if (MSVC AND NOT (CMAKE_C_COMPILER_ID STREQUAL "Intel"))
  target_compile_definitions(fftw3_private_options INTERFACE /bigobj)
endif ()
target_compile_options(fftw3_private_options INTERFACE
    $<$<BOOL:HAVE_SSE>:${SSE_FLAG}>
    $<$<BOOL:HAVE_SSE2>:${SSE2_FLAG}>
    $<$<BOOL:HAVE_AVX>:${AVX_FLAG}>
    $<$<BOOL:HAVE_AVX2>:${AVX2_FLAG}>
    $<$<BOOL:HAVE_FMA>:${FMA_FLAG}>)
target_link_libraries(fftw3_private_options INTERFACE $<$<BOOL:HAVE_LIBM>:m>)

target_include_directories(fftw3_private_options INTERFACE "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>")
target_include_directories(fftw3_public_options INTERFACE $<INSTALL_INTERFACE:include>) # FIXME

# Set double and single precision libraries.
add_library(fftw3 STATIC ${SOURCEFILES})
add_library(fftw3f STATIC ${SOURCEFILES})
add_library(FFTW3::fftw3 ALIAS fftw3)
add_library(FFTW3::fftw3f ALIAS fftw3f)

target_include_directories(fftw3 PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/double)
target_include_directories(fftw3f PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/float)

set(targets fftw3 fftw3f)
foreach (target ${targets})
  target_link_libraries(${target} PRIVATE fftw3_private_options)
  target_link_libraries(${target} PUBLIC fftw3_public_options)
  if (Threads_FOUND)
    target_link_libraries(${target} PRIVATE Threads::Threads ${CMAKE_THREAD_LIBS_INIT})
  elseif (OPENMP_FOUND)
    target_link_libraries(${target} PRIVATE Threads::Threads ${CMAKE_THREAD_LIBS_INIT} OpenMP::OpenMP_C)
  endif ()

  set_target_properties(${target} PROPERTIES SOVERSION 3.3.10 VERSION 3)
endforeach ()

# Test/Bench:
if (FFTW3_BUILD_TESTS)
  set(bench_source_files ${fftw_libbench2_SOURCE} tests/bench.c tests/hook.c tests/fftw-bench.c)
  add_executable(bench ${bench_source_files})
  target_link_libraries(bench PRIVATE fftw3)

  add_executable(benchf ${bench_source_files})
  target_link_libraries(benchf PRIVATE fftw3f)

  # We have made the config.h private, but bench needs to include it, so include it explicitly here...
  target_include_directories(bench PRIVATE ${PROJECT_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR}/double)
  target_include_directories(benchf PRIVATE ${PROJECT_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR}/float)

  enable_testing()
  if (Threads_FOUND)
    macro(fftw_add_test problem)
      add_test(NAME ${problem} COMMAND bench -s ${problem})
      add_test(NAME ${problem}f COMMAND benchf -s ${problem})
    endmacro()
    fftw_add_test(32x64)
    fftw_add_test(ib256)
  endif ()
endif ()

# Install targets and public header:
install(
    TARGETS fftw3_private_options fftw3_public_options fftw3 fftw3f
    EXPORT FFTW3
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
install (FILES api/fftw3.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# CMake config files:
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${PROJECT_SOURCE_DIR}/FFTW3Config.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/FFTW3Config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/fftw3)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/FFTW3ConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion)
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/FFTW3Config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/FFTW3ConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/fftw3)

# Export:
install(EXPORT FFTW3
    FILE FFTW3LibraryDepends.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/fftw3
    NAMESPACE FFTW3::
)

#export(TARGETS ${fftw3_lib} NAMESPACE FFTW3:: FILE ${PROJECT_BINARY_DIR}/FFTW3LibraryDepends.cmake)
#install(EXPORT FFTW3LibraryDepends
#    NAMESPACE FFTW3::
#    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/fftw3${PREC_SUFFIX}
#    COMPONENT Development)
