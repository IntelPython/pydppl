find_package(GTest REQUIRED)
# We need thread support for gtest
find_package(Threads REQUIRED)

# Emulate autotools like make check target to build tests
set(CMAKE_CTEST_COMMAND ctest --progress --output-on-failure -j 4)
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})
enable_testing()

include_directories(${CMAKE_SOURCE_DIR}/include)
link_directories(${GTEST_LIB_DIR})

# Copy the spir-v input files to test build directory
set(spirv-test-files
    multi_kernel.spv
)
foreach(tf ${spirv-test-files})
    file(COPY ${tf} DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
endforeach()

if(DPCTL_GENERATE_COVERAGE)
    file(GLOB_RECURSE sources ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)
    file(GLOB_RECURSE dpctl_sources ${CMAKE_CURRENT_SOURCE_DIR}/../source/*.cpp)

    # Add profiling flags
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-instr-generate -fcoverage-mapping")

    # Add all dpctl sources into a single executable so that we can run coverage
    # analysis and generate a report.
    add_executable(dpctl_c_api_tests
        EXCLUDE_FROM_ALL
        ${sources}
        ${dpctl_sources}
    )
    target_link_libraries(dpctl_c_api_tests
        ${CMAKE_THREAD_LIBS_INIT}
        GTest::GTest
        DPCTLSyclInterface
        ${LEVEL_ZERO_LIBRARY}
        ${DPCPP_OPENCL_LIBRARY}
    )
    add_custom_target(llvm-cov
        COMMAND ${CMAKE_MAKE_PROGRAM} dpctl_c_api_tests
        COMMAND ${CMAKE_CURRENT_BINARY_DIR}/dpctl_c_api_tests
        COMMAND ${LLVM_PROFDATA_EXE} merge -sparse default.profraw -o dpctl.profdata
        COMMAND ${LLVM_COV_EXE} report ${CMAKE_CURRENT_BINARY_DIR}/dpctl_c_api_tests -instr-profile=dpctl.profdata ${dpctl_sources}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
    add_custom_target(lcov-genhtml
        COMMAND ${CMAKE_MAKE_PROGRAM} llvm-cov
        COMMAND ${LLVM_COV_EXE} export -format=lcov ${CMAKE_CURRENT_BINARY_DIR}/dpctl_c_api_tests -instr-profile=dpctl.profdata ${dpctl_sources} > dpctl.lcov
        COMMAND ${GENHTML_EXE} ${CMAKE_CURRENT_BINARY_DIR}/dpctl.lcov --output-directory ${COVERAGE_OUTPUT_DIR}/dpctl-c-api-coverage
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
else()
    file(GLOB_RECURSE sources ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)
    add_executable(dpctl_c_api_tests EXCLUDE_FROM_ALL ${sources})
    target_link_libraries(dpctl_c_api_tests
        ${CMAKE_THREAD_LIBS_INIT}
        GTest::GTest
        DPCTLSyclInterface
        ${LEVEL_ZERO_LIBRARY}
    )
endif()

gtest_discover_tests(dpctl_c_api_tests)
add_dependencies(check dpctl_c_api_tests)
