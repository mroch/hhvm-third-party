# Phase 1 - see if they have a built-in version of ocamlc which is 'good enough'
find_program(OCAML NAMES ocaml ocaml)
find_program(OCAMLC NAMES ocamlc.opt ocamlc)
find_program(OCAMLOPT NAMES ocamlopt.opt ocamlopt)
find_program(OCAMLBUILD NAMES ocamlbuild.native ocamlbuild)

set(OCAMLC_FOUND FALSE)

if (OCAMLC)
  # We found a local version of ocamlc - check the version.
  message(STATUS "Found ocamlc: ${OCAMLC}")
  execute_process(COMMAND "${OCAMLC}" -version
    OUTPUT_VARIABLE OCAMLC_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE)

  message(STATUS "Got ocaml version ${OCAMLC_VERSION}")
  if ("${OCAMLC_VERSION}" VERSION_LESS "4.03")
    message(STATUS "System version of ocaml is too old")
  else()
    set(OCAMLC_FOUND TRUE)
  endif()
endif()

if (NOT OCAMLC_FOUND)
  # Either we couldn't find a reasonable version of ocamlc or it was too old.
  message(STATUS "Building ocaml from third-party")

  set(OCAML_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/build")
  set(OCAML "${OCAML_PREFIX}/bin/ocaml")
  set(OCAMLC "${OCAML_PREFIX}/bin/ocamlc.opt")
  set(OCAMLOPT "${OCAML_PREFIX}/bin/ocamlopt.opt")
  set(OCAMLBUILD "${OCAML_PREFIX}/bin/ocamlbuild.native")

  # TODO: what about dependencies so we rebuild ocaml when it changes?
  # NOTE: The weirdness building world.opt twice is because (at least on arm64)
  # ocaml (4.03) seems to have parallel build dependency problems.
  add_custom_command(
    OUTPUT ${OCAML} ${OCAMLC} ${OCAMLOPT}
    COMMAND ./configure -prefix "${OCAML_PREFIX}" -no-graph
    COMMAND \$\(MAKE\) -k world.opt || true
    COMMAND \$\(MAKE\) -j1 world.opt
    COMMAND \$\(MAKE\) install
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/src"
    COMMENT "Compiling ocaml")

  add_custom_command(
    OUTPUT ${OCAMLBUILD}
    # Hack up the path before building ocamlbuild.
    # (OCamlBuild doesn't understand how to override its build tools properly -
    # they call "ocamlc -where" directly from configure.make)
    # This would be better as 'cmake -E env' but that's only available in cmake 3.1+
    COMMAND /bin/bash -c 'PATH="${OCAML_PREFIX}/bin:$ENV{PATH}" \$\(MAKE\) -f configure.make
        OCAML_NATIVE=true
        OCAMLBUILD_BINDIR=${OCAML_PREFIX}/bin
        OCAMLBUILD_LIBDIR=${OCAML_PREFIX}/lib'
    COMMAND /bin/bash -c 'PATH="${OCAML_PREFIX}/bin:$ENV{PATH}" \$\(MAKE\)'
    COMMAND \$\(MAKE\) install CHECK_IF_PREINSTALLED=false
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/ocamlbuild"
    DEPENDS ${OCAMLC}
    COMMENT "Compiling ocamlbuild")

  set(OCAMLC_FOUND TRUE)
endif()

add_custom_target(ocaml
  DEPENDS ${OCAMLC} ${OCAMLOPT} ${OCAMLBUILD})

set(OCAML_EXECUTABLE ${OCAML} CACHE FILEPATH "path to ocaml" FORCE)
mark_as_advanced(OCAML_EXECUTABLE)
set(OCAMLC_EXECUTABLE ${OCAMLC} CACHE FILEPATH "path to ocamlc" FORCE)
mark_as_advanced(OCAMLC_EXECUTABLE)
set(OCAMLOPT_EXECUTABLE ${OCAMLOPT} CACHE FILEPATH "path to ocamlopt" FORCE)
mark_as_advanced(OCAMLOPT_EXECUTABLE)
set(OCAMLBUILD_EXECUTABLE ${OCAMLBUILD} CACHE FILEPATH "path to ocamlbuild" FORCE)
mark_as_advanced(OCAMLBUILD_EXECUTABLE)
