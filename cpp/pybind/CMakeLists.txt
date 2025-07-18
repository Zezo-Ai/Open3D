if (NOT Python3_EXECUTABLE)
    message(FATAL_ERROR "Python 3 not found in top level file")
endif()

# We need to get python version to configure some meta files
set(PYTHON_VERSION "${Python3_VERSION_MAJOR}.${Python3_VERSION_MINOR}")


pybind11_add_module(pybind)

add_subdirectory(camera)
add_subdirectory(core)
add_subdirectory(data)
add_subdirectory(geometry)
add_subdirectory(io)
add_subdirectory(ml)
add_subdirectory(pipelines)
add_subdirectory(t)
add_subdirectory(t/geometry)
add_subdirectory(t/io)
add_subdirectory(t/pipelines)
add_subdirectory(utility)
add_subdirectory(visualization)

target_sources(pybind PRIVATE
    docstring.cpp
    open3d_pybind.cpp
    pybind_utils.cpp
)

# Include with `#include "pybind/geometry/xxx.h`
target_include_directories(pybind PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/.."
)

# Suppress Pybind11 warnings
target_include_directories(pybind SYSTEM PRIVATE
    ${PYBIND11_INCLUDE_DIR}
    ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}
)

open3d_show_and_abort_on_warning(pybind)
open3d_set_global_properties(pybind)
target_link_libraries(pybind PRIVATE Open3D::Open3D)


if (WIN32)
    target_link_options(pybind PUBLIC "/force:multiple")
    # TODO(Sameer): Only export PyInit_pybind, open3d_core_cuda_device_count
elseif (APPLE)
    file(GENERATE OUTPUT pybind.map CONTENT
        [=[_PyInit_pybind
open3d_core_cuda_device_count
        ]=])
    target_link_options(pybind PRIVATE $<$<CONFIG:Release>:
        -Wl,-exported_symbols_list
        "${CMAKE_CURRENT_BINARY_DIR}/pybind.map" >)
elseif (UNIX)   # Linux
    file(GENERATE OUTPUT pybind.map CONTENT
        [=[{
    global:
        PyInit_pybind;
        open3d_core_cuda_device_count;
    local:
        *;
};]=])
    target_link_options(pybind PRIVATE $<$<CONFIG:Release>:
        "-Wl,--version-script=${CMAKE_CURRENT_BINARY_DIR}/pybind.map" >)
    target_link_options(pybind PRIVATE "-flto=auto")
endif()


# At `make`: open3d.so (or the equivalents) will be created at
# PYTHON_COMPILED_MODULE_DIR. The default location is
# `build/lib/${CMAKE_BUILD_TYPE}/Python/{cpu|cuda}`
set(PYTHON_COMPILED_MODULE_DIR
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/Python/$<IF:$<BOOL:${BUILD_CUDA_MODULE}>,cuda,cpu>")

if (UNIX AND NOT APPLE)
    # Use RPATH instead of RUNPATH in pybind so that needed libc++.so can find child dependant libc++abi.so in RPATH
    # https://stackoverflow.com/questions/69662319/managing-secondary-dependencies-of-shared-libraries
    target_link_options(pybind PRIVATE "LINKER:--disable-new-dtags")
endif()
set_target_properties(pybind PROPERTIES
                      FOLDER "Python"
                      LIBRARY_OUTPUT_DIRECTORY "${PYTHON_COMPILED_MODULE_DIR}"
                      ARCHIVE_OUTPUT_DIRECTORY "${PYTHON_COMPILED_MODULE_DIR}")

if (BUILD_SHARED_LIBS)
    if (WIN32)    # CMake does not add soversion suffix to WIN32 DLLs
        set(_libopen3d_soname "Open3D.dll")
    elseif(APPLE)
        set(_libopen3d_soname "libOpen3D.${OPEN3D_ABI_VERSION}.dylib")
    else()
        set(_libopen3d_soname "libOpen3D.so.${OPEN3D_ABI_VERSION}")
    endif()
    add_custom_command(TARGET pybind POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            $<TARGET_FILE:Open3D> $<TARGET_FILE_DIR:pybind>/${_libopen3d_soname}
    )
endif()
# Include additional libraries that may be absent from the user system
# eg: libc++.so, libc++abi.so (needed by filament) for Linux.
# libc++.so is a linker script including libc++.so.1 and libc++abi.so, so append 1 to libc++.so
set(PYTHON_EXTRA_LIBRARIES $<TARGET_FILE:TBB::tbb>)
if (BUILD_GUI AND CMAKE_SYSTEM_NAME STREQUAL "Linux")
    list(APPEND PYTHON_EXTRA_LIBRARIES ${CPP_LIBRARY}.1 ${CPPABI_LIBRARY})
endif()
if (WITH_OPENMP AND APPLE AND NOT BUILD_SHARED_LIBS)
# Package libomp v11.1.0, if it is not installed. Later version cause crash on
# x86_64 if PyTorch is already imported. Case of shared libopen3d.dylib is not
# handled.
# https://github.com/microsoft/LightGBM/issues/4229
    list(APPEND PYTHON_EXTRA_LIBRARIES ${OpenMP_libomp_LIBRARY})
    execute_process(COMMAND brew list --versions libomp
        COMMAND cut "-d\ " -f2
        OUTPUT_VARIABLE libomp_ver OUTPUT_STRIP_TRAILING_WHITESPACE)
    if(libomp_ver VERSION_GREATER_EQUAL 12.0.0)
        message(SEND_ERROR "libomp ${libomp_ver} found, which can lead to interoperability problems with other Python libraries using libomp.  Please use libomp v11.1 to build Open3D.")
    endif()
    file(GENERATE OUTPUT update_pybind_libomp.sh
        CONTENT [=[libomp_library=$(dyld_info -dependents "$<TARGET_FILE:pybind>" |
grep libomp | tr -d '[:space:]')
install_name_tool -change $libomp_library @rpath/$(basename $libomp_library) \
"$<TARGET_FILE:pybind>"]=])
    add_custom_command(TARGET pybind POST_BUILD
        COMMAND bash update_pybind_libomp.sh
        COMMENT "Updating pybind to use packaged libomp")
endif()

# Use `make python-package` to create the python package in the build directory
# The python package will be created at PYTHON_PACKAGE_DST_DIR. It contains:
# 1) Pure-python code and misc files, copied from python/package
# 2) The compiled python-C++ module, i.e. open3d.so (or the equivalents)
# 3) Configured files and supporting files
# Note: `make python-package` clears PYTHON_COMPILED_MODULE_DIR first every time
set(PYTHON_PACKAGE_SRC_DIR "${PROJECT_SOURCE_DIR}/python")
set(PYTHON_PACKAGE_DST_DIR "${CMAKE_BINARY_DIR}/lib/python_package")
message(STATUS "PYPI_PACKAGE_NAME: ${PYPI_PACKAGE_NAME}")


# add the open3d python module first
set(COMPILED_MODULE_PATH_LIST $<TARGET_FILE:pybind>)
# add the open3d DSO / DLL if shared
if (BUILD_SHARED_LIBS)
    list(APPEND COMPILED_MODULE_PATH_LIST $<TARGET_FILE_DIR:pybind>/${_libopen3d_soname})
endif()

set(GENERATED_OUTPUTS "")

# add additional optional compiled modules
if (BUILD_TENSORFLOW_OPS)
    list(APPEND COMPILED_MODULE_PATH_LIST $<TARGET_FILE:open3d_tf_ops> )
    add_custom_command(OUTPUT "${CMAKE_BINARY_DIR}/lib/ml/tf/python/ops/ops.py"
            COMMAND ${Python3_EXECUTABLE} generate_tf_ops_wrapper.py --input "${PYTHON_PACKAGE_SRC_DIR}/open3d/ml/tf/python/ops/ops.py.in" --output "${CMAKE_BINARY_DIR}/lib/ml/tf/python/ops/ops.py" --lib $<TARGET_FILE:open3d_tf_ops>
                        DEPENDS open3d_tf_ops
                        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                        COMMENT "Generating python ops.py" )

    list(APPEND GENERATED_OUTPUTS "${CMAKE_BINARY_DIR}/lib/ml/tf/python/ops/ops.py")
    # find tensorflow to get some info for the _build_config.py
    find_package(Tensorflow)
endif()

# add additional optional compiled modules
if (BUILD_PYTORCH_OPS)
    list( APPEND COMPILED_MODULE_PATH_LIST $<TARGET_FILE:open3d_torch_ops> )
    set(DEPENDENCIES_DIR "$<TARGET_FILE_DIR:TBB::tbb>")
    add_custom_command( OUTPUT "${CMAKE_BINARY_DIR}/lib/ml/torch/python/ops.py" "${CMAKE_BINARY_DIR}/lib/ml/torch/python/return_types.py"
            COMMAND ${Python3_EXECUTABLE} generate_torch_ops_wrapper.py --input_ops_py_in "${PYTHON_PACKAGE_SRC_DIR}/open3d/ml/torch/python/ops.py.in" --input_return_types_py_in "${PYTHON_PACKAGE_SRC_DIR}/open3d/ml/torch/python/return_types.py.in" --output_dir "${CMAKE_BINARY_DIR}/lib/ml/torch/python/" --lib $<TARGET_FILE:open3d_torch_ops> --tensorflow_ops_dir "${CMAKE_CURRENT_SOURCE_DIR}/../open3d/ml/tensorflow" --dependencies_dir "${DEPENDENCIES_DIR}"
                        DEPENDS open3d_torch_ops
                        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                        COMMENT "Generating python ops.py and return_types.py" )

    list(APPEND GENERATED_OUTPUTS "${CMAKE_BINARY_DIR}/lib/ml/torch/python/ops.py" "${CMAKE_BINARY_DIR}/lib/ml/torch/python/return_types.py")
    # get the pytorch version information again here for _build_config.py
    # because it is not safe to call find_package(Pytorch) again.
    execute_process(
        COMMAND ${Python3_EXECUTABLE} "-c"
                "import torch; print(torch.__version__, end='')"
        OUTPUT_VARIABLE Pytorch_VERSION)
endif()

if (BUNDLE_OPEN3D_ML)
    find_path(
        OPEN3D_ML_ROOT
        NAMES set_open3d_ml_root.sh
        HINTS $ENV{OPEN3D_ML_ROOT}
        DOC "Path to the Open3D-ML repo. This should be set if BUNDLE_OPEN3D_ML is enabled. Alternatively set an env var with the same name to populate this var."
        REQUIRED
        NO_DEFAULT_PATH
    )

    ExternalProject_Add(
        open3d_ml
        PREFIX "${CMAKE_BINARY_DIR}/open3d_ml"
        GIT_REPOSITORY "${OPEN3D_ML_ROOT}"
        GIT_TAG origin/main
        GIT_SHALLOW
        BUILD_IN_SOURCE ON
        # do not configure
        CONFIGURE_COMMAND ""
        # do not build
        BUILD_COMMAND ""
        # do not install
        INSTALL_COMMAND ""
        )
    list(APPEND GENERATED_OUTPUTS open3d_ml)
endif()

configure_file("_build_config.py.in"
               "${CMAKE_BINARY_DIR}/lib/_build_config.py.in")
file(GENERATE
    OUTPUT "${PYTHON_COMPILED_MODULE_DIR}/_build_config.py"
    INPUT "${CMAKE_BINARY_DIR}/lib/_build_config.py.in"
)

add_custom_target(python-package
    COMMAND ${CMAKE_COMMAND}
            -DPYTHON_PACKAGE_SRC_DIR=${PYTHON_PACKAGE_SRC_DIR}
            -DPYTHON_PACKAGE_DST_DIR=${PYTHON_PACKAGE_DST_DIR}
            -DPYTHON_COMPILED_MODULE_DIR=${PYTHON_COMPILED_MODULE_DIR}
            -DPYTHON_VERSION=${PYTHON_VERSION}
            "-DCOMPILED_MODULE_PATH_LIST=${COMPILED_MODULE_PATH_LIST}"
            "-DPYTHON_EXTRA_LIBRARIES=${PYTHON_EXTRA_LIBRARIES}"
            -DBUILD_JUPYTER_EXTENSION=${BUILD_JUPYTER_EXTENSION}
            -DBUILD_TENSORFLOW_OPS=${BUILD_TENSORFLOW_OPS}
            -DBUILD_PYTORCH_OPS=${BUILD_PYTORCH_OPS}
            -DBUNDLE_OPEN3D_ML=${BUNDLE_OPEN3D_ML}
            -DOPEN3D_ML_ROOT=${OPEN3D_ML_ROOT}
            -DBUILD_GUI=${BUILD_GUI}
            -DBUILD_CUDA_MODULE=${BUILD_CUDA_MODULE}
            -DBUILD_SYCL_MODULE=${BUILD_SYCL_MODULE}
            -DGUI_RESOURCE_DIR=${GUI_RESOURCE_DIR}
            -DPROJECT_EMAIL=${PROJECT_EMAIL}
            -DPROJECT_HOMEPAGE_URL=${PROJECT_HOMEPAGE_URL}
            -DPROJECT_DOCS=${PROJECT_DOCS}
            -DPROJECT_CODE=${PROJECT_CODE}
            -DPROJECT_ISSUES=${PROJECT_ISSUES}
            -DPROJECT_VERSION=${OPEN3D_VERSION_FULL}
            -DPROJECT_DESCRIPTION=${PROJECT_DESCRIPTION}
            -DPROJECT_VERSION_THREE_NUMBER=${PROJECT_VERSION_THREE_NUMBER}
            -DPYPI_PACKAGE_NAME=${PYPI_PACKAGE_NAME}
            -P ${CMAKE_CURRENT_SOURCE_DIR}/make_python_package.cmake
    VERBATIM
    DEPENDS ${GENERATED_OUTPUTS}
)

# Use `make pip-package` to create the pip package in the build directory
add_custom_target(pip-package
    COMMAND ${Python3_EXECUTABLE} setup.py bdist_wheel --dist-dir pip_package
    COMMAND echo "pip wheel created at ${PYTHON_PACKAGE_DST_DIR}/pip_package"
    WORKING_DIRECTORY ${PYTHON_PACKAGE_DST_DIR}
    DEPENDS python-package
)

# Use `make install-pip-package` to install pip wheel package to the current
# python environment.
add_custom_target(install-pip-package
    COMMAND ${CMAKE_COMMAND}
            -DPYTHON_PACKAGE_DST_DIR=${PYTHON_PACKAGE_DST_DIR}
            -DPython3_EXECUTABLE=${Python3_EXECUTABLE}
            -P ${CMAKE_CURRENT_SOURCE_DIR}/make_install_pip_package.cmake
    DEPENDS pip-package
)

# FOR DEBUGGING ONLY Use `make install-python-package` to build and install
# python package in the current python environment. This is substantially
# faster than `make install-pip-package`. However this approach does not create
# wheel or egg files and does not take care of dependencies thus not suitable
# for deployment.
# Ref: https://stackoverflow.com/a/33791008/1255535
add_custom_target(install-python-package
    COMMAND ${Python3_EXECUTABLE} setup.py install --single-version-externally-managed --root=/
    WORKING_DIRECTORY ${PYTHON_PACKAGE_DST_DIR}
    DEPENDS python-package
)
