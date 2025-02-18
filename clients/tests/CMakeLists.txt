# ########################################################################
# Copyright (c) 2018-2021 Advanced Micro Devices, Inc.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.
#
# ########################################################################

find_package(GTest REQUIRED)

# Download some test matrices
set(TEST_MATRICES
  SNAP/amazon0312
  Muite/Chebyshev4
  FEMLAB/sme3Dc
  Williams/webbase-1M
  Bova/rma10
  JGD_BIBD/bibd_22_8
  Williams/mac_econ_fwd500
  Williams/mc2depi
  Hamm/scircuit
  Sandia/ASIC_320k
  GHS_psdef/bmwcra_1
  HB/nos1
  HB/nos2
  HB/nos3
  HB/nos4
  HB/nos5
  HB/nos6
  HB/nos7
  DNVS/shipsec1
)

set(TEST_MD5HASH
  f567e5f5029d052e3004bc69bb3f13f5
  e39879103dafab21f4cf942e0fe42a85
  a95eee14d980a9cfbbaf5df4a3c64713
  2d4c239daad6f12d66a1e6a2af44cbdb
  a899a0c48b9a58d081c52ffd88a84955
  455d5b699ea10232bbab5bc002219ae6
  f1b0e56fbb75d1d6862874e3d7d33060
  8c8633eada6455c1784269b213c85ea6
  3e62f7ea83914f7e20019aefb2a5176f
  fcfaf8a25c8f49b8d29f138f3c65c08f
  8a3cf5448a4fe73dcbdb5a16b326715f
  b203f7605cb1f20f83280061068f7ec7
  b0f812ffcc9469f0bf9be701205522c4
  f185514062a0eeabe86d2909275fe1dc
  04b781415202db404733ca0c159acbef
  c98e35f1cfd1ee8177f37bdae155a6e7
  c39375226aa5c495293003a5f637598f
  9a6481268847e6cf0d70671f2ff1ddcd
  73372e7d6a0848f8b19d64a924fab73e
)

if(NOT TARGET hipsparse)
  set(CONVERT_SOURCE ${CMAKE_SOURCE_DIR}/../deps/convert.cpp)
else()
  set(CONVERT_SOURCE ${CMAKE_SOURCE_DIR}/deps/convert.cpp)
endif()

if(BUILD_ADDRESS_SANITIZER)
  execute_process(COMMAND ${CMAKE_CXX_COMPILER} ${CONVERT_SOURCE} -O3 -fsanitize=address -shared-libasan -o ${PROJECT_BINARY_DIR}/mtx2csr.exe)
else()
  execute_process(COMMAND ${CMAKE_CXX_COMPILER} ${CONVERT_SOURCE} -O3 -o ${PROJECT_BINARY_DIR}/mtx2csr.exe)
endif()

list(LENGTH TEST_MATRICES len)
math(EXPR len1 "${len} - 1")

foreach(i RANGE 0 ${len1})
  list(GET TEST_MATRICES ${i} m)
  list(GET TEST_MD5HASH ${i} md5)

  string(REPLACE "/" ";" sep_m ${m})
  list(GET sep_m 0 dir)
  list(GET sep_m 1 mat)

  # Download test matrices if not already downloaded
  set(CMAKE_MATRICES_DIR ${PROJECT_BINARY_DIR}/matrices)
  if(NOT EXISTS "${CMAKE_MATRICES_DIR}/${mat}.bin")
    message("-- Downloading and extracting test matrix ${m}.tar.gz")
    file(DOWNLOAD https://sparse.tamu.edu/MM/${m}.tar.gz ${CMAKE_MATRICES_DIR}/${mat}.tar.gz
        INACTIVITY_TIMEOUT 3
        STATUS DL)

    list(GET DL 0 stat)
    list(GET DL 1 msg)

    if(NOT stat EQUAL 0)
      message("-- Timeout has been reached, trying mirror ...")
      # Try again using ufl links
      file(DOWNLOAD https://www.cise.ufl.edu/research/sparse/MM/${m}.tar.gz ${CMAKE_MATRICES_DIR}/${mat}.tar.gz
          INACTIVITY_TIMEOUT 3
          STATUS DL)

      list(GET DL 0 stat)
      list(GET DL 1 msg)

      if(NOT stat EQUAL 0)
        message(FATAL_ERROR "${msg}")
      endif()
    endif()

    # Check MD5 hash before continuing
    file(MD5 ${CMAKE_MATRICES_DIR}/${mat}.tar.gz hash)

    # Compare hash
    if(NOT hash STREQUAL md5)
      message(FATAL_ERROR "${mat}.tar.gz is corrupted")
    endif()
    execute_process(COMMAND tar xf ${mat}.tar.gz WORKING_DIRECTORY ${CMAKE_MATRICES_DIR})
    
    file(RENAME ${CMAKE_MATRICES_DIR}/${mat}/${mat}.mtx ${CMAKE_MATRICES_DIR}/${mat}.mtx)
    execute_process(COMMAND ${PROJECT_BINARY_DIR}/mtx2csr.exe ${mat}.mtx ${mat}.bin WORKING_DIRECTORY ${CMAKE_MATRICES_DIR})
    # TODO: add 'COMMAND_ERROR_IS_FATAL ANY' once cmake supported version is 3.19
    file(REMOVE_RECURSE ${CMAKE_MATRICES_DIR}/${mat}.tar.gz ${CMAKE_MATRICES_DIR}/${mat} ${CMAKE_MATRICES_DIR}/${mat}.mtx)
	
  endif()
endforeach()

set(HIPSPARSE_TEST_SOURCES
  hipsparse_gtest_main.cpp
  test_axpyi.cpp
  test_gthr.cpp
  test_gthrz.cpp
  test_roti.cpp
  test_sctr.cpp
  test_bsrmv.cpp
  test_bsrxmv.cpp
  test_bsrsv2.cpp
  test_csrsv2.cpp
  test_bsrmm.cpp
  test_bsrsm2.cpp
  test_csrsm2.cpp
  test_gemmi.cpp
  test_gemvi.cpp
  test_csrgeam2.cpp
  test_csrgemm2_a.cpp
  test_csrgemm2_b.cpp
  test_bsrilu02.cpp
  test_csrilu02.cpp
  test_bsric02.cpp
  test_csric02.cpp
  test_nnz.cpp
  test_csr2dense.cpp
  test_csc2dense.cpp
  test_dense2csr.cpp
  test_prune_dense2csr.cpp
  test_prune_dense2csr_by_percentage.cpp
  test_dense2csc.cpp
  test_csr2coo.cpp
  test_csr2bsr.cpp
  test_bsr2csr.cpp
  test_gebsr2csr.cpp
  test_csr2csr_compress.cpp
  test_prune_csr2csr.cpp
  test_prune_csr2csr_by_percentage.cpp
  test_coo2csr.cpp
  test_identity.cpp
  test_csrsort.cpp
  test_cscsort.cpp
  test_coosort.cpp
  test_csru2csr.cpp
  test_csrilusv.cpp
  test_gebsr2gebsr.cpp
  test_csr2gebsr.cpp
  test_gebsr2gebsc.cpp
  test_spmat_descr.cpp
  test_spvec_descr.cpp
  test_dnvec_descr.cpp
  test_spmv_coo.cpp
  test_spmv_coo_aos.cpp
  test_spmv_csr.cpp
  test_axpby.cpp
  test_gather.cpp
  test_scatter.cpp
  test_rot.cpp
  test_spvv.cpp
  test_dense_to_sparse_csr.cpp
  test_dense_to_sparse_csc.cpp
  test_dense_to_sparse_coo.cpp
  test_sparse_to_dense_csr.cpp
  test_sparse_to_dense_csc.cpp
  test_sparse_to_dense_coo.cpp
  test_spmm_csr.cpp
  test_spmm_coo.cpp
  test_spmm_bell.cpp
  test_spgemm_csr.cpp
  test_sddmm_csr.cpp
  test_sddmm_csc.cpp
  test_sddmm_coo.cpp
  test_sddmm_coo_aos.cpp
  test_gtsv2_strided_batch.cpp
  test_gtsv.cpp
  test_gtsv2_nopivot.cpp
  test_csrcolor.cpp
  test_spsv_csr.cpp
  test_spsv_coo.cpp
  test_spsm_csr.cpp
  test_spsm_coo.cpp
)


if(NOT BUILD_CUDA)
    list(APPEND HIPSPARSE_TEST_SOURCES
        test_doti.cpp
        test_dotci.cpp
        test_csr2csc.cpp
        test_csrgemm.cpp
        test_csrgeam.cpp
        test_csrmv.cpp
        test_csrmm.cpp
        test_hybmv.cpp
        test_csr2hyb.cpp
        test_hyb2csr.cpp
    )
endif()

set(HIPSPARSE_CLIENTS_COMMON
  ../common/arg_check.cpp
  ../common/unit.cpp
  ../common/utility.cpp
  ../common/hipsparse_template_specialization.cpp
)

add_executable(hipsparse-test ${HIPSPARSE_TEST_SOURCES} ${HIPSPARSE_CLIENTS_COMMON})

# Set GOOGLE_TEST definition
target_compile_definitions(hipsparse-test PRIVATE GOOGLE_TEST)

# Internal common header
target_include_directories(hipsparse-test PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>)

# Target link libraries
target_link_libraries(hipsparse-test PRIVATE GTest::GTest roc::hipsparse)

# Add OpenMP if available
if(OPENMP_FOUND AND THREADS_FOUND)
  target_link_libraries(hipsparse-test PRIVATE OpenMP::OpenMP_CXX ${OpenMP_CXX_FLAGS})
endif()

if(NOT BUILD_CUDA)
  target_link_libraries(hipsparse-test PRIVATE hip::host)
else()
  target_compile_definitions(hipsparse-test PRIVATE __HIP_PLATFORM_NVIDIA__)
  target_include_directories(hipsparse-test PRIVATE ${HIP_INCLUDE_DIRS})
  target_link_libraries(hipsparse-test PRIVATE ${CUDA_LIBRARIES})
endif()

set_target_properties(hipsparse-test PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging")

if (WIN32)
  # for now adding in all .dll as dependency chain is not cmake based on win32
  file( GLOB third_party_dlls
    LIST_DIRECTORIES OFF
    CONFIGURE_DEPENDS
    $ENV{HIP_DIR}/bin/*.dll
    $ENV{HIP_DIR}/bin/hipinfo.exe
    ${ROCSPARSE_PATH}/bin/librocsparse.dll
    ${CMAKE_SOURCE_DIR}/rtest.*
    C:/Windows/System32/libomp140*.dll
  )
  foreach( file_i ${third_party_dlls})
    add_custom_command( TARGET hipsparse-test POST_BUILD COMMAND ${CMAKE_COMMAND} ARGS -E copy ${file_i} ${PROJECT_BINARY_DIR}/staging/ )
  endforeach( file_i )

endif()

add_test(hipsparse-test hipsparse-test)
