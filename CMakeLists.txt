project(umap-build)
cmake_minimum_required(VERSION 3.5.1)
include(ExternalProject)
set(BUILD_TOP ${PROJECT_SOURCE_DIR})
set(BUILD_SRC ${PROJECT_SOURCE_DIR}/src)
set(BUILD_BIN ${PROJECT_SOURCE_DIR}/bin)
set(BUILD_DL  ${PROJECT_SOURCE_DIR}/tmp)
set(BUILD_OUT ${PROJECT_SOURCE_DIR}/build)
file(MAKE_DIRECTORY ${BUILD_SRC})
file(MAKE_DIRECTORY ${BUILD_BIN})
file(MAKE_DIRECTORY ${BUILD_DL})
file(MAKE_DIRECTORY ${BUILD_OUT})
file(MAKE_DIRECTORY ${BUILD_OUT}/include)
file(MAKE_DIRECTORY ${BUILD_OUT}/lib)
file(MAKE_DIRECTORY ${BUILD_OUT}/bin)

function(hash_cfitsio_tar match)
  set(match 0)
  file(SHA512 ${BUILD_DL}/cfitsio3450.tar.gz CFITS_SHA512)
  if (CFITS_SHA512 STREQUAL "f553f9bbedc3ffabf939bb507d3c41d6dc0d889b3a0835c23a7aff90ba98d7f9a6217527d69bc8984ce9bd9664681baeec5b1555d2ab51aaca003fdb5a08127e")
    execute_process(COMMAND tar -xf ${BUILD_DL}/cfitsio3450.tar.gz -C ${BUILD_SRC}/ WORKING_DIRECTORY ${BUILD_TOP})
    set(match 1)
  else()
    message("Hash verification failed for cfitsio tar file.")
  endif()
endfunction()

if (NOT EXISTS ${BUILD_SRC}/cfitsio/CMakeLists.txt)
  hash_cfitsio_tar(FILE_VALID)
  if (NOT FILE_VALID)
    if (NOT EXISTS ${BUILD_DL}/cfitsio3450.tar.gz)
      message("Downloading cfitsio...")
      execute_process(COMMAND wget http://heasarc.gsfc.nasa.gov/FTP/software/fitsio/c/cfitsio3450.tar.gz WORKING_DIRECTORY ${BUILD_DL})
      hash_cfitsio_tar(DL_FILE_VALID)
      if (NOT DL_FILE_VALID)
        return()
      endif()
    endif()
  endif()
endif()

OPTION (ENABLE_TESTS "Selects whether tests are built." Off)
OPTION (ENABLE_LOGGING "Build umap with Logging enabled" Off)
OPTION (SHARED_LIBS "Build with shared libraries" Off)

set(UMAP_INSTALL_PATH ${BUILD_OUT})
set(BUILD_SHARED_LIBS ${SHARED_LIBS})
set(CMAKE_INSTALL_PREFIX ${BUILD_OUT})
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

ExternalProject_Add(
  umap
  DOWNLOAD_COMMAND ""
  CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${BUILD_OUT} -DBUILD_SHARED_LIBS=${SHARED_LIBS}
  SOURCE_DIR ${BUILD_SRC}/umap
  BINARY_DIR ${BUILD_BIN}/umap
  INSTALL_DIR ${BUILD_OUT}/
)

ExternalProject_Add(
  umap-apps
  DOWNLOAD_COMMAND ""
  CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${BUILD_OUT} -DBUILD_SHARED_LIBS=${SHARED_LIBS} -DUMAP_INSTALL_PATH=${BUILD_OUT} -DCFITS_INCLUDE_PATH=${BUILD_OUT}/include -DCFITS_LIBRARY_PATH=${BUILD_OUT}/lib
  SOURCE_DIR ${BUILD_SRC}/umap-apps
  BINARY_DIR ${BUILD_BIN}/umap-apps
  INSTALL_DIR ${BUILD_OUT}/
  DEPENDS umap cfitsio
)

