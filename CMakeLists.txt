cmake_minimum_required(VERSION 3.15)

if (NOT VCPKG_LIBRARY_LINKAGE)
    set(VCPKG_LIBRARY_LINKAGE static)
endif()

# By default, Use ezvcpkg to install dependencies. But don't use
# ezvcpkg if it appears that this configuration is using vcpkg
# manifest mode already, either by building cesium-native directly,
# or via a higher level project that has added cesium-native as a
# subdirectory and is using vcpkg to statisfy its dependencies.

#In a fresh build, before CMakeCache.txt exists, determine whether to
#use ezvcpkg. After this, CESIUM_USE_EZVCPG will be in the cache.
set(CESIUM_USE_EZVCPKG_DEFAULT ON)
if (VCPKG_MANIFEST_MODE)
    set(CESIUM_USE_EZVCPKG_DEFAULT OFF)
elseif (CMAKE_TOOLCHAIN_FILE)
    get_filename_component(toolchainFile "${CMAKE_TOOLCHAIN_FILE}" NAME)
    if(toolchainFile STREQUAL "vcpkg.cmake")
        set(CESIUM_USE_EZVCPKG_DEFAULT OFF)
    endif()
endif()

option(CESIUM_USE_EZVCPKG "use ezvcpkg helper" ${CESIUM_USE_EZVCPKG_DEFAULT})

if(CESIUM_USE_EZVCPKG)
  # Keep vcpkg from running in manifset mode. It will try to because
  # this directory contains a vcpkg.json manifest.
  set(VCPKG_MANIFEST_MODE OFF CACHE BOOL "vcpkg manifest mode should not be enabled with ezvcpg!")
  include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/ezvcpkg/ezvcpkg.cmake)
endif()

include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/detect-vcpkg-triplet.cmake)

# Note: VCPKG_TRIPLET is not used by vcpkg! The relevant variables are
# VCPKG_TARGET_TRIPLET and VCPKG_HOST_TRIPLET. ezvcpkg and our custom
# installation code use VCPKG_TRIPLET, so initialize it here.

if (VCPKG_TARGET_TRIPLET)
    set(VCPKG_TRIPLET "${VCPKG_TARGET_TRIPLET}")
endif()

if (NOT VCPKG_TRIPLET)
    if (DEFINED ENV{VCPKG_TRIPLET})
        set(VCPKG_TRIPLET "$ENV{VCPKG_TRIPLET}")
    elseif(DETECTED_VCPKG_TRIPLET_ERROR)
        message(FATAL_ERROR "${DETECTED_VCPKG_TRIPLET_ERROR}")
    elseif(DETECTED_VCPKG_TRIPLET STREQUAL "x64-windows")
        # cesium-native requires static linking on Windows
        set(VCPKG_TRIPLET "x64-windows-static-md")
    else()
        set(VCPKG_TRIPLET "${DETECTED_VCPKG_TRIPLET}")
    endif()
    if (NOT CESIUM_USE_EZVCPKG)
        set(VCPKG_TARGET_TRIPLET "${VCPKG_TRIPLET}")
    endif()
endif()

message(STATUS "VCPKG_TRIPLET ${VCPKG_TRIPLET}")
message(STATUS "VCPKG_TARGET_TRIPLET ${VCPKG_TARGET_TRIPLET}")

if (NOT VCPKG_OVERLAY_PORTS)
    if (DEFINED ENV{VCPKG_OVERLAY_PORTS})
        set(VCPKG_OVERLAY_PORTS "$ENV{VCPKG_OVERLAY_PORTS}")
    endif()
endif()

if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/extern/vcpkg/ports")
    list(APPEND VCPKG_OVERLAY_PORTS "${CMAKE_CURRENT_SOURCE_DIR}/extern/vcpkg/ports")
endif()

message(STATUS "VCPKG_OVERLAY_PORTS ${VCPKG_OVERLAY_PORTS}")

if (NOT VCPKG_OVERLAY_TRIPLETS)
    if (DEFINED ENV{VCPKG_OVERLAY_TRIPLETS})
        set(VCPKG_OVERLAY_TRIPLETS "$ENV{VCPKG_OVERLAY_TRIPLETS}")
    endif()
endif()

message(STATUS "VCPKG_OVERLAY_TRIPLETS ${VCPKG_OVERLAY_TRIPLETS}")

# These packages are used in the public headers of Cesium libraries, so we need to distribute the headers and binaries
# with the installation
# Note that fmt is a public dependency of the vcpkg version of spdlog
# STB is not technically part of the public interface, but it is used by the downstream Cesium for Unreal project
set(PACKAGES_PUBLIC asyncplusplus expected-lite fmt glm rapidjson spdlog stb ada-url)
# These packages are used in the code and produce binaries, but are not part of the public interface.  Therefore we need
# to distribute the binaries for linking, but not the headers, as downstream consumers don't need them
# OpenSSL and abseil are both dependencies of s2geometry
set(PACKAGES_PRIVATE
    abseil curl draco ktx modp-base64 meshoptimizer openssl s2geometry
    libjpeg-turbo sqlite3 tinyxml2 libwebp zlib-ng picosha2
    earcut-hpp cpp-httplib[core] libmorton zstd
)

# Packages only used for testing
set(PACKAGES_TEST doctest)

if(CESIUM_USE_EZVCPKG)
    set(PACKAGES_ALL ${PACKAGES_PUBLIC})
    list(APPEND PACKAGES_ALL ${PACKAGES_PRIVATE})
    list(APPEND PACKAGES_ALL ${PACKAGES_TEST})

    ezvcpkg_fetch(
        COMMIT dbe35ceb30c688bf72e952ab23778e009a578f18
        PACKAGES ${PACKAGES_ALL}
        # Clean the build trees after building, so that we don't use a ton a disk space on the CI cache
        CLEAN_BUILDTREES
        # Update the cmake toolchain so it can find the above packages
        UPDATE_TOOLCHAIN
        # Force the installation of each package one at a time, or the Travis CI build will time out waiting for output
        SERIALIZE
    )
endif()

if (NOT CMAKE_TOOLCHAIN_FILE)
    message(FATAL_ERROR "Specify the VCPKG toolchain on the command line as '-DCMAKE_TOOLCHAIN_FILE=<VCPKG_ROOT>/scripts/buildsystems/vcpkg.cmake'")
else()
    message(STATUS "CMAKE_TOOLCHAIN_FILE ${CMAKE_TOOLCHAIN_FILE}")
endif()

# Set defaults that should be set BEFORE compiler / IDE detection
include("cmake/defaults.cmake")

project(cesium-native
    VERSION 0.48.0
    LANGUAGES CXX C
)

include(GNUInstallDirs)
include(CMakeDependentOption)

set(PACKAGE_BASE_DIR "${EZVCPKG_PACKAGES_DIR}")
set(PACKAGE_BUILD_DIR "${EZVCPKG_DIR}")

if(NOT EZVCPKG_PACKAGES_DIR)
    set(PACKAGE_BUILD_DIR "${VCPKG_INSTALLED_DIR}/${VCPKG_TRIPLET}/")
    set(PACKAGE_BASE_DIR "$ENV{VCPKG_ROOT}/packages")
endif()

message(STATUS "PACKAGE_BASE_DIR ${PACKAGE_BASE_DIR}")
message(STATUS "PACKAGE_BUILD_DIR ${PACKAGE_BUILD_DIR}")


option(CESIUM_INSTALL_STATIC_LIBS "Whether to install the static libraries of cesium-native and its dependencies." ON)
option(CESIUM_INSTALL_HEADERS "Whether to install the header files of cesium-native and its public dependencies." ON)
option(CESIUM_ENABLE_CLANG_TIDY "Enable clang-tidy targets for static code analysis." ON)

cmake_dependent_option(
    CESIUM_ENABLE_CLANG_TIDY_ON_BUILD
    "Run clang-tidy while building. Will slow down the build process. Available only if CESIUM_ENABLE_CLANG_TIDY is ON."
    OFF
    CESIUM_ENABLE_CLANG_TIDY
    OFF
)

set(CESIUM_CLANG_TIDY_USE_THREADS 14 CACHE STRING "Sets the number of threads for run-clang-tidy to use.")

if(CESIUM_INSTALL_STATIC_LIBS OR CESIUM_INSTALL_HEADERS AND EZVCPKG_PACKAGES_DIR)
    foreach(PACKAGE ${PACKAGES_PUBLIC})
        string(REGEX REPLACE "\[.*\]" "" PACKAGE ${PACKAGE})
        set(PACKAGE_DIR ${PACKAGE_BASE_DIR}/${PACKAGE}_${VCPKG_TRIPLET})
        message(DEBUG "PACKAGE_DIR ${PACKAGE_DIR}")

        if(CESIUM_INSTALL_HEADERS AND NOT PACKAGE IN_LIST CESIUM_EXCLUDE_INSTALL_HEADERS)
            install(
                DIRECTORY ${PACKAGE_DIR}/include/
                TYPE INCLUDE
            )
        endif()

        if (CESIUM_INSTALL_STATIC_LIBS AND NOT PACKAGE IN_LIST CESIUM_EXCLUDE_INSTALL_STATIC_LIBS AND EXISTS ${PACKAGE_DIR}/lib)
            install(
                DIRECTORY $<IF:$<CONFIG:Debug>,${PACKAGE_DIR}/debug/lib/,${PACKAGE_DIR}/lib/>
                DESTINATION ${CMAKE_INSTALL_LIBDIR}
            )
        endif()
    endforeach()
endif()

if(CESIUM_INSTALL_STATIC_LIBS AND NOT VCPKG_MANIFEST_MODE)
    foreach(PACKAGE ${PACKAGES_PRIVATE})
        set(PACKAGE_DIR ${PACKAGE_BASE_DIR}/${PACKAGE}_${VCPKG_TRIPLET})
        message(DEBUG "PACKAGE_DIR ${PACKAGE_DIR}")
        if (NOT PACKAGE IN_LIST CESIUM_EXCLUDE_INSTALL_STATIC_LIBS AND EXISTS ${PACKAGE_DIR}/lib)
            install(
                DIRECTORY $<IF:$<CONFIG:Debug>,${PACKAGE_DIR}/debug/lib/,${PACKAGE_DIR}/lib/>
                DESTINATION ${CMAKE_INSTALL_LIBDIR}
            )
        endif()
    endforeach()
endif()

if(NOT DEFINED CMAKE_C_COMPILER_LAUNCHER AND NOT DEFINED CMAKE_CXX_COMPILER_LAUNCHER)
    find_program(CCACHE_FOUND ccache)
    find_program(SCCACHE_FOUND sccache)
    if (SCCACHE_FOUND)
        message("setting SCCACHE to ${SCCACHE_FOUND}")
        set(CMAKE_C_COMPILER_LAUNCHER ${SCCACHE_FOUND})
        set(CMAKE_CXX_COMPILER_LAUNCHER ${SCCACHE_FOUND})
    elseif(CCACHE_FOUND)
        message("setting CCACHE to ${CCACHE_FOUND}")
        set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE_FOUND})
        set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_FOUND})
    endif()
endif()

# ccache/sccache only works with /Z7, not /Zi, so tweak the Debug and RelWithDebInfo build flags in the presence of a compiler cache
if(MSVC AND (DEFINED CMAKE_C_COMPILER_LAUNCHER))
    message(DEBUG "Setting MSVC flags to /Z7 for ccache compatibility.  Current flags: ${CMAKE_CXX_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "/Z7" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "/Z7" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "/Z7" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
    string(REPLACE "/Zi" "/Z7" CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
    message(DEBUG "New flags: ${CMAKE_CXX_FLAGS_DEBUG}")
endif()

option(CESIUM_TRACING_ENABLED "Whether to enable the Cesium performance tracing framework (CESIUM_TRACE_* macros)." OFF)
option(CESIUM_COVERAGE_ENABLED "Whether to enable code coverage" OFF)
option(CESIUM_TESTS_ENABLED "Whether to enable tests" ON)
option(CESIUM_GLM_STRICT_ENABLED "Whether to force strict GLM compile definitions." ON)
option(CESIUM_DISABLE_DEFAULT_ELLIPSOID "Whether to disable the WGS84 default value for ellipsoid parameters across cesium-native." OFF)
option(CESIUM_MSVC_STATIC_RUNTIME_ENABLED "Whether to enable static linking for MSVC runtimes" OFF)
option(CESIUM_DEBUG_TILE_UNLOADING "Whether to enable tracking of tile _doNotUnloadSubtreeCount modifications for tile unloading debugging." OFF)
option(CESIUM_CLANG_TIME_TRACE "Whether to enable the -ftime-trace compilation option when building with Clang." OFF)

if (CESIUM_MSVC_STATIC_RUNTIME_ENABLED)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

if (CESIUM_TRACING_ENABLED)
    add_compile_definitions(CESIUM_TRACING_ENABLED=1)
endif()

# Set defaults that need to be set AFTER compiler / IDE detection
include("cmake/compiler.cmake")

if(CESIUM_ENABLE_CLANG_TIDY)
    setup_clang_tidy(
        PROJECT_BUILD_DIRECTORY
        "${PROJECT_BINARY_DIR}"
        PROJECT_SOURCE_DIRECTORIES
        "${PROJECT_SOURCE_DIR}"
        ENABLE_CLANG_TIDY_ON_BUILD
        ${CESIUM_ENABLE_CLANG_TIDY_ON_BUILD}
    )
endif()

# Add Modules
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/modules")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/extern/cmake-modules/")
if (CESIUM_COVERAGE_ENABLED AND NOT MSVC)
    include(CodeCoverage)
    append_coverage_compiler_flags()
    setup_target_for_coverage_gcovr_html(
        NAME cesium-native-tests-coverage
        EXECUTABLE ctest -j ${PROCESSOR_COUNT}
        EXCLUDE "${PROJECT_SOURCE_DIR}/extern/*" "${PROJECT_BINARY_DIR}"
        DEPENDENCIES cesium-native-tests
    )
endif()

if (NOT DEFINED GLOB_USE_CONFIGURE_DEPENDS)
    set(GLOB_USE_CONFIGURE_DEPENDS OFF CACHE BOOL
        "Controls if cesium-native targets should use configure_depends or not for globbing their sources"
    )
endif()

# On the CI builds, I have to do this explicitly for some reason or it fails to find the vcpkg packages.
# The toolchain is supposed to manage this, but I haven't figured out why it isn't yet.

list(APPEND CMAKE_PREFIX_PATH "${PACKAGE_BUILD_DIR}/share/s2")
list(APPEND CMAKE_PREFIX_PATH "${PACKAGE_BUILD_DIR}/share")
list(APPEND CMAKE_PREFIX_PATH "${PACKAGE_BUILD_DIR}")

# Find the VCPKG dependnecies
# Note that while we could push these into the extern/CMakeLists.txt as an organization tidy-up, that would require
# us to update the minimum version of CMake to 3.24 and to add the GLOBAL option to the find_package calls, otherwise
# they won't be visible in this scope nor any of the subdirectories for the actual libraries.
#
# However, for some of the vcpkg built libraries where they don't provide a prope cmake config file, we have to declare
# and imporeted library target ourselves. This is the case for modp_b64::modp_b64, picosha2::picosha2 and earcut. In
# these cases, we *do* have the somewhat ugly and verbose details in the extern/CMakeLists.txt file.
#
# XXX Above comment should be obsoleted by these first calls to
# find_package, which resolve to our own modules that provide
# targets. If needed, they can be installed with CMake config files
# etc.
find_package(zlib-ng REQUIRED)
find_package(modp_b64 REQUIRED)

find_package(ada CONFIG REQUIRED)
find_package(Async++ CONFIG REQUIRED)
find_package(CURL REQUIRED)
find_package(doctest CONFIG REQUIRED)
find_package(draco CONFIG REQUIRED)
find_package(expected-lite CONFIG REQUIRED)
find_package(glm CONFIG REQUIRED)
find_package(httplib CONFIG REQUIRED)
find_package(Ktx CONFIG REQUIRED)
find_package(libmorton CONFIG REQUIRED)
find_package(libjpeg-turbo CONFIG REQUIRED)
find_package(meshoptimizer CONFIG REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(s2 CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(tinyxml2 CONFIG REQUIRED)
find_package(unofficial-sqlite3 CONFIG REQUIRED)
find_package(WebP CONFIG REQUIRED)

# Private Library (s2geometry)
add_subdirectory(extern EXCLUDE_FROM_ALL)

# Public Targets
add_subdirectory(CesiumUtility)
add_subdirectory(CesiumGltf)
add_subdirectory(CesiumGeometry)
add_subdirectory(CesiumGeospatial)
add_subdirectory(CesiumJsonReader)
add_subdirectory(CesiumJsonWriter)
add_subdirectory(CesiumGltfContent)
add_subdirectory(CesiumGltfReader)
add_subdirectory(CesiumGltfWriter)
add_subdirectory(CesiumAsync)
add_subdirectory(Cesium3DTiles)
add_subdirectory(Cesium3DTilesReader)
add_subdirectory(Cesium3DTilesWriter)
add_subdirectory(Cesium3DTilesContent)
add_subdirectory(CesiumRasterOverlays)
add_subdirectory(Cesium3DTilesSelection)
add_subdirectory(CesiumClientCommon)
add_subdirectory(CesiumIonClient)
add_subdirectory(CesiumITwinClient)
add_subdirectory(CesiumQuantizedMeshTerrain)
add_subdirectory(CesiumVectorData)
add_subdirectory(CesiumCurl)

# Private Targets
if (CESIUM_TESTS_ENABLED)
    # enable_testing() MUST be called before add_subdirectory or no tests
    # will be found by ctest
    enable_testing()
    add_subdirectory(CesiumNativeTests)
endif()


add_subdirectory(doc)

include(CMakePackageConfigHelpers)

if(CESIUM_INSTALL_STATIC_LIBS AND CESIUM_INSTALL_HEADERS)
  install(EXPORT CesiumExports
    FILE cesium-nativeTargets.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/cesium-native/cmake)

  install(FILES
    "${CMAKE_CURRENT_LIST_DIR}/cmake/modules/Findzlib-ng.cmake"
    "${CMAKE_CURRENT_LIST_DIR}/cmake/modules/Findmodp_b64.cmake"
    DESTINATION ${CMAKE_INSTALL_DATADIR}/cesium-native/cmake)

  configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/config/Config.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/cesium-nativeConfig.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_DATADIR}/cesium-native/cmake)

  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/cesium-nativeConfig.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/cesium-native/cmake)
endif()
