cmake_minimum_required(VERSION 2.8.9)
include(CheckCSourceCompiles)
include(CheckIncludeFile)

set(LIBNAME jsongit2)
project(libjsongit2 C)

#
# You can override where libjsongit2 gets its dependent library and includes
# from, at the cmake commandline.  For git2, eg
#
# cmake .. -DJG2_GIT2_INC_PATH=/usr/local/include \
#          -DJG2_GIT2_LIB_PATH=/usr/local/lib/libgit2.so
#
# and for libarchive, eg
#
# cmake .. -DJG2_ARCHIVE_INC_PATH=/usr/local/include \
#          -DJG2_ARCHIVE_LIB_PATH=/usr/local/lib/libgit2.so
#


set(JG2_INSTALL_LIB_DIR       lib     CACHE PATH "Install dir for libraries")
set(JG2_INSTALL_BIN_DIR       bin     CACHE PATH "Install dir for executables")
set(JG2_INSTALL_INCLUDE_DIR   include CACHE PATH "Install dir for header files")
set(JG2_INSTALL_DATA_DIR      share   CACHE PATH "Install dir for data files")
set(CMAKE_MACOSX_RPATH TRUE)

set(CPACK_PACKAGE_NAME "${LIBNAME}")
set(CPACK_PACKAGE_VERSION_MAJOR "1")
set(CPACK_PACKAGE_VERSION_MINOR "0")
set(CPACK_PACKAGE_VERSION_PATCH "0")
set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
set(CPACK_PACKAGE_VENDOR "andy@warmcat.com")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${LIBNAME} ${PACKAGE_VERSION}")
set(SOVERSION "1")

if (CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR (CMAKE_C_COMPILER_ID MATCHES "Clang") OR (CMAKE_CXX_COMPILER_ID MATCHES "Clang"))
    include (CheckCCompilerFlag)
    
    set(CMAKE_C_FLAGS "-Wall -Wsign-compare -Wignored-qualifiers -Wtype-limits -Wuninitialized -Werror -Wundef ${CMAKE_C_FLAGS}" )
    
    CHECK_C_COMPILER_FLAG(-fvisibility=hidden JG2_HAVE_VISIBILITY)
    if (JG2_HAVE_VISIBILITY)
          set(CMAKE_C_FLAGS  "-fvisibility=hidden ${CMAKE_C_FLAGS}" )
    endif()
endif()

set(DEPLIBS pthread)

#
# libgit2 paths
#
find_path(   JG2_GIT2_INC_PATH NAMES "git2.h")
find_library(JG2_GIT2_LIB_PATH NAMES "git2")

if (JG2_GIT2_INC_PATH AND JG2_GIT2_LIB_PATH)
	set(DEPLIBS ${JG2_GIT2_LIB_PATH} ${DEPLIBS})
	include_directories(BEFORE "${JG2_GIT2_INC_PATH}")
else()
	message(FATAL_ERROR " Unable to find libgit2")
endif()

#
# libarchive paths
#
find_path(   JG2_ARCHIVE_INC_PATH NAMES "archive.h")
find_library(JG2_ARCHIVE_LIB_PATH NAMES "archive")

if (JG2_ARCHIVE_INC_PATH AND JG2_ARCHIVE_LIB_PATH)
	set(DEPLIBS ${JG2_ARCHIVE_LIB_PATH} ${DEPLIBS})
	include_directories(BEFORE "${JG2_ARCHIVE_INC_PATH}")
	set(JG2_HAVE_ARCHIVE_H "Y")
endif()

set(CMAKE_REQUIRED_INCLUDES "${JG2_ARCHIVE_INC_PATH}")
CHECK_C_SOURCE_COMPILES(
	"#include <git2.h>
	int main(int argc, char **argv) { return GIT_BLAME_USE_MAILMAP; }
	" JG2_HAVE_BLAME_MAILMAP)

set(SOURCES lib/cache.c
	    lib/main.c
	    lib/repostate.c
	    lib/util.c

	    lib/job/job.c
	    lib/job/reflist.c
	    lib/job/log.c
	    lib/job/commit.c
	    lib/job/tree.c
	    lib/job/plain.c
	    lib/job/repos.c
	    lib/job/blame.c
	    lib/job/blog.c

	    lib/conf/gitolite.c
	    lib/conf/scan-repos.c

	    lib/email/md5.c
	    lib/email/email.c

	    lib/lac/lac.c
	    lib/lac/cached-file.c
)

if (JG2_HAVE_ARCHIVE_H)
	set(SOURCES ${SOURCES} lib/job/snapshot.c)
	set(DEPLIBS ${DEPLIBS} archive)
else()
	set(SOURCES ${SOURCES} lib/job/no-snapshot.c)
endif()


configure_file("cmake/config.h.in" "${PROJECT_BINARY_DIR}/jg2-config.h")

add_library(${LIBNAME} SHARED ${SOURCES})

set(HDR_PUBLIC "include/libjsongit2.h" "${PROJECT_BINARY_DIR}/jg2-config.h")

set_target_properties(${LIBNAME} PROPERTIES PUBLIC_HEADER "${HDR_PUBLIC}")
	
target_include_directories(${LIBNAME} PRIVATE "${PROJECT_BINARY_DIR}"
					      "${PROJECT_SOURCE_DIR}/include")
					      
target_link_libraries(${LIBNAME} ${DEPLIBS})

set(CPACK_COMPONENT_LIBRARIES_DISPLAY_NAME "Libraries")
set(CPACK_COMPONENT_DEV_DISPLAY_NAME "Development files")

message("----------------------------- dependent libs -----------------------------")
message(" libgit2:    include: ${JG2_GIT2_INC_PATH}, lib: ${JG2_GIT2_LIB_PATH}")
message(" libgit2:    has GIT_BLAME_USE_MAILMAP: ${JG2_HAVE_BLAME_MAILMAP}")
if (NOT JG2_HAVE_ARCHIVE_H)
	message(" libarchive: not found")
else()
	message(" libarchive: include: ${JG2_ARCHIVE_INC_PATH}, lib: ${JG2_ARCHIVE_LIB_PATH}")
endif()

add_executable(jg2-example examples/minimal/jg2-example.c)
target_link_libraries(jg2-example jsongit2)
target_include_directories(jg2-example PRIVATE "${PROJECT_SOURCE_DIR}/include")

add_executable(jg2-threadchurn examples/threadchurn/threadchurn.c)
target_link_libraries(jg2-threadchurn jsongit2 pthread)
target_include_directories(jg2-threadchurn PRIVATE "${PROJECT_SOURCE_DIR}/include")

install(TARGETS
		${LIBNAME}
		jg2-example
		jg2-threadchurn

	LIBRARY DESTINATION "${JG2_INSTALL_LIB_DIR}${LIB_SUFFIX}" COMPONENT libraries
	ARCHIVE DESTINATION "${JG2_INSTALL_LIB_DIR}${LIB_SUFFIX}" COMPONENT libraries
	RUNTIME DESTINATION "${JG2_INSTALL_BIN_DIR}" COMPONENT libraries
	PUBLIC_HEADER DESTINATION "${JG2_INSTALL_INCLUDE_DIR}" COMPONENT dev)
