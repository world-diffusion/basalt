cmake_minimum_required(VERSION 3.24)

project(basalt CXX)

option(BASALT_COMPILE_CKAAR_STRIPPED "Build stripped-down basalt library" ON)
option(BASALT_BUILD_SHARED_LIB "Build basalt core library as shared" ON)
option(BASALT_FULLY_STATIC_BINARIES "Link executables fully static" ON)
option(BASALT_INSTANTIATIONS_DOUBLE "Instantiate templates for Scalar=double." ON)
option(BASALT_INSTANTIATIONS_FLOAT "Instantiate templates for Scalar=float." ON)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

if(NOT CXX_MARCH)
  set(CXX_MARCH native)
endif()

# ------------------------------------------------------------------------
# Create the main library target up front (sources/links attached below).
# Keeping this early lets every later section configure the target
# directly instead of accumulating global CMAKE_* state.
# ------------------------------------------------------------------------
if(BASALT_BUILD_SHARED_LIB)
  add_library(basalt SHARED)
else()
  add_library(basalt STATIC)
endif()
add_library(basalt::basalt ALIAS basalt)

target_compile_features(basalt PUBLIC cxx_std_17)
set_target_properties(basalt PROPERTIES
  CXX_STANDARD_REQUIRED ON
  CXX_EXTENSIONS OFF
  POSITION_INDEPENDENT_CODE ON
)

# ------------------------------------------------------------------------
# CCache (compiler launcher is inherently global/toolchain-level, so this
# is one of the few things that legitimately stays as a CMAKE_* variable)
# ------------------------------------------------------------------------
if (NOT CMAKE_C_COMPILER_LAUNCHER AND NOT CMAKE_CXX_COMPILER_LAUNCHER)
  find_program(CCACHE_PROGRAM ccache)
  if(CCACHE_PROGRAM)
    message(STATUS "Found ccache: ${CCACHE_PROGRAM}")
    set(CMAKE_C_COMPILER_LAUNCHER   ${CCACHE_PROGRAM})
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_PROGRAM})
  else()
    message(STATUS "Didn't find ccache")
  endif()
else()
  message(STATUS "Compiler launcher already set. Not configuring ccache.")
  message(STATUS "CMAKE_C_COMPILER_LAUNCHER: ${CMAKE_C_COMPILER_LAUNCHER}")
  message(STATUS "CMAKE_CXX_COMPILER_LAUNCHER: ${CMAKE_CXX_COMPILER_LAUNCHER}")
endif()

# ------------------------------------------------------------------------
# Link-option helpers (already target-scoped in the original file — kept
# as-is, just moved up next to the target that uses them).
# ------------------------------------------------------------------------
function(basalt_add_static_link_options target)
  if(BASALT_FULLY_STATIC_BINARIES AND UNIX AND NOT APPLE)
    target_link_options(${target} PRIVATE
      "$<$<CONFIG:Release>:-static>"
      "$<$<CONFIG:Release>:-static-libstdc++>"
      "$<$<CONFIG:Release>:-static-libgcc>"
    )
  endif()
endfunction()

function(basalt_add_gui_link_options target)
  if(BASALT_FULLY_STATIC_BINARIES AND UNIX AND NOT APPLE)
    target_link_options(${target} PRIVATE
      "$<$<CONFIG:Release>:-static-libstdc++>"
      "$<$<CONFIG:Release>:-static-libgcc>"
    )
  endif()
endfunction()

function(basalt_add_macos_runtime_paths target)
  if(APPLE)
    set_property(TARGET ${target} APPEND PROPERTY BUILD_RPATH "@executable_path/../lib")
    set_property(TARGET ${target} PROPERTY INSTALL_RPATH "@executable_path/../lib")
  endif()
endfunction()

# ------------------------------------------------------------------------
# Warning / optimization flags -> applied per-target via
# target_compile_options instead of a global CMAKE_CXX_FLAGS string.
# Any executable built in this project should call this too, so it's a
# function rather than a one-off block.
# ------------------------------------------------------------------------
function(basalt_add_common_compile_options target)
  set(_flags
    -Wall -Wextra -Werror
    -Wno-error=unused-variable
    -Wno-error=array-bounds
    -ftemplate-backtrace-limit=0
    -DEIGEN_DONT_PARALLELIZE
  )

  if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    list(APPEND _flags
      -Wno-exceptions -fcolor-diagnostics -Wno-error=deprecated-declarations
      -Wno-error=missing-template-arg-list-after-template-kw
    )
    if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang"
       AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 20)
      list(APPEND _flags -frelaxed-template-template-args)
    endif()
    if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 10)
      list(APPEND _flags -Wno-misleading-indentation -Wno-error=deprecated-copy)
    endif()
    if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 15)
      list(APPEND _flags -Wno-error=unused-but-set-variable)
    endif()
  else()
    list(APPEND _flags
      -Wno-error=maybe-uninitialized -Wno-error=implicit-fallthrough -Wno-error=deprecated-declarations
    )
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 10.0)
      list(APPEND _flags -Wno-error=uninitialized)
    endif()
    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 9.0)
      list(APPEND _flags -Wno-error=deprecated-copy)
    endif()
    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 10.0)
      list(APPEND _flags -Wno-parentheses)
    endif()
  endif()

  if(NOT (APPLE AND CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64"))
    list(APPEND _flags -march=${CXX_MARCH})
  else()
    message(STATUS "Running on Apple ${CMAKE_SYSTEM_PROCESSOR}. Disabled -march flag.")
  endif()

  target_compile_options(${target} PRIVATE ${_flags})

  # Per-config flags (previously CMAKE_CXX_FLAGS_<CONFIG>) as generator
  # expressions scoped to this target only.
  target_compile_options(${target} PRIVATE
    "$<$<CONFIG:Debug>:-O0;-g;-DEIGEN_INITIALIZE_MATRICES_BY_NAN>"
    "$<$<CONFIG:RelWithDebInfo>:-O3;-g;-DEIGEN_INITIALIZE_MATRICES_BY_NAN>"
    "$<$<CONFIG:Release>:-O3;-DNDEBUG>"
    "$<$<CONFIG:CIDebug>:-O0;-DEIGEN_INITIALIZE_MATRICES_BY_NAN>"
    "$<$<CONFIG:CIRelWithDebInfo>:-O3;-DEIGEN_INITIALIZE_MATRICES_BY_NAN>"
  )

  if(NOT TBB_USE_GLIBCXX_VERSION AND UNIX AND NOT APPLE
     AND CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    string(REPLACE "." "0" _tbb_glibcxx_version ${CMAKE_CXX_COMPILER_VERSION})
    target_compile_definitions(${target} PRIVATE TBB_USE_GLIBCXX_VERSION=${_tbb_glibcxx_version})
  endif()
endfunction()

basalt_add_common_compile_options(basalt)
set_target_properties(basalt PROPERTIES COMPILE_WARNING_AS_ERROR OFF)
# ------------------------------------------------------------------------
# Platform detection / target-scoped platform quirks
# ------------------------------------------------------------------------
if(APPLE)
  set_target_properties(basalt PROPERTIES FIND_FRAMEWORK LAST)

  if(CMAKE_SYSTEM_VERSION VERSION_LESS 19.0.0)
    target_include_directories(basalt PRIVATE "/usr/local/opt/llvm/include/c++/v1")
    target_link_directories(basalt PRIVATE "/usr/local/opt/llvm/lib")
    target_compile_options(basalt PRIVATE "-nostdinc++")
    get_target_property(_basalt_implicit_incs basalt CXX_IMPLICIT_INCLUDE_DIRECTORIES)
    if(_basalt_implicit_incs)
      list(REMOVE_ITEM _basalt_implicit_incs "/usr/local/opt/llvm/include/c++/v1")
      set_target_properties(basalt PROPERTIES CXX_IMPLICIT_INCLUDE_DIRECTORIES "${_basalt_implicit_incs}")
    endif()
  endif()

  if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    message(STATUS "Detected macOS with non-Apple clang")
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    message(STATUS "Detected macOS with Apple clang")
    if(CMAKE_SYSTEM_VERSION VERSION_LESS 18.0.0)
      message(WARNING "Detected Darwin version ${CMAKE_SYSTEM_VERSION}, which is earlier than macos 10.14 Mojave.")
    endif()
  else()
    message(WARNING "Detected macOS with unsupported compiler ${CMAKE_CXX_COMPILER_ID}")
  endif()
elseif(UNIX)
  set(STD_CXX_FS stdc++fs)

  if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    message(STATUS "Detected Linux with clang.")
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    message(STATUS "Detected Linux with gcc.")
  else()
    message(WARNING "Detected Linux with unsupported compiler ${CMAKE_CXX_COMPILER_ID}")
  endif()
else()
  message(WARNING "Only Linux and macOS are currently supported")
endif()

if(BASALT_INSTANTIATIONS_DOUBLE)
  target_compile_definitions(basalt PUBLIC BASALT_INSTANTIATIONS_DOUBLE)
endif()
if(BASALT_INSTANTIATIONS_FLOAT)
  target_compile_definitions(basalt PUBLIC BASALT_INSTANTIATIONS_FLOAT)
endif()

# NOTE: CMAKE_REQUIRED_FLAGS intentionally left as-is — it's a documented
# input variable consumed by check_cxx_source_compiles()/friends, not
# project state, so it doesn't count as a "leak" the same way the old
# CMAKE_CXX_FLAGS accumulation did.
set(CMAKE_REQUIRED_FLAGS "-Wno-error")

# ------------------------------------------------------------------------
# Dependencies
# ------------------------------------------------------------------------
if(NOT TARGET Eigen3::Eigen)
  find_package(Eigen3 CONFIG REQUIRED)
endif()
find_package(TBB CONFIG REQUIRED)
find_package(OpenCV CONFIG REQUIRED COMPONENTS core imgproc calib3d features2d imgcodecs)
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  file(GLOB BASALT_VCPKG_BOOST_CONFIG "${CMAKE_BINARY_DIR}/vcpkg_installed/*/share/boost/BoostConfig.cmake")
  if(BASALT_VCPKG_BOOST_CONFIG)
    list(GET BASALT_VCPKG_BOOST_CONFIG 0 BASALT_VCPKG_BOOST_CONFIG)
    get_filename_component(BASALT_VCPKG_BOOST_DIR "${BASALT_VCPKG_BOOST_CONFIG}" DIRECTORY)
    set(Boost_DIR "${BASALT_VCPKG_BOOST_DIR}" CACHE PATH "Boost config directory" FORCE)
  endif()
  find_package(Boost CONFIG REQUIRED COMPONENTS date_time filesystem format program_options regex thread)
endif()
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  find_package(lz4 CONFIG REQUIRED)
endif()
if(NOT TARGET fmt::fmt)
  find_package(fmt CONFIG REQUIRED)
endif()
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  find_package(CLI11 CONFIG REQUIRED)
endif()
find_package(magic_enum CONFIG REQUIRED)
if(NOT TARGET nlohmann_json::nlohmann_json)
  find_package(nlohmann_json CONFIG REQUIRED)
endif()
find_package(opengv CONFIG REQUIRED)
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  find_package(rosbag CONFIG REQUIRED)
endif()
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  find_package(Pangolin CONFIG REQUIRED)
endif()
if(NOT TARGET basalt-headers)
  find_package(basalt-headers CONFIG REQUIRED)
endif()
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  find_package(GTest CONFIG REQUIRED)
endif()

get_target_property(BASALT_EIGEN_INCLUDE_DIRS Eigen3::Eigen INTERFACE_INCLUDE_DIRECTORIES)
if(NOT BASALT_EIGEN_INCLUDE_DIRS)
  message(FATAL_ERROR "Eigen3::Eigen does not publish include directories")
endif()

function(basalt_normalize_eigen_includes target_name)
  if(NOT TARGET ${target_name})
    return()
  endif()

  get_target_property(_include_dirs ${target_name} INTERFACE_INCLUDE_DIRECTORIES)
  if(NOT _include_dirs)
    return()
  endif()

  set(_filtered_include_dirs "")
  set(_saw_eigen_include_dir OFF)
  foreach(_dir IN LISTS _include_dirs)
    if(_dir MATCHES ".*/eigen3?(/unsupported)?$")
      set(_saw_eigen_include_dir ON)
    else()
      list(APPEND _filtered_include_dirs "${_dir}")
    endif()
  endforeach()

  if(_saw_eigen_include_dir)
    list(APPEND _filtered_include_dirs ${BASALT_EIGEN_INCLUDE_DIRS})
    list(REMOVE_DUPLICATES _filtered_include_dirs)
    set_target_properties(${target_name} PROPERTIES
      INTERFACE_INCLUDE_DIRECTORIES "${_filtered_include_dirs}")
  endif()
endfunction()

if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  foreach(_target opengv basalt::basalt-headers ${BASALT_PANGO_TARGETS})
    basalt_normalize_eigen_includes(${_target})
  endforeach()
else()
  foreach(_target opengv basalt::basalt-headers)
    basalt_normalize_eigen_includes(${_target})
  endforeach()
endif()


# fix for debian vs. ubuntu magic enum install path
get_target_property(MAGIC_ENUM_INCLUDE_DIRS
    magic_enum::magic_enum
    INTERFACE_INCLUDE_DIRECTORIES
)
foreach(dir IN LISTS MAGIC_ENUM_INCLUDE_DIRS)
    if(EXISTS "${dir}/magic_enum.hpp")
        # Ubuntu layout
        set(MAGIC_ENUM_ROOT "${dir}")
    elseif(EXISTS "${dir}/magic_enum/magic_enum.hpp")
        # Debian layout
        set(MAGIC_ENUM_ROOT "${dir}/magic_enum")
    endif()
endforeach()

add_library(basalt::opencv INTERFACE IMPORTED)
set_property(TARGET basalt::opencv PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${OpenCV_INCLUDE_DIRS})
set_property(TARGET basalt::opencv PROPERTY INTERFACE_LINK_LIBRARIES ${OpenCV_LIBS})

# Build bundled apriltag library (needs basalt::opencv above)
add_subdirectory(thirdparty/apriltag EXCLUDE_FROM_ALL)

if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  add_library(basalt::cli11 INTERFACE IMPORTED)
  set_property(TARGET basalt::cli11 PROPERTY INTERFACE_LINK_LIBRARIES CLI11::CLI11)
endif()

if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  set(BASALT_PANGO_TARGETS ${Pangolin_LIBRARIES})
  enable_testing()
endif()

# ------------------------------------------------------------------------
# basalt library: sources, includes, links, install
# ------------------------------------------------------------------------
install(TARGETS basalt
  EXPORT BasaltTargets
  LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
  ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
  RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
  FILE_SET HEADERS DESTINATION ${CMAKE_INSTALL_PREFIX}/include)

target_sources(basalt
  PUBLIC
    FILE_SET HEADERS
    BASE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include
    FILES
      include/basalt/calibration/vignette.h
      include/basalt/hash_bow/hash_bow.h
      include/basalt/linearization/block_diagonal.hpp
      include/basalt/linearization/imu_block.hpp
      include/basalt/linearization/landmark_block.hpp
      include/basalt/linearization/landmark_block_abs_dynamic.hpp
      include/basalt/linearization/linearization_abs_qr.hpp
      include/basalt/linearization/linearization_abs_sc.hpp
      include/basalt/linearization/linearization_base.hpp
      include/basalt/linearization/linearization_rel_sc.hpp
      include/basalt/optical_flow/frame_to_frame_optical_flow.h
      include/basalt/optical_flow/multiscale_frame_to_frame_optical_flow.h
      include/basalt/optical_flow/optical_flow.h
      include/basalt/optical_flow/patch.h
      include/basalt/optical_flow/patch_optical_flow.h
      include/basalt/optical_flow/patterns.h
      include/basalt/optimization/accumulator.h
      include/basalt/optimization/linearize.h
      include/basalt/optimization/poses_linearize.h
      include/basalt/optimization/poses_optimize.h
      include/basalt/optimization/spline_linearize.h
      include/basalt/optimization/spline_optimize.h
      include/basalt/utils/ba_utils.h
      include/basalt/utils/cast_utils.hpp
      include/basalt/utils/common_types.h
      include/basalt/utils/filesystem.h
      include/basalt/utils/format.hpp
      include/basalt/utils/imu_types.h
      include/basalt/utils/keypoints.h
      include/basalt/utils/nfr.h
      include/basalt/utils/sim_utils.h
      include/basalt/utils/system_utils.h
      include/basalt/utils/test_utils.h
      include/basalt/utils/time_utils.hpp
      include/basalt/utils/tracks.h
      include/basalt/utils/union_find.h
      include/basalt/utils/vio_config.h
      include/basalt/utils/vis_utils.h
      include/basalt/vi_estimator/ba_base.h
      include/basalt/vi_estimator/landmark_database.h
      include/basalt/vi_estimator/marg_helper.h
      include/basalt/vi_estimator/nfr_mapper.h
      include/basalt/vi_estimator/sc_ba_base.h
      include/basalt/vi_estimator/sqrt_ba_base.h
      include/basalt/vi_estimator/sqrt_keypoint_vio.h
      include/basalt/vi_estimator/sqrt_keypoint_vo.h
      include/basalt/vi_estimator/vio_estimator.h
  PRIVATE
    src/calibration/vignette.cpp
    src/linearization/landmark_block.cpp
    src/linearization/linearization_abs_qr.cpp
    src/linearization/linearization_abs_sc.cpp
    src/linearization/linearization_base.cpp
    src/linearization/linearization_rel_sc.cpp
    src/optical_flow/optical_flow.cpp
    src/utils/keypoints.cpp
    src/utils/system_utils.cpp
    src/utils/time_utils.cpp
    src/utils/vio_config.cpp
    src/vi_estimator/ba_base.cpp
    src/vi_estimator/landmark_database.cpp
    src/vi_estimator/marg_helper.cpp
    src/vi_estimator/nfr_mapper.cpp
    src/vi_estimator/sc_ba_base.cpp
    src/vi_estimator/sqrt_ba_base.cpp
    src/vi_estimator/sqrt_keypoint_vio.cpp
    src/vi_estimator/sqrt_keypoint_vo.cpp
    src/vi_estimator/vio_estimator.cpp
)

if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  target_sources(basalt
    PUBLIC
      FILE_SET HEADERS
      BASE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include
      FILES
        include/basalt/calibration/aprilgrid.h
        include/basalt/calibration/calibration_helper.h
        include/basalt/calibration/cam_calib.h
        include/basalt/calibration/cam_imu_calib.h
        include/basalt/device/rs_t265.h
        include/basalt/io/dataset_io.h
        include/basalt/io/dataset_io_euroc.h
        include/basalt/io/dataset_io_kitti.h
        include/basalt/io/dataset_io_rosbag.h
        include/basalt/io/dataset_io_uzh.h
        include/basalt/io/marg_data_io.h
    PRIVATE
      src/calibration/aprilgrid.cpp
      src/calibration/calibraiton_helper.cpp
      src/io/dataset_io.cpp
      src/io/marg_data_io.cpp
  )
endif()

target_link_libraries(basalt
  PUBLIC ${STD_CXX_FS} Eigen3::Eigen basalt::opencv basalt::basalt-headers TBB::tbb
  PRIVATE magic_enum::magic_enum apriltag opengv nlohmann_json::nlohmann_json fmt::fmt)
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  target_link_libraries(basalt PRIVATE rosbag)
endif()

target_include_directories(basalt SYSTEM PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
if(MAGIC_ENUM_ROOT)
    target_include_directories(basalt PRIVATE "${MAGIC_ENUM_ROOT}")
else()
    message(FATAL_ERROR "Cannot locate magic_enum headers")
endif()

if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  target_include_directories(basalt PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/ros/include>)
endif()

# ------------------------------------------------------------------------
# Non-stripped executables / tests
# ------------------------------------------------------------------------
if(NOT BASALT_COMPILE_CKAAR_STRIPPED)
  add_executable(basalt_calibrate src/calibrate.cpp src/calibration/cam_calib.cpp)
  target_link_libraries(basalt_calibrate basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
  basalt_add_common_compile_options(basalt_calibrate)
  basalt_add_gui_link_options(basalt_calibrate)
  basalt_add_macos_runtime_paths(basalt_calibrate)

  add_executable(basalt_calibrate_imu src/calibrate_imu.cpp src/calibration/cam_imu_calib.cpp)
  target_link_libraries(basalt_calibrate_imu basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
  basalt_add_common_compile_options(basalt_calibrate_imu)
  basalt_add_gui_link_options(basalt_calibrate_imu)
  basalt_add_macos_runtime_paths(basalt_calibrate_imu)

  add_executable(basalt_vio_sim src/vio_sim.cpp)
  target_link_libraries(basalt_vio_sim basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
  basalt_add_common_compile_options(basalt_vio_sim)
  basalt_add_gui_link_options(basalt_vio_sim)
  basalt_add_macos_runtime_paths(basalt_vio_sim)

  add_executable(basalt_mapper_sim src/mapper_sim.cpp)
  target_link_libraries(basalt_mapper_sim basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
  basalt_add_common_compile_options(basalt_mapper_sim)
  basalt_add_gui_link_options(basalt_mapper_sim)
  basalt_add_macos_runtime_paths(basalt_mapper_sim)

  # mapper sim native doesn't use template free interface
  if(BASALT_INSTANTIATIONS_DOUBLE)
    add_executable(basalt_mapper_sim_naive src/mapper_sim_naive.cpp)
    target_link_libraries(basalt_mapper_sim_naive basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
    basalt_add_common_compile_options(basalt_mapper_sim_naive)
    basalt_add_gui_link_options(basalt_mapper_sim_naive)
    basalt_add_macos_runtime_paths(basalt_mapper_sim_naive)
  endif()

  add_executable(basalt_mapper src/mapper.cpp)
  target_link_libraries(basalt_mapper basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
  basalt_add_common_compile_options(basalt_mapper)
  basalt_add_gui_link_options(basalt_mapper)
  basalt_add_macos_runtime_paths(basalt_mapper)

  add_executable(basalt_opt_flow src/opt_flow.cpp)
  target_link_libraries(basalt_opt_flow basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
  basalt_add_common_compile_options(basalt_opt_flow)
  basalt_add_gui_link_options(basalt_opt_flow)
  basalt_add_macos_runtime_paths(basalt_opt_flow)

  add_executable(basalt_vio src/vio.cpp)
  target_link_libraries(basalt_vio basalt ${BASALT_PANGO_TARGETS} basalt::cli11 fmt::fmt)
  basalt_add_common_compile_options(basalt_vio)
  basalt_add_gui_link_options(basalt_vio)
  basalt_add_macos_runtime_paths(basalt_vio)

  add_executable(basalt_time_alignment src/time_alignment.cpp)
  target_link_libraries(basalt_time_alignment basalt ${BASALT_PANGO_TARGETS} basalt::cli11)
  basalt_add_common_compile_options(basalt_time_alignment)
  basalt_add_gui_link_options(basalt_time_alignment)
  basalt_add_macos_runtime_paths(basalt_time_alignment)

  add_executable(basalt_kitti_eval src/kitti_eval.cpp)
  target_link_libraries(basalt_kitti_eval basalt::basalt-headers basalt::cli11)
  basalt_add_common_compile_options(basalt_kitti_eval)
  basalt_add_static_link_options(basalt_kitti_eval)
  basalt_add_macos_runtime_paths(basalt_kitti_eval)

  find_package(realsense2 QUIET)
  if(realsense2_FOUND)
    add_executable(basalt_rs_t265_record src/rs_t265_record.cpp src/device/rs_t265.cpp)
    target_link_libraries(basalt_rs_t265_record basalt realsense2::realsense2 ${BASALT_PANGO_TARGETS} basalt::cli11)
    basalt_add_common_compile_options(basalt_rs_t265_record)
    basalt_add_gui_link_options(basalt_rs_t265_record)
    basalt_add_macos_runtime_paths(basalt_rs_t265_record)

    add_executable(basalt_rs_t265_vio src/rs_t265_vio.cpp src/device/rs_t265.cpp)
    target_link_libraries(basalt_rs_t265_vio basalt realsense2::realsense2 ${BASALT_PANGO_TARGETS} basalt::cli11)
    basalt_add_common_compile_options(basalt_rs_t265_vio)
    basalt_add_gui_link_options(basalt_rs_t265_vio)
    basalt_add_macos_runtime_paths(basalt_rs_t265_vio)

    # librealsense-file and rosbag both provide overlapping rosbag symbols.
    # Keep this relaxation local to the two T265 tools.
    if(UNIX AND NOT APPLE)
      find_library(UDEV_LIB udev REQUIRED)
      target_link_libraries(basalt_rs_t265_record "${UDEV_LIB}")
      target_link_libraries(basalt_rs_t265_vio "${UDEV_LIB}")
      target_link_options(basalt_rs_t265_record PRIVATE "LINKER:--allow-multiple-definition")
      target_link_options(basalt_rs_t265_vio PRIVATE "LINKER:--allow-multiple-definition")
    elseif(APPLE)
      find_library(SECURITY_FRAMEWORK Security REQUIRED)
      target_link_libraries(basalt_rs_t265_record "${SECURITY_FRAMEWORK}")
      target_link_libraries(basalt_rs_t265_vio "${SECURITY_FRAMEWORK}")
      target_link_options(basalt_rs_t265_record PRIVATE "LINKER:-ld_classic")
      target_link_options(basalt_rs_t265_vio PRIVATE "LINKER:-ld_classic")
      target_link_options(basalt_rs_t265_record PRIVATE "LINKER:-multiply_defined,suppress")
      target_link_options(basalt_rs_t265_vio PRIVATE "LINKER:-multiply_defined,suppress")
    endif()
  endif()

  # NOTE: basalt itself is intentionally omitted here — it's already
  # installed (with its FILE_SET HEADERS) by the unconditional
  # install(TARGETS basalt ...) above. Installing the same FILE_SET twice
  # for one target is a hard CMake error, not just redundant.
  install(TARGETS basalt_calibrate basalt_calibrate_imu basalt_vio_sim basalt_mapper_sim basalt_mapper basalt_opt_flow basalt_vio basalt_kitti_eval basalt_time_alignment
    EXPORT BasaltTargets
    RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)

  if(BASALT_INSTANTIATIONS_DOUBLE)
    install(TARGETS basalt_mapper_sim_naive
      EXPORT BasaltTargets
      RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
      LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
      ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)
  endif()

  if(realsense2_FOUND)
    install(TARGETS basalt_rs_t265_record basalt_rs_t265_vio
      EXPORT BasaltTargets
      RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
      LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
      ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)
  endif()

  file(GLOB SCRIPTS_TO_INSTALL "${CMAKE_CURRENT_SOURCE_DIR}/scripts/basalt_*.py")
  install(PROGRAMS ${SCRIPTS_TO_INSTALL} DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)

  file(GLOB CONFIG_FILES "${CMAKE_CURRENT_SOURCE_DIR}/data/*.json")
  install(FILES ${CONFIG_FILES}
    DESTINATION ${CMAKE_INSTALL_PREFIX}/etc/basalt)
  install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/data/udev"
    DESTINATION ${CMAKE_INSTALL_PREFIX}/etc/basalt)

  add_subdirectory(test)
endif()
