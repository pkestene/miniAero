cmake_minimum_required(VERSION 3.1)

project(miniAero CXX C)

# options
option (USE_MPI "Activate / want MPI build" ON)

# C++11 is for Kokkos
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS OFF)


#
# default local cmake macro repository
#
#set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})


#
# Prevent from build in source tree
#
include(${PROJECT_SOURCE_DIR}/cmake/preventBuildInSource.cmake)

#
# Init build type: Release, Debug, ...
#
include(${PROJECT_SOURCE_DIR}/cmake/initBuildType.cmake)

#####################################################################
# External packages: MPI, ...
#####################################################################


#####################################################################
#find_package(MPI REQUIRED)
find_package(MPI)
if (USE_MPI)
  if(MPI_CXX_FOUND)
    message(STATUS "MPI support found")
    message(STATUS "MPI compile flags: " ${MPI_CXX_COMPILE_FLAGS})
    message(STATUS "MPI include path: " ${MPI_CXX_INCLUDE_PATH})
    message(STATUS "MPI LINK flags path: " ${MPI_CXX_LINK_FLAGS})
    message(STATUS "MPI libraries: " ${MPI_CXX_LIBRARIES})
    #include_directories(SYSTEM ${MPI_CXX_INCLUDE_PATH})

    #set(CMAKE_EXE_LINKER_FLAGS ${MPI_CXX_LINK_FLAGS})
    find_program(OMPI_INFO
      NAMES ompi_info
      HINTS ${MPI_CXX_LIBRARIES}/../bin)
    if (OMPI_INFO)
      execute_process(COMMAND ${OMPI_INFO}
        OUTPUT_VARIABLE _output)
      if (_output MATCHES "smcuda")
        message(STATUS "Found OpenMPI with CUDA support built.")
      else()
        message(WARNING "OpenMPI found, but it is not built with CUDA support.")
	set( CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -DMPI_CUDA_AWARE_OFF" )
      endif()
    endif()
  else()
    message(WARNING "Not compiling with MPI. Suppress this warning with -DUSE_MPI=OFF")
    set(USE_MPI OFF)
  endif()
endif()


#
# common flags
#

if (KOKKOS_ENABLE_CUDA)
  set( CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -DCUDA" )
endif()

if (USE_MPI)
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DWITH_MPI=1 -DATOMICS_FLUX" )
endif()

############################ Build Kokkos ######################################
add_subdirectory(kokkos_src)

############################ Build ${PROJECT_NAME} #############################
set(EXE_SRC
  Cell.C
  Cell.h
  Cells.h
  CopyGhost.C
  CopyGhost.h
  ElementTopo.C
  ElementTopo.h
  ElementTopoHexa8.C
  ElementTopoHexa8.h
  Extrapolate_BC.h
  Face.C
  Face.h
  Faces.h
  Flux.h
  GasModel.h
  GreenGauss.h
  Inflow_BC.h
  Initial_Conditions.h
  Main.C
  Main.h
  MathTools.h
  MathToolsDevice.h
  MemoryUsage.C
  MemoryUsage.h
  MeshData.h
  MeshProcessor.C
  MeshProcessor.h
  NoSlip_BC.h
  Options.h
  Parallel3DMesh.C
  Parallel3DMesh.h
  Roe_Flux.h
  StencilLimiter.h
  Tangent_BC.h
  TimeSolverExplicitRK4.h
  VanAlbadaLimiter.h
  VenkatLimiter.h
  ViewTypes.h
  Viscous_Flux.h
  YAML_Default.h
  YAML_Doc.C
  YAML_Doc.h
  YAML_Element.C
  YAML_Element.h
  )

if(KOKKOS_ENABLE_CUDA)
  set(EXE_NAME "${PROJECT_NAME}.cuda")
else()
  set(EXE_NAME "${PROJECT_NAME}.host")
endif()

add_executable(${EXE_NAME}
  ${EXE_SRC}
  )
target_include_directories(${EXE_NAME}
  PUBLIC
  ${CMAKE_SOURCE_DIR}
  ${CMAKE_SOURCE_DIR}/kokkos_src/core/src
  ${CMAKE_SOURCE_DIR}/kokkos_src/containers/src
  ${CMAKE_SOURCE_DIR}/kokkos_src/algorithms/src
  ${CMAKE_BINARY_DIR}/kokkos_src
  )
target_link_libraries(${EXE_NAME}
  kokkos
  )

############################ Testing ###########################################
enable_testing()

if(USE_MPI)
  add_test(NAME 3D_Sod_Parallel
    COMMAND mpirun -np 4 ${CMAKE_BINARY_DIR}/${EXE_NAME}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tests/3D_Sod_Parallel)
  add_test(NAME FlatPlate_Parallel
    COMMAND mpirun -np 4 ${CMAKE_BINARY_DIR}/${EXE_NAME}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tests/FlatPlate_Parallel)
  add_test(NAME Ramp_Parallel
    COMMAND mpirun -np 4 ${CMAKE_BINARY_DIR}/${EXE_NAME}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tests/Ramp_Parallel)
else()
  add_test(NAME 3D_Sod_Serial
    COMMAND ${CMAKE_BINARY_DIR}/${EXE_NAME}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tests/3D_Sod_Serial)
  add_test(NAME FlatPlate_Serial
    COMMAND mpirun -np 4 ${CMAKE_BINARY_DIR}/${EXE_NAME}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tests/FlatPlate_Serial)
  add_test(NAME Ramp_Serial
    COMMAND mpirun -np 4 ${CMAKE_BINARY_DIR}/${EXE_NAME}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tests/Ramp_Serial)
endif()

############################ PRINT CONFIGURE STATUS ############################

message(STATUS "")
message(STATUS "****************** ${PROJECT_NAME} Settings ******************")

message(STATUS "MPI      enabled : ${USE_MPI}")
message(STATUS "CUDA     enabled : ${KOKKOS_ENABLE_CUDA}")

message(STATUS "****************** *************** ******************")

