get_property(TORCHAUDIO_THIRD_PARTIES GLOBAL PROPERTY TORCHAUDIO_THIRD_PARTIES)

################################################################################
# libtorchaudio
################################################################################
set(
  LIBTORCHAUDIO_SOURCES
  lfilter.cpp
  overdrive.cpp
  utils.cpp
  )

set(
  LIBTORCHAUDIO_INCLUDE_DIRS
  ${PROJECT_SOURCE_DIR}
  )

set(
  LIBTORCHAUDIO_LINK_LIBRARIES
  torch
  ${TORCHAUDIO_THIRD_PARTIES}
  )

set(
  LIBTORCHAUDIO_COMPILE_DEFINITIONS)

#------------------------------------------------------------------------------#
# START OF CUSTOMIZATION LOGICS
#------------------------------------------------------------------------------#
if(BUILD_RNNT)
  list(
    APPEND
    LIBTORCHAUDIO_SOURCES
    rnnt/cpu/compute_alphas.cpp
    rnnt/cpu/compute_betas.cpp
    rnnt/cpu/compute.cpp
    rnnt/compute_alphas.cpp
    rnnt/compute_betas.cpp
    rnnt/compute.cpp
    rnnt/autograd.cpp
    )
  if (USE_CUDA)
    list(
      APPEND
      LIBTORCHAUDIO_SOURCES
      rnnt/gpu/compute_alphas.cu
      rnnt/gpu/compute_betas.cu
      rnnt/gpu/compute.cu
      )
  endif()
endif()

if(USE_CUDA)
  list(
    APPEND
    LIBTORCHAUDIO_INCLUDE_DIRS
    ${CUDA_TOOLKIT_INCLUDE}
    )
  list(
    APPEND
    LIBTORCHAUDIO_LINK_LIBRARIES
    ${C10_CUDA_LIBRARY}
    ${CUDA_CUDART_LIBRARY}
    )
  list(
    APPEND
    LIBTORCHAUDIO_COMPILE_DEFINITIONS
    USE_CUDA
  )
endif()

if(BUILD_KALDI)
  list(APPEND LIBTORCHAUDIO_SOURCES kaldi.cpp)
  list(APPEND LIBTORCHAUDIO_COMPILE_DEFINITIONS INCLUDE_KALDI)
endif()

if(BUILD_SOX)
  list(
    APPEND
    LIBTORCHAUDIO_SOURCES
    sox/io.cpp
    sox/utils.cpp
    sox/effects.cpp
    sox/effects_chain.cpp
    sox/types.cpp
    )
  list(
    APPEND
    LIBTORCHAUDIO_COMPILE_DEFINITIONS
    INCLUDE_SOX
    )
endif()

if(OpenMP_CXX_FOUND)
  list(
    APPEND
    LIBTORCHAUDIO_LINK_LIBRARIES
    OpenMP::OpenMP_CXX
    )
endif()

#------------------------------------------------------------------------------#
# END OF CUSTOMIZATION LOGICS
#------------------------------------------------------------------------------#
function (define_library name source include_dirs link_libraries compile_defs)
  add_library(${name} SHARED ${source})
  target_include_directories(${name} PRIVATE ${include_dirs})
  target_link_libraries(${name} ${link_libraries})
  target_compile_definitions(${name} PRIVATE ${compile_defs})
  set_target_properties(${name} PROPERTIES PREFIX "")
  if (MSVC)
    set_target_properties(${name} PROPERTIES SUFFIX ".pyd")
  endif(MSVC)
  install(
    TARGETS ${name}
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION lib  # For Windows
    )
endfunction()

define_library(
  libtorchaudio
  "${LIBTORCHAUDIO_SOURCES}"
  "${LIBTORCHAUDIO_INCLUDE_DIRS}"
  "${LIBTORCHAUDIO_LINK_LIBRARIES}"
  "${LIBTORCHAUDIO_COMPILE_DEFINITIONS}"
  )


################################################################################
# libtorchaudio_decoder.so
################################################################################
if (BUILD_CTC_DECODER)
  set(
    LIBTORCHAUDIO_DECODER_SOURCES
    decoder/src/decoder/LexiconDecoder.cpp
    decoder/src/decoder/Trie.cpp
    decoder/src/decoder/Utils.cpp
    decoder/src/decoder/lm/KenLM.cpp
    decoder/src/decoder/lm/ZeroLM.cpp
    decoder/src/dictionary/Dictionary.cpp
    decoder/src/dictionary/String.cpp
    decoder/src/dictionary/System.cpp
    decoder/src/dictionary/Utils.cpp
    )
  set(
    LIBTORCHAUDIO_DECODER_DEFINITIONS
    BUILD_CTC_DECODER
    )
  define_library(
    libtorchaudio_decoder
    "${LIBTORCHAUDIO_DECODER_SOURCES}"
    "${PROJECT_SOURCE_DIR}"
    "torch;kenlm"
    "${LIBTORCHAUDIO_COMPILE_DEFINITIONS};${LIBTORCHAUDIO_DECODER_DEFINITIONS}"
    )
endif()

# TODO: Add libtorchaudio_decoder
if (APPLE)
  set(TORCHAUDIO_LIBRARY libtorchaudio CACHE INTERNAL "")
else()
  set(TORCHAUDIO_LIBRARY -Wl,--no-as-needed libtorchaudio -Wl,--as-needed CACHE INTERNAL "")
endif()

################################################################################
# libtorchaudio_ffmpeg
################################################################################
if(BUILD_FFMPEG)
  set(
    LIBTORCHAUDIO_FFMPEG_SOURCES
    ffmpeg/prototype.cpp
    ffmpeg/decoder.cpp
    ffmpeg/ffmpeg.cpp
    ffmpeg/filter_graph.cpp
    ffmpeg/buffer.cpp
    ffmpeg/sink.cpp
    ffmpeg/stream_processor.cpp
    ffmpeg/streamer.cpp
    )
  find_package(FFMPEG 4.1 REQUIRED COMPONENTS avdevice avfilter avformat avcodec avutil)
  define_library(
    libtorchaudio_ffmpeg
    "${LIBTORCHAUDIO_FFMPEG_SOURCES}"
    "${LIBTORCHAUDIO_INCLUDE_DIRS};${FFMPEG_INCLUDE_DIRS}"
    "torch;${FFMPEG_LIBRARIES}"
    "${LIBTORCHAUDIO_COMPILE_DEFINITIONS}"
  )
endif()

################################################################################
# _torchaudio.so
################################################################################
if (BUILD_TORCHAUDIO_PYTHON_EXTENSION)
  # See https://github.com/pytorch/pytorch/issues/38122
  find_library(TORCH_PYTHON_LIBRARY torch_python PATHS "${TORCH_INSTALL_PREFIX}/lib")
  if (WIN32)
    find_package(Python3 ${PYTHON_VERSION} EXACT COMPONENTS Development)
    set(ADDITIONAL_ITEMS Python3::Python)
  endif()
  function(define_extension name sources libraries definitions)
    add_library(${name} SHARED ${sources})
    target_compile_definitions(${name} PRIVATE "${definitions}")
    target_include_directories(
      ${name} PRIVATE ${PROJECT_SOURCE_DIR} ${Python_INCLUDE_DIR})
    target_link_libraries(
      ${name}
      ${libraries}
      ${TORCH_PYTHON_LIBRARY}
      ${ADDITIONAL_ITEMS}
      )
    set_target_properties(${name} PROPERTIES PREFIX "")
    if (MSVC)
      set_target_properties(${name} PROPERTIES SUFFIX ".pyd")
    endif(MSVC)
    if (APPLE)
      # https://github.com/facebookarchive/caffe2/issues/854#issuecomment-364538485
      # https://github.com/pytorch/pytorch/commit/73f6715f4725a0723d8171d3131e09ac7abf0666
      set_target_properties(${name} PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
    endif()
    install(
      TARGETS ${name}
      LIBRARY DESTINATION .
      RUNTIME DESTINATION .  # For Windows
      )
  endfunction()

  set(
    EXTENSION_SOURCES
    pybind/pybind.cpp
    )
  #----------------------------------------------------------------------------#
  # START OF CUSTOMIZATION LOGICS
  #----------------------------------------------------------------------------#
  if(BUILD_SOX)
    list(
      APPEND
      EXTENSION_SOURCES
      pybind/sox/effects.cpp
      pybind/sox/effects_chain.cpp
      pybind/sox/io.cpp
      pybind/sox/utils.cpp
      )
  endif()
  #----------------------------------------------------------------------------#
  # END OF CUSTOMIZATION LOGICS
  #----------------------------------------------------------------------------#
  define_extension(
    _torchaudio
    "${EXTENSION_SOURCES}"
    libtorchaudio
    "${LIBTORCHAUDIO_COMPILE_DEFINITIONS}"
    )
  if(BUILD_CTC_DECODER)
    set(
      DECODER_EXTENSION_SOURCES
      decoder/bindings/pybind.cpp
      )
    define_extension(
      _torchaudio_decoder
      "${DECODER_EXTENSION_SOURCES}"
      "libtorchaudio_decoder"
      "${LIBTORCHAUDIO_DECODER_DEFINITIONS}"
      )
  endif()
endif()
