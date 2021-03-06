include(cmake/Macros/icecc.cmake) # this must be the first line!

cmake_minimum_required(VERSION 3.1)
cmake_policy(VERSION 3.1)

# Don't ignore targets that do not exist, inside add_dependencies calls.
cmake_policy(SET CMP0046 NEW)

project(pysidebindings)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Macros/
                      ${CMAKE_MODULE_PATH})

option(USE_PYTHON_VERSION "Use specific python version to build pyside2." "")

if (USE_PYTHON_VERSION)
    find_package(PythonInterp ${USE_PYTHON_VERSION} REQUIRED)
    find_package(PythonLibs ${USE_PYTHON_VERSION} REQUIRED)
else()
    find_package(PythonInterp 2.6)
    find_package(PythonLibs 2.6)
endif()

macro(get_python_extension_suffix)
  # Result of imp.get_suffixes() depends on the platform, but generally looks something like:
  # [('.cpython-34m-x86_64-linux-gnu.so', 'rb', 3), ('.cpython-34m.so', 'rb', 3),
  # ('.abi3.so', 'rb', 3), ('.so', 'rb', 3), ('.py', 'r', 1), ('.pyc', 'rb', 2)]
  # We pick the first most detailed one, strip of the file extension part.

  execute_process(
    COMMAND ${PYTHON_EXECUTABLE} -c "if True:
       import imp, re
       first_suffix = imp.get_suffixes()[0][0]
       res = re.search(r'^(.+)\\.', first_suffix)
       if res:
           first_suffix = res.group(1)
       else:
           first_suffix = ''
       print(first_suffix)
       "
    OUTPUT_VARIABLE PYTHON_EXTENSION_SUFFIX
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  message("PYTHON_EXTENSION_SUFFIX: " ${PYTHON_EXTENSION_SUFFIX})
endmacro()


if (NOT PYTHON_EXTENSION_SUFFIX)
  get_python_extension_suffix()
endif()

if (NOT PYTHON_CONFIG_SUFFIX)
  if (PYTHON_VERSION_MAJOR EQUAL 2)
      set(PYTHON_CONFIG_SUFFIX "-python${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}")
      if (PYTHON_EXTENSION_SUFFIX)
          set(PYTHON_CONFIG_SUFFIX "${PYTHON_CONFIG_SUFFIX}${PYTHON_EXTENSION_SUFFIX}")
      endif()
  elseif (PYTHON_VERSION_MAJOR EQUAL 3)
      set(PYTHON_CONFIG_SUFFIX "${PYTHON_EXTENSION_SUFFIX}")
  endif()
endif()

if (NOT PYTHON_SHARED_LIBRARY_SUFFIX)
  set(PYTHON_SHARED_LIBRARY_SUFFIX "${PYTHON_CONFIG_SUFFIX}")

  # Append a "v" to disambiguate the python version and the pyside version in the
  # shared library file name.
  if (APPLE AND PYTHON_VERSION_MAJOR EQUAL 2)
      set(PYTHON_SHARED_LIBRARY_SUFFIX "${PYTHON_SHARED_LIBRARY_SUFFIX}v")
  endif()
endif()


message(STATUS "PYTHON_EXTENSION_SUFFIX:      ${PYTHON_EXTENSION_SUFFIX}")
message(STATUS "PYTHON_CONFIG_SUFFIX:         ${PYTHON_CONFIG_SUFFIX}")
message(STATUS "PYTHON_SHARED_LIBRARY_SUFFIX: ${PYTHON_SHARED_LIBRARY_SUFFIX}")

find_package(Shiboken2 2.0.0 REQUIRED)
find_package(Qt5 5.6.0 REQUIRED COMPONENTS Core)
add_definitions(${Qt5Core_DEFINITIONS})

find_file(GL_H "gl.h" PATH_SUFFIXES "GL")
message("result:" "${GL_H}")
include(FindQt5Extra)

set(XVFB_EXEC "")
option(USE_XVFB "Uses xvfb-run with the unit tests to avoid QtGui tests popping windows on the screen." FALSE)
if(USE_XVFB)
    find_program(XVFB_RUN NAMES xvfb-run)
    if (NOT ${XVFB_RUN} MATCHES "XVFB_RUN-NOTFOUND")
        set(XVFB_EXEC ${XVFB_RUN} -a)
        message(STATUS "Using xvfb-run to perform QtGui tests.")
    endif()
endif()

option(BUILD_TESTS "Build tests." TRUE)
option(ENABLE_VERSION_SUFFIX "Used to use current version in suffix to generated files. This is used to allow multiples versions installed simultaneous." FALSE)
set(LIB_SUFFIX "" CACHE STRING "Define suffix of directory name (32/64)" )
set(LIB_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX}" CACHE PATH "The subdirectory relative to the install prefix where libraries will be installed (default is /lib${LIB_SUFFIX})" FORCE)
if(CMAKE_HOST_APPLE)
    set(ALTERNATIVE_QT_INCLUDE_DIR "" CACHE PATH "Deprecated. CMake now finds the proper include dir itself.")
    set(OSX_USE_LIBCPP "OFF" CACHE BOOL "Explicitly link the libc++ standard library (useful for osx deployment targets lower than 10.9.")
    if(OSX_USE_LIBCPP)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
    endif()
endif()

# Force usage of the C++11 standard, without a silent fallback
# to C++98 if the compiler does not support C++11.
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Qt5: QT_INCLUDE_DIR does no longer exist.
# On Windows, macOS, and Linux it can be computed from Qt5Core_INCLUDE_DIRS, which contains
# a list of include directories. We take the first one.
message(STATUS "*** Qt5Core_INCLUDE_DIRS = ${Qt5Core_INCLUDE_DIRS}")
list(GET Qt5Core_INCLUDE_DIRS 0 QT_INCLUDE_DIR)

# On macOS, check if Qt is a framework build. This affects how include paths should be handled.
get_target_property(QtCore_is_framework Qt5::Core FRAMEWORK)

if (QtCore_is_framework)
    # Get the path to the framework dir.
    get_filename_component(QT_FRAMEWORK_INCLUDE_DIR "${QT_INCLUDE_DIR}/../" ABSOLUTE)
    message(STATUS "*** QT_FRAMEWORK_INCLUDE_DIR is ${QT_FRAMEWORK_INCLUDE_DIR}")

    # QT_INCLUDE_DIR points to the QtCore.framework directory, so we need to adjust this to point
    # to the actual include directory, which has include files for non-framework parts of Qt.
    get_filename_component(QT_INCLUDE_DIR "${QT_INCLUDE_DIR}/../../include" ABSOLUTE)
endif()

if(MSVC)
    # Qt5: this flag has changed from /Zc:wchar_t- in Qt4.X
    set(CMAKE_CXX_FLAGS "/Zc:wchar_t /GR /EHsc /DNOCOLOR /DWIN32 /D_WINDOWS /D_SCL_SECURE_NO_WARNINGS") # XXX
else()
    if(CMAKE_HOST_UNIX AND NOT CYGWIN)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -fvisibility=hidden -Wno-strict-aliasing")
    endif()
    set(CMAKE_CXX_FLAGS_DEBUG "-g")
    option(ENABLE_GCC_OPTIMIZATION "Enable specific GCC flags to optimization library size and performance. Only available on Release Mode" 0)
    if(ENABLE_GCC_OPTIMIZATION)
        set(CMAKE_BUILD_TYPE Release)
        set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG -Os -Wl,-O1")
        if(NOT CMAKE_HOST_APPLE)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wl,--hash-style=gnu")
        endif()
    endif()

    if(CMAKE_HOST_APPLE)
        # ALTERNATIVE_QT_INCLUDE_DIR is deprecated, because CMake takes care of finding the proper
        # include folders using the qmake found in the environment. Only use it for now in case
        # something goes wrong with the cmake process.
        if(ALTERNATIVE_QT_INCLUDE_DIR AND NOT QT_INCLUDE_DIR)
            set(QT_INCLUDE_DIR ${ALTERNATIVE_QT_INCLUDE_DIR})
        endif()
    endif()
endif()
message(STATUS "*** computed QT_INCLUDE_DIR as ${QT_INCLUDE_DIR}")

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE ${SHIBOKEN_BUILD_TYPE})
endif()

set(BINDING_NAME PySide2)
set(BINDING_API_MAJOR_VERSION "2")
set(BINDING_API_MINOR_VERSION "0")
set(BINDING_API_MICRO_VERSION "0")
set(BINDING_API_RELEASE_LEVEL "alpha") # alpha, beta, rc, or final
set(BINDING_API_SERIAL 0) # leave as 0 when release level is final
set(BINDING_API_VERSION "${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}.${BINDING_API_MICRO_VERSION}" CACHE STRING "PySide version" FORCE)
if (BINDING_API_RELEASE_LEVEL STREQUAL "final")
    set(BINDING_API_VERSION_FULL "${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}.${BINDING_API_MICRO_VERSION}"
        CACHE STRING "PySide version [full]" FORCE)
else()
    set(BINDING_API_VERSION_FULL "${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}.${BINDING_API_MICRO_VERSION}~${BINDING_API_RELEASE_LEVEL}${BINDING_API_SERIAL}"
        CACHE STRING "PySide version [full]" FORCE)
endif()

string(TIMESTAMP PYSIDE_BUILD_DATE "%Y-%m-%dT%H:%M:%S+00:00" UTC)
if (PYSIDE_BUILD_DATE)
    set(PYSIDE_BUILD_DATE "__build_date__ = '${PYSIDE_BUILD_DATE}'")
endif()

find_package(Git)
if(GIT_FOUND)
    # Check if current source folder is inside a git repo, so that commit information can be
    # queried.
    execute_process(
      COMMAND ${GIT_EXECUTABLE} rev-parse --git-dir
      OUTPUT_VARIABLE PYSIDE_SOURCE_IS_INSIDE_REPO
      ERROR_QUIET
      OUTPUT_STRIP_TRAILING_WHITESPACE)

    if(PYSIDE_SOURCE_IS_INSIDE_REPO)
        # Force git dates to be UTC-based.
        set(ENV{TZ} UTC)
        execute_process(
          COMMAND ${GIT_EXECUTABLE} --no-pager show --date=format-local:%Y-%m-%dT%H:%M:%S+00:00 -s --format=%cd HEAD
          OUTPUT_VARIABLE PYSIDE_BUILD_COMMIT_DATE
          OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(PYSIDE_BUILD_COMMIT_DATE)
            set(PYSIDE_BUILD_COMMIT_DATE "__build_commit_date__ = '${PYSIDE_BUILD_COMMIT_DATE}'")
        endif()
        unset(ENV{TZ})

        execute_process(
          COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
          OUTPUT_VARIABLE PYSIDE_BUILD_COMMIT_HASH
          OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(PYSIDE_BUILD_COMMIT_HASH)
            set(PYSIDE_BUILD_COMMIT_HASH "__build_commit_hash__ = '${PYSIDE_BUILD_COMMIT_HASH}'")
        endif()

        execute_process(
          COMMAND ${GIT_EXECUTABLE} describe HEAD
          OUTPUT_VARIABLE PYSIDE_BUILD_COMMIT_HASH_DESCRIBED
          OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(PYSIDE_BUILD_COMMIT_HASH_DESCRIBED)
            set(PYSIDE_BUILD_COMMIT_HASH_DESCRIBED "__build_commit_hash_described__ = '${PYSIDE_BUILD_COMMIT_HASH_DESCRIBED}'")
        endif()

    endif()
endif()

include(PySideModules)

macro(COLLECT_MODULE_IF_FOUND shortname)
    set(name "Qt5${shortname}")
    find_package(${name})
    set(_name_found "${name}_FOUND")
    set(_name_dir "${name}_DIR")
    get_filename_component(_qt_dir ${QT_INCLUDE_DIR}   DIRECTORY)
    get_filename_component(_module_dir "${${_name_dir}}/../../../" ABSOLUTE)

    # Determine essential/optional/missing
    set(module_state "missing")
    list(FIND ALL_ESSENTIAL_MODULES "${shortname}" essentialIndex)
    if(${essentialIndex} EQUAL -1)
        list(FIND ALL_OPTIONAL_MODULES "${shortname}" optionalIndex)
        if(NOT ${optionalIndex} EQUAL -1)
            set(module_state "optional")
        endif()
    else()
        set(module_state "essential")
    endif()

    if(${${_name_found}} AND ${_qt_dir} STREQUAL ${_module_dir})
        message(STATUS "${module_state} module ${name} found (${ARGN})")
        # record the shortnames for the tests
        list(APPEND all_module_shortnames ${shortname})
    else()
        if("${module_state}" STREQUAL "optional")
            message(STATUS "optional module ${name} skipped")
        elseif("${module_state}" STREQUAL "essential")
            message(STATUS "skipped module ${name} is essential!\n"
                           "   We do not guarantee that all tests are working.")
        else()
            message(FATAL_ERROR "module ${name} MISSING")
        endif()
    endif()
endmacro()

# Set default values for pyside2_global.h
set (Qt5X11Extras_FOUND "0")
set (Qt5Test_FOUND "0")
set (Qt5Widgets_FOUND "0")

# Collect all essential modules.
# note: the order of this list is relevant for dependencies.
# For instance: Qt5Printsupport must come before Qt5WebKitWidgets.
set(ALL_ESSENTIAL_MODULES Core Gui Widgets PrintSupport Sql Network Test Concurrent)
if(UNIX AND NOT APPLE)
    list(APPEND ALL_ESSENTIAL_MODULES X11Extras)
endif()
if(WIN32)
    list(APPEND ALL_ESSENTIAL_MODULES WinExtras)
endif()
if(APPLE)
    list(APPEND ALL_ESSENTIAL_MODULES MacExtras)
endif()

# Collect all optional modules.
set(ALL_OPTIONAL_MODULES Xml XmlPatterns Help Multimedia MultimediaWidgets OpenGL Qml Quick QuickWidgets Script ScriptTools TextToSpeech Charts Svg DataVisualization)
find_package(Qt5UiTools)
if(Qt5UiTools_FOUND)
    list(APPEND ALL_OPTIONAL_MODULES UiTools)
else()
    set(DISABLE_QtUiTools 1)
endif()
if(WIN32)
    list(APPEND ALL_OPTIONAL_MODULES AxContainer)
endif()
list(APPEND ALL_OPTIONAL_MODULES WebChannel WebEngineWidgets WebKit WebKitWidgets WebSockets)

# Modules to be built unless specified by -DMODULES on command line
if (NOT MODULES)
    set(MODULES "${ALL_ESSENTIAL_MODULES};${ALL_OPTIONAL_MODULES}")
endif()

foreach(m ${MODULES})
    COLLECT_MODULE_IF_FOUND(${m})
endforeach()

string(REGEX MATCHALL "[0-9]+" qt_version_helper "${Qt5Core_VERSION}")

list(GET qt_version_helper 0 QT_VERSION_MAJOR)
list(GET qt_version_helper 1 QT_VERSION_MINOR)
unset(qt_version_helper)

set(PYSIDE_QT_VERSION "${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}" CACHE STRING "Qt version used to compile PySide" FORCE)
if(ENABLE_VERSION_SUFFIX)
      set(pyside_SUFFIX "-${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}")
endif()

# no more supported: include(${QT_USE_FILE})

# Configure OS support
set(ENABLE_X11 "0")
set(ENABLE_MAC "0")
set(ENABLE_WIN "0")
set(ENABLE_SIMULATOR "0")

# no more Maemo, no more simulator
if(CMAKE_HOST_APPLE)
    set(ENABLE_MAC "1")
    set(AUTO_OS "mac")
elseif(CMAKE_HOST_WIN32)
    set(ENABLE_WIN "1")
    set(AUTO_OS "win")
elseif(CMAKE_HOST_UNIX)
    set(ENABLE_X11 "1")
    set(AUTO_OS "x11")
else()
    message(FATAL_ERROR "OS not supported")
endif()
message(STATUS "Detected OS: ${AUTO_OS}")

if (WIN32)
    set(PATH_SEP "\;")
else()
    set(PATH_SEP ":")
endif()

# Define supported Qt Version
set(SUPPORTED_QT_VERSION "${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}")

set(BINDING_VERSION ${BINDING_API_VERSION}.${QT_VERSION_MAJOR}.${QT_VERSION_MINOR})

# uninstall target
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake"
               "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
               IMMEDIATE @ONLY)
add_custom_target(uninstall "${CMAKE_COMMAND}"
                  -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")


set(ARCHIVE_NAME pyside-qt${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}+${BINDING_API_VERSION_FULL})
add_custom_target(dist
    COMMAND mkdir -p "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}" &&
            git log > "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}/ChangeLog" &&
            git archive --prefix=${ARCHIVE_NAME}/ HEAD --format=tar --output="${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar" &&
            tar -C "${CMAKE_BINARY_DIR}" --owner=root --group=root -r "${ARCHIVE_NAME}/ChangeLog" -f "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar" &&
            bzip2 -f9 "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar" &&
            echo "Source package created at ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar.bz2.\n"
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

if (NOT PYTHON_SITE_PACKAGES)
    execute_process(
        COMMAND ${SHIBOKEN_PYTHON_INTERPRETER} -c "if True:
            from distutils import sysconfig
            from os.path import sep
            print(sysconfig.get_python_lib(1, 0, prefix='${CMAKE_INSTALL_PREFIX}').replace(sep, '/'))
            "
        OUTPUT_VARIABLE PYTHON_SITE_PACKAGES
        OUTPUT_STRIP_TRAILING_WHITESPACE)
    if (NOT PYTHON_SITE_PACKAGES)
        message(FATAL_ERROR "Could not detect Python module installation directory.")
    elseif (APPLE)
        message(STATUS "!!! The generated bindings will be installed on ${PYTHON_SITE_PACKAGES}, is it right!?")
    endif()
endif()

set(GENERATOR_EXTRA_FLAGS --generator-set=shiboken
                          --enable-parent-ctor-heuristic
                          --enable-pyside-extensions
                          --enable-return-value-heuristic
                          --use-isnull-as-nb_nonzero)
# 2017-04-24 The protected hack can unfortunately not be disabled, because
# Clang does produce linker errors when we disable the hack.
# But the ugly workaround in Python is replaced by a shiboken change.
if(WIN32 OR DEFINED AVOID_PROTECTED_HACK)
    message(STATUS "PySide2 will be generated avoiding the protected hack!")
    set(GENERATOR_EXTRA_FLAGS ${GENERATOR_EXTRA_FLAGS} --avoid-protected-hack)
    add_definitions(-DAVOID_PROTECTED_HACK)
else()
    message(STATUS "PySide will be generated using the protected hack!")
endif()

add_subdirectory(libpyside)
find_package(Qt5Designer)
if(Qt5UiTools_FOUND AND Qt5Designer_FOUND)
    add_subdirectory(plugins)
endif()

# project directories
add_subdirectory(PySide2)
if (BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif ()

find_program(SPHINX_BUILD sphinx-build)
find_program(DOT_EXEC dot)

if (QT_SRC_DIR AND SPHINX_BUILD AND DOT_EXEC)
    add_subdirectory(doc)
else ()
    set(DOCS_TARGET_DISABLED_MESSAGE "apidoc generation targets disabled.")
    if (NOT QT_SRC_DIR)
        message(STATUS "QT_SRC_DIR variable not set, ${DOCS_TARGET_DISABLED_MESSAGE}")
    elseif (NOT SPHINX_BUILD)
        message(STATUS "sphinx-build command not found, ${DOCS_TARGET_DISABLED_MESSAGE}")
    elseif (NOT DOT_EXEC)
        message(STATUS "graphviz not found, ${DOCS_TARGET_DISABLED_MESSAGE}")
    else()
        message(STATUS "Unknown issue occurred, ${DOCS_TARGET_DISABLED_MESSAGE}")
    endif()
endif()
