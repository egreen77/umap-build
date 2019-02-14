project(umap-build)
cmake_minimum_required(VERSION 3.5.1)
include(ExternalProject)
set(BUILD_TOP ${PROJECT_SOURCE_DIR})
set(BUILD_SRC ${PROJECT_SOURCE_DIR}/src)
set(BUILD_BIN ${PROJECT_BINARY_DIR}/build/bin)
set(BUILD_DL  ${PROJECT_BINARY_DIR}/tmp)
set(BUILD_OUT ${PROJECT_BINARY_DIR}/build)
file(MAKE_DIRECTORY ${BUILD_SRC})
file(MAKE_DIRECTORY ${BUILD_BIN})
file(MAKE_DIRECTORY ${BUILD_DL})
file(MAKE_DIRECTORY ${BUILD_OUT})
file(MAKE_DIRECTORY ${BUILD_OUT}/include)
file(MAKE_DIRECTORY ${BUILD_OUT}/lib)
file(MAKE_DIRECTORY ${BUILD_OUT}/bin)

function(hash_cfitsio_tar rv)
  set(${rv} 0 PARENT_SCOPE)
  file(SHA512 ${BUILD_DL}/cfitsio3450.tar.gz CFITS_SHA512)
  message("Hashing cfitsio tar file...")
  if (CFITS_SHA512 STREQUAL "f553f9bbedc3ffabf939bb507d3c41d6dc0d889b3a0835c23a7aff90ba98d7f9a6217527d69bc8984ce9bd9664681baeec5b1555d2ab51aaca003fdb5a08127e")
    message("Extracting cfitsio tar file...")
    execute_process(COMMAND tar -xf ${BUILD_DL}/cfitsio3450.tar.gz -C ${BUILD_SRC}/ WORKING_DIRECTORY ${BUILD_TOP})
    set(${rv} 1 PARENT_SCOPE)
    return()
  else()
    message("Hash verification failed for cfitsio tar file.")
  endif()
endfunction()

function(fetch_cfits rv)
  message("Downloading cfitsio...")
  execute_process(COMMAND wget http://heasarc.gsfc.nasa.gov/FTP/software/fitsio/c/cfitsio3450.tar.gz WORKING_DIRECTORY ${BUILD_DL})
  hash_cfitsio_tar("TARFILE_VALID")
  set(${rv} TARFILE_VALID PARENT_SCOPE)
endfunction()

function(get_cfits)
  if (NOT EXISTS ${BUILD_SRC}/cfitsio/CMakeLists.txt)
    if (NOT EXISTS ${BUILD_DL}/cfitsio3450.tar.gz)
      fetch_cfits("DL_CFITS_HASH_VALID")
      if (NOT DL_CFITS_HASH_VALID)
        message( FATAL_ERROR "Failed to fetch a valid cfitsio tarball." )
      endif()
    else()
      hash_cfitsio_tar("CFITS_HASH_VALID")
      if (NOT CFITS_HASH_VALID)
        fetch_cfits("DL_CFITS_HASH_VALID")
        if (NOT DL_CFITS_HASH_VALID)
          message( FATAL_ERROR "Failed to fetch a valid cfitsio tarball." )
        endif()
      endif()
    endif()
  endif()
endfunction()


OPTION (ENABLE_CFITS "Enable CFITSIO" Off)
OPTION (ENABLE_TESTS "Selects whether tests are built." Off)
OPTION (ENABLE_LOGGING "Build umap with Logging enabled" Off)
OPTION (SHARED_LIBS "Build with shared libraries" Off)
message(" === OPTION SUMMARY ===")
message("ENABLE_CFITS        ${ENABLE_CFITS}")
message("ENABLE_TESTS        ${ENABLE_TESTS}")
message("ENABLE_LOGGING      ${ENABLE_LOGGING}")
message("SHARED_LIBS         ${SHARED_LIBS}")

set(UMAP_INSTALL_PATH ${BUILD_OUT})
set(BUILD_SHARED_LIBS ${SHARED_LIBS})
set(CMAKE_INSTALL_PREFIX ${BUILD_OUT})

# Arguments and deps for umap-apps ExternalProject
set(APPS_CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${BUILD_OUT} -DBUILD_SHARED_LIBS=${SHARED_LIBS} -DUMAP_INSTALL_PATH=${BUILD_OUT})
set(APPS_DEPS umap)

if (ENABLE_CFITS)
  get_cfits()
  set(CFITS_INCLUDE_PATH ${BUILD_OUT}/include)
  set(CFITS_LIBRARY_PATH ${BUILD_OUT}/lib)
  ExternalProject_Add(
    cfitsio
    DOWNLOAD_COMMAND ""
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${BUILD_OUT} -DBUILD_SHARED_LIBS=${SHARED_LIBS}
    SOURCE_DIR ${BUILD_SRC}/cfitsio
    BINARY_DIR ${BUILD_BIN}/cfitsio
    INSTALL_DIR ${BUILD_OUT}/
  )
  set(APPS_CMAKE_ARGS ${APPS_CMAKE_ARGS} -DCFITS_INCLUDE_PATH=${CFITS_INCLUDE_PATH} -DCFITS_LIBRARY_PATH=${CFITS_LIBRARY_PATH})
  set(APPS_DEPS ${APPS_DEPS} cfitsio)
endif()

ExternalProject_Add(
  umap
  DOWNLOAD_COMMAND ""
  CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${BUILD_OUT} -DBUILD_SHARED_LIBS=${SHARED_LIBS} -DENABLE_TESTS=${ENABLE_TESTS} -DENABLE_LOGGING=${ENABLE_LOGGING}
  SOURCE_DIR ${BUILD_SRC}/umap
  BINARY_DIR ${BUILD_BIN}/umap
  INSTALL_DIR ${BUILD_OUT}/
)

ExternalProject_Add(
  umap-apps
  DOWNLOAD_COMMAND ""
  CMAKE_ARGS ${APPS_CMAKE_ARGS}
  SOURCE_DIR ${BUILD_SRC}/umap-apps
  BINARY_DIR ${BUILD_BIN}/umap-apps
  INSTALL_DIR ${BUILD_OUT}/
  DEPENDS ${APPS_DEPS}
)

