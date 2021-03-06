# CMakeLists.txt file for unit testing OpenMP Library
include(FindPythonInterp)
include(CheckTypeSize)
include(CheckLibraryExists)

if(NOT PYTHONINTERP_FOUND)
  libomp_warning_say("Could not find Python.")
  libomp_warning_say("The check-libomp target will not be available!")
  return()
endif()

# Some tests use math functions
check_library_exists(m sqrt "" LIBOMP_HAVE_LIBM)

macro(pythonize_bool var)
  if (${var})
    set(${var} True)
  else()
    set(${var} False)
  endif()
endmacro()

pythonize_bool(LIBOMP_USE_HWLOC)
pythonize_bool(LIBOMP_OMPT_SUPPORT)
pythonize_bool(LIBOMP_OMPT_BLAME)
pythonize_bool(LIBOMP_OMPT_TRACE)
pythonize_bool(LIBOMP_HAVE_LIBM)

set(LIBOMP_TEST_CFLAGS "" CACHE STRING
  "Extra compiler flags to send to the test compiler")

if(${LIBOMP_STANDALONE_BUILD})
  # Make sure we can use the console pool for recent cmake and ninja > 1.5
  if(CMAKE_VERSION VERSION_LESS 3.1.20141117)
    set(cmake_3_2_USES_TERMINAL)
  else()
    set(cmake_3_2_USES_TERMINAL USES_TERMINAL)
  endif()
  set(LIBOMP_TEST_COMPILER ${CMAKE_C_COMPILER} CACHE STRING
    "Compiler to use for testing OpenMP library")
  set(LIBOMP_TEST_OPENMP_FLAG -fopenmp CACHE STRING
    "OpenMP compiler flag to use for testing OpenMP library")
  find_program(LIBOMP_LLVM_LIT_EXECUTABLE
    NAMES llvm-lit lit.py lit
    PATHS ${OPENMP_LLVM_TOOLS_DIR})
  if(NOT LIBOMP_LLVM_LIT_EXECUTABLE)
    libomp_say("Cannot find llvm-lit.")
    libomp_say("Please put llvm-lit in your PATH, set LIBOMP_LLVM_LIT_EXECUTABLE to its full path or point OPENMP_LLVM_TOOLS_DIR to its directory")
    libomp_warning_say("The check-libomp target will not be available!")
    return()
  endif()
  find_program(FILECHECK_EXECUTABLE
    NAMES FileCheck
    PATHS ${OPENMP_LLVM_TOOLS_DIR})
  if (NOT FILECHECK_EXECUTABLE)
    # set to empty string so that lit can disable dependent tests
    set(FILECHECK_EXECUTABLE "")
  endif()
  # Set lit arguments
  # The -j 1 lets the actual tests run with the entire machine.
  # We have one test thread that spawns the tests serially.  This allows
  # Each test to use the entire machine.
  set(LIBOMP_LIT_ARGS_DEFAULT "-sv --show-unsupported --show-xfail")
  if(MSVC OR XCODE)
    set(LIBOMP_LIT_ARGS_DEFAULT "${LIBOMP_LIT_ARGS_DEFAULT} --no-progress-bar")
  endif()
  set(LIBOMP_LIT_ARGS "${LIBOMP_LIT_ARGS_DEFAULT}" CACHE STRING
    "Default options for lit")
  separate_arguments(LIBOMP_LIT_ARGS)
  add_custom_target(check-libomp
    COMMAND ${PYTHON_EXECUTABLE} ${LIBOMP_LLVM_LIT_EXECUTABLE} ${LIBOMP_LIT_ARGS} ${CMAKE_CURRENT_BINARY_DIR}
    DEPENDS omp
    COMMENT "Running libomp tests"
    ${cmake_3_2_USES_TERMINAL}
  )
else()
  # LLVM source tree build, test just-built clang
  if(NOT MSVC)
    set(LIBOMP_TEST_COMPILER ${LLVM_RUNTIME_OUTPUT_INTDIR}/clang)
  else()
    set(LIBOMP_TEST_COMPILER ${LLVM_RUNTIME_OUTPUT_INTDIR}/clang.exe)
  endif()
  set(LIBOMP_TEST_OPENMP_FLAG -fopenmp=libomp)
  set(FILECHECK_EXECUTABLE ${LLVM_RUNTIME_OUTPUT_INTDIR}/FileCheck)
  # Use add_lit_testsuite() from LLVM CMake.
  add_lit_testsuite(check-libomp
    "Running libomp tests"
    ${CMAKE_CURRENT_BINARY_DIR}
    DEPENDS clang clang-headers FileCheck omp
  )
endif()

# Configure the lit.site.cfg.in file
set(AUTO_GEN_COMMENT "## Autogenerated by libomp configuration.\n# Do not edit!")
configure_file(lit.site.cfg.in lit.site.cfg @ONLY)

