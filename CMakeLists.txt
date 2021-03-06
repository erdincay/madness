cmake_minimum_required (VERSION 3.1.0)
project(MADNESS CXX C ASM)

# Check CMake version
if(CMAKE_VERSION VERSION_LESS 3.5.1)  
  message("WARNING: If parallel build fails halfway through, reduce the value of the parallel build to 'make -j 4'. Or try upgrading to CMake 3.5.1.")
endif()

# Set MADNESS version
set(MADNESS_MAJOR_VERSION 0)
set(MADNESS_MINOR_VERSION 10)
set(MADNESS_MICRO_VERSION 1)
set(MADNESS_VERSION "${MADNESS_MAJOR_VERSION}.${MADNESS_MINOR_VERSION}.${MADNESS_MICRO_VERSION}")

# Add module directory and modules =============================================

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/modules/")
include(AppendFlags)
include(CheckIncludeFile)
include(CheckTypeSize)
include(CheckCXXSourceCompiles)
include(CheckFunctionExists)
include(CMakeDependentOption)
include(AddUnittests)
include(AddMADLibrary)
include(CMakePackageConfigHelpers)
include(CopyTargetProperties)
include(FeatureSummary)

# Set install paths ============================================================

set(MADNESS_INSTALL_BINDIR "bin" 
    CACHE PATH "MADNESS binary install directory")
set(MADNESS_INSTALL_INCLUDEDIR "include" 
    CACHE PATH "MADNESS INCLUDE install directory")
set(MADNESS_INSTALL_LIBDIR "lib"
    CACHE PATH "MADNESS LIB install directory")
set(MADNESS_INSTALL_DATADIR "share/madness/${MADNESS_VERSION}/data"
    CACHE PATH "MADNESS DATA install directory")
# TODO this matches Automake value, should be adjusted to the convention: share/madness/${MADNESS_VERSION}/doc
set(MADNESS_INSTALL_DOCDIR "share/doc/madness"
    CACHE PATH "MADNESS DOC install directory")
set(MADNESS_INSTALL_CMAKEDIR "lib/cmake/madness"
    CACHE PATH "MADNESS CMAKE install directory")

# Enable optional libraries ====================================================

# Always search for libraries that are 'ON' by default. If a library is not
# found, it is disable without error. If the library is 'OFF' by default,
# failure to find the library is an error. 

option(ENABLE_MPI "Enable Message Passing Interface (MPI) Library" ON)
option(ENABLE_MKL "Search for Intel Math Kernel Library (MKL) for BLAS and LAPACK support" ON)
option(ENABLE_ACML "Search for AMD Core Math Library (ACML) for BLAS and LAPACK support" ON)
option(ENABLE_TCMALLOC_MINIMAL "Enable use of tcmalloc_minimal library from Google Performance Tools (gperftools)" OFF)
cmake_dependent_option(ENABLE_GPERFTOOLS "Enable use of Google Performance Tools (gperftools)" ON
    "NOT ENABLE_TCMALLOC_MINIMAL" OFF)
option(ENABLE_LIBXC "Enables use of the libxc library of density functionals" ON)
option(ENABLE_PAPI "Enables use of PAPI" OFF)

option(ENABLE_TBB "Enables use of Intel Thread Building Blocks (TBB) as the task scheduler" ON)
option(ENABLE_PARSEC "Enables use of PaRSEC as the task scheduler" OFF)
if(ENABLE_PARSEC AND ENABLE_TBB)
  message(FATAL_ERROR "TBB and PaRSEC cannot be both enabled.")
endif()

option(ENABLE_ELEMENTAL "Enable Elemental library for distributed-memory linear algebra" OFF)
if (ENABLE_ELEMENTAL)
  if (DEFINED ELEMENTAL_TAG)
    if("${ELEMENTAL_TAG}" STREQUAL "")
      message(FATAL_ERROR "Invalid value given for ELEMENTAL_TAG; specify a valid hash or tag.")
    endif()
    message(STATUS "Will compile Elemental source (tag=${ELEMENTAL_TAG}) in the build directory")
    set(ENABLE_ELEMENTAL_EMBEDDED OFF)
    set(MADNESS_HAS_ELEMENTAL_EMBEDDED 0)
  else (DEFINED ELEMENTAL_TAG)
    message(STATUS "Will use the embedded Elemental source(tag=v0.84)")
    set(ENABLE_ELEMENTAL_EMBEDDED ON)
    set(MADNESS_HAS_ELEMENTAL_EMBEDDED 1)
  endif (DEFINED ELEMENTAL_TAG)
  set(MADNESS_HAS_ELEMENTAL 1)
else (ENABLE_ELEMENTAL)
  set(MADNESS_HAS_ELEMENTAL 0)
endif (ENABLE_ELEMENTAL)
add_feature_info(Elemental ENABLE_ELEMENTAL "compiles parallel linear-algebra library Elemental as part of MADNESS")

# Configure options ============================================================

option(ENABLE_GENTENSOR "Enable generic tensor support" OFF)
add_feature_info(GENTENSOR ENABLE_GENTENSOR "enables use of MRA Function compression that allows computing in 6 dimensions")

if(ENABLE_GENTENSOR)
  add_definitions(-DUSE_GENTENSOR)
endif()
set(TENSOR_USE_GENTENSOR ${ENABLE_GENTENSOR} CACHE BOOL "enable generic tensors")

option(ENABLE_TASK_PROFILER
    "Enable task profiler that collects per-task start and stop times." OFF)
add_feature_info(TASK_PROFILER ENABLE_TASK_PROFILER "supports task-level tracing of program execution")
set(MADNESS_TASK_PROFILING ${ENABLE_TASK_PROFILER} CACHE BOOL
    "Enable task profiler that collects per-task start and stop times.")

option(ENABLE_WORLD_PROFILE "Enables profiling" OFF)
add_feature_info(WORLD_PROFILE ENABLE_WORLD_PROFILE "supports simple profiling of MADworld runtime")
set(WORLD_PROFILE_ENABLE ${ENABLE_WORLD_PROFILE} CACHE BOOL 
    "Enables world profiling")

option(ENABLE_MEM_STATS "Gather memory statistics (expensive)" OFF)
add_feature_info(MEM_STATS ENABLE_MEM_STATS "gather memory statistics (expensive)")
set(WORLD_GATHER_MEM_STATS ${ENABLE_MEM_STATS} CACHE BOOL "Gather memory statistics (expensive)")

option(ENABLE_MEM_PROFILE "Turn on instrumented memory profiling (print_meminfo)" OFF)
add_feature_info(MEM_PROFILE ENABLE_MEM_PROFILE "instrumented memory profiling")
set(WORLD_MEM_PROFILE_ENABLE ${ENABLE_MEM_PROFILE} CACHE BOOL "Turn on instrumented memory profiling (print_meminfo)")

option(ENABLE_TENSOR_BOUNDS_CHECKING
    "Enable checking of bounds in tensors ... slow but useful for debugging" OFF)
add_feature_info(TENSOR_BOUNDS_CHECKING ENABLE_TENSOR_BOUNDS_CHECKING
    "Enable checking of bounds in tensors ... slow but useful for debugging")
set(TENSOR_BOUNDS_CHECKING ${ENABLE_TENSOR_BOUNDS_CHECKING} CACHE BOOL
    "Enable checking of bounds in tensors ... slow but useful for debugging")

option(ENABLE_TENSOR_INSTANCE_COUNT
    "Enable counting of allocated tensors for memory leak detection" OFF)
add_feature_info(TENSOR_INSTANCE_COUNT ENABLE_TENSOR_INSTANCE_COUNT
    "Enable counting of allocated tensors for memory leak detection")
set(TENSOR_INSTANCE_COUNT CACHE BOOL
    "Enable counting of allocated tensors for memory leak detection")

option(ENABLE_SPINLOCKS
    "Enables use of spinlocks instead of mutexes (faster unless over subscribing processors)" ON)
add_feature_info(SPINLOCKS ENABLE_SPINLOCKS
    "Enables use of spinlocks instead of mutexes (faster unless over subscribing processors)")
set(USE_SPINLOCKS ${ENABLE_SPINLOCKS} CACHE BOOL
    "Enables use of spinlocks instead of mutexs (faster unless over subscribing processors)")

option(ENABLE_NEVER_SPIN
    "Disables use of spinlocks (notably for use inside virtual machines)" OFF)
add_feature_info(NEVER_SPIN ENABLE_NEVER_SPIN
    "Disables use of spinlocks (notably for use inside virtual machines)")
set(NEVER_SPIN ${ENABLE_NEVER_SPIN} CACHE BOOL
    "Disables use of spinlocks (notably for use inside virtual machines)")

option(ENABLE_BSEND_ACKS 
    "Use MPI Send instead of MPI Bsend for huge message acknowledgements" ON)
add_feature_info(BSEND_ACKS ENABLE_BSEND_ACKS
    "Use MPI Send instead of MPI Bsend for huge message acknowledgements")
set(MADNESS_USE_BSEND_ACKS ${ENABLE_BSEND_ACKS} CACHE BOOL
    "Use MPI Send instead of MPI Bsend for huge message acknowledgements")

option(DISABLE_WORLD_GET_DEFAULT "Disables World::get_default()" OFF)
add_feature_info(WORLD_GET_DEFAULT_DISABLE DISABLE_WORLD_GET_DEFAULT "Disables World::get_default()")
set(WORLD_GET_DEFAULT_DISABLED ${DISABLE_WORLD_GET_DEFAULT} CACHE BOOL 
    "Disables World::get_default()")

option(ENABLE_UNITTESTS "Enables unit tests targets" ON)
add_feature_info(UNITTESTS ENABLE_UNITTESTS
    "Enables unit tests targets")

set(FORTRAN_INTEGER_SIZE 4 CACHE STRING "The fortran integer size (4 or 8 bytes) used for BLAS and LAPACK function calls")
if(NOT (FORTRAN_INTEGER_SIZE EQUAL 4 OR FORTRAN_INTEGER_SIZE EQUAL 8))
  message(FATAL_ERROR "Incorrect fortran integer size '${FORTRAN_INTEGER_SIZE}'\n"
                       "FORTRAN_INTEGER_SIZE must be equal to 4 or 8")
endif()
add_feature_info("FORTRAN_INTEGER_SIZE=${FORTRAN_INTEGER_SIZE}" TRUE "assumes Fortran integers to be ${FORTRAN_INTEGER_SIZE} bytes long")

if (CMAKE_BUILD_TYPE STREQUAL Release OR
    CMAKE_BUILD_TYPE STREQUAL RelWithDebInfo OR
    CMAKE_BUILD_TYPE STREQUAL MinSizeRel)
  set (DEFAULT_ASSERTION_TYPE "disable")
else ()
  set (DEFAULT_ASSERTION_TYPE "throw")
endif()
set(ASSERTION_TYPE ${DEFAULT_ASSERTION_TYPE} CACHE STRING "Define MADNESS assertion behavior (abort|assert|disable|throw)")
if(ASSERTION_TYPE STREQUAL "abort")
  set(MADNESS_ASSERTIONS_ABORT 1)
elseif(ASSERTION_TYPE STREQUAL "assert")
  set(MADNESS_ASSERTIONS_ASSERT 1)
elseif(ASSERTION_TYPE STREQUAL "disable")
  set(MADNESS_ASSERTIONS_DISABLE 1)
elseif(ASSERTION_TYPE STREQUAL "throw")
  set(MADNESS_ASSERTIONS_THROW 1)
else()
  message(WARNING "Unsupported ASSERTION_TYPE '${ASSERTION_TYPE}'")
  set(ASSERTION_TYPE "throw")
  set(MADNESS_ASSERTIONS_THROW 1)
endif()
message(STATUS "Assertion type: ${ASSERTION_TYPE}")
add_feature_info("ASSERTION_TYPE=${ASSERTION_TYPE}" TRUE "controls how MADNESS assertions (MADNESS_ASSERT) are handled")

set(MPI_THREAD "multiple" CACHE STRING "Thread level for MPI (multiple|serialized)")
if(MPI_THREAD STREQUAL "multiple")
  set(MADNESS_MPI_THREAD_LEVEL "MPI_THREAD_MULTIPLE")
elseif(MPI_THREAD STREQUAL "serialized")
  set(MADNESS_MPI_THREAD_LEVEL "MPI_THREAD_SERIALIZED")
  if (ENABLE_ELEMENTAL)
    message (FATAL_ERROR "Use of Elemental precludes MPI_THREAD=serialized, must use MPI_THREAD=multiple")
  endif (ENABLE_ELEMENTAL)
else()
  message(FATAL_ERROR "Invalid value for MPI_THREAD '${MPI_THREAD}'; valid values are 'multiple' or 'serialized'")
endif()
add_feature_info("MPI_THREAD=${MPI_THREAD}" TRUE "controls the level of thread-safety support in MPI")

# Enable support for shared libraries ==========================================

get_property(SUPPORTS_SHARED GLOBAL PROPERTY TARGET_SUPPORTS_SHARED_LIBS)
cmake_dependent_option(BUILD_SHARED_LIBS "Enable shared libraries" ON
      "SUPPORTS_SHARED" OFF)
if(BUILD_SHARED_LIBS)
  set(BLA_STATIC FALSE)
  set(CMAKE_MACOSX_RPATH TRUE)
else()
  set(BLA_STATIC TRUE)
  set(CMAKE_MACOSX_RPATH FALSE)
endif()

set(CMAKE_SKIP_RPATH FALSE)
set(CMAKE_SKIP_BUILD_RPATH  FALSE)
set(CMAKE_SKIP_INSTALL_RPATH FALSE)

# Get host and platform information ============================================

if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
  set(ON_A_MAC 1)
  # look for frameworks and appbundles last
  set(CMAKE_FIND_FRAMEWORK LAST)
  set(CMAKE_FIND_APPBUNDLE LAST)
endif()

if($ENV{USERNAME})
set(MADNESS_CONFIGURATION_USER "$ENV{USERNAME}")
elseif($ENV{USER})
set(MADNESS_CONFIGURATION_USER "$ENV{USER}")
else()
set(MADNESS_CONFIGURATION_USER "$ENV{USER}")
endif()
cmake_host_system_information(RESULT MADNESS_CONFIGURATION_HOST QUERY HOSTNAME)
string(TIMESTAMP MADNESS_CONFIGURATION_DATE)

set(MAD_BIND_DEFAULT "-1 -1 -1" CACHE STRING "The default binding for threads")

# Check if the target platform is CRAY XE
check_cxx_source_compiles(
    "
    #ifndef __CRAYXE
    #error choke me
    #endif
    int main() { return 0; }
    " HAVE_CRAYXE)
# Check if the target platform is CRAY XE
check_cxx_source_compiles(
    "
    #ifndef __CRAYXT
    #error choke me
    #endif
    int main() { return 0; }
    " HAVE_CRAYXT)
if(HAVE_CRAYXE)
  set(AMD_QUADCORE_TUNE ON CACHE BOOL "Target for tuning mtxmq kernels")
  set(USE_SPINLOCKS ON CACHE BOOL 
      "Enables use of spinlocks instead of mutexs (faster unless over subscribing processors)" FORCE)
  set(MAD_BIND_DEFAULT "1 0 2" CACHE STRING "The default binding for threads" FORCE)
  set(MPI_C_COMPILER cc CACHE STRING "CRAY MPI C compiler")
  set(MPI_CXX_COMPILER CC CACHE STRING "CRAY MPI C++ compiler")
endif()

# Check if the target platform is BG/P
check_cxx_source_compiles(
    "
    #ifndef __bgp__
    #error choke me
    #endif
    int main() { return 0; }
    " HAVE_IBMBGP)

# Check if the target platform is BG/Q
check_cxx_source_compiles(
    "
    #ifndef __bgq__
    #error choke me
    #endif
    int main() { return 0; }
    " HAVE_IBMBGQ)

if(HAVE_IBMBGQ OR HAVE_IBMBGP)
  set(USE_SPINLOCKS ON CACHE BOOL 
      "Enables use of spinlocks instead of mutexs (faster unless over subscribing processors)" FORCE)
endif()


# Check if the target is x86_64
check_cxx_source_compiles(
    "
    #if !(defined(__x86_64__) || defined(_M_X64))
    #error Not x86_64
    #endif
    int main() { return 0; }
    " USE_X86_64_ASM)

if(NOT USE_X86_64_ASM)
  # Check if the target is x86
  check_cxx_source_compiles(
      "
      #if !(defined(__i386) || defined(_M_IX86))
      #error Not x86
      #endif
      int main() { return 0; }
      " USE_X86_32_ASM)
endif()

# Check compiler feature support ===============================================

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Check for system include files
check_include_file(sys/stat.h HAVE_SYS_STAT_H)
check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(unistd.h HAVE_UNISTD_H)
if(MADNESS_TASK_PROFILING)
  check_include_file(execinfo.h HAVE_EXECINFO_H)
  check_include_file(cxxabi.h HAVE_CXXABI_H)
  if(NOT (HAVE_EXECINFO_H AND HAVE_CXXABI_H))
    message(FATAL_ERROR "Unable to find required header files execinfo.h and/or cxxabi.h")
  endif()
endif()

# Check type support
check_type_size("int64_t" HAVE_INT64_T)
check_type_size("long double" HAVE_LONG_DOUBLE)
check_type_size("long long" HAVE_LONG_LONG)
check_cxx_source_compiles(
    "
    #include <sys/types.h>
    int main() { typedef pid_t test_t; return 0; }
    " CHECK_SYS_TYPES_H_HAS_PID_T)
if(NOT CHECK_SYS_TYPES_H_HAS_PID_T)
  set(pid_t 1)
endif()

# Check function support
check_function_exists(memset HAVE_MEMSET)
check_function_exists(posix_memalign HAVE_POSIX_MEMALIGN)
if(HAVE_POSIX_MEMALIGN)
  # look for both version of posix_memalign, with and without throw()
  check_cxx_source_compiles(
      "
      #include <stddef.h>
      #include <stdlib.h>
      extern \"C\"  int posix_memalign(void **memptr, size_t alignment, size_t size) throw();
      int main() { void *m; posix_memalign(&m, 16, 1024); }
      " CHECK_STDLIB_H_HAS_POSIX_MEMALIGN_THROW)
  if(NOT CHECK_STDLIB_H_HAS_POSIX_MEMALIGN_THROW)
    check_cxx_source_compiles(
        "
        #include <stddef.h>
        #include <stdlib.h>
        extern \"C\"  int posix_memalign(void **memptr, size_t alignment, size_t size);
        int main() { void *m; posix_memalign(&m, 16, 1024); }
        " CHECK_STDLIB_H_HAS_POSIX_MEMALIGN)
  endif()
  if(NOT CHECK_STDLIB_H_HAS_POSIX_MEMALIGN_THROW AND NOT CHECK_STDLIB_H_HAS_POSIX_MEMALIGN)
    set(MISSING_POSIX_MEMALIGN_PROTO 1)
  endif()

else()
  message(WARNING "posix_memalign NOT FOUND ... enabling override of new/delete ... THIS WILL BE SLOW")
endif()
check_function_exists(pow HAVE_POW)
check_function_exists(random HAVE_RANDOM)
check_function_exists(sleep HAVE_SLEEP)
check_function_exists(strchr HAVE_STRCHR)
# look for both version of posix_memalign, with and without throw()
check_cxx_source_compiles(
    "
    #include <cmath>
    #include <cstdlib>
    long (*absptr)(long) = &std::abs; 
    long a = -1;  
    long b = std::abs(a);
    int main() { return 0; }
    " HAVE_STD_ABS_LONG)
if(NOT HAVE_STD_ABS_LONG)
  check_cxx_source_compiles(
      "
      #include <cmath>
      #include <cstdlib>
      long (*labsptr)(long) = &std::labs; 
      long a = -1l;  
      long b = labs(a);
      int main() { return 0; }
      " HAVE_LABS)
endif()
if(NOT CHECK_STDLIB_H_HAS_POSIX_MEMALIGN_THROW AND NOT CHECK_STDLIB_H_HAS_POSIX_MEMALIGN)
  set(MISSING_POSIX_MEMALIGN_PROTO 1)
endif()

# Check for thread local storage keyword support.
# thread_local, __thread , __thread_local, or __declspec(thread)
if(NOT DEFINED THREAD_LOCAL_KEYWORD)
  foreach(_thread_local_keyword thread_local __thread __thread_local)
    check_cxx_source_compiles(
        "
        ${_thread_local_keyword} int i = 0;
        int main() { i = 1; return 0; }
        " THREAD_LOCAL_SUPPORT)
    if(THREAD_LOCAL_SUPPORT AND _thread_local_keyword STREQUAL "thread_local")
      message(STATUS "Thread local keyword: thread_local")
      unset(THREAD_LOCAL_KEYWORD CACHE)
      break()
    elseif(THREAD_LOCAL_SUPPORT)
      message(STATUS "Thread local keyword: ${_thread_local_keyword}")
      set(THREAD_LOCAL_KEYWORD "${_thread_local_keyword}"
          CACHE STRING "thread local storage keyword, 'thread_local' in C++>11")
      break()
    else()
      unset(THREAD_LOCAL_SUPPORT CACHE)
    endif()
  endforeach()
  
  if(NOT DEFINED THREAD_LOCAL_SUPPORT)
    message(FATAL_ERROR "Unable to detect mandatory support for thread-local storage")
  endif()
endif()


# Check for restrict keyword support
# restrict, __restrict, __restrict__, or _Restrict
if(NOT DEFINED RESTRICT_KEYWORD)
  foreach(_restrict_keyword restrict __restrict __restrict__ _Restrict)
    check_cxx_source_compiles(
        "
        int*  ${_restrict_keyword} i = nullptr;
        int main() { *i = 1; return 0; }
        " RESTRICT_SUPPORT)
    if(RESTRICT_SUPPORT AND _restrict_keyword STREQUAL "restrict")
      unset(RESTRICT_KEYWORD CACHE)
      break()
    elseif(RESTRICT_SUPPORT)
      set(RESTRICT_KEYWORD "${_restrict_keyword}"
          CACHE STRING "C++ equivialent of the C 'restrict' keyword")
      break()
    else()
      unset(RESTRICT_SUPPORT CACHE)
    endif()
  endforeach()

  if(NOT DEFINED RESTRICT_SUPPORT)
    # Set the restrict keyword to nothing so that it is not used
    set(RESTRICT_KEYWORD ""
        CACHE STRING "C++ equivialent of the C 'restrict' keyword")
  endif()
  
  # Print restrict keyword search results
  message(STATUS "Restrict keyword: ${RESTRICT_KEYWORD}")
endif()

check_cxx_source_compiles(
    "
    template <typename T>
    static inline void f(T* a) {};
    template <typename T> void g(T* a) { f(a); }
    template void g(int* a);
    int main() { return 0; }
    " HAVE_UNQUALIFIED_STATIC_DECL)

# Check linker feature support ===============================================

include(CheckDisablePIESupport)
check_disablepie_support(LINKER_HAS_DISABLEPIE_SUPPORT DISABLEPIE_LINKER_FLAG)

# Check for applications =======================================================

find_package(Doxygen)
find_package(LATEX)
find_program(XTERM_EXECUTABLE xterm)
if(XTERM_EXECUTABLE)
  set(HAVE_XTERM 1)
  message(STATUS "Found xterm: ${XTERM_EXECUTABLE}")
endif()
find_program(GDB_EXECUTABLE gdb)
if(GDB_EXECUTABLE)
  set(HAVE_GDB 1)
  message(STATUS "Found gdb: ${GDB_EXECUTABLE}")
endif()
find_package(PythonInterp)

# ccache for caching builds
find_program(CCACHE ccache)
if(CCACHE)
    message (STATUS "Found ccache: ${CCACHE}")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ${CCACHE})
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ${CCACHE})
endif(CCACHE)

# Check for external dependencies ==============================================
add_custom_target(External)
include(external/pthread.cmake)
include(external/mpi.cmake)
include(external/papi.cmake)
include(external/lapack.cmake)
include(external/libxc.cmake)
include(external/libunwind.cmake)
include(external/gperftools.cmake)
include(external/tbb.cmake)
include(external/parsec.cmake)

if (DEFINED ELEMENTAL_TAG)
  include(external/elemental.cmake)
endif (DEFINED ELEMENTAL_TAG)

# Add project subdirectories ===================================================

# Create build and install libraries libraries
add_custom_target(libraries)
add_custom_target(install-libraries)

include_directories(${CMAKE_SOURCE_DIR}/src ${CMAKE_BINARY_DIR}/src)
set(CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE TRUE)
set(CMAKE_INCLUDE_CURRENT_DIR TRUE)
if(ENABLE_UNITTESTS)
  include(CTest)
  enable_testing()
  set(MADNESS_HAS_GOOGLE_TEST 1)
  add_custom_target(unittests)
  add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND}) # to be GNU compatible
endif()
add_subdirectory(src)
add_subdirectory(doc)

# Generate configure files =====================================================

configure_file(
  ${PROJECT_SOURCE_DIR}/cmake/config.h.in
  ${PROJECT_BINARY_DIR}/src/madness/config.h
)

configure_file(
  ${PROJECT_SOURCE_DIR}/cmake/doxygen.cfg.in
  ${PROJECT_BINARY_DIR}/doc/doxygen.cfg
  @ONLY
)

CONFIGURE_FILE(
  ${PROJECT_SOURCE_DIR}/config/MADNESS.pc.in
  ${PROJECT_BINARY_DIR}/config/MADNESS.pc
)

# install config files
install(FILES ${PROJECT_BINARY_DIR}/config/MADNESS.pc
    DESTINATION lib/pkgconfig)

# sample CMakeLists.txt
configure_file(
  ${PROJECT_SOURCE_DIR}/doc/devsamp/CMakeLists.txt.sample.in
  ${PROJECT_BINARY_DIR}/doc/devsamp/CMakeLists.txt.sample
  @ONLY
)

# Create the version file
write_basic_package_version_file(madness-config-version.cmake
  VERSION ${MADNESS_VERSION} COMPATIBILITY AnyNewerVersion)

# Create the targets file
export(EXPORT madness
  FILE "${PROJECT_BINARY_DIR}/madness-targets.cmake")
export(EXPORT madworld
  FILE "${PROJECT_BINARY_DIR}/madworld-targets.cmake")

# Create the configure file
configure_package_config_file(cmake/madness-config.cmake.in
    "${MADNESS_BINARY_DIR}/madness-config.cmake"
  INSTALL_DESTINATION "${MADNESS_INSTALL_CMAKEDIR}"
  PATH_VARS CMAKE_INSTALL_PREFIX MADNESS_INSTALL_BINDIR 
            MADNESS_INSTALL_INCLUDEDIR MADNESS_INSTALL_LIBDIR
            MADNESS_INSTALL_DATADIR MADNESS_INSTALL_DOCDIR 
            MADNESS_INSTALL_CMAKEDIR)

# Install config, version, and target files
install(EXPORT madness
    FILE "madness-targets.cmake"
    DESTINATION "${MADNESS_INSTALL_CMAKEDIR}" 
    COMPONENT madness-config)
install(EXPORT madworld
    FILE "madworld-targets.cmake"
    DESTINATION "${MADNESS_INSTALL_CMAKEDIR}" 
    COMPONENT madness-config)
install(FILES
    "${MADNESS_BINARY_DIR}/madness-config.cmake"
    "${MADNESS_BINARY_DIR}/madness-config-version.cmake"
    DESTINATION "${MADNESS_INSTALL_CMAKEDIR}" 
    COMPONENT madness-config)
add_custom_target(install-config
     COMMAND ${CMAKE_COMMAND} -DCOMPONENT=madness-config -P ${CMAKE_BINARY_DIR}/cmake_install.cmake
     COMMENT "Installing MADNESS config components")

feature_summary(WHAT ALL
                DESCRIPTION "=== MADNESS Package/Feature Info ===")
    
