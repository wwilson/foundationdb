set(FDB_C_SRCS
  fdb_c.cpp
  foundationdb/fdb_c.h
  foundationdb/fdb_c_internal.h
  foundationdb/fdb_c_types.h)

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/foundationdb)

set(asm_file ${CMAKE_CURRENT_BINARY_DIR}/fdb_c.g.S)

set(os "linux")
set(cpu "intel")
if(APPLE)
  set(os "osx")
elseif(WIN32)
  set(os "windows")
  set(asm_file ${CMAKE_CURRENT_BINARY_DIR}/fdb_c.g.asm)
endif()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
  set(cpu "aarch64")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(ppc64le|powerpc64le)")
  set(cpu "ppc64le")
endif()

set(IS_ARM_MAC NO)
if(APPLE AND CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64")
  set(IS_ARM_MAC YES)
endif()

add_custom_command(OUTPUT ${asm_file} ${CMAKE_CURRENT_BINARY_DIR}/fdb_c_function_pointers.g.h
  COMMAND $<TARGET_FILE:Python::Interpreter> ${CMAKE_CURRENT_SOURCE_DIR}/generate_asm.py ${os} ${cpu}
      ${CMAKE_CURRENT_SOURCE_DIR}/fdb_c.cpp
      ${asm_file}
      ${CMAKE_CURRENT_BINARY_DIR}/fdb_c_function_pointers.g.h
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/generate_asm.py ${CMAKE_CURRENT_SOURCE_DIR}/fdb_c.cpp
  COMMENT "Generate C bindings")
add_custom_target(fdb_c_generated DEPENDS ${asm_file}
      ${CMAKE_CURRENT_BINARY_DIR}/fdb_c_function_pointers.g.h)

vexillographer_compile(TARGET fdb_c_options LANG c OUT ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_options.g.h
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_options.g.h)

include(GenerateExportHeader)

if(OPEN_FOR_IDE)
  add_library(fdb_c OBJECT ${FDB_C_SRCS} ${asm_file})
else()
  add_library(fdb_c SHARED ${FDB_C_SRCS} ${asm_file})
  strip_debug_symbols(fdb_c)
endif()
add_dependencies(fdb_c fdb_c_generated fdb_c_options)
add_dependencies(fdbclient fdb_c_options)
add_dependencies(fdbclient_sampling fdb_c_options)
target_link_libraries(fdb_c PRIVATE $<BUILD_INTERFACE:fdbclient>)
if(APPLE)
  set(symbols ${CMAKE_CURRENT_BINARY_DIR}/fdb_c.symbols)
  add_custom_command(OUTPUT ${symbols}
    COMMAND $<TARGET_FILE:Python::Interpreter> ${CMAKE_CURRENT_SOURCE_DIR}/symbolify.py
        ${CMAKE_CURRENT_SOURCE_DIR}/foundationdb/fdb_c.h
        ${CMAKE_CURRENT_SOURCE_DIR}/foundationdb/fdb_c_internal.h
        ${symbols}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/symbolify.py ${CMAKE_CURRENT_SOURCE_DIR}/foundationdb/fdb_c.h ${CMAKE_CURRENT_SOURCE_DIR}/foundationdb/fdb_c_internal.h
    COMMENT "Generate exported_symbols_list")
  add_custom_target(exported_symbols_list DEPENDS ${symbols})
  add_dependencies(fdb_c exported_symbols_list)
  target_link_options(fdb_c PRIVATE "LINKER:-no_weak_exports,-exported_symbols_list,${symbols}")
elseif(WIN32)
else()
  target_link_options(fdb_c PRIVATE "LINKER:--version-script=${CMAKE_CURRENT_SOURCE_DIR}/fdb_c.map,-z,nodelete,-z,noexecstack")
endif()
target_include_directories(fdb_c PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/foundationdb>)
if(WIN32)
  enable_language(ASM_MASM)
  set_property(SOURCE ${asm_file} PROPERTY LANGUAGE ASM_MASM)
endif()

# The tests don't build on windows
if(NOT WIN32)
  set(MAKO_SRCS
    test/mako/async.hpp
    test/mako/async.cpp
    test/mako/blob_granules.hpp
    test/mako/blob_granules.cpp
    test/mako/future.hpp
    test/mako/limit.hpp
    test/mako/logger.hpp
    test/mako/mako.cpp
    test/mako/mako.hpp
    test/mako/operations.hpp
    test/mako/operations.cpp
    test/mako/process.hpp
    test/mako/shm.hpp
    test/mako/stats.hpp
    test/mako/time.hpp
    test/mako/utils.cpp
    test/mako/utils.hpp)
  add_subdirectory(test/unit/third_party)
  find_package(Threads REQUIRED)
  set(UNIT_TEST_SRCS
    test/unit/unit_tests.cpp
    test/unit/fdb_api.cpp
    test/unit/fdb_api.hpp)

  set(UNIT_TEST_VERSION_510_SRCS test/unit/unit_tests_version_510.cpp)
  set(TRACE_PARTIAL_FILE_SUFFIX_TEST_SRCS test/unit/trace_partial_file_suffix_test.cpp)
  set(DISCONNECTED_TIMEOUT_UNIT_TEST_SRCS
    test/unit/disconnected_timeout_tests.cpp
    test/unit/fdb_api.cpp
    test/unit/fdb_api.hpp)

  add_library(fdb_cpp INTERFACE)
  target_sources(fdb_cpp INTERFACE test/fdb_api.hpp)
  target_include_directories(fdb_cpp INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/test)
  target_link_libraries(fdb_cpp INTERFACE fmt::fmt)

  set(API_TESTER_SRCS
    test/apitester/fdb_c_api_tester.cpp
    test/apitester/TesterApiWorkload.cpp
    test/apitester/TesterApiWorkload.h
    test/apitester/TesterTestSpec.cpp
    test/apitester/TesterTestSpec.h
    test/apitester/TesterBlobGranuleCorrectnessWorkload.cpp
    test/apitester/TesterCancelTransactionWorkload.cpp
    test/apitester/TesterCorrectnessWorkload.cpp
    test/apitester/TesterKeyValueStore.cpp
    test/apitester/TesterKeyValueStore.h
    test/apitester/TesterOptions.h
    test/apitester/TesterScheduler.cpp
    test/apitester/TesterScheduler.h
    test/apitester/TesterTransactionExecutor.cpp
    test/apitester/TesterTransactionExecutor.h
    test/apitester/TesterUtil.cpp
    test/apitester/TesterUtil.h
    test/apitester/TesterWorkload.cpp
    test/apitester/TesterWorkload.h
    ../../flow/SimpleOpt.h
  )

  if(OPEN_FOR_IDE)
    add_library(fdb_c_performance_test OBJECT test/performance_test.c test/test.h)
    add_library(fdb_c_ryw_benchmark OBJECT test/ryw_benchmark.c test/test.h)
    add_library(fdb_c_txn_size_test OBJECT test/txn_size_test.c test/test.h)
    add_library(fdb_c_client_memory_test OBJECT test/client_memory_test.cpp test/unit/fdb_api.cpp test/unit/fdb_api.hpp)
    add_library(mako OBJECT ${MAKO_SRCS})
    add_library(fdb_c_setup_tests OBJECT test/unit/setup_tests.cpp)
    add_library(fdb_c_unit_tests OBJECT ${UNIT_TEST_SRCS})
    add_library(fdb_c_unit_tests_version_510 OBJECT ${UNIT_TEST_VERSION_510_SRCS})
    add_library(trace_partial_file_suffix_test OBJECT ${TRACE_PARTIAL_FILE_SUFFIX_TEST_SRCS})
    add_library(disconnected_timeout_unit_tests OBJECT ${DISCONNECTED_TIMEOUT_UNIT_TEST_SRCS})
    add_library(fdb_c_api_tester OBJECT ${API_TESTER_SRCS})
  else()
    add_executable(fdb_c_performance_test test/performance_test.c test/test.h)
    add_executable(fdb_c_ryw_benchmark test/ryw_benchmark.c test/test.h)
    add_executable(fdb_c_txn_size_test test/txn_size_test.c test/test.h)
    add_executable(fdb_c_client_memory_test test/client_memory_test.cpp test/unit/fdb_api.cpp test/unit/fdb_api.hpp)
    add_executable(mako ${MAKO_SRCS})
    add_executable(fdb_c_setup_tests test/unit/setup_tests.cpp)
    add_executable(fdb_c_unit_tests ${UNIT_TEST_SRCS})
    add_executable(fdb_c_unit_tests_version_510 ${UNIT_TEST_VERSION_510_SRCS})
    add_executable(trace_partial_file_suffix_test ${TRACE_PARTIAL_FILE_SUFFIX_TEST_SRCS})
    add_executable(disconnected_timeout_unit_tests ${DISCONNECTED_TIMEOUT_UNIT_TEST_SRCS})
    add_executable(fdb_c_api_tester ${API_TESTER_SRCS})
    strip_debug_symbols(fdb_c_performance_test)
    strip_debug_symbols(fdb_c_ryw_benchmark)
    strip_debug_symbols(fdb_c_txn_size_test)
    strip_debug_symbols(fdb_c_client_memory_test)
  endif()
  target_link_libraries(fdb_c_performance_test PRIVATE fdb_c Threads::Threads)
  target_link_libraries(fdb_c_ryw_benchmark PRIVATE fdb_c Threads::Threads)
  target_link_libraries(fdb_c_txn_size_test PRIVATE fdb_c Threads::Threads)
  target_link_libraries(fdb_c_client_memory_test PRIVATE fdb_c Threads::Threads)

  add_dependencies(fdb_c_setup_tests doctest)
  add_dependencies(fdb_c_unit_tests doctest)
  add_dependencies(fdb_c_unit_tests_version_510 doctest)
  add_dependencies(disconnected_timeout_unit_tests doctest)
  target_include_directories(fdb_c_setup_tests PUBLIC ${DOCTEST_INCLUDE_DIR})
  target_include_directories(fdb_c_unit_tests PUBLIC ${DOCTEST_INCLUDE_DIR})
  target_include_directories(fdb_c_unit_tests_version_510 PUBLIC ${DOCTEST_INCLUDE_DIR})
  target_include_directories(disconnected_timeout_unit_tests PUBLIC ${DOCTEST_INCLUDE_DIR})
  target_link_libraries(fdb_c_setup_tests PRIVATE fdb_c Threads::Threads)
  target_link_libraries(fdb_c_unit_tests PRIVATE fdb_c Threads::Threads fdbclient rapidjson)
  target_link_libraries(fdb_c_unit_tests_version_510 PRIVATE fdb_c Threads::Threads)
  target_link_libraries(trace_partial_file_suffix_test PRIVATE fdb_c Threads::Threads flow)
  target_link_libraries(disconnected_timeout_unit_tests PRIVATE fdb_c Threads::Threads)

if(USE_SANITIZER)
  target_link_libraries(fdb_c_api_tester PRIVATE fdb_c fdb_cpp toml11_target Threads::Threads fmt::fmt boost_asan)
else()
  target_link_libraries(fdb_c_api_tester PRIVATE fdb_c fdb_cpp toml11_target Threads::Threads fmt::fmt boost_target)
endif()

  # do not set RPATH for mako
  set_property(TARGET mako PROPERTY SKIP_BUILD_RPATH TRUE)
  if (USE_SANITIZER)
	target_link_libraries(mako PRIVATE fdb_c fdbclient fmt::fmt Threads::Threads fdb_cpp boost_asan)
  else()
	target_link_libraries(mako PRIVATE fdb_c fdbclient fmt::fmt Threads::Threads fdb_cpp boost_target)
  endif()

  if(NOT OPEN_FOR_IDE)
    # Make sure that fdb_c.h is compatible with c90
    add_executable(fdb_c90_test test/fdb_c90_test.c)
    set_property(TARGET fdb_c90_test PROPERTY C_STANDARD 90)
    target_compile_options(fdb_c90_test PRIVATE -Wall -Wextra -Wpedantic -Werror)
    target_link_libraries(fdb_c90_test PRIVATE fdb_c)
  endif()

  if(OPEN_FOR_IDE)
    set(FDB_C_TARGET $<TARGET_OBJECTS:fdb_c>)
  else()
    set(FDB_C_TARGET $<TARGET_FILE:fdb_c>)
  endif()
  add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so
    COMMAND ${CMAKE_COMMAND} -E copy ${FDB_C_TARGET} ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so
    DEPENDS fdb_c
    COMMENT "Copy libfdb_c to use as external client for test")
  add_custom_target(external_client DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so)
  add_dependencies(fdb_c_unit_tests external_client)
  add_dependencies(disconnected_timeout_unit_tests external_client)
  add_dependencies(fdb_c_api_tester external_client)

  add_fdbclient_test(
    NAME fdb_c_setup_tests
    COMMAND $<TARGET_FILE:fdb_c_setup_tests>)
  add_fdbclient_test(
    NAME fdb_c_unit_tests
    COMMAND $<TARGET_FILE:fdb_c_unit_tests>
            @CLUSTER_FILE@
            fdb)
  add_fdbclient_test(
    NAME fdb_c_unit_tests_version_510
    COMMAND $<TARGET_FILE:fdb_c_unit_tests_version_510>
            @CLUSTER_FILE@
            fdb)
  add_fdbclient_test(
    NAME trace_partial_file_suffix_test
    COMMAND $<TARGET_FILE:trace_partial_file_suffix_test>
            @CLUSTER_FILE@
            fdb)
  add_fdbclient_test(
    NAME fdb_c_external_client_unit_tests
    COMMAND $<TARGET_FILE:fdb_c_unit_tests>
            @CLUSTER_FILE@
            fdb
            ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so
            )
  add_unavailable_fdbclient_test(
    NAME disconnected_timeout_unit_tests
    COMMAND $<TARGET_FILE:disconnected_timeout_unit_tests>
            @CLUSTER_FILE@
            )
  add_unavailable_fdbclient_test(
    NAME disconnected_timeout_external_client_unit_tests
    COMMAND $<TARGET_FILE:disconnected_timeout_unit_tests>
            @CLUSTER_FILE@
            ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so
            )
  add_fdbclient_test(
    NAME fdb_c_api_tests
    DISABLE_LOG_DUMP
    COMMAND ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/run_c_api_tests.py
            --cluster-file
            @CLUSTER_FILE@
            --tester-binary
            $<TARGET_FILE:fdb_c_api_tester>
            --external-client-library
            ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so
            --test-dir
            ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests
            --tmp-dir
            @TMP_DIR@
            --log-dir
            @LOG_DIR@
            )

  add_fdbclient_test(
    NAME fdb_c_api_tests_blob_granule
    DISABLE_LOG_DUMP
    API_TEST_BLOB_GRANULES_ENABLED
    COMMAND ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/run_c_api_tests.py
            --cluster-file
            @CLUSTER_FILE@
            --tester-binary
            $<TARGET_FILE:fdb_c_api_tester>
            --external-client-library
            ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so
            --test-dir
            ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/blobgranuletests
            --blob-granule-local-file-path
            @DATA_DIR@/fdbblob/
            --tmp-dir
            @TMP_DIR@
            --log-dir
            @LOG_DIR@
            )

  add_fdbclient_test(
    NAME fdb_c_api_tests_with_tls
    DISABLE_LOG_DUMP
    TLS_ENABLED
    COMMAND ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/run_c_api_tests.py
            --cluster-file
            @CLUSTER_FILE@
            --tester-binary
            $<TARGET_FILE:fdb_c_api_tester>
            --external-client-library
            ${CMAKE_CURRENT_BINARY_DIR}/libfdb_c_external.so
            --test-dir
            ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests
            --tmp-dir
            @TMP_DIR@
            --log-dir
            @LOG_DIR@
            --tls-cert-file
            @CLIENT_CERT_FILE@
            --tls-key-file
            @CLIENT_KEY_FILE@
            --tls-ca-file
            @SERVER_CA_FILE@
            )

  if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64" AND NOT USE_SANITIZER)
    add_test(NAME fdb_c_upgrade_single_threaded_630api
      COMMAND ${CMAKE_SOURCE_DIR}/tests/TestRunner/upgrade_test.py
            --build-dir ${CMAKE_BINARY_DIR}
            --test-file ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests/upgrade/MixedApiWorkloadSingleThr.toml
            --upgrade-path "6.3.23" "7.0.0" "7.1.5" "7.2.0"
            --process-number 1
            )

    add_test(NAME fdb_c_upgrade_single_threaded_700api
      COMMAND ${CMAKE_SOURCE_DIR}/tests/TestRunner/upgrade_test.py
            --build-dir ${CMAKE_BINARY_DIR}
            --test-file ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests/upgrade/MixedApiWorkloadSingleThr.toml
            --upgrade-path "7.0.0" "7.1.5" "7.2.0"
            --process-number 1
            )

    add_test(NAME fdb_c_upgrade_multi_threaded_630api
        COMMAND ${CMAKE_SOURCE_DIR}/tests/TestRunner/upgrade_test.py
            --build-dir ${CMAKE_BINARY_DIR}
            --test-file ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests/upgrade/MixedApiWorkloadMultiThr.toml
            --upgrade-path "6.3.23" "7.0.0" "7.1.5" "7.2.0" "7.1.5"
            --process-number 3
          )

    add_test(NAME fdb_c_upgrade_multi_threaded_700api
        COMMAND ${CMAKE_SOURCE_DIR}/tests/TestRunner/upgrade_test.py
            --build-dir ${CMAKE_BINARY_DIR}
            --test-file ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests/upgrade/MixedApiWorkloadMultiThr.toml
            --upgrade-path "7.0.0" "7.1.5" "7.2.0" "7.1.5"
            --process-number 3
          )

    add_test(NAME fdb_c_upgrade_multi_threaded_710api
        COMMAND ${CMAKE_SOURCE_DIR}/tests/TestRunner/upgrade_test.py
            --build-dir ${CMAKE_BINARY_DIR}
            --test-file ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests/upgrade/MixedApiWorkloadMultiThr.toml
            --upgrade-path "7.1.5" "7.2.0" "7.1.5"
            --process-number 3
          )

    add_test(NAME fdb_c_cluster_wiggle
          COMMAND ${CMAKE_SOURCE_DIR}/tests/TestRunner/upgrade_test.py
              --build-dir ${CMAKE_BINARY_DIR}
              --test-file ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests/upgrade/MixedApiWorkloadMultiThr.toml
              --upgrade-path "7.2.0" "wiggle"
              --disable-log-dump
              --process-number 3
              --redundancy double
            )

    add_test(NAME fdb_c_wiggle_and_upgrade
          COMMAND ${CMAKE_SOURCE_DIR}/tests/TestRunner/upgrade_test.py
              --build-dir ${CMAKE_BINARY_DIR}
              --test-file ${CMAKE_SOURCE_DIR}/bindings/c/test/apitester/tests/upgrade/MixedApiWorkloadMultiThr.toml
              --upgrade-path "7.0.0" "wiggle" "7.2.0"
              --disable-log-dump
              --process-number 3
              --redundancy double
            )

  endif()

endif()

set(c_workloads_srcs
  test/workloads/workloads.cpp
  test/workloads/workloads.h
  test/workloads/SimpleWorkload.cpp)

if(OPEN_FOR_IDE)
  add_library(c_workloads OBJECT ${c_workloads_srcs})
else()
  add_library(c_workloads SHARED ${c_workloads_srcs})
endif()
set_target_properties(c_workloads PROPERTIES
  LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/share/foundationdb")
target_link_libraries(c_workloads PUBLIC fdb_c)

if (NOT WIN32 AND NOT APPLE AND NOT OPEN_FOR_IDE)
  target_link_options(c_workloads PRIVATE "LINKER:--version-script=${CMAKE_CURRENT_SOURCE_DIR}/external_workload.map,-z,nodelete")
endif()

# TODO: re-enable once the old vcxproj-based build system is removed.
#generate_export_header(fdb_c EXPORT_MACRO_NAME "DLLEXPORT"
#  EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_export.h)

set(targets_export_name "FoundationDB-Client")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(version_config "${generated_dir}/${targets_export_name}ConfigVersion.cmake")
set(project_config "${generated_dir}/${targets_export_name}Config.cmake")
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${version_config}" VERSION ${GENERIC_LIB_VERSION} COMPATIBILITY AnyNewerVersion
)
configure_file("${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in" "${project_config}" @ONLY)

fdb_install(
  TARGETS fdb_c
  EXPORT ${targets_export_name}
  DESTINATION lib
  COMPONENT clients)
fdb_install(
  FILES foundationdb/fdb_c.h
  ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_options.g.h
  ${CMAKE_SOURCE_DIR}/fdbclient/vexillographer/fdb.options
  ${CMAKE_SOURCE_DIR}/bindings/c/foundationdb/fdb_c_types.h
  DESTINATION include
  DESTINATION_SUFFIX /foundationdb
  COMPONENT clients)
fdb_install(
  FILES "${project_config}" "${version_config}"
  DESTINATION lib
  DESTINATION_SUFFIX "/cmake/${targets_export_name}"
  COMPONENT clients)
fdb_configure_and_install(
  FILE "${PROJECT_SOURCE_DIR}/cmake/foundationdb-client.pc.in"
  DESTINATION lib
  DESTINATION_SUFFIX "/pkgconfig"
  COMPONENT clients)
fdb_install(
  EXPORT ${targets_export_name}
  DESTINATION lib
  DESTINATION_SUFFIX "/cmake/${targets_export_name}"
  COMPONENT clients)
