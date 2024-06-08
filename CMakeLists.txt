function(add_simple_commonlibsse_ng_plugin)
  set(options OPTIONAL USE_ADDRESS_LIBRARY USE_SIGNATURE_SCANNING
              STRUCT_DEPENDENT EXCLUDE_FROM_ALL)
  set(oneValueArgs
      NAME
      AUTHOR
      EMAIL
      VERSION
      MINIMUM_SKSE_VERSION
      SRC_FOLDER
      LICENSE)
  set(multiValueArgs COMPATIBLE_RUNTIMES SOURCES)
  cmake_parse_arguments(PARSE_ARGV 0 ADD_SIMPLE_COMMONLIBSSE_NG_PLUGIN
                        "${options}" "${oneValueArgs}" "${multiValueArgs}")

  set_from_environment(OUTPUT_DIRS)
  message("Using toolchain file ${CMAKE_TOOLCHAIN_FILE}.")
  message("CLIBNGPlugin.cmake options:")
  option(COPY_BUILD
         "Copy the build output to the Skyrim directory, also need OUTPUT_DIRS"
         ON)
  message("\tCopy build output: ${COPY_BUILD} to OUTPUT_DIRS: ${OUTPUT_DIRS}")

  set(SRC_FOLDER "src")
  if(DEFINED ADD_SIMPLE_COMMONLIBSSE_NG_PLUGIN_SRC_FOLDER)
    set(SRC_FOLDER "${ADD_SIMPLE_COMMONLIBSSE_NG_PLUGIN_SRC_FOLDER}")
  endif()

  _add_cxx_files("${SRC_FOLDER}")

  set(CommonLibPath "extern/CommonLibSSE-NG")

  set(BUILD_TESTS OFF)
  add_subdirectory(${CommonLibPath} ${CommonLibName} EXCLUDE_FROM_ALL)
  include(${CommonLibPath}/cmake/CommonLibSSE.cmake)

  add_commonlibsse_plugin(${PROJECT_NAME} ${ARGN} SOURCES ${SOURCE_FILES}
                          ${HEADER_FILES})

  target_compile_features("${PROJECT_NAME}" PRIVATE cxx_std_23)

  _generate_version_rc("${ADD_SIMPLE_COMMONLIBSSE_NG_PLUGIN_LICENSE}"
                       "${ADD_SIMPLE_COMMONLIBSSE_NG_PLUGIN_AUTHOR}")
  target_sources("${PROJECT_NAME}"
                 PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/cmake/version.rc)

  target_precompile_headers("${PROJECT_NAME}" PRIVATE include/PCH.h)

  target_include_directories(
    ${PROJECT_NAME}
    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FOLDER})

  set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
  set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DEBUG OFF)
  set(CMAKE_OPTIMIZE_DEPENDENCIES ON)

  if(CMAKE_GENERATOR MATCHES "Visual Studio")
    _msvc_specific_init()
  endif()

  if(COPY_BUILD)
    _setup_copy_build()
  endif()
endfunction()

function(_add_cxx_files SRC_FOLDER)
  file(
    GLOB_RECURSE INCLUDE_FILES
    LIST_DIRECTORIES false
    CONFIGURE_DEPENDS "include/*.h" "include/*.hpp")

  source_group(
    TREE ${CMAKE_CURRENT_SOURCE_DIR}/include
    PREFIX "Header Files"
    FILES ${INCLUDE_FILES})

  file(
    GLOB_RECURSE HEADER_FILES
    LIST_DIRECTORIES false
    CONFIGURE_DEPENDS "${SRC_FOLDER}/*.h" "${SRC_FOLDER}/*.hpp")

  source_group(
    TREE ${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FOLDER}
    PREFIX "Header Files"
    FILES ${HEADER_FILES})

  list(APPEND HEADER_FILES ${INCLUDE_FILES})
  list(REMOVE_ITEM HEADER_FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/PCH.h)
  set(HEADER_FILES
      ${HEADER_FILES}
      PARENT_SCOPE)

  file(
    GLOB_RECURSE SOURCE_FILES
    LIST_DIRECTORIES false
    CONFIGURE_DEPENDS "${SRC_FOLDER}/*.cpp")

  set(SOURCE_FILES
      ${SOURCE_FILES}
      PARENT_SCOPE)

  source_group(
    TREE ${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FOLDER}
    PREFIX "Source Files"
    FILES ${SOURCE_FILES})

  set_property(GLOBAL PROPERTY USE_FOLDERS ON)
endfunction()

macro(set_from_environment VARIABLE)
  if(NOT DEFINED ${VARIABLE} AND DEFINED ENV{${VARIABLE}})
    set(${VARIABLE} $ENV{${VARIABLE}})
  endif()
endmacro()

function(_msvc_specific_init)
  # https://gitlab.kitware.com/cmake/cmake/-/issues/24922#note_1371990
  if(MSVC_VERSION GREATER_EQUAL 1936 AND MSVC_IDE) # 17.6+
    # When using /std:c++latest, "Build ISO C++23 Standard Library Modules"
    # defaults to "Yes". Default to "No" instead.
    #
    # As of CMake 3.26.4, there isn't a way to control this property
    # (https://gitlab.kitware.com/cmake/cmake/-/issues/24922), We'll use the
    # MSBuild project system instead
    # (https://learn.microsoft.com/en-us/cpp/build/reference/vcxproj-file-structure)
    file(
      CONFIGURE
      OUTPUT
      "${CMAKE_BINARY_DIR}/Directory.Build.props"
      CONTENT
      [==[
<Project>
    <ItemDefinitionGroup>
        <ClCompile>
            <BuildStlModules>false</BuildStlModules>
        </ClCompile>
    </ItemDefinitionGroup>
</Project>
      ]==]
      @ONLY)
  endif()

  add_compile_definitions(_UNICODE)

  target_compile_definitions(${PROJECT_NAME} PRIVATE "$<$<CONFIG:DEBUG>:DEBUG>")

  set(SC_RELEASE_OPTS
      "/Zi;/fp:fast;/GL;/Gy-;/Gm-;/Gw;/sdl-;/GS-;/guard:cf-;/O2;/Ob2;/Oi;/Ot;/Oy;/fp:except-"
  )

  target_compile_options(
    "${PROJECT_NAME}"
    PRIVATE /MP
            /W3
            /WX
            /permissive-
            /Zc:alignedNew
            /Zc:auto
            /Zc:__cplusplus
            /Zc:externC
            /Zc:externConstexpr
            /Zc:forScope
            /Zc:hiddenFriend
            /Zc:implicitNoexcept
            /Zc:lambda
            /Zc:noexceptTypes
            /Zc:preprocessor
            /Zc:referenceBinding
            /Zc:rvalueCast
            /Zc:sizedDealloc
            /Zc:strictStrings
            /Zc:ternary
            /Zc:threadSafeInit
            /Zc:trigraphs
            /Zc:wchar_t
            /wd4200 # nonstandard extension used : zero-sized array in
                    # struct/union
            /arch:AVX)

  target_compile_options(${PROJECT_NAME} PUBLIC "$<$<CONFIG:DEBUG>:/fp:strict>")
  target_compile_options(${PROJECT_NAME} PUBLIC "$<$<CONFIG:DEBUG>:/ZI>")
  target_compile_options(${PROJECT_NAME} PUBLIC "$<$<CONFIG:DEBUG>:/Od>")
  target_compile_options(${PROJECT_NAME} PUBLIC "$<$<CONFIG:DEBUG>:/Gy>")
  target_compile_options(${PROJECT_NAME}
                         PUBLIC "$<$<CONFIG:RELEASE>:${SC_RELEASE_OPTS}>")

  target_link_options(
    ${PROJECT_NAME} PRIVATE /WX
    "$<$<CONFIG:DEBUG>:/INCREMENTAL;/OPT:NOREF;/OPT:NOICF>"
    "$<$<CONFIG:RELEASE>:/LTCG;/INCREMENTAL:NO;/OPT:REF;/OPT:ICF;/DEBUG:FULL>")
endfunction()

function(_generate_version_rc LICENCE AUTHOR)
  file(
    CONFIGURE
    OUTPUT
    "${CMAKE_BINARY_DIR}/cmake/version.rc"
    CONTENT
    [==[
#include <winres.h>

1 VERSIONINFO
  FILEVERSION @PROJECT_VERSION_MAJOR@, @PROJECT_VERSION_MINOR@, @PROJECT_VERSION_PATCH@, 0
  PRODUCTVERSION @PROJECT_VERSION_MAJOR@, @PROJECT_VERSION_MINOR@, @PROJECT_VERSION_PATCH@, 0
  FILEFLAGSMASK 0x17L
#ifdef _DEBUG
  FILEFLAGS 0x1L
#else
  FILEFLAGS 0x0L
#endif
  FILEOS 0x4L
  FILETYPE 0x1L
  FILESUBTYPE 0x0L
BEGIN
  BLOCK "StringFileInfo"
  BEGIN
    BLOCK "040904b0"
    BEGIN
      VALUE "FileDescription", "@PROJECT_NAME@"
      VALUE "FileVersion", "@PROJECT_VERSION@.0"
      VALUE "InternalName", "@PROJECT_NAME@"
      VALUE "LegalCopyright", "@LICENCE@"
      VALUE "ProductName", "@PROJECT_NAME@"
      VALUE "ProductVersion", "@PROJECT_VERSION@.0"
      VALUE "Author", "@AUTHOR@"
    END
  END
  BLOCK "VarFileInfo"
  BEGIN
    VALUE "Translation", 0x409, 1200
  END
END
  ]==]
    @ONLY)
endfunction()

function(_setup_copy_build)
  if(DEFINED OUTPUT_DIRS)
    foreach(OUTPUT_DIR ${OUTPUT_DIRS})
      add_custom_command(
        TARGET ${PROJECT_NAME}
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory "${OUTPUT_DIR}/SKSE/Plugins")

      message("Will copy dll and pdb to ${OUTPUT_DIR}/SKSE/Plugins on build")
      add_custom_command(
        TARGET ${PROJECT_NAME}
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${PROJECT_NAME}>
                ${OUTPUT_DIR}/SKSE/Plugins/
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_PDB_FILE:${PROJECT_NAME}>
                ${OUTPUT_DIR}/SKSE/Plugins/)
    endforeach()
  else()
    message(
      WARNING
        "Variable OUTPUT_DIRS is not defined. Skipping post-build copy command."
    )
  endif()
endfunction()
