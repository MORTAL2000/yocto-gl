cmake_minimum_required (VERSION 3.9)

project (yocto-gl)

option(YOCTO_OPENGL "Build OpenGL apps" ON)
option(YOCTO_EMBREE "Use Intel's Embree raytracer" OFF)
option(YOCTO_TESTING "Enable testing" ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE "Release")
endif()

set(CMAKE_BINARY_DIR ${CMAKE_SOURCE_DIR}/bin)
if(NOT GENERATOR_IS_MULTI_CONFIG AND CMAKE_BUILD_TYPE MATCHES Debug)
  set(CMAKE_BINARY_DIR ${CMAKE_SOURCE_DIR}/bin/debug)
endif(NOT GENERATOR_IS_MULTI_CONFIG AND CMAKE_BUILD_TYPE MATCHES Debug)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
if(GENERATOR_IS_MULTI_CONFIG)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR})
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR})
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR})
endif(GENERATOR_IS_MULTI_CONFIG)

# platform compile flags
if(APPLE)
  add_definitions(-Wall)
endif(APPLE)
if(MSVC)
  add_definitions(/D_CRT_SECURE_NO_WARNINGS /wd4018 /wd4244 /wd4305 /wd4800 /wd4267)
  set(CMAKE_CXX_FLAGS "/EHsc")
endif(MSVC)

add_library(yocto 
  yocto/yocto_math.h yocto/yocto_commonio.h 
  yocto/yocto_modelio.h yocto/yocto_modelio.cpp
  yocto/yocto_bvh.h yocto/yocto_bvh.cpp
  yocto/yocto_shape.h yocto/yocto_shape.cpp 
  yocto/yocto_image.h yocto/yocto_image.cpp
  yocto/yocto_trace.h yocto/yocto_trace.cpp
  yocto/yocto_sceneio.h yocto/yocto_sceneio.cpp
)

if(UNIX AND NOT APPLE)
  find_package(Threads REQUIRED)
  target_link_libraries(yocto Threads::Threads)
endif(UNIX AND NOT APPLE)

if(YOCTO_EMBREE)
  add_definitions(-DYOCTO_EMBREE)
  if(APPLE)
    include_directories(/usr/local/include)
    link_directories(/usr/local/lib)
    target_link_libraries(yocto libembree3.dylib)
  endif(APPLE)
  if(MSVC)
    include_directories("/Program Files/Intel/Embree3 X64/include" "C:/Program Files/Intel/Embree3 X64/include")
    link_directories("/Program Files/Intel/Embree3 X64/lib" "C:/Program Files/Intel/Embree3 X64/lib")
    target_link_libraries(yocto embree3 tbb tbbmalloc)
  endif(MSVC)
endif(YOCTO_EMBREE)

add_executable(yscntrace apps/yscntrace.cpp)
add_executable(yscnproc apps/yscnproc.cpp)
add_executable(yimgproc apps/yimgproc.cpp)
add_executable(ymshproc apps/ymshproc.cpp)

target_link_libraries(yscntrace yocto)
target_link_libraries(yscnproc yocto)
target_link_libraries(yimgproc yocto)
target_link_libraries(ymshproc yocto)

if(YOCTO_OPENGL)
  add_definitions(-DYOCTO_OPENGL)
  find_package(OpenGL REQUIRED)
  set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL " " FORCE)
  set(GLFW_BUILD_TESTS OFF CACHE BOOL " " FORCE)
  set(GLFW_BUILD_DOCS OFF CACHE BOOL " " FORCE)
  set(GLFW_INSTALL OFF CACHE BOOL " " FORCE)
  add_subdirectory(apps/ext/glfw glfw)

  include_directories(apps/)
  add_definitions(-DIMGUI_IMPL_OPENGL_LOADER_GLAD)

  add_library(yocto_opengl apps/yocto_opengl.h apps/yocto_opengl.cpp 
    apps/ext/imgui/imgui.cpp apps/ext/imgui/imgui_draw.cpp apps/ext/imgui/imgui_widgets.cpp
    apps/ext/imgui/imgui_impl_glfw.cpp apps/ext/imgui/imgui_impl_opengl3.cpp apps/ext/glad/glad.c)
  target_link_libraries(yocto_opengl glfw ${OPENGL_gl_LIBRARY})
  if(UNIX AND NOT APPLE)
    target_link_libraries(yocto_opengl dl)
  endif(UNIX AND NOT APPLE)

  add_executable(yscnview apps/yscnview.cpp)
  add_executable(yimshproc apps/yimshproc.cpp)
  add_executable(yscnitrace apps/yscnitrace.cpp)
  add_executable(yscnitraces apps/yscnitraces.cpp)
  add_executable(yimgview apps/yimgview.cpp)
  add_executable(yimgviews apps/yimgviews.cpp)

  target_link_libraries(yscnview yocto yocto_opengl ${GL_EXTRA_LIBRARIES})
  target_link_libraries(yimshproc yocto yocto_opengl ${GL_EXTRA_LIBRARIES})
  target_link_libraries(yscnitrace yocto yocto_opengl ${GL_EXTRA_LIBRARIES})
  target_link_libraries(yscnitraces yocto yocto_opengl ${GL_EXTRA_LIBRARIES})
  target_link_libraries(yimgview yocto yocto_opengl ${GL_EXTRA_LIBRARIES})
  target_link_libraries(yimgviews yocto yocto_opengl ${GL_EXTRA_LIBRARIES})
endif(YOCTO_OPENGL)

if(YOCTO_TESTING)
  enable_testing()
  set(test-yimgproc ${CMAKE_SOURCE_DIR}/tests/_results/test-yimgproc.sh)
  set(test-yscntrace ${CMAKE_SOURCE_DIR}/tests/_results/test-yscntrace.sh)
  set(test-yscnproc ${CMAKE_SOURCE_DIR}/tests/_results/test-yscnproc.sh)
  set(testdir ${CMAKE_SOURCE_DIR}/tests)
  set(outdir ${CMAKE_SOURCE_DIR}/tests/_output)
  set(resdir ${CMAKE_SOURCE_DIR}/tests/_results)
  set(diffdir ${CMAKE_SOURCE_DIR}/tests/_difference)
  # image io
  foreach(filename "uvgrid.png")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yimgproc-${filename}
            COMMAND ${test-yimgproc} ${testdir}/textures/${filename} ${outdir}/yimgproc-${name}.png ${resdir}/yimgproc-${name}.png ${diffdir}/yimgproc-${name}.png
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})    
  endforeach()
  # procedurla images
  foreach(name "images1" "images2")
    add_test(NAME yimgproc-${name}
            COMMAND ${test-yimgproc} ${name}.ypreset ${outdir}/yimgproc-${name}.png ${resdir}/yimgproc-${name}.png ${diffdir}/yimgproc-${name}.png
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})    
  endforeach()
  # eyelight
  foreach(filename "features1.yaml" "materials1.yaml" "materials2.yaml" "materials3.yaml" "materials4.yaml" "shapes1.yaml" "shapes2.yaml" "shapes3.yaml" "arealights1.yaml" "environments1.yaml")
    set(tracer "eyelight")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yscntrace-${tracer}-${filename}
            COMMAND ${test-yscntrace} ${testdir}/${filename} ${outdir}/yscntrace-${tracer}-${name}.png ${resdir}/yscntrace-${tracer}-${name}.png ${diffdir}/yscntrace-${tracer}-${name}.png "-t ${tracer} -s 4 -r 640"
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endforeach()
  # path
  foreach(filename "features1.yaml" "materials1.yaml" "materials2.yaml" "materials3.yaml" "materials4.yaml" "shapes1.yaml" "shapes2.yaml" "shapes3.yaml" "arealights1.yaml" "environments1.yaml")
    set(tracer "path")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yscntrace-${tracer}-${filename}
            COMMAND ${test-yscntrace} ${testdir}/${filename} ${outdir}/yscntrace-${tracer}-${name}.png ${resdir}/yscntrace-${tracer}-${name}.png ${diffdir}/yscntrace-${tracer}-${name}.png "-t ${tracer} -s 16 -r 640"
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endforeach()
  # naive
  foreach(filename "features1.yaml" "materials1.yaml" "materials2.yaml" "materials3.yaml" "shapes1.yaml" "shapes2.yaml" "shapes3.yaml" "arealights1.yaml" "environments1.yaml")
    set(tracer "naive")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yscntrace-${tracer}-${filename}
            COMMAND ${test-yscntrace} ${testdir}/${filename} ${outdir}/yscntrace-${tracer}-${name}.png ${resdir}/yscntrace-${tracer}-${name}.png ${diffdir}/yscntrace-${tracer}-${name}.png "-t ${tracer} -s 16 -r 640"
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endforeach()
  if(YOCTO_EMBREE)
  # embree
  foreach(filename "features1.yaml" "materials1.yaml" "materials2.yaml" "materials3.yaml" "shapes1.yaml" "shapes2.yaml" "shapes3.yaml" "arealights1.yaml" "environments1.yaml")
    set(tracer "path")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yscntrace-embree-${tracer}-${filename}
            COMMAND ${test-yscntrace} ${testdir}/${filename} ${outdir}/yscntrace-embree-${tracer}-${name}.png ${resdir}/yscntrace-embree-${tracer}-${name}.png ${diffdir}/yscntrace-embree-${tracer}-${name}.png "-t ${tracer} --bvh-embree -s 16 -r 640"
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endforeach()
  endif(YOCTO_EMBREE)
  # yaml output
  foreach(filename "features1.yaml" "shapes1.yaml" "shapes2.yaml" "shapes3.yaml")
    set(format "yaml")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yscnproc-${format}-${filename}
            COMMAND ${test-yscnproc} ${testdir}/${filename} ${outdir}/${format}/${name} ${outdir}/${format}/${name}/${name}.${format} ${outdir}/yscnproc-${format}-${name}.png ${resdir}/yscnproc-${format}-${name}.png ${diffdir}/yscnproc-${format}-${name}.png "-t path -s 8 -r 640"
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endforeach()
  # obj output
  foreach(filename "features1.yaml" "shapes1.yaml" "shapes2.yaml" "shapes3.yaml")
    set(format "obj")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yscnproc-${format}-${filename}
            COMMAND ${test-yscnproc} ${testdir}/${filename} ${outdir}/${format}/${name} ${outdir}/${format}/${name}/${name}.${format} ${outdir}/yscnproc-${format}-${name}.png ${resdir}/yscnproc-${format}-${name}.png ${diffdir}/yscnproc-${format}-${name}.png "-t path -s 8 -r 640"
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endforeach()
  # pbrt output
  foreach(filename "features1.yaml" "shapes1.yaml" "shapes2.yaml" "shapes3.yaml")
    set(format "pbrt")
    get_filename_component(name ${filename} NAME_WE)
    add_test(NAME yscnproc-${format}-${filename}
            COMMAND ${test-yscnproc} ${testdir}/${filename} ${outdir}/${format}/${name} ${outdir}/${format}/${name}/${name}.${format} ${outdir}/yscnproc-${format}-${name}.png ${resdir}/yscnproc-${format}-${name}.png ${diffdir}/yscnproc-${format}-${name}.png "-t path -s 8 -r 640"
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endforeach()
endif(YOCTO_TESTING)
