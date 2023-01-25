# (C) Copyright 2017-2020 UCAR
#
# This software is licensed under the terms of the Apache Licence Version 2.0
# which can be obtained at http://www.apache.org/licenses/LICENSE-2.0.

# ##########
# fv3-bundle
# ##########

cmake_minimum_required( VERSION 3.12 FATAL_ERROR )

find_package( ecbuild 3.5 REQUIRED HINTS ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/../ecbuild)

project( fv3-bundle VERSION 1.1.0 LANGUAGES C CXX Fortran )

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include( ecbuild_bundle )

# Default release mode
set( ECBUILD_DEFAULT_BUILD_TYPE Release )

# Enable MPI
set( ENABLE_MPI ON CACHE BOOL "Compile with MPI")

# Initialize
ecbuild_bundle_initialize()

# Use external jedi-cmake or build in bundle
if(DEFINED ENV{jedi_cmake_ROOT})
  include( $ENV{jedi_cmake_ROOT}/share/jedicmake/Functions/git_functions.cmake )
else()
  ecbuild_bundle( PROJECT jedicmake GIT "https://github.com/jcsda/jedi-cmake.git" BRANCH develop UPDATE RECURSIVE )
  include( jedicmake/cmake/Functions/git_functions.cmake )
endif()

# ECMWF libs
# ----------
#ecbuild_bundle( PROJECT eckit    GIT "https://github.com/ecmwf/eckit.git" TAG 1.20.2 )
#ecbuild_bundle( PROJECT fckit    GIT "https://github.com/ecmwf/fckit.git" TAG 0.9.5 )
#ecbuild_bundle( PROJECT atlas    GIT "https://github.com/ecmwf/atlas.git" TAG 0.31.1 )

# External (optional) observation operators
# ------------------------------
option("BUNDLE_SKIP_GEOS-AERO" "Don't build GEOS-AERO" "ON") # Skip geos-aero build unless user passes -DBUNDLE_SKIP_GEOS-AERO=OFF
option("BUNDLE_SKIP_ROPP-UFO"  "Don't build ROPP-UFO"  "ON") # Skip ropp-ufo build unless user passes -DBUNDLE_SKIP_ROPP-UFO=OFF

ecbuild_bundle( PROJECT geos-aero GIT "https://github.com/jcsda-internal/geos-aero.git" BRANCH develop UPDATE )
ecbuild_bundle( PROJECT ropp-ufo GIT "https://github.com/jcsda-internal/ropp-test.git"  BRANCH develop UPDATE )

# External (required) observation operators
# ------------------------------
ecbuild_bundle( PROJECT crtm GIT "https://github.com/jcsda/crtm.git" TAG v2.4-jedi.2 )

# Core JEDI repositories
# ----------------------
ecbuild_bundle( PROJECT oops  GIT "https://github.com/jcsda/oops.git"  TAG e8cbd61084a4574906eafc0e9c5450da674f185a )
ecbuild_bundle( PROJECT vader GIT "https://github.com/jcsda/vader.git" TAG 922863dc6afc7d6a644f1deedb6771744ce57f9c )
ecbuild_bundle( PROJECT saber GIT "https://github.com/jcsda/saber.git" TAG eb86f4b43699c0c39202fe5e0982471f5383c4e9 )
ecbuild_bundle( PROJECT ioda  GIT "https://github.com/jcsda/ioda.git"  TAG b02248f298a027d26ef1c9a3d9f42f807db1b576 )
ecbuild_bundle( PROJECT ufo   GIT "https://github.com/jcsda/ufo.git"   TAG 334fdbb8867d49c5eff3a33d6c3a64184e1ce6a2 )

# Options for building with certain models
# ----------------------------------------
set(FV3_FORECAST_MODEL "FV3CORE" CACHE STRING "Choose which MODEL to build with")
set_property(CACHE FV3_FORECAST_MODEL PROPERTY STRINGS "FV3CORE" "GEOS")
if(NOT FV3_FORECAST_MODEL MATCHES "^(FV3CORE|GEOS)$")
  ecbuild_error("Valid options for -DFV3_FORECAST_MODEL are FV3CORE|GEOS")
endif()
ecbuild_info("Building with FV3-based model: ${FV3_FORECAST_MODEL}")

# FMS and FV3 dynamical core model are needed for FV3CORE
# -------------------------------------------------------
if (FV3_FORECAST_MODEL MATCHES "FV3CORE")
  option("BUNDLE_SKIP_FMS" "Don't build FMS" "OFF")
  ecbuild_bundle( PROJECT fms GIT "https://github.com/jcsda/FMS.git" BRANCH release-stable UPDATE )
  ecbuild_bundle( PROJECT fv3 GIT "https://github.com/jcsda/GFDL_atmos_cubed_sphere.git" BRANCH release-stable UPDATE )
endif()

# fv3-jedi linear model
# ---------------------
ecbuild_bundle( PROJECT fv3-jedi-lm GIT "https://github.com/jcsda/fv3-jedi-linearmodel.git" BRANCH develop UPDATE )

# fv3-jedi and associated repositories
# ------------------------------------
ecbuild_bundle( PROJECT femps    GIT "https://github.com/jcsda/femps.git"    BRANCH develop UPDATE )
ecbuild_bundle( PROJECT fv3-jedi GIT "https://github.com/jcsda/fv3-jedi.git" TAG 9c42256c95a31c3c59604dddbbc60b51ae49a9f9 )

# ioda, ufo, fv3-jedi and saber test data
#----------------------------------------

# If IODA branch is being built set GIT_BRANCH_FUNC to IODA's current branch.
# If a tagged version of IODA is being built set GIT_TAG_FUNC to ioda's current tag. In this case,
# IODA test files will be download from UCAR DASH and ioda-data repo will not be cloned.
# When LOCAL_PATH_JEDI_TESTFILES is set to the directory of IODA test files stored
# in a local directory, ioda-data repo will not be cloned

find_branch_name(REPO_DIR_NAME ioda)
# When LOCAL_PATH_JEDI_TESTFILES is set to the directory of IODA test files stored
# in a local directory, ioda-data repo will not be cloned
if( NOT DEFINED ENV{LOCAL_PATH_JEDI_TESTFILES} AND NOT DEFINED ${GIT_TAG_FUNC} )
  ecbuild_bundle( PROJECT ioda-data GIT "https://github.com/JCSDA-internal/ioda-data.git" BRANCH develop UPDATE )
endif()

# If IODA's current branch is available in ioda-data repo, that branch will be checked out
branch_checkout (REPO_DIR_NAME ioda-data
                 BRANCH ${GIT_BRANCH_FUNC} )

# same procedure for ufo-data
find_branch_name(REPO_DIR_NAME ufo)
if( NOT DEFINED ENV{LOCAL_PATH_JEDI_TESTFILES} AND NOT DEFINED ${GIT_TAG_FUNC} )
  ecbuild_bundle( PROJECT ufo-data GIT "https://github.com/JCSDA-internal/ufo-data.git" BRANCH develop UPDATE )
endif()

# If UFO's current branch is available in ufo-data repo, that branch will be checked out
branch_checkout (REPO_DIR_NAME ufo-data
                 BRANCH ${GIT_BRANCH_FUNC} )

# same procedure for fv3-jedi-data
find_branch_name(REPO_DIR_NAME fv3-jedi)
if( NOT DEFINED ENV{LOCAL_PATH_JEDI_TESTFILES} AND NOT DEFINED ${GIT_TAG_FUNC} )
  ecbuild_bundle( PROJECT fv3-jedi-data GIT "https://github.com/JCSDA-internal/fv3-jedi-data.git" BRANCH develop UPDATE )
endif()

# If fv3-jedi's current branch is available in fv3-jedi-data repo, that branch will be checked out
branch_checkout (REPO_DIR_NAME fv3-jedi-data
                 BRANCH ${GIT_BRANCH_FUNC} )

# Build Doxygen documentation
# ---------------------------
option(BUILD_FV3_BUNDLE_DOC "Build documentation" OFF)
if(BUILD_FV3_BUNDLE_DOC)
   add_subdirectory( Documentation )
endif(BUILD_FV3_BUNDLE_DOC)

ecbuild_bundle_finalize()

include(cmake/cdash-integration.cmake)
include(CTest)
