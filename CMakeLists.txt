cmake_minimum_required(VERSION 3.22)
project(ros2_grpc_server)

# C++17 is required by gRPC
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)

find_program(GRPC_CPP_PLUGIN_EXECUTABLE grpc_cpp_plugin REQUIRED)

find_package(gRPC CONFIG QUIET)
if(NOT gRPC_FOUND)
    message(STATUS "gRPC CONFIG not found, using manual library detection")
    
    # Find gRPC libraries manually
    find_library(GRPC_LIBRARY NAMES grpc++ REQUIRED)
    find_library(GRPC_UNSECURE_LIBRARY NAMES grpc REQUIRED)
    
    message(STATUS "Found gRPC library: ${GRPC_LIBRARY}")
    message(STATUS "Found gRPC unsecure library: ${GRPC_UNSECURE_LIBRARY}")
    
    # Create imported target for gRPC::grpc++
    if(NOT TARGET gRPC::grpc++)
        add_library(gRPC::grpc++ UNKNOWN IMPORTED)
        set_target_properties(gRPC::grpc++ PROPERTIES
            IMPORTED_LOCATION "${GRPC_LIBRARY}"
            INTERFACE_LINK_LIBRARIES "${GRPC_UNSECURE_LIBRARY}"
        )
    message(STATUS "Setup the gRPC::grpc++ library")
    endif()
else()
    message(STATUS "Found gRPC via CONFIG")
endif()
find_package(Protobuf REQUIRED)

include_directories(
    ${PROJECT_SOURCE_DIR}/include
)

# ---------- Protobuf & gRPC generation
# Set the directory where .proto files are located
set(PROTO_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/proto")
file(GLOB PROTO_SRC "${PROTO_SRC_DIR}/*.proto")

# Specify output directory for header, source, and script files
set(GENERATED_PROTO_CPP_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include/msgs")
set(GENERATED_PROTO_PY_DIR "${CMAKE_CURRENT_SOURCE_DIR}/script/msgs")

# Make folders if they dont exist
file(MAKE_DIRECTORY ${GENERATED_PROTO_CPP_DIR} ${GENERATED_PROTO_PY_DIR})

# Generate header and source files directory
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${PROTO_SRC})
# These will be in CMAKE_CURRENT_BINARY_DIR by default
include_directories(${GENERATED_PROTO_CPP_DIR} ${CMAKE_CURRENT_BINARY_DIR})

# gRPC C++/Python stubs
set(GRPC_SRCS)
set(GRPC_HDRS)
set(GRPC_PYS)

# Generated C++ and Python code from .proto files
foreach(proto_file ${PROTO_SRC})
    get_filename_component(proto_name ${proto_file} NAME_WE)
    message(STATUS "Processing Proto definition: ${proto_name}")

    # Add custom command to generate C++ code for each .proto file
    set(pb_cc   ${GENERATED_PROTO_CPP_DIR}/${proto_name}.pb.cc)
    set(pb_h    ${GENERATED_PROTO_CPP_DIR}/${proto_name}.pb.h)
    set(grpc_cc ${GENERATED_PROTO_CPP_DIR}/${proto_name}.grpc.pb.cc)
    set(grpc_h  ${GENERATED_PROTO_CPP_DIR}/${proto_name}.grpc.pb.h)

    add_custom_command(
        OUTPUT ${pb_cc} ${pb_h} ${grpc_cc} ${grpc_h}
        COMMAND ${Protobuf_PROTOC_EXECUTABLE}
        ARGS --proto_path=${PROTO_SRC_DIR}
            --grpc_out=${GENERATED_PROTO_CPP_DIR}
            --cpp_out=${GENERATED_PROTO_CPP_DIR}
            --plugin=protoc-gen-grpc=${GRPC_CPP_PLUGIN_EXECUTABLE}
            ${proto_file}
        DEPENDS ${proto_file} ${GRPC_CPP_PLUGIN_EXECUTABLE}
        COMMENT "Compiling ${proto_file} to C++ header & source"
    )
    list(APPEND GRPC_SRCS ${grpc_cc})
    list(APPEND GRPC_HDRS ${grpc_h})

    # Add custom command to generate Python code for each .proto file
    set(grpc_pb2_py  ${GENERATED_PROTO_PY_DIR}/${proto_name}_pb2.py)
    set(grpc_py      ${GENERATED_PROTO_PY_DIR}/${proto_name}_pb2_grpc.py)

    add_custom_command(
        OUTPUT ${grpc_pb2_py} ${grpc_py}
        COMMAND python3 -m grpc_tools.protoc
        ARGS --proto_path=${PROTO_SRC_DIR}
            --python_out=${GENERATED_PROTO_PY_DIR}
            --grpc_python_out=${GENERATED_PROTO_PY_DIR}
            ${proto_file}
        DEPENDS ${proto_file}
        COMMENT "Compiling ${proto_file} to Python"
    )
    list(APPEND GRPC_PYS ${grpc_pb2_py})
    list(APPEND GRPC_PYS ${grpc_py})
endforeach()

# Create a custom target to trigger code generation for all proto files
add_custom_target(generate_protos ALL
    DEPENDS ${GRPC_SRCS} ${GRPC_HDRS} ${GRPC_PYS}
)

# ------------------------------




# ------------------------------ add executable ------------------------------
# ----- Build C++ test Code ----- #
option(BUILD_CPP_SOURCE "Build cpp source" ON)
if(BUILD_CPP_SOURCE)
    add_executable(pose_server src/pose_server.cpp)
    add_dependencies(pose server
        generate_protos
    )
    target_sources(pose_server PRIVATE
        ${GENERATED_PROTO_CPP_DIR}/pose_messages.grpc.pb.cc
        ${GENERATED_PROTO_CPP_DIR}/pose_messages.pb.cc
    )
    target_link_libraries(pose_server 
        gRPC::grpc++ 
        protobuf::libprotobuf
    )

    add_executable(pose_client src/pose_client.cpp)
    target_sources(pose_client PRIVATE
        ${GENERATED_PROTO_CPP_DIR}/pose_messages.grpc.pb.cc
        ${GENERATED_PROTO_CPP_DIR}/pose_messages.pb.cc
    )
    target_link_libraries(pose_client 
        gRPC::grpc++ 
        protobuf::libprotobuf
    )
endif()

# ------------ ROS 2
add_executable(float32_server
    src/float32_server.cpp
)
add_dependencies(float32_server
    generate_protos
)
target_sources(float32_server PRIVATE
    ${GENERATED_PROTO_CPP_DIR}/robot_service.grpc.pb.cc
    ${PROTO_SRCS}
)
target_link_libraries(float32_server 
    gRPC::grpc++ 
    protobuf::libprotobuf
)

# Linking required packages
ament_target_dependencies(float32_server
    rclcpp
    std_msgs
    geometry_msgs
)

# Installation Settings
install(TARGETS
    float32_server
    DESTINATION lib/${PROJECT_NAME}
)

ament_package()
