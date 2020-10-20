cmake_minimum_required(VERSION 3.5)

project(etcd3cppapi CXX)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include (ExternalProject)
include(FindProtobuf)
find_package(Protobuf REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_search_module(GRPC REQUIRED grpc)
pkg_search_module(GRPCPP REQUIRED grpc++)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -g -ggdb -lgrpc++_reflection -std=c++14 ")

# Protobuf
find_package(Protobuf 3.0 REQUIRED)
# find_package(GRPC++ REQUIRED)
file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/proto")
set(SRC_PROTO_FILES 
    "proto/gogo.proto"
    "proto/http.proto"
    "proto/annotations.proto"
    "proto/auth.proto"
    "proto/kv.proto"
    "proto/rpc.proto"
)

list(LENGTH SRC_PROTO_FILES LEN)
math(EXPR PROTO_LIST_NUM "${LEN} - 1")


foreach(PROTO_FILE ${SRC_PROTO_FILES})
    set(CC_EXT ".pb.cc")
    set(H_EXT ".pb.h")
    get_filename_component(FILE_WE ${PROTO_FILE} NAME_WE)
    get_filename_component(FILE_LOC ${PROTO_FILE} DIRECTORY)
    set(CC_FILE "${CMAKE_CURRENT_BINARY_DIR}/${FILE_LOC}/${FILE_WE}${CC_EXT}")
    set(H_FILE "${CMAKE_CURRENT_BINARY_DIR}/${FILE_LOC}/${FILE_WE}${H_EXT}")
    
    add_custom_command (OUTPUT ${CC_FILE} ${H_FILE}
        COMMENT "Generating ${CC_FILE} and ${H_FILE} from ${CMAKE_CURRENT_SOURCE_DIR}/${PROTO_FILE}..."
        COMMAND ${PROTOBUF_PROTOC_EXECUTABLE} -I ${CMAKE_CURRENT_SOURCE_DIR}/proto/ -I ${CMAKE_CURRENT_SOURCE_DIR}/google/protobuf/ --cpp_out=${CMAKE_CURRENT_BINARY_DIR}/proto/ --plugin=`which grpc_cpp_plugin` ${CMAKE_CURRENT_SOURCE_DIR}/${PROTO_FILE}
        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/${PROTO_FILE}
    )
    
    set(CC_FILES ${CC_FILES} ${CC_FILE})
    set(H_FILES ${H_FILES} ${H_FILE})
endforeach()

add_custom_command (OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/proto/rpc.grpc.pb.cc ${CMAKE_CURRENT_BINARY_DIR}/proto/rpc.grpc.pb.h
    COMMENT "Generating  rpc.grpc.pb.cc and rpc.grpc.pb.h..."
    message("Generating  rpc.grpc.pb.cc and rpc.grpc.pb.h...")
    COMMAND  ${PROTOBUF_PROTOC_EXECUTABLE} -I ${CMAKE_CURRENT_SOURCE_DIR}/proto/ --grpc_out=${CMAKE_CURRENT_BINARY_DIR}/proto/ --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ${CMAKE_CURRENT_SOURCE_DIR}/proto/rpc.proto   
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/proto/rpc.proto
)

include_directories(${CMAKE_CURRENT_BINARY_DIR}/proto
                    ${CMAKE_CURRENT_SOURCE_DIR}/src/)

add_library(${PROJECT_NAME} SHARED ${CC_FILES} 
                               ${H_FILES} 
                               ${CMAKE_CURRENT_BINARY_DIR}/proto/rpc.grpc.pb.cc 
                               ${CMAKE_CURRENT_BINARY_DIR}/proto/rpc.grpc.pb.h
                              )

target_link_libraries(${PROJECT_NAME} pthread grpc++_reflection ${GRPC++_LIBRARY} ${PROTOBUF_LIBRARY})



