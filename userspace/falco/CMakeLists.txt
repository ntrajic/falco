#
# Copyright (C) 2019 The Falco Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
# the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
# specific language governing permissions and limitations under the License.
#

configure_file("${SYSDIG_SOURCE_DIR}/userspace/sysdig/config_sysdig.h.in" config_sysdig.h)

add_custom_command(
  OUTPUT
    ${CMAKE_CURRENT_BINARY_DIR}/output.grpc.pb.cc ${CMAKE_CURRENT_BINARY_DIR}/output.grpc.pb.h
    ${CMAKE_CURRENT_BINARY_DIR}/output.pb.cc ${CMAKE_CURRENT_BINARY_DIR}/output.pb.h
    ${CMAKE_CURRENT_BINARY_DIR}/schema.pb.cc ${CMAKE_CURRENT_BINARY_DIR}/schema.pb.h
  COMMENT "Generate gRPC code"
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/output.proto
  COMMAND ${PROTOC} -I ${CMAKE_CURRENT_SOURCE_DIR} --cpp_out=. ${CMAKE_CURRENT_SOURCE_DIR}/output.proto
          ${CMAKE_CURRENT_SOURCE_DIR}/schema.proto
  COMMAND ${PROTOC} -I ${CMAKE_CURRENT_SOURCE_DIR} --grpc_out=. --plugin=protoc-gen-grpc=${GRPC_CPP_PLUGIN}
          ${CMAKE_CURRENT_SOURCE_DIR}/output.proto
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})

add_executable(
  falco
  configuration.cpp
  logger.cpp
  falco_outputs.cpp
  event_drops.cpp
  statsfilewriter.cpp
  falco.cpp
  "${SYSDIG_SOURCE_DIR}/userspace/sysdig/fields_info.cpp"
  webserver.cpp
  grpc_context.cpp
  grpc_server_impl.cpp
  grpc_server.cpp
  utils.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/output.grpc.pb.cc
  ${CMAKE_CURRENT_BINARY_DIR}/output.pb.cc
  ${CMAKE_CURRENT_BINARY_DIR}/schema.pb.cc)

add_dependencies(falco civetweb)

if(USE_BUNDLED_DEPS)
add_dependencies(falco yamlcpp)
endif()

target_include_directories(
  falco
  PUBLIC
    "${SYSDIG_SOURCE_DIR}/userspace/sysdig"
    "${PROJECT_SOURCE_DIR}/userspace/engine"
    "${PROJECT_BINARY_DIR}/userspace/falco"
    "${PROJECT_BINARY_DIR}/driver/src"
    "${YAMLCPP_INCLUDE_DIR}"
    "${CIVETWEB_INCLUDE_DIR}"
    "${GRPC_INCLUDE}"
    "${PROTOBUF_INCLUDE}"
    "${CMAKE_CURRENT_BINARY_DIR}"
    "${DRAIOS_DEPENDENCIES_DIR}/yaml-${DRAIOS_YAML_VERSION}/target/include")

target_link_libraries(
  falco
  falco_engine
  sinsp
  "${GRPCPP_LIB}"
  "${GRPC_LIB}"
  "${PROTOBUF_LIB}"
  "${LIBYAML_LIB}"
  "${YAMLCPP_LIB}"
  "${CIVETWEB_LIB}")

configure_file(config_falco.h.in config_falco.h)

add_custom_command(
  TARGET falco
  COMMAND bash ${CMAKE_CURRENT_SOURCE_DIR}/verify_engine_fields.sh ${CMAKE_SOURCE_DIR} ${OPENSSL_BINARY}
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  COMMENT "Comparing engine fields checksum in falco_engine.h to actual fields")

# add_custom_target(verify_engine_fields DEPENDS verify_engine_fields.sh falco_engine.h)

# add_dependencies(verify_engine_fields falco)

install(TARGETS falco DESTINATION ${FALCO_BIN_DIR})
install(
  DIRECTORY lua
  DESTINATION ${FALCO_SHARE_DIR}
  FILES_MATCHING
  PATTERN *.lua)
