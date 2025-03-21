# This is the top-level CMakeLists.txt file for the libical project.
#
# Pass the following variables to cmake to control the build:
# (See doc/UsingLibical.txt for more information)
#
# -DWITH_CXX_BINDINGS=[true|false]
#  Build the C++ bindings.
#  Default=true
#
# -DICAL_ERRORS_ARE_FATAL=[true|false]
#  Set to make icalerror_* calls abort instead of internally signaling an error
#  Default=false
#  Notes:
#   Change the behavior at runtime using the icalerror_set_errors_are_fatal() function.
#   Query the behavior at runtime using the icalerror_get_errors_are_fatal() function.
#
# -DICAL_ALLOW_EMPTY_PROPERTIES=[true|false]
#  Set to prevent empty properties from being replaced with X-LIC-ERROR properties.
#  Default=false
#
# -DUSE_BUILTIN_TZDATA=[true|false]
#  Set to build using our own timezone data.
#  Default=false (use the system timezone data on non-Windows systems)
#  ALWAYS true on Windows systems
#
# -DSTATIC_ONLY=[true|false]
#  Set to build static libraries only.
#  Turns-off GObject Introspection.
#  Default=false (build shared and static libs)
#
# -DSHARED_ONLY=[true|false]
#  Set to build shared (dynamic) libraries only.
#  Default=false (build shared and static libs)
#  Takes precedence over STATIC_ONLY
#
# -DENABLE_LTO_BUILD=[true|false]
#  Build a link-time optimized version (requires gcc or clang)
#  Default=false (do not build a link time optimized version)
#
# -DGOBJECT_INTROSPECTION=[true|false]
#  Set to build GObject introspection "typelib" files
#  Requires GObject Introspection development package v0.6.7 or higher
#  Default=false (do not generate the introspection files)
#
# -DICAL_BUILD_DOCS=[true|false]
#  Configure for the API documentation and User Manual.  The 'docs' target will not be available.
#  Default=true
#
# -DICAL_GLIB_VAPI=[true|false]
#  Set to build Vala "vapi" files
#  Requires Vala package
#  Default=false (build the libical-glib interface)
#
# -DICAL_GLIB=[true|false]
#  Set to build libical-glib (GObject) interface
#  Requires glib development package v2.20 or higher
#  Default=true (build the libical-glib interface)
#
# -DENABLE_GTK_DOC=[true|false]
#  Set to build libical-glib developer documentation
#  Requires gtk-doc and ICAL_BUILD_DOCS option to be true.
#  Default=true (build gtk-doc developer documentation)
#
# -DUSE_32BIT_TIME_T=[true|false]
#  Set to build using a 32bit time_t (ignored unless building with MSVC on Windows)
#  Default=false (use the default size of time_t)
#

# ## DO NOT USE IF YOU ARE AN END-USER.  FOR THE DEVELOPERS ONLY!!
## Special CMake Options for Developers
#
# -DABI_DUMPER=[true|false]
#  Build for the abi-dumper (requires gcc)
#  Default=false
#
# -DADDRESS_SANITIZER=[true|false]
#  Build with the address sanitizer (requires gcc or clang)
#  Default=false
#
# -DTHREAD_SANITIZER=[true|false]
#  Build with the thread sanitizer (requires gcc or clang)
#  Default=false
#

cmake_minimum_required(VERSION 3.11.0) #first line, to shutup a cygwin warning
project(libical C) #CXX is optional for the bindings

cmake_policy(SET CMP0003 NEW)
if(POLICY CMP0042)
  cmake_policy(SET CMP0042 NEW)
endif()
if(POLICY CMP0054)
  cmake_policy(SET CMP0054 NEW)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules)

#Include CMake capabilities
include(LibIcalMacrosInternal)
include(FeatureSummary)

# Enable the test harness
enable_testing()

if(WINCE)
  find_package(Wcecompat REQUIRED)
  include_directories(${WCECOMPAT_INCLUDE_DIR})
  set(CMAKE_REQUIRED_INCLUDES ${CMAKE_REQUIRED_INCLUDES} ${WCECOMPAT_INCLUDE_DIR})
  set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES} ${WCECOMPAT_LIBRARIES})
endif()

set(LIBICAL_LIB_MAJOR_VERSION "3")
set(LIBICAL_LIB_MINOR_VERSION "0")
set(LIBICAL_LIB_PATCH_VERSION "95")
set(LIBICAL_LIB_VERSION_STRING
  "${LIBICAL_LIB_MAJOR_VERSION}.${LIBICAL_LIB_MINOR_VERSION}.${LIBICAL_LIB_PATCH_VERSION}"
)

set(PROJECT_VERSION "${LIBICAL_LIB_MAJOR_VERSION}.${LIBICAL_LIB_MINOR_VERSION}")
set(PROJECT_URL "http://libical.github.io/libical/")

# library build types
set(LIBRARY_TYPE SHARED)

libical_option(STATIC_ONLY "Build static libraries only." False)
if(STATIC_ONLY)
  set(LIBRARY_TYPE STATIC)
endif()

libical_option(SHARED_ONLY "Build shared (dynamic) libraries only. Takes precedence over STATIC_ONLY." False)
if(SHARED_ONLY)
  set(STATIC_ONLY False)
  set(LIBRARY_TYPE SHARED)
endif()

if(NOT STATIC_ONLY AND NOT SHARED_ONLY)
  add_feature_info(
    "Build types"
    TRUE
    "build both shared and static libraries"
  )
endif()

if(STATIC_ONLY)
  add_definitions(
    -DLIBICAL_ICAL_STATIC_DEFINE
    -DLIBICAL_ICALSS_STATIC_DEFINE
    -DLIBICAL_VCAL_STATIC_DEFINE
  )
endif()

# must have Perl to create the derived stuff
find_package(Perl REQUIRED)
set_package_properties(Perl PROPERTIES
  TYPE REQUIRED
  PURPOSE "Required by the libical build system."
)

# Ensure finding 64bit libs when using 64-bit compilers
if(CMAKE_CL_64)
  set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS True)
endif()

# libicu is highly recommended for RSCALE support
#  libicu can be found at http://www.icu-project.org
#  RSCALE info at http://tools.ietf.org/html/rfc7529
if(NOT "$ENV{ICU_BASE}" STREQUAL "") #support the old ICU_BASE env
  set(ICU_ROOT $ENV{ICU_BASE})
endif()
find_package(ICU COMPONENTS uc i18n)
set_package_properties(ICU PROPERTIES
  TYPE RECOMMENDED
  PURPOSE "For RSCALE (RFC7529) support"
)
add_feature_info(
  "RSCALE support (RFC7529)"
  ICU_FOUND
  "build in RSCALE support"
)
if(ICU_FOUND)
  set(REQUIRES_PRIVATE_ICU "Requires.private: icu-i18n") #for libical.pc
  set(HAVE_LIBICU 1)
  if(ICU_VERSION VERSION_GREATER 50)
    set(HAVE_ICU_DANGI TRUE)
  else()
    set(HAVE_ICU_DANGI FALSE)
  endif()
  if(ICU_GENCCODE_EXECUTABLE)
    get_filename_component(ICU_EXEC ${ICU_GENCCODE_EXECUTABLE} DIRECTORY)
  elseif(ICU_UCONV_EXECUTABLE)
    get_filename_component(ICU_EXEC ${ICU_UCONV_EXECUTABLE} DIRECTORY)
  elseif(ICU_ICUINFO_EXECUTABLE)
    get_filename_component(ICU_EXEC ${ICU_ICUINFO_EXECUTABLE} DIRECTORY)
  elseif(ICU_ICU-CONFIG_EXECUTABLE)
    get_filename_component(ICU_EXEC ${ICU_ICU-CONFIG_EXECUTABLE} DIRECTORY)
  elseif(ICU_MAKECONV_EXECUTABLE)
    get_filename_component(ICU_EXEC ${ICU_MAKECONV_EXECUTABLE} DIRECTORY)
  else()
    message(FATAL_ERROR
      "Unable find locate the ICU runtime path. Is your ICU installation broken?")
  endif()
  set(ICU_BINARY_DIR ${ICU_EXEC} CACHE DOC STRING "Runtime binaries directory for the ICU library")
endif()

# compile in Berkeley DB support
find_package(BDB)
set_package_properties(BDB PROPERTIES
  TYPE OPTIONAL
  PURPOSE "For Berkeley DB storage support"
)
add_feature_info(
  "Berkeley DB storage support"
  BDB_FOUND
  "build in support for Berkeley DB storage"
)
if(BDB_FOUND)
  set(HAVE_BDB True)
endif()

# MSVC specific definitions
if(WIN32)
  if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE -D_CRT_NONSTDC_NO_DEPRECATE -DYY_NO_UNISTD_H)
    libical_option(USE_32BIT_TIME_T "Build using a 32bit time_t (ignored unless building with MSVC on Windows)." False)
    if(USE_32BIT_TIME_T)
      add_definitions(-D_USE_32BIT_TIME_T)
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4290") #C++ exception specification ignored
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4068") #unknown pragma
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4028") #formal parameter differs
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4068") #unknown pragma
  endif()
  add_definitions(-DBIG_ENDIAN=0 -DLITTLE_ENDIAN=1 -DBYTE_ORDER=BIG_ENDIAN)
endif()

# Use GNUInstallDirs

include(GNUInstallDirs)
set(LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR} CACHE STRING "Library directory name" FORCE)
set(INCLUDE_INSTALL_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE STRING "Include directory name" FORCE)
set(SHARE_INSTALL_DIR ${CMAKE_INSTALL_DATAROOTDIR} CACHE STRING "Share directory name")

# set the output paths
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)
if(WIN32)
  set(LIBRARY_OUTPUT_PATH ${EXECUTABLE_OUTPUT_PATH})
  set(LIB_INSTALL_DIR lib)
  set(BIN_INSTALL_DIR bin)
else()
  set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/lib)
endif()

if(APPLE)
  set(CMAKE_INSTALL_NAME_DIR ${LIB_DESTINATION})
endif()

libical_option(ICAL_ERRORS_ARE_FATAL "icalerror_* calls will abort instead of internally signaling an error." False)
if(ICAL_ERRORS_ARE_FATAL)
  set(ICAL_ERRORS_ARE_FATAL 1)
else()
  set(ICAL_ERRORS_ARE_FATAL 0)
endif()

libical_option(ICAL_ALLOW_EMPTY_PROPERTIES "Prevents empty properties from being replaced with X-LIC-ERROR properties." False)
if(ICAL_ALLOW_EMPTY_PROPERTIES)
  set(ICAL_ALLOW_EMPTY_PROPERTIES 1)
else()
  set(ICAL_ALLOW_EMPTY_PROPERTIES 0)
endif()

if(WIN32 OR WINCE)
  set(DEF_USE_BUILTIN_TZDATA False)
else()
  set(DEF_USE_BUILTIN_TZDATA True)
endif()
libical_option(USE_BUILTIN_TZDATA "Build using our builtin timezone data, else use the system timezone data. ALWAYS true on Windows." ${DEF_USE_BUILTIN_TZDATA})
if(USE_BUILTIN_TZDATA)
  set(USE_BUILTIN_TZDATA 1)
else()
  set(USE_BUILTIN_TZDATA 0)
endif()
if(WIN32 OR WINCE)
  #Always use builtin tzdata on Windows systems.
  if(NOT USE_BUILTIN_TZDATA)
    message(STATUS
      "Currently unable to use system tzdata on Windows. Falling back to builtin tzdata"
    )
    set(USE_BUILTIN_TZDATA 1)
  endif()
endif()

include(ConfigureChecks.cmake)
add_definitions(-DHAVE_CONFIG_H)
configure_file(config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)

set(INSTALL_TARGETS_DEFAULT_ARGS
  RUNTIME DESTINATION ${BIN_INSTALL_DIR}
  LIBRARY DESTINATION ${LIB_INSTALL_DIR}
  ARCHIVE DESTINATION ${LIB_INSTALL_DIR}
)

set(MIN_GOBJECT_INTROSPECTION "0.6.7")
libical_option(GOBJECT_INTROSPECTION "Build GObject introspection \"typelib\" files. Requires GObject Introspection development package ${MIN_GOBJECT_INTROSPECTION} or higher." False)
if(GOBJECT_INTROSPECTION)
  find_package(GObjectIntrospection ${MIN_GOBJECT_INTROSPECTION})
  set_package_properties(GObjectIntrospection PROPERTIES
    TYPE REQUIRED
    URL "https://wiki.gnome.org/Projects/GObjectIntrospection"
    PURPOSE "Required in order to build the GObject introspection \"typelib\" files."
  )
  if(GObjectIntrospection_FOUND)
    set(HAVE_INTROSPECTION TRUE)
    if(STATIC_ONLY)
      message(FATAL_ERROR
        "You are attempting to build with GObject Introspection enabled, "
        "however that option is not supported when building static libraries only. "
        "Please disable the static only option to cmake (-DSTATIC_ONLY=False) "
        "if you really want to build with GObject Introspection. Alternatively, "
        "you can disable GObject Introspection (by passing -DGOBJECT_INTROSPECTION=False to cmake)")
    endif()
  endif()
endif()

libical_option(ICAL_GLIB_VAPI "Build Vala \"vapi\" files." False)
if(ICAL_GLIB_VAPI)
  if(NOT GOBJECT_INTROSPECTION)
    message(FATAL_ERROR
      "You requested to build the Vala vapi but have not enabled the GObject Introspection. "
      "Please try again also passing -DGOBJECT_INTROSPECTION=True to cmake"
    )
  endif()

  find_program(VALAC valac DOC "the Vala compiler")
  if(NOT VALAC)
    message(FATAL_ERROR
      "valac, the Vala compiler was not found. "
      "Install it or disable Vala bindings with -DICAL_GLIB_VAPI=False"
    )
  endif()

  find_program(VAPIGEN vapigen DOC "tool to generate the Vala API")
  if(NOT VAPIGEN)
    message(FATAL_ERROR
      "vapigen, the tool for generating the Vala API was not found. "
      "Install it or disable Vala bindings with -DICAL_GLIB_VAPI=False"
    )
  endif()
endif()

set(MIN_GLIB "2.32")
set(MIN_LIBXML "2.7.3")
libical_option(ICAL_GLIB "Build libical-glib interface. Requires glib ${MIN_GLIB} and libxml ${MIN_LIBXML} development packages or higher." True)
if(ICAL_GLIB)
  find_package(GLib ${MIN_GLIB})
  set_package_properties(GLib PROPERTIES
    TYPE OPTIONAL
    PURPOSE "For the optional libical-glib interface"
  )
  find_package(LibXML ${MIN_LIBXML})
  set_package_properties(LibXML PROPERTIES
    TYPE OPTIONAL
    DESCRIPTION "a library providing XML and HTML support"
    URL "http://xmlsoft.org"
    PURPOSE "For the optional libical-glib interface"
  )
  if(GLIB_FOUND AND LIBXML_FOUND)
    set(HAVE_GLIB TRUE)
  elseif(GLIB_FOUND)
    message(FATAL_ERROR
      "You requested to build libical-glib, but the necessary development package "
      "is missing or too low a version (libxml ${MIN_LIBXML} or higher is required)"
    )
  elseif(LIBXML_FOUND)
    message(FATAL_ERROR
      "You requested to build libical-glib, but the necessary development package "
      "is missing or too low a version (glib ${MIN_GLIB} or higher is required)"
    )
  else()
    message(FATAL_ERROR
      "You requested to build libical-glib, but the necessary development packages "
      "are missing or too low a version "
      "(glib ${MIN_GLIB} and libxml ${MIN_LIBXML} or higher are required)"
      "Alternatively, disable the libical-glib build (by passing -DICAL_GLIB=False to cmake)"
    )
  endif()
endif()

#
# Compiler settings
#
if(CMAKE_COMPILER_IS_GNUCC OR "${CMAKE_C_COMPILER_ID}" MATCHES "Clang")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden -Wno-deprecated -Wall -Wno-unknown-pragmas -Wextra -Winit-self -Wmissing-include-dirs -Wunused -Wno-div-by-zero -Wundef -Wpointer-arith -Wtype-limits -Wwrite-strings -Werror=return-type")
  libical_add_cflag(-Wunused-but-set-variable UNUSED_BUT_SET)
  libical_add_cflag(-Wlogical-op LOGICAL_OP)
  libical_add_cflag(-Wsizeof-pointer-memaccess POINTER_MEMACCESS)
  libical_add_cflag(-Wformat-security FORMAT_SECURITY)
  libical_add_cflag(-Wredundant-decls REDUNDANT_DECLS)
  if(CMAKE_SYSTEM_NAME MATCHES Linux OR CMAKE_SYSTEM_NAME STREQUAL GNU)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_XOPEN_SOURCE=500 -D_DEFAULT_SOURCE -D_GNU_SOURCE")
  endif()
endif()
if("${CMAKE_C_COMPILER_ID}" MATCHES "Clang")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Qunused-arguments")
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden -Weffc++ -Wno-deprecated -Wall -Wextra -Woverloaded-virtual -Winit-self -Wmissing-include-dirs -Wunused -Wno-div-by-zero -Wundef -Wpointer-arith -Wtype-limits -Wwrite-strings -Werror=return-type")
  libical_add_cxxflag(-Wunused-but-set-variable UNUSED_BUT_SET)
  libical_add_cxxflag(-Wlogical-op LOGICAL_OP)
  libical_add_cxxflag(-Wsizeof-pointer-memaccess POINTER_MEMACCESS)
  libical_add_cxxflag(-Wreorder REORDER)
  libical_add_cxxflag(-Wformat-security FORMAT_SECURITY)
  libical_add_cxxflag(-Wredundant-decls REDUNDANT_DECLS)
  if(CMAKE_SYSTEM_NAME MATCHES Linux OR CMAKE_SYSTEM_NAME STREQUAL GNU)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_XOPEN_SOURCE=500 -D_DEFAULT_SOURCE -D_GNU_SOURCE")
  endif()
endif()
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Qunused-arguments")
endif()

########################################################

libical_option(WITH_CXX_BINDINGS "Build the C++ bindings." True)
if(WITH_CXX_BINDINGS)
  enable_language(CXX)
  if(CMAKE_CXX_COMPILER)
    add_definitions(-DWITH_CXX_BINDINGS)
  else()
    message(STATUS
      "Warning: Building the C++ bindings is not possible since a C++ compiler could not be found. "
      "Turning-off C++ bindings"
    )
    set(WITH_CXX_BINDINGS False)
  endif()
endif()

#some test programs need to know if we are using 32-bit time
if(SIZEOF_TIME_T EQUAL 4)
  set(USE_32BIT_TIME_T TRUE)
endif()

################ Developer Options #####################
libical_option(ABI_DUMPER "Build for abi-dumper (developer-only option)." False)
if(ABI_DUMPER)
  if(CMAKE_COMPILER_IS_GNUCC)
    set(CMAKE_C_FLAGS "-g -Og")
    set(CMAKE_CXX_FLAGS "-g -Og")
  else()
    message(FATAL_ERROR
      "You are trying to build for the abi-dumper using a non-GCC compiler.")
  endif()
endif()

libical_option(ADDRESS_SANITIZER "Build with the address sanitizer (developer-only option)." False)
if(ADDRESS_SANITIZER)
  if(CMAKE_COMPILER_IS_GNUCC OR "${CMAKE_C_COMPILER_ID}" MATCHES "Clang")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address -g")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -g")
  else()
    message(FATAL_ERROR
      "You are trying to build with the address sanitizer using a non-GCC or Clang compiler.")
  endif()
  if(GOBJECT_INTROSPECTION)
    message(STATUS
      "Warning: Building the address sanitizer with GObject Introspection is is not supported. "
      "Turning-off GObject Introspection"
    )
    set(HAVE_INTROSPECTION False)
  endif()
  if(ICAL_GLIB)
    message(STATUS
      "Warning: Building the address sanitizer with the GObject interface is not supported. "
      "Turning-off the GObject interface"
    )
    set(ICAL_GLIB False)
  endif()
endif()

libical_option(THREAD_SANITIZER "Build with the thread sanitizer (developer-only option)." False)
if(THREAD_SANITIZER)
  if(CMAKE_COMPILER_IS_GNUCC OR "${CMAKE_C_COMPILER_ID}" MATCHES "Clang")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=thread -O1 -g")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=thread -O1 -g")
  else()
    message(FATAL_ERROR
      "You are trying to build with the thread sanitizer using a non-GCC or Clang compiler.")
  endif()
  if(GOBJECT_INTROSPECTION)
    message(STATUS
      "Warning: Building the thread sanitizer with GObject Introspection is is not supported. "
      "Turning-off GObject Introspection"
    )
    set(HAVE_INTROSPECTION False)
  endif()
  if(ICAL_GLIB)
    message(STATUS
      "Warning: Building the thread sanitizer with the GObject interface is not supported. "
      "Turning-off the GObject interface"
    )
    set(ICAL_GLIB False)
  endif()
endif()

libical_option(ENABLE_LTO_BUILD "Build a link-time optimized version." False)
if(ENABLE_LTO_BUILD)
  if(CMAKE_COMPILER_IS_GNUCC)
    libical_add_cflag(-flto LTO)
    if(C_SUPPORTS_LTO)
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wlto-type-mismatch -fuse-linker-plugin")
      set(CMAKE_AR "gcc-ar")
      set(CMAKE_RANLIB "gcc-ranlib")
    else()
      message(FATAL_ERROR
        "Your C compiler ${CMAKE_C_COMPILER_ID} does not support the LTO building.")
    endif()
    if(WITH_CXX_BINDINGS AND CMAKE_COMPILER_IS_GNUCXX)
      include(CheckCXXCompilerFlag)
      libical_add_cxxflag(-flto LTO)
      if(CXX_SUPPORTS_LTO)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wlto-type-mismatch -fuse-linker-plugin")
      else()
        message(FATAL_ERROR
          "Your C++ compiler ${CMAKE_CXX_COMPILER_ID} does not support LTO building.")
      endif()
    endif()
  else()
    message(FATAL_ERROR
      "Build link-time optimization using a non-GCC compiler is currently not supported.")
  endif()
endif()

################# build subdirs ########################

add_subdirectory(design-data)
add_subdirectory(scripts)
add_subdirectory(test-data)
add_subdirectory(src)
add_subdirectory(examples)
if(USE_BUILTIN_TZDATA)
  # use our zoneinfo if cmake is passed -DUSE_BUILTIN_TZDATA
  add_subdirectory(zoneinfo)
endif()

libical_option(ICAL_BUILD_DOCS "Build documentation" True)
if(ICAL_BUILD_DOCS)
  add_subdirectory(doc) # needs to go last, for the build source files
endif()

########### create and install pkg-config file #########

set(VERSION "${LIBICAL_LIB_VERSION_STRING}")
set(prefix "${CMAKE_INSTALL_PREFIX}")
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/lib")
set(includedir "\${prefix}/include")
set(PTHREAD_LIBS "${CMAKE_THREAD_LIBS_INIT}")

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/libical.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/libical.pc
  @ONLY
)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libical.pc DESTINATION ${LIB_INSTALL_DIR}/pkgconfig)

########### Create and install the CMake Config files ##########
include(CMakePackageConfigHelpers)

configure_package_config_file(
  LibIcalConfig.cmake.in ${libical_BINARY_DIR}/LibIcalConfig.cmake
  INSTALL_DESTINATION ${LIB_INSTALL_DIR}/cmake/LibIcal
  PATH_VARS LIB_INSTALL_DIR INCLUDE_INSTALL_DIR
)

# Create a version file
write_basic_package_version_file(
  ${libical_BINARY_DIR}/LibIcalConfigVersion.cmake
  VERSION ${LIBICAL_LIB_VERSION_STRING}
  COMPATIBILITY SameMajorVersion
)

install(
  FILES ${libical_BINARY_DIR}/LibIcalConfigVersion.cmake ${libical_BINARY_DIR}/LibIcalConfig.cmake
  DESTINATION ${LIB_INSTALL_DIR}/cmake/LibIcal
)

install(
  EXPORT icalTargets
  DESTINATION ${LIB_INSTALL_DIR}/cmake/LibIcal
  FILE LibIcalTargets.cmake
)

########## By popular demand, add an uninstall target ##########

if(NOT TARGET uninstall)
  configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/uninstall.cmake"
    IMMEDIATE
    @ONLY
  )

  add_custom_target(uninstall
    COMMAND "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/uninstall.cmake"
  )
endif()

feature_summary(WHAT ALL INCLUDE_QUIET_PACKAGES FATAL_ON_MISSING_REQUIRED_PACKAGES)
