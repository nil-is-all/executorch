# Copyright (c) Meta Platforms, Inc. and affiliates.
# All rights reserved.
#
# This source code is licensed under the BSD-style license found in the
# LICENSE file in the root directory of this source tree.

# Flatbuffer schema header lib. Please this file formatted by running:
# ~~~
# cmake-format -i CMakeLists.txt
# ~~~

# The include directory that will contain the generated schema headers.
set(_program_schema__include_dir "${CMAKE_BINARY_DIR}/schema/include")
set(_program_schema__output_dir
    "${_program_schema__include_dir}/executorch/schema"
)
# Source root directory for executorch.
if(NOT EXECUTORCH_ROOT)
  set(EXECUTORCH_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/..)
endif()

function(generate_program_schema _schema_srcs _schema_name)
  set(_schema_outputs)
  foreach(fbs_file ${_schema_srcs})
    string(REGEX REPLACE "[.]fbs$" "_generated.h" generated "${fbs_file}")
    list(APPEND _schema_outputs "${_program_schema__output_dir}/${generated}")
  endforeach()

  # Generate the headers from the .fbs files.
  add_custom_command(
    OUTPUT ${_schema_outputs}
    COMMAND flatc --cpp --cpp-std c++11 --gen-mutable --scoped-enums -o
            "${_program_schema__output_dir}" ${_schema_srcs}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS flatc ${_schema_srcs}
    COMMENT "Generating ${_schema_name} headers"
    VERBATIM
  )

  add_library(${_schema_name} INTERFACE ${_schema_outputs})
  set_target_properties(${_schema_name} PROPERTIES LINKER_LANGUAGE CXX)

  # exir lets users set the alignment of tensor data embedded in the flatbuffer,
  # and some users need an alignment larger than the default, which is typically
  # 32.
  target_compile_definitions(
    ${_schema_name}
    INTERFACE FLATBUFFERS_MAX_ALIGNMENT=${EXECUTORCH_FLATBUFFERS_MAX_ALIGNMENT}
  )

  target_include_directories(
    ${_schema_name}
    INTERFACE
      $<BUILD_INTERFACE:${_program_schema__include_dir}>
      $<BUILD_INTERFACE:${EXECUTORCH_ROOT}/third-party/flatbuffers/include>
  )

endfunction()

# Generate common schema
set(common_schema_srcs scalar_type.fbs)
generate_program_schema("${common_schema_srcs}" "common_schema")

# For the other schemas
set(program_schema_srcs program.fbs)
generate_program_schema("${program_schema_srcs}" "program_schema")
add_dependencies(program_schema common_schema)

install(
  TARGETS common_schema program_schema
  EXPORT ExecuTorchTargets
  DESTINATION ${CMAKE_INSTALL_LIBDIR}
  INCLUDES
  DESTINATION ${_common_include_directories}
)
