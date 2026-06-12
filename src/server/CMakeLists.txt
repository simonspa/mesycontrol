if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.30)
    cmake_policy(SET CMP0167 NEW)
endif()

set(Boost_USE_MULTITHREADED ON)
set(Boost_USE_STATIC_LIBS   OFF)
find_package(Boost 1.87 REQUIRED COMPONENTS chrono program_options date_time log log_setup regex thread)
find_package(Threads REQUIRED)

include(FindProtobuf)
find_package(Protobuf REQUIRED)

include_directories(${CMAKE_CURRENT_BINARY_DIR})

protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${MESYCONTROL_PROTO_IN})
add_custom_target(mesycontrol_proto DEPENDS ${PROTO_HDRS} ${PROTO_SRCS})

add_library(mesycontrol SHARED)
add_dependencies(mesycontrol mesycontrol_proto)
target_sources(mesycontrol PRIVATE
    logging.cc
    mrc_comm.cc
    mrc1_connection.cc
    mrc1_reply_parser.cc
    mrc1_request_queue.cc
    poller.cc
    protocol.cc
    ${PROTO_SRCS}
    ${PROTO_HDRS}
    tcp_connection.cc
    tcp_connection.cc
    tcp_connection_manager.cc
    tcp_server.cc
    ${MESYCONTROL_GLOBAL_SOURCES}
)
# Expose the dir with generated mesycontrol.pb.h
target_include_directories(mesycontrol PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)

target_link_libraries(mesycontrol
    PUBLIC Boost::disable_autolinking
    PUBLIC ${Boost_LIBRARIES}
    PUBLIC Threads::Threads
    PUBLIC protobuf::libprotobuf
    )

target_compile_definitions(mesycontrol PRIVATE -DBOOST_BIND_GLOBAL_PLACEHOLDERS)
target_compile_features(mesycontrol PRIVATE cxx_std_17)

add_executable(mesycontrol_server
    main.cc
    )

target_link_libraries(mesycontrol_server
    PRIVATE mesycontrol
    )

target_compile_definitions(mesycontrol_server PRIVATE -DBOOST_BIND_GLOBAL_PLACEHOLDERS)
target_compile_features(mesycontrol_server PRIVATE cxx_std_17)
target_include_directories(mesycontrol_server PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

option(BOOST_ASIO_ENABLE_HANDLER_TRACKING "enable boost::asio handler tracking" OFF)
option(BOOST_ASIO_ENABLE_BUFFER_DEBUGGING "enable boost::asio buffer debugging" OFF)

if(BOOST_ASIO_ENABLE_HANDLER_TRACKING)
    target_compile_definitions(mesycontrol_server PRIVATE BOOST_ASIO_ENABLE_HANDLER_TRACKING)
endif()

if(BOOST_ASIO_ENABLE_BUFFER_DEBUGGING)
    target_compile_definitions(mesycontrol_server PRIVATE BOOST_ASIO_ENABLE_BUFFER_DEBUGGING)
endif()

if(WIN32)
    # Link against the DLLs needed by boost::asio
    target_link_libraries(mesycontrol_server PRIVATE ws2_32 wsock32)
    # Statically link against the GCC and C++ libraries to avoid
    # depending on DLLs installed with MinGW.
    target_link_options(mesycontrol_server PRIVATE -static-libgcc -static-libstdc++)
else(WIN32)
    target_link_libraries(mesycontrol_server PRIVATE rt)
endif(WIN32)

if (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "armv7l")
    target_link_libraries(mesycontrol_server PRIVATE atomic)
endif()

install(TARGETS mesycontrol RUNTIME DESTINATION lib)
install(TARGETS mesycontrol_server RUNTIME DESTINATION bin)
install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    DESTINATION include
    FILES_MATCHING
    PATTERN "*.h")
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/mesycontrol.pb.h" DESTINATION include)

if (NOT WIN32)
    # Invoke a script to install additional shared library dependencies.
    install(SCRIPT install_server_deps.cmake)
endif()

# vim:tw=0
