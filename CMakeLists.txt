cmake_minimum_required (VERSION 2.6)
project (DIAMOND)
include(CheckCXXCompilerFlag)

option(BUILD_STATIC "BUILD_STATIC" OFF)
option(EXTRA "EXTRA" OFF)
option(STATIC_LIBGCC "STATIC_LIBGCC" OFF)
option(STATIC_LIBSTDC++ "STATIC_LIBSTDC++" OFF)
option(X86 "X86" ON)
option(STRICT_BAND "STRICT_BAND" ON)
option(LEFTMOST_SEED_FILTER "LEFTMOST_SEED_FILTER" ON)
option(SEQ_MASK "SEQ_MASK" ON)
option(DP_STAT "DP_STAT" OFF)
option(SINGLE_THREADED "SINGLE_THREADED" OFF)
option(EIGEN_BLAS "EIGEN_BLAS" OFF)
option(WITH_ZSTD "WITH_ZSTD" OFF)
set(MAX_SHAPE_LEN 19)
set(BLAST_INCLUDE_DIR "" CACHE STRING "BLAST_INCLUDE_DIR")
set(BLAST_LIBRARY_DIR "" CACHE STRING "BLAST_LIBRARY_DIR")

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm.*|ARM.*)")
  set(X86 OFF)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64.*|AARCH64.*)")
  set(X86 OFF)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "PPC64*|ppc64*|powerpc64*")
  set(X86 OFF)
  # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maltivec")
  add_definitions(-DEIGEN_DONT_VECTORIZE)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^s390")
  set(X86 OFF)
endif()

if(STATIC_LIBSTDC++)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libstdc++")
endif()

if(STRICT_BAND)
  add_definitions(-DSTRICT_BAND)
endif()

if(SINGLE_THREADED)
  add_definitions(-DSINGLE_THREADED)
endif()

if(SEQ_MASK)
  add_definitions(-DSEQ_MASK)
endif()

if(LEFTMOST_SEED_FILTER)
  add_definitions(-DLEFTMOST_SEED_FILTER)
endif()

if(DP_STAT)
  add_definitions(-DDP_STAT)
endif()

if(EXTRA)
  add_definitions(-DEXTRA)
endif()

add_definitions(-DMAX_SHAPE_LEN=${MAX_SHAPE_LEN})

IF(STATIC_LIBGCC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libgcc")
endif()

if(BUILD_STATIC)
    set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
    set(BUILD_SHARED_LIBRARIES OFF)
    set(CMAKE_EXE_LINKER_FLAGS "-static")
endif()

check_cxx_compiler_flag("-std=gnu++14" HAS_GNUPP14)
if (HAS_GNUPP14)
  if(${CMAKE_VERSION} VERSION_LESS "3.1.0")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++14")
  else()
    set(CMAKE_CXX_STANDARD 14)
  endif()
else()
  if(${CMAKE_VERSION} VERSION_LESS "3.1.0")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11")
  else()
    set(CMAKE_CXX_STANDARD 11)
  endif()
endif()

if(CMAKE_BUILD_MARCH)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${CMAKE_BUILD_MARCH}")
endif()

find_package(ZLIB REQUIRED)
find_package(Threads REQUIRED)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release)
endif()

if (${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wno-implicit-fallthrough -Wreturn-type -Wno-unused -Wno-unused-parameter -Wno-unused-variable -Wno-uninitialized -Wno-deprecated-copy -Wno-unknown-warning-option")
  # -fsanitize=address -fno-omit-frame-pointer
endif()

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-pragma-clang-attribute -Wno-overloaded-virtual -Wno-missing-braces")
endif()

if (CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fabi-version=7")
  message("Setting -fabi-version for GCC 4.x")
endif()

set(DISPATCH_OBJECTS
"src/dp/swipe/banded_3frame_swipe.cpp"
"src/search/stage2.cpp"
"src/tools/benchmark.cpp"
"src/dp/swipe/swipe_wrapper.cpp"
"src/util/tantan.cpp"
"src/dp/scan_diags.cpp"
"src/dp/ungapped_simd.cpp"
)

if(EXTRA)
  LIST(APPEND DISPATCH_OBJECTS "src/tools/benchmark_swipe.cpp")
endif()

add_library(arch_generic OBJECT ${DISPATCH_OBJECTS})
target_compile_options(arch_generic PUBLIC -DDISPATCH_ARCH=ARCH_GENERIC -DARCH_ID=0 -DEigen=Eigen_GENERIC)
target_include_directories(arch_generic PRIVATE "${CMAKE_SOURCE_DIR}/src/lib")
if(X86)
add_library(arch_sse4_1 OBJECT ${DISPATCH_OBJECTS})
target_include_directories(arch_sse4_1 PRIVATE "${CMAKE_SOURCE_DIR}/src/lib")
add_library(arch_avx2 OBJECT ${DISPATCH_OBJECTS})
target_include_directories(arch_avx2 PRIVATE "${CMAKE_SOURCE_DIR}/src/lib")
if (${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC)
	target_compile_options(arch_sse4_1 PUBLIC -DDISPATCH_ARCH=ARCH_SSE4_1 -DARCH_ID=1 -D__SSSE3__ -D__SSE4_1__ -D__POPCNT__ -DEigen=Eigen_SSE4_1)
    target_compile_options(arch_avx2 PUBLIC -DDISPATCH_ARCH=ARCH_AVX2 -DARCH_ID=2 /arch:AVX2 -D__SSSE3__ -D__SSE4_1__ -D__POPCNT__ -DEigen=Eigen_AVX2)
else()
	target_compile_options(arch_sse4_1 PUBLIC -DDISPATCH_ARCH=ARCH_SSE4_1 -DARCH_ID=1 -mssse3 -mpopcnt -msse4.1 -DEigen=Eigen_SSE4_1)
    target_compile_options(arch_avx2 PUBLIC -DDISPATCH_ARCH=ARCH_AVX2 -DARCH_ID=2 -mssse3 -mpopcnt -msse4.1 -msse4.2 -mavx -mavx2 -DEigen=Eigen_AVX2)
endif()
endif(X86)

set(OBJECTS
  src/run/main.cpp
  src/basic/config.cpp
  src/stats/score_matrix.cpp
  src/data/queries.cpp
  src/data/reference.cpp
  src/data/seed_histogram.cpp
  src/output/daa/daa_record.cpp
  src/util/command_line_parser.cpp
  src/util/seq_file_format.cpp
  src/util/util.cpp
  src/basic/basic.cpp
  src/basic/hssp.cpp
  src/dp/ungapped_align.cpp
  src/run/tools.cpp
  src/chaining/greedy_align.cpp
  src/output/output_format.cpp
  src/output/clustering_variables.cpp
  src/output/clustering_format.cpp
  src/output/join_blocks.cpp
  src/data/frequent_seeds.cpp
  src/align/legacy/query_mapper.cpp
  src/output/blast_tab_format.cpp
  src/output/blast_pairwise_format.cpp
  src/run/double_indexed.cpp
  src/output/sam_format.cpp
  src/align/align.cpp
  src/search/setup.cpp
  src/data/taxonomy.cpp
  src/masking/masking.cpp
  src/data/seed_set.cpp
  src/util/simd.cpp
  src/output/taxon_format.cpp
  src/output/daa/view.cpp
  src/output/output_sink.cpp
  src/output/target_culling.cpp
  src/align/legacy/banded_swipe_pipeline.cpp
  src/util/io/compressed_stream.cpp
  src/util/io/deserializer.cpp
  src/util/io/file_sink.cpp
  src/util/io/file_source.cpp
  src/util/io/input_file.cpp
  src/util/io/input_stream_buffer.cpp
  src/util/io/output_file.cpp
  src/util/io/output_stream_buffer.cpp
  src/util/io/serializer.cpp
  src/util/io/temp_file.cpp
  src/util/io/text_input_file.cpp
  src/data/taxon_list.cpp
  src/data/taxonomy_nodes.cpp
  src/util/algo/MurmurHash3.cpp
  src/search/stage0.cpp
  src/data/seed_array.cpp
  src/output/paf_format.cpp
  src/util/system/system.cpp
  src/util/algo/greedy_vertex_cover.cpp
  src/util/sequence/sequence.cpp
  src/tools/tsv_record.cpp
  src/tools/tools.cpp
  src/util/system/getRSS.cpp
  src/lib/tantan/LambdaCalculator.cc
  src/util/algo/upgma.cpp
  src/util/algo/upgma_mc.cpp
  src/util/algo/edge_vec.cpp
  src/util/string/string.cpp
  src/align/extend.cpp
  src/test/simulate.cpp
  src/test/test.cpp
  src/align/ungapped.cpp
  src/align/gapped_score.cpp
  src/align/gapped_final.cpp
  src/align/full_db.cpp
  src/align/culling.cpp
  src/cluster/medoid.cpp
  src/cluster/cluster_registry.cpp
  src/cluster/multi_step_cluster.cpp
  src/cluster/mcl.cpp
  src/align/output.cpp
  src/tools/roc.cpp
  src/test/data.cpp
  src/test/test_cases.cpp
  src/chaining/smith_waterman.cpp
  src/tools/merge_tsv.cpp
  src/output/xml_format.cpp
  src/align/gapped_filter.cpp
  src/util/parallel/filestack.cpp
  src/util/parallel/parallelizer.cpp
  src/util/parallel/multiprocessing.cpp
  src/tools/benchmark_io.cpp
  src/align/memory.cpp
  src/lib/alp/njn_dynprogprob.cpp
  src/lib/alp/njn_dynprogproblim.cpp
  src/lib/alp/njn_dynprogprobproto.cpp
  src/lib/alp/njn_ioutil.cpp
  src/lib/alp/njn_localmaxstat.cpp
  src/lib/alp/njn_localmaxstatmatrix.cpp
  src/lib/alp/njn_localmaxstatutil.cpp
  src/lib/alp/njn_random.cpp
  src/lib/alp/sls_alignment_evaluer.cpp
  src/lib/alp/sls_alp.cpp
  src/lib/alp/sls_alp_data.cpp
  src/lib/alp/sls_alp_regression.cpp
  src/lib/alp/sls_alp_sim.cpp
  src/lib/alp/sls_basic.cpp
  src/lib/alp/sls_pvalues.cpp
  src/align/global_ranking/global_ranking.cpp
  src/align/global_ranking/extend.cpp
  src/tools/rocid.cpp
  src/lib/blast/blast_seg.cpp
  src/lib/blast/blast_filter.cpp
  src/lib/blast/nlm_linear_algebra.cpp
  src/stats/matrices/blosum45.cpp
  src/stats/matrices/blosum50.cpp
  src/stats/matrices/blosum62.cpp
  src/stats/matrices/blosum80.cpp
  src/stats/matrices/blosum90.cpp
  src/stats/matrices/pam250.cpp
  src/stats/matrices/pam30.cpp
  src/stats/matrices/pam70.cpp
  src/stats/stats.cpp
  src/stats/cbs.cpp
  src/stats/comp_based_stats.cpp
  src/stats/hauser_correction.cpp
  src/stats/matrix_adjust.cpp
  src/stats/matrix_adjust_eigen.cpp
  src/data/index.cpp
  src/data/dmnd/dmnd.cpp
  src/data/sequence_file.cpp
  src/tools/find_shapes.cpp
  src/data/block.cpp
  src/run/config.cpp
  src/data/sequence_set.cpp
  src/align/global_ranking/table.cpp
  src/output/daa/daa_write.cpp
  src/search/seed_complexity.cpp
  src/tools/view.cpp
  src/util/string/tsv.cpp
  src/tools/join.cpp
  src/basic/value.cpp
  src/masking/motifs.cpp
  src/align/alt_hsp.cpp
)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}")

if(BLAST_INCLUDE_DIR)
  set(BLAST_OBJ "src/data/blastdb/blastdb.cpp")
endif()

if(WITH_ZSTD OR BLAST_INCLUDE_DIR)
  set(ZSTD_OBJ "src/util/io/zstd_stream.cpp")
endif()

if(X86)
  add_executable(diamond $<TARGET_OBJECTS:arch_generic> $<TARGET_OBJECTS:arch_sse4_1> $<TARGET_OBJECTS:arch_avx2> ${OBJECTS} ${BLAST_OBJ} ${ZSTD_OBJ})
else()
  add_executable(diamond $<TARGET_OBJECTS:arch_generic> ${OBJECTS} ${BLAST_OBJ} ${ZSTD_OBJ})
endif()

target_include_directories(diamond PRIVATE
  "${ZLIB_INCLUDE_DIR}"
  "${CMAKE_SOURCE_DIR}/src/lib")

if(BLAST_INCLUDE_DIR)
  function(find_blast_lib var library)
    find_library(${var} ${library} PATHS ${BLAST_LIBRARY_DIR} "/usr/lib/ncbi-blast+")
    if(${${var}} STREQUAL "${var}-NOTFOUND")
      message(FATAL_ERROR "Unable to find BLAST library: ${library}")
    endif()
    message("Found BLAST library: ${${var}}")
  endfunction(find_blast_lib)
  message("BLAST_INCLUDE_DIR: ${BLAST_INCLUDE_DIR}")
  message("BLAST_LIBRARY_DIR: ${BLAST_LIBRARY_DIR}")
  target_include_directories(diamond PRIVATE "${BLAST_INCLUDE_DIR}")
  find_blast_lib(SEQDB_LIBRARY seqdb)
  find_blast_lib(BIBLIO_LIBRARY biblio)
  find_blast_lib(BLASTDB_LIBRARY blastdb)
  find_blast_lib(BLASTDB_FORMAT_LIBRARY blastdb_format)
  find_blast_lib(GENERAL_LIBRARY general)
  find_blast_lib(GENOME_COLLECTION_LIBRARY genome_collection)
  find_blast_lib(MEDLINE_LIBRARY medline)
  find_blast_lib(PUB_LIBRARY pub)
  find_blast_lib(SEQ_LIBRARY seq)
  find_blast_lib(SEQCODE_LIBRARY seqcode)
  find_blast_lib(SEQSET_LIBRARY seqset)
  find_blast_lib(SEQUTIL_LIBRARY sequtil)
  find_blast_lib(SUBMIT_LIBRARY submit)
  find_blast_lib(XOBJMGR_LIBRARY xobjmgr)
  find_blast_lib(XOBJUTIL_LIBRARY xobjutil)
  find_blast_lib(XSER_LIBRARY xser)
  find_blast_lib(XNCBI_LIBRARY xncbi)
  find_blast_lib(XUTIL_LIBRARY xutil)
  find_blast_lib(LMDB_LIBRARY lmdb)
  target_link_libraries(diamond ${SEQDB_LIBRARY} ${BLASTDB_LIBRARY} ${BLASTDB_FORMAT_LIBRARY} ${GENERAL_LIBRARY}
    ${XOBJUTIL_LIBRARY} ${XOBJMGR_LIBRARY} ${GENOME_COLLECTION_LIBRARY} ${SEQ_LIBRARY} ${SEQCODE_LIBRARY}
    ${SEQSET_LIBRARY} ${SEQUTIL_LIBRARY} ${PUB_LIBRARY} ${MEDLINE_LIBRARY} ${BIBLIO_LIBRARY} ${SUBMIT_LIBRARY}
    ${XSER_LIBRARY} ${XNCBI_LIBRARY} ${XUTIL_LIBRARY} ${LMDB_LIBRARY})
  add_definitions(-DWITH_BLASTDB)
  set_target_properties(diamond PROPERTIES INSTALL_RPATH_USE_LINK_PATH TRUE)
  if (${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC)
    find_library(DBGHELP_LIBRARY Dbghelp)
    target_link_libraries(diamond ${DBGHELP_LIBRARY})
  else()
    target_link_libraries(diamond ${CMAKE_DL_LIBS})
  endif()
endif()

if(EIGEN_BLAS)
  add_definitions(-DEIGEN_USE_BLAS -DEIGEN_USE_LAPACKE)
  find_package(LAPACKE)
  target_include_directories(diamond PRIVATE "${LAPACKE_INCLUDE_DIR}")
  target_link_libraries(diamond "${LAPACKE_LIBRARIES_DEP}")
endif()

if(WITH_ZSTD OR BLAST_INCLUDE_DIR)
  find_path(${ZSTD_INCLUDE_DIR} NAMES zstd.h PATHS ${ZSTD_INCLUDE_DIR})
  find_library(ZSTD_LIBRARY NAMES libzstd.a libzstd_static)
  target_include_directories(diamond PRIVATE "${ZSTD_INCLUDE_DIR}")
  target_link_libraries(diamond ${ZSTD_LIBRARY})
  add_definitions(-DWITH_ZSTD)
endif()

target_link_libraries(diamond ${ZLIB_LIBRARY} ${CMAKE_THREAD_LIBS_INIT})

install(TARGETS diamond DESTINATION bin)
