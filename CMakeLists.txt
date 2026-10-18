cmake_minimum_required(VERSION 3.16)
project(hcdformat NONE)

# ── ROS 2 / colcon build of the Rust-backed hcdformat package ────────────────────────────────────────
#
# hcdformat is Rust-canonical: the Python API IS the native binding (`import hcdf` loads the extension,
# which ships as `hcdf/__init__.abi3.so`) and the `hcdf` CLI is the RUST binary. So this is an ament_cmake
# package that builds the binding + the CLI FROM SOURCE with a Rust toolchain; it is deliberately NOT an
# ament_python (pure-Python) package any more. Build-machine requirements (developer/CI tools; no
# universal rosdep key across distros, so they are documented here + in the README rather than declared
# as <depend>s in package.xml):
#
#   * a Rust toolchain (cargo + rustc, stable), install via rustup: https://rustup.rs
#   * maturin >= 1.7, < 2, via `pip install 'maturin>=1.7,<2'` (builds the abi3 binding wheel)
#
# What colcon installs:
#   * the abi3 binding wheel (maturin) unpacked into this prefix's python site-packages → `import hcdf`
#   * the native `hcdf` CLI binary into lib/hcdformat → `ros2 run hcdformat hcdf ...`
#   * schema, completion, and connectivity registry data into share/hcdformat

find_package(ament_cmake REQUIRED)
find_package(Python3 REQUIRED COMPONENTS Interpreter)

# The Rust toolchain gate: a clear configure-time error if cargo is missing (see the rustup note above).
find_program(CARGO_EXECUTABLE cargo REQUIRED
  DOC "Rust cargo; install a stable toolchain via rustup (https://rustup.rs)")

set(_repo "${CMAKE_CURRENT_SOURCE_DIR}")
set(_py_manifest "${_repo}/rust/hcdformat-py/Cargo.toml")
set(_rs_manifest "${_repo}/rust/hcdformat-rs/Cargo.toml")
set(_wheel_dir "${CMAKE_CURRENT_BINARY_DIR}/wheel")
set(_cargo_target "${CMAKE_CURRENT_BINARY_DIR}/cargo-target")

# ── Cross-compilation (conda/rattler-build packaging) ─────────────────────────────────────────────
# rattler-build exports `target_platform` for the package being built; when it names a platform that
# differs from the build machine, map it to the Rust target triple and pass --target to BOTH maturin
# and cargo. The target std comes from the conda rust-std-<triple> package; the linker is the conda
# cross C compiler ($CC). With no target_platform in the environment (plain colcon/rustup builds),
# everything below stays empty and the build is native.
set(_rust_target "")
if(DEFINED ENV{target_platform})
  if("$ENV{target_platform}" STREQUAL "linux-aarch64")
    set(_rust_target "aarch64-unknown-linux-gnu")
  elseif("$ENV{target_platform}" STREQUAL "osx-arm64")
    set(_rust_target "aarch64-apple-darwin")
  elseif("$ENV{target_platform}" STREQUAL "osx-64")
    set(_rust_target "x86_64-apple-darwin")
  endif()
endif()

set(_rust_env "")                                   # command prefix: cmake -E env <cross vars>
set(_rust_target_args "")                          # --target <triple>
set(_cargo_release_dir "${_cargo_target}/release") # cargo nests <triple>/ under it when crossing
if(_rust_target)
  if(NOT DEFINED ENV{CC})
    message(FATAL_ERROR
      "hcdformat: cross target ${_rust_target} needs \$CC (the conda cross C compiler) in the environment")
  endif()
  string(TOUPPER "${_rust_target}" _rust_target_upper)
  string(REPLACE "-" "_" _rust_target_upper "${_rust_target_upper}")
  set(_rust_env "${CMAKE_COMMAND}" -E env
    "CARGO_TARGET_${_rust_target_upper}_LINKER=$ENV{CC}"
    "PYO3_CROSS_PYTHON_VERSION=${Python3_VERSION_MAJOR}.${Python3_VERSION_MINOR}")
  set(_rust_target_args --target "${_rust_target}")
  set(_cargo_release_dir "${_cargo_target}/${_rust_target}/release")
  message(STATUS "hcdformat: cross-compiling Rust for ${_rust_target} (linker: $ENV{CC})")
endif()

# 1) Build the abi3 binding wheel with maturin. maturin compiles the Rust core + PyO3 binding and packs
#    it as `hcdf/__init__.abi3.so` (the extension IS the package) alongside the generated type stubs and
#    the data files, one abi3 wheel with no shipped `.py`.
add_custom_command(
  OUTPUT "${_wheel_dir}/.built"
  COMMAND ${_rust_env} "${Python3_EXECUTABLE}" -m maturin build --release
          --manifest-path "${_py_manifest}" --out "${_wheel_dir}"
          ${_rust_target_args}
  COMMAND "${CMAKE_COMMAND}" -E touch "${_wheel_dir}/.built"
  COMMENT "hcdformat: building the abi3 Rust binding wheel (maturin)"
  VERBATIM)

# 2) Build the native `hcdf` CLI binary (cargo, feature=cli) so `ros2 run hcdformat hcdf` works. This is
#    the SAME Rust CLI the wheel's console script drives (via the run_cli binding): one implementation.
add_custom_command(
  OUTPUT "${_cargo_release_dir}/hcdf"
  COMMAND ${_rust_env} "${CARGO_EXECUTABLE}" build --release --features cli --bin hcdf
          --manifest-path "${_rs_manifest}" --target-dir "${_cargo_target}"
          ${_rust_target_args}
  COMMENT "hcdformat: building the native hcdf CLI binary (cargo --features cli)"
  VERBATIM)

# 1b) Unpack the wheel INTO THE BUILD TREE (a wheel is a zip; stdlib zipfile is enough). This must
#     NOT happen in install(CODE)/pip at install time: the pixi-build-ros backend builds its package
#     from CMake's install manifest, so files dropped into the prefix by an install-time subprocess
#     never reach the .conda. Unpacking here lets install(DIRECTORY) below list every payload file in
#     the manifest — and sidesteps pip's platform-tag check when cross-compiling.
set(_wheel_unpacked "${CMAKE_CURRENT_BINARY_DIR}/wheel-unpacked")
set(_unpack_script "${CMAKE_CURRENT_BINARY_DIR}/unpack_wheel.cmake")
file(WRITE "${_unpack_script}"
"file(GLOB _whl \"${_wheel_dir}/*.whl\")
if(NOT _whl)
  message(FATAL_ERROR \"hcdformat: no wheel found in ${_wheel_dir}; did the maturin build run?\")
endif()
list(GET _whl 0 _whl0)
message(STATUS \"hcdformat: unpacking binding wheel \${_whl0} -> ${_wheel_unpacked}\")
execute_process(
  COMMAND \"${Python3_EXECUTABLE}\" -m zipfile -e \"\${_whl0}\" \"${_wheel_unpacked}\"
  RESULT_VARIABLE _rc)
if(NOT _rc EQUAL 0)
  message(FATAL_ERROR \"hcdformat: wheel unpack failed (rc=\${_rc})\")
endif()
")

add_custom_command(
  OUTPUT "${_wheel_dir}/.unpacked"
  COMMAND "${CMAKE_COMMAND}" -E remove_directory "${_wheel_unpacked}"  # drop any stale payload first
  COMMAND "${CMAKE_COMMAND}" -P "${_unpack_script}"
  COMMAND "${CMAKE_COMMAND}" -E touch "${_wheel_dir}/.unpacked"
  DEPENDS "${_wheel_dir}/.built"
  COMMENT "hcdformat: unpacking the binding wheel"
  VERBATIM)

add_custom_target(hcdformat_build ALL
  DEPENDS "${_wheel_dir}/.built" "${_wheel_dir}/.unpacked" "${_cargo_release_dir}/hcdf")

# 3a) Install the binding: the unpacked wheel payload -> this prefix's python site-packages. Every
#     payload file lands in CMake's install manifest (the backend packages exactly that set).
set(_py_site "lib/python${Python3_VERSION_MAJOR}.${Python3_VERSION_MINOR}/site-packages")
install(DIRECTORY "${_wheel_unpacked}/" DESTINATION "${_py_site}")

# 3b) Install the native CLI binary and the schema/reference data.
install(PROGRAMS "${_cargo_release_dir}/hcdf" DESTINATION lib/${PROJECT_NAME})
install(FILES
  "${_repo}/hcdf.xsd"
  "${_repo}/hcdf-stream-profile.xsd"
  "${_repo}/hcdf.schema.json"
  "${_repo}/hcdf.completions.json"
  DESTINATION share/${PROJECT_NAME})
install(FILES
  "${_repo}/registries/connectivity-profiles.json"
  "${_repo}/registries/connectivity-profile-registry.schema.json"
  DESTINATION share/${PROJECT_NAME}/registries)

ament_package()
