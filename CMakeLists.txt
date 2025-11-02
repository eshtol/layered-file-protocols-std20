cmake_minimum_required(VERSION 3.5)
project(layered-file-protocols LANGUAGES C CXX)

include(CheckFunctionExists)
include(CTest)
include(GNUInstallDirs)
include(TestBigEndian)


option(
    BUILD_DOC
    "Build documentation"
    FALSE
)
option(
    BUILD_EXAMPLES
    "Build examples"
    FALSE
)


set(CMAKE_CXX_STANDARD 20)
set(CMAKE_C_STANDARD 99)
test_big_endian(LFP_BIG_ENDIAN)
if (LFP_BIG_ENDIAN)
    message(STATUS "System is big endian")
else ()
    message(STATUS "System is little endian")
endif ()

if (NOT MSVC)
    # assuming gcc-style options
    # add warnings in debug mode
    list(APPEND lfp-warnings-c++
        -Wall
        -Wextra
        -pedantic
        -Wformat-nonliteral
        -Wcast-align
        -Wpointer-arith
        -Wmissing-declarations
        -Wcast-qual
        -Wwrite-strings
        -Wchar-subscripts
        -Wredundant-decls
    )
endif()

check_function_exists(_ftelli64 HAVE_FTELLI64)
check_function_exists(_fseeki64 HAVE_FSEEKI64)
check_function_exists(ftello HAVE_FTELLO)
check_function_exists(fseeko HAVE_FSEEKO)

add_library(lfp
    src/lfp.cpp
    src/cfile.cpp
    src/memfile.cpp
    src/tapeimage.cpp
    src/rp66.cpp
)
add_library(lfp::lfp ALIAS lfp)

target_include_directories(lfp
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

target_compile_options(lfp
    BEFORE
    PRIVATE
        $<$<CONFIG:Debug>:${lfp-warnings-c++}>
        $<$<CONFIG:RelWithDebInfo>:${lfp-warnings-c++}>
        # Assures recommended stack unwinding on MVSC.
        # Beyond users control as without it lfp won't work correctly
        $<$<CXX_COMPILER_ID:MSVC>:/EHsc>
)

target_compile_definitions(lfp
    PRIVATE
        $<$<BOOL:${LFP_BIG_ENDIAN}>:IS_BIG_ENDIAN>
        $<$<NOT:$<BOOL:${LFP_BIG_ENDIAN}>>:IS_LITTLE_ENDIAN>
        $<$<BOOL:${HAVE_FTELLI64}>:HAVE_FTELLI64>
        $<$<BOOL:${HAVE_FSEEKI64}>:HAVE_FSEEKI64>
        $<$<BOOL:${HAVE_FTELLO}>:HAVE_FTELLO>
        $<$<BOOL:${HAVE_FSEEKO}>:HAVE_FSEEKO>
)

install(
    TARGETS lfp
    EXPORT lfp-export
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(DIRECTORY include/ DESTINATION include)
install(
    EXPORT
        lfp-export
    NAMESPACE
        lfp::
    DESTINATION
        ${CMAKE_INSTALL_DATADIR}/lfp/cmake
    FILE
        lfp-config.cmake
)
export(
    TARGETS
        lfp
    NAMESPACE
        lfp::
    FILE
        lfp-config.cmake
)

if (BUILD_DOC)
    add_subdirectory(docs)
endif ()

if (BUILD_EXAMPLES)
    add_subdirectory(examples)
endif ()

if (NOT BUILD_TESTING)
    return ()
endif ()

if (NOT TARGET Catch2::Catch2)
    add_subdirectory(external/catch2)
endif ()

add_executable(unit-tests
    test/cfile.cpp
    test/main.cpp
    test/memfile.cpp
    test/tapeimage.cpp
    test/rp66.cpp
)

target_compile_options(unit-tests
    BEFORE
    PRIVATE
        $<$<CXX_COMPILER_ID:MSVC>:/EHsc>
)

target_compile_definitions(unit-tests
    PRIVATE
        $<$<BOOL:${LFP_BIG_ENDIAN}>:IS_BIG_ENDIAN>
        $<$<NOT:$<BOOL:${LFP_BIG_ENDIAN}>>:IS_LITTLE_ENDIAN>
)
target_link_libraries(unit-tests
    lfp::lfp
    Catch2::Catch2
)
add_test(NAME unit-tests COMMAND unit-tests)
