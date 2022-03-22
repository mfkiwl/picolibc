# Copyright © 2021 Amazon.com, Inc. or its affiliates.
# SPDX-License-Identifier: Apache-2.0

if(CONFIG_PICOLIBC)

  set(ENV{PICOLIBC_FLAGS})

  function(picolibc_option)
    foreach(arg ${ARGV})
      if(NOT "$ENV{PICOLIBC_FLAGS}" STREQUAL "")
	set(ENV{PICOLIBC_FLAGS} "$ENV{PICOLIBC_FLAGS};")
      endif()
      set(ENV{PICOLIBC_FLAGS} "$ENV{PICOLIBC_FLAGS}${arg}")
    endforeach()
  endfunction()

  function(picolibc_option_ifdef feature_toggle)
    if(${${feature_toggle}})
      picolibc_option(${ARGN})
    endif()
  endfunction()

  function(picolibc_option_ifndef feature_toggle)
    if(${${feature_toggle}})
    else()
      picolibc_option(${ARGN})
    endif()
  endfunction()

  function(picolibc_option_bool feature_toggle key)
    if(${${feature_toggle}})
      picolibc_option("-D${key}=true")
    else()
      picolibc_option("-D${key}=false")
    endif()
  endfunction()

  picolibc_option_ifdef(CONFIG_PICOLIBC_SIZE_OPTIMIZATIONS "--buildtype" "minsize")
  picolibc_option_ifdef(CONFIG_PICOLIBC_SPEED_OPTIMIZATIONS "--buildtype" "release")
  picolibc_option_ifdef(CONFIG_PICOLIBC_DEBUG_OPTIMIZATIONS "--buildtype" "debugoptimized")
  picolibc_option_ifdef(CONFIG_PICOLIBC_NO_OPTIMIZATIONS "--buildtype" "plain")
  picolibc_option_bool(CONFIG_PICOLIBC_FAST_STRCMP "fast-strcmp")
  picolibc_option_bool(CONFIG_PICOLIBC_IO_C99_FORMATS "io-c99-formats")
  picolibc_option_bool(CONFIG_PICOLIBC_IO_LONG_LONG "io-long-long")
  picolibc_option_bool(CONFIG_PICOLIBC_IO_POS_ARGS "io-pos-args")
  picolibc_option_ifdef(CONFIG_PICOLIBC_IO_FLOAT "-Dformat-default=double")
  picolibc_option_ifndef(CONFIG_PICOLIBC_IO_FLOAT "-Dformat-default=integer")
  picolibc_option_bool(CONFIG_PICOLIBC_IO_FLOAT_EXACT "io-float-exact")
  picolibc_option_bool(CONFIG_PICOLIBC_LOCALE_INFO "newlib-locale-info")
  picolibc_option_bool(CONFIG_PICOLIBC_LOCALE_EXTENDED_INFO "newlib-locale-info-extended")
  picolibc_option_bool(CONFIG_PICOLIBC_MULTIBYTE "newlib-mb")
  picolibc_option_bool(CONFIG_PICOLIBC_PICOEXIT "picoexit")
  picolibc_option_bool(CONFIG_PICOLIBC_MULTITHREAD "newlib-multithread")
  picolibc_option_bool(CONFIG_THREAD_LOCAL_STORAGE "thread-local-storage")
  picolibc_option_bool(CONFIG_PICOLIBC_GLOBAL_ERRNO "newlib-global-errno")

  set(PICOLIBC_DIR ${ZEPHYR_CURRENT_MODULE_DIR})

  list(JOIN TOOLCHAIN_C_FLAGS " " PICOLIBC_C_FLAGS)
  set(PICOLIBC_FLAGS "$ENV{PICOLIBC_FLAGS}")
 
  set(ENV{PICOLIBC_FLAGS})

  if(CONFIG_BIG_ENDIAN)
    set(PICOLIBC_ENDIAN big)
  else()
    set(PICOLIBC_ENDIAN little)
  endif()

  set(PICOLIBC_INSTALL_DIR "${CMAKE_BINARY_DIR}/picolibc")

  include(ExternalProject)

  ExternalProject_Add(PicolibcBuild
    SOURCE_DIR "${PICOLIBC_DIR}"
    CONFIGURE_COMMAND ${PICOLIBC_DIR}/zephyr/configure
    "CONFIG_ARCH=${CONFIG_ARCH}"
    "CMAKE_BINARY_DIR=${CMAKE_BINARY_DIR}"
    "CMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
    "CMAKE_OBJCOPY=${CMAKE_OBJCOPY}"
    "CMAKE_AS=${CMAKE_AS}"
    "CMAKE_AR=${CMAKE_AR}"
    "CMAKE_RANLIB=${CMAKE_RANLIB}"
    "CMAKE_NM=${CMAKE_NM}"
    "CMAKE_STRIP=${CMAKE_STRIP}"
    "TOOLCHAIN_C_FLAGS=${PICOLIBC_C_FLAGS}"
    "CROSS_COMPILE=${CROSS_COMPILE}"
    "CROSS_COMPILE_TARGET=${CROSS_COMPILE_TARGET}"
    "PICOLIBC_DIR=${PICOLIBC_DIR}"
    "PICOLIBC_ENDIAN=${PICOLIBC_ENDIAN}"
    "PICOLIBC_INSTALL_DIR=${PICOLIBC_INSTALL_DIR}" ${PICOLIBC_FLAGS}
    BUILD_COMMAND ninja -C build
    INSTALL_COMMAND meson install -C build --quiet
    TEST_COMMAND ""
    )

  set_target_properties(PicolibcBuild
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PICOLIBC_INSTALL_DIR}")

  # Zephyr normally uses -ffreestanding, which with current GNU toolchains
  # means that the flag macros used by picolibc <inttypes.h> to signal
  # support for PRI.64 macros are not present.  To make them available we
  # need to hook into the include path before the system files and
  # explicitly include the picolibc header that provides those macros.
  zephyr_include_directories(include)

  # define __LINUX_ERRNO_EXTENSIONS__ so we get errno defines like -ESHUTDOWN
  # used by the network stack
  zephyr_compile_definitions(__LINUX_ERRNO_EXTENSIONS__)

  zephyr_link_libraries(
    c
    gcc # Lib C depends on libgcc.
    )

  set(Picolibc_SPECS "${PICOLIBC_INSTALL_DIR}/picolibc.specs")

  zephyr_link_libraries(
    --specs=${Picolibc_SPECS}
    -T/dev/null
    )

  zephyr_compile_options(
    --specs=${Picolibc_SPECS}
    -D_GNU_SOURCE
    )

endif()
