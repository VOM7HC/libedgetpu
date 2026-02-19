cmake_minimum_required(VERSION 3.16)

project(libedgetpu LANGUAGES C CXX)

set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

option(EDGETPU_BUILD_THROTTLED "Build throttled libedgetpu variant" ON)
option(EDGETPU_USE_GOLD_LINKER "Use GNU gold linker (-fuse-ld=gold)" OFF)

set(TFROOT "" CACHE PATH "Path to TensorFlow source root")
if(NOT TFROOT)
  message(FATAL_ERROR "TFROOT must be set (e.g. -DTFROOT=/path/to/tensorflow)")
endif()
if(NOT EXISTS "${TFROOT}/tensorflow/lite/c/common.c")
  message(FATAL_ERROR "Invalid TFROOT: ${TFROOT}/tensorflow/lite/c/common.c not found")
endif()
if(NOT EXISTS "${TFROOT}/tensorflow/lite/util.cc")
  message(FATAL_ERROR "Invalid TFROOT: ${TFROOT}/tensorflow/lite/util.cc not found")
endif()

find_program(FLATC_EXECUTABLE flatc)
if(NOT FLATC_EXECUTABLE)
  message(FATAL_ERROR "flatc not found. Install flatbuffers-compiler or set FLATC_EXECUTABLE.")
endif()

set(EDGETPU_PUBLIC_DEP_INCLUDE_DIRS)
set(EDGETPU_DEP_LIBRARIES)

find_package(Flatbuffers CONFIG QUIET)
if(TARGET flatbuffers::flatbuffers)
  list(APPEND EDGETPU_DEP_LIBRARIES flatbuffers::flatbuffers)
else()
  find_library(FLATBUFFERS_LIB flatbuffers REQUIRED)
  find_path(FLATBUFFERS_INCLUDE_DIR flatbuffers/flatbuffers.h REQUIRED)
  list(APPEND EDGETPU_DEP_LIBRARIES ${FLATBUFFERS_LIB})
  list(APPEND EDGETPU_PUBLIC_DEP_INCLUDE_DIRS ${FLATBUFFERS_INCLUDE_DIR})
endif()

find_package(absl CONFIG QUIET)
if(TARGET absl::flags)
  list(APPEND EDGETPU_DEP_LIBRARIES
    absl::flags
    absl::flags_internal
    absl::flags_reflection
    absl::flags_marshalling
    absl::hash
    absl::hashtablez_sampler
    absl::raw_hash_set
    absl::str_format_internal
  )
else()
  set(ABSL_REQUIRED_LIBS
    absl_flags
    absl_flags_internal
    absl_flags_reflection
    absl_flags_marshalling
    absl_hash
    absl_hashtablez_sampler
    absl_raw_hash_set
    absl_str_format_internal
  )
  foreach(lib ${ABSL_REQUIRED_LIBS})
    find_library(ABSL_LIB_${lib} ${lib} REQUIRED)
    list(APPEND EDGETPU_DEP_LIBRARIES ${ABSL_LIB_${lib}})
  endforeach()
  find_path(ABSL_INCLUDE_DIR absl/types/optional.h REQUIRED)
  list(APPEND EDGETPU_PUBLIC_DEP_INCLUDE_DIRS ${ABSL_INCLUDE_DIR})
endif()

set(LIBUSB_INCLUDE_DIRS)
if(WIN32)
  find_path(LIBUSB_INCLUDE_DIR NAMES libusb.h PATH_SUFFIXES libusb-1.0 REQUIRED)
  find_library(LIBUSB_LIBRARY NAMES libusb-1.0 REQUIRED)
  set(LIBUSB_LIBRARIES ${LIBUSB_LIBRARY})
  set(LIBUSB_INCLUDE_DIRS ${LIBUSB_INCLUDE_DIR})
else()
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(LIBUSB REQUIRED libusb-1.0)
  set(LIBUSB_LIBRARIES ${LIBUSB_LIBRARIES})
  set(LIBUSB_INCLUDE_DIRS ${LIBUSB_INCLUDE_DIRS})
endif()

set(LIBEDGETPU_ROOT ${CMAKE_CURRENT_SOURCE_DIR})
set(LIBEDGETPU_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/generated)

set(GENERATED_FLATBUFFER_HEADERS
  ${LIBEDGETPU_BINARY_DIR}/executable/executable_generated.h
  ${LIBEDGETPU_BINARY_DIR}/api/driver_options_generated.h
)

add_custom_command(
  OUTPUT ${LIBEDGETPU_BINARY_DIR}/executable/executable_generated.h
  COMMAND ${CMAKE_COMMAND} -E make_directory ${LIBEDGETPU_BINARY_DIR}/executable
  COMMAND ${FLATC_EXECUTABLE} --cpp --gen-object-api --force-empty --gen-mutable
          -o ${LIBEDGETPU_BINARY_DIR}/executable
          ${LIBEDGETPU_ROOT}/executable/executable.fbs
  DEPENDS ${LIBEDGETPU_ROOT}/executable/executable.fbs
  VERBATIM
)

add_custom_command(
  OUTPUT ${LIBEDGETPU_BINARY_DIR}/api/driver_options_generated.h
  COMMAND ${CMAKE_COMMAND} -E make_directory ${LIBEDGETPU_BINARY_DIR}/api
  COMMAND ${FLATC_EXECUTABLE} --cpp --gen-object-api --force-empty --gen-mutable
          -o ${LIBEDGETPU_BINARY_DIR}/api
          ${LIBEDGETPU_ROOT}/api/driver_options.fbs
  DEPENDS ${LIBEDGETPU_ROOT}/api/driver_options.fbs
  VERBATIM
)

set(FIRMWARE_HEADER ${LIBEDGETPU_BINARY_DIR}/driver/usb/usb_latest_firmware.h)
add_custom_command(
  OUTPUT ${FIRMWARE_HEADER}
  COMMAND ${CMAKE_COMMAND}
          -DOUTPUT=${FIRMWARE_HEADER}
          -DFIRMWARE_SOURCES=${LIBEDGETPU_ROOT}/driver/usb/apex_latest_single_ep.bin\;${LIBEDGETPU_ROOT}/driver/usb/apex_latest_multi_ep.bin
          -P ${LIBEDGETPU_ROOT}/cmake/GenerateFirmwareHeader.cmake
  DEPENDS
    ${LIBEDGETPU_ROOT}/driver/usb/apex_latest_single_ep.bin
    ${LIBEDGETPU_ROOT}/driver/usb/apex_latest_multi_ep.bin
    ${LIBEDGETPU_ROOT}/cmake/GenerateFirmwareHeader.cmake
  VERBATIM
)

add_custom_target(libedgetpu_generated_headers
  DEPENDS ${GENERATED_FLATBUFFER_HEADERS} ${FIRMWARE_HEADER}
)

set(LIBEDGETPU_C_SRCS
  ${TFROOT}/tensorflow/lite/c/common.c
)

set(LIBEDGETPU_COMMON_CXX_SRCS
  api/allocated_buffer.cc
  api/buffer.cc
  api/driver_options_helper.cc
  api/layer_information.cc
  api/tensor_util.cc
  api/watchdog.cc
  driver/aligned_allocator.cc
  driver/allocator.cc
  driver/beagle/beagle_kernel_top_level_handler.cc
  driver/beagle/beagle_pci_driver_provider.cc
  driver/beagle/beagle_top_level_handler.cc
  driver/beagle/beagle_top_level_interrupt_manager.cc
  driver/beagle/beagle_usb_driver_provider.cc
  driver/device_buffer.cc
  driver/device_buffer_mapper.cc
  driver/dma_chunker.cc
  driver/dma_info.cc
  driver/dma_info_extractor.cc
  driver/driver.cc
  driver/driver_factory.cc
  driver/executable_util.cc
  driver/instruction_buffers.cc
  driver/interrupt/grouped_interrupt_controller.cc
  driver/interrupt/interrupt_controller.cc
  driver/interrupt/top_level_interrupt_manager.cc
  driver/kernel/kernel_coherent_allocator.cc
  driver/kernel/kernel_event_handler.cc
  driver/kernel/kernel_interrupt_handler.cc
  driver/kernel/kernel_mmu_mapper.cc
  driver/kernel/kernel_registers.cc
  driver/memory/buddy_address_space.cc
  driver/memory/buddy_allocator.cc
  driver/memory/dual_address_space.cc
  driver/memory/mmio_address_space.cc
  driver/memory/mmu_mapper.cc
  driver/memory/nop_address_space.cc
  driver/mmio/coherent_allocator.cc
  driver/mmio_driver.cc
  driver/package_registry.cc
  driver/package_verifier.cc
  driver/real_time_dma_scheduler.cc
  driver/registers/registers.cc
  driver/request.cc
  driver/run_controller.cc
  driver/scalar_core_controller.cc
  driver/single_queue_dma_scheduler.cc
  driver/single_tpu_request.cc
  driver/usb/local_usb_device.cc
  driver/usb/usb_dfu_commands.cc
  driver/usb/usb_dfu_util.cc
  driver/usb/usb_driver.cc
  driver/usb/usb_io_request.cc
  driver/usb/usb_ml_commands.cc
  driver/usb/usb_registers.cc
  driver/usb/usb_standard_commands.cc
  driver_shared/time_stamper/driver_time_stamper.cc
  port/blocking_counter.cc
  port/default/port_from_tf/logging.cc
  port/default/port_from_tf/status.cc
  port/default/port_from_tf/statusor.cc
  port/default/stringprintf.cc
  port/shared_mutex.cc
  tflite/custom_op.cc
  tflite/custom_op_data.cc
  tflite/custom_op_direct.cc
  tflite/custom_op_user_data_direct.cc
  tflite/edgetpu_c.cc
  tflite/edgetpu_delegate_for_custom_op.cc
  tflite/edgetpu_delegate_for_custom_op_tflite_plugin.cc
  ${TFROOT}/tensorflow/lite/util.cc
)

set(LIBEDGETPU_PLATFORM_SRCS)
if(WIN32)
  list(APPEND LIBEDGETPU_PLATFORM_SRCS
    driver/driver_factory_windows.cc
    driver/beagle/beagle_pci_driver_provider_windows.cc
    driver/kernel/windows/kernel_coherent_allocator_windows.cc
    driver/kernel/windows/kernel_event_handler_windows.cc
    driver/kernel/windows/kernel_event_windows.cc
    driver/kernel/windows/kernel_registers_windows.cc
    driver/usb/libusb_options_windows.cc
    port/fileio_windows.cc
    port/timer_windows.cc
  )
elseif(APPLE)
  list(APPEND LIBEDGETPU_PLATFORM_SRCS
    driver/driver_factory_darwin.cc
    driver/usb/libusb_options_default.cc
    port/timer_portable.cc
  )
else()
  list(APPEND LIBEDGETPU_PLATFORM_SRCS
    driver/driver_factory_default.cc
    driver/beagle/beagle_pci_driver_provider_linux.cc
    driver/kernel/linux/kernel_coherent_allocator_linux.cc
    driver/kernel/linux/kernel_event_handler_linux.cc
    driver/kernel/linux/kernel_event_linux.cc
    driver/kernel/linux/kernel_registers_linux.cc
    driver/usb/libusb_options_default.cc
    port/timer_portable.cc
  )
endif()

set(LIBEDGETPU_DIRECT_VARIANT_SRCS
  tflite/edgetpu_context_direct.cc
  tflite/edgetpu_manager_direct.cc
)

function(add_libedgetpu_target target_name throttle_enabled output_dir)
  add_library(${target_name} SHARED
    ${LIBEDGETPU_C_SRCS}
    ${LIBEDGETPU_COMMON_CXX_SRCS}
    ${LIBEDGETPU_PLATFORM_SRCS}
    ${LIBEDGETPU_DIRECT_VARIANT_SRCS}
  )

  add_dependencies(${target_name} libedgetpu_generated_headers)

  target_compile_options(${target_name} PRIVATE -Wall)
  if(throttle_enabled)
    target_compile_definitions(${target_name} PRIVATE THROTTLE_EDGE_TPU)
  endif()

  target_compile_definitions(${target_name} PRIVATE DARWINN_PORT_DEFAULT)

  target_include_directories(${target_name} PRIVATE
    ${LIBEDGETPU_ROOT}
    ${TFROOT}
    ${LIBEDGETPU_BINARY_DIR}
    ${LIBUSB_INCLUDE_DIRS}
    ${EDGETPU_PUBLIC_DEP_INCLUDE_DIRS}
  )

  if(WIN32)
    target_compile_definitions(${target_name} PRIVATE EDGETPU_COMPILE_LIBRARY)
    target_link_libraries(${target_name} PRIVATE setupapi cfgmgr32)
  endif()

  target_link_libraries(${target_name} PRIVATE
    ${EDGETPU_DEP_LIBRARIES}
    ${LIBUSB_LIBRARIES}
  )

  if(EDGETPU_USE_GOLD_LINKER AND NOT WIN32 AND NOT APPLE)
    target_link_options(${target_name} PRIVATE -fuse-ld=gold)
  endif()

  if(NOT WIN32 AND NOT APPLE)
    target_link_options(${target_name} PRIVATE
      -Wl,--version-script=${LIBEDGETPU_ROOT}/tflite/public/libedgetpu.lds
    )
  endif()

  set_target_properties(${target_name} PROPERTIES
    OUTPUT_NAME edgetpu
    VERSION 1.0
    SOVERSION 1
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${output_dir}
  )
endfunction()

add_libedgetpu_target(edgetpu_direct OFF direct/k8)
add_custom_target(libedgetpu DEPENDS edgetpu_direct)
if(EDGETPU_BUILD_THROTTLED)
  add_libedgetpu_target(edgetpu_throttled ON throttled/k8)
  add_custom_target(libedgetpu-throttled DEPENDS edgetpu_throttled)
endif()
