# Copyright 2019 Google
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# CMake file for the firebase_analytics library

cmake_minimum_required (VERSION 3.1)
set (CMAKE_CXX_STANDARD 11)

include(download_pod_headers)

project(firebase_analytics NONE)
enable_language(C)
enable_language(CXX)

# Analytics generates header files for default events, parameters, and
# properties based on the iOS SDK, that are used across all platforms.
set(analytics_generated_headers_dir
    "${FIREBASE_GEN_FILE_DIR}/analytics/src/include/firebase/analytics")
set(event_names_header "${analytics_generated_headers_dir}/event_names.h")
set(parameter_names_header
    "${analytics_generated_headers_dir}/parameter_names.h")
set(user_property_names_header
    "${analytics_generated_headers_dir}/user_property_names.h")
file(MAKE_DIRECTORY ${analytics_generated_headers_dir})

# Generate the header file by invoking the generate_constants python script.
function(generate_analytics_header OBJC_FILE CPP_FILE)
  add_custom_command(
    OUTPUT ${CPP_FILE}
    COMMAND python "${CMAKE_CURRENT_LIST_DIR}/generate_constants.py"
      "--objc_header=${OBJC_FILE}"
      "--cpp_header=${CPP_FILE}"
    DEPENDS ${OBJC_FILE}
    COMMENT "Generating ${CPP_FILE}"
  )
endfunction()

# Call the above function for all of the header files to generate.
generate_analytics_header(
  "${CMAKE_CURRENT_LIST_DIR}/ios_headers/FIREventNames.h"
  "${event_names_header}"
)
generate_analytics_header(
  "${CMAKE_CURRENT_LIST_DIR}/ios_headers/FIRParameterNames.h"
  "${parameter_names_header}"
)
generate_analytics_header(
  "${CMAKE_CURRENT_LIST_DIR}/ios_headers/FIRUserPropertyNames.h"
  "${user_property_names_header}"
)

# Common source files used by all platforms
set(common_SRCS
    src/analytics_common.cc
    # List the generated files, so that they are generated for the build.
    ${event_names_header}
    ${parameter_names_header}
    ${user_property_names_header})

# Source files used by the Android implementation.
set(android_SRCS
    src/analytics_android.cc)

# Source files used by the iOS implementation.
set(ios_SRCS
    src/analytics_ios.mm)

# Source files used by the stub implementation.
set(stub_SRCS
    src/analytics_stub.cc)

if(ANDROID)
  set(analytics_platform_SRCS
      "${android_SRCS}")
elseif(IOS)
  set(analytics_platform_SRCS
      "${ios_SRCS}")
else()
  set(analytics_platform_SRCS
      "${stub_SRCS}")
endif()

add_library(firebase_analytics STATIC
    ${common_SRCS}
    ${analytics_platform_SRCS})

# Set up the dependency on Firebase App.
target_link_libraries(firebase_analytics
    firebase_app)
# Public headers all refer to each other relative to the src/include directory,
# while private headers are relative to the entire C++ SDK directory.
target_include_directories(firebase_analytics
  PUBLIC
    ${CMAKE_CURRENT_LIST_DIR}/src/include
    ${FIREBASE_GEN_FILE_DIR}/analytics/src/include
  PRIVATE
    ${FIREBASE_CPP_SDK_ROOT_DIR}
    ${FIREBASE_GEN_FILE_DIR}
)
target_compile_definitions(firebase_analytics
  PRIVATE
    -DINTERNAL_EXPERIMENTAL=1
)
# Automatically include headers that might not be declared.
if(MSVC)
  add_definitions(/FI"assert.h" /FI"string.h" /FI"stdint.h")
else()
  add_definitions(-include assert.h -include string.h)
endif()

if(IOS)
  # Enable Automatic Reference Counting (ARC).
  set_property(
    TARGET firebase_analytics
    APPEND_STRING PROPERTY
    COMPILE_FLAGS "-fobjc-arc")

  # Setup the target for getting the Firebase iOS SDK headers through Cocoapods.
  set(pod_target_name "download_analytics_pod_headers")
  set(pods_dir "${PROJECT_BINARY_DIR}/Pods")
  set(pod_list "")
  list(APPEND pod_list "'Firebase/Core', '5.20.2'")
  list(APPEND pod_list "'Firebase/Analytics', '5.20.2'")

  setup_pod_headers_target("${pod_target_name}" "${pods_dir}" "${pod_list}")

  # Add the Cocoapod headers to the include directories
  set(base_header_dir "${pods_dir}/Pods/Headers/Public")
  target_include_directories(firebase_analytics
    PRIVATE
      ${base_header_dir}/FirebaseCore
      ${base_header_dir}/FirebaseAnalytics/FirebaseAnalytics
  )

  # Add a dependency to downloading the headers onto analytics.
  add_dependencies(firebase_analytics ${pod_target_name})
endif()
