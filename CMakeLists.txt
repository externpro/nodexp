cmake_minimum_required(VERSION 3.31)
set(CMAKE_PROJECT_TOP_LEVEL_INCLUDES .devcontainer/cmake/xproinc.cmake)
project(nodexp VERSION 22.19.0)
# https://nodejs.org/dist/v22.19.0/SHASUMS256.txt
set(SHA256_win64 ea3fad0e67a991d8477d8c01344b56e69c676ccb733f065b22436994b1253f86) # node-v22.19.0-win-x64.zip
set(SHA256_hders 12a22367bd09674d04b95265f14295d737892bd682b99aad3122364d36e1ac8f) # node-v22.19.0-headers.tar.xz
set(SHA256_ndlib 4af0951712fb05a686a03e0592880e195ba53e5eb70e224d7ea7b8b76f2a3e86) # win-x64/node.lib
set(SHA256_Linux c0649af18e6a24f6fe5535a3e86b341dd49a8e71117c8b68bde973ef834f16f2) # node-v22.19.0-linux-x64.tar.xz
set(SHA256_arm64 0b2d9f564b6594222a62c82e1df2efe119dd4a4aff29644f4dd325bf360b6bcc) # node-v22.19.0-linux-arm64.tar.xz
set(SHA256_macos 1c3a9e78da501bbc1f0c99fbbb69bb7c722bc7a9bf30128b21ea502f3905892a) # node-v22.19.0-darwin-arm64.tar.xz
include(FetchContent)
cmake_policy(SET CMP0168 OLD) # don't want to see download progress
string(TOLOWER ${CMAKE_SYSTEM_NAME} os)
set(ver ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH})
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  set(os win)
  set(arch x64)
  set(ext zip)
  set(sha ${SHA256_win64})
  set(url_hders https://nodejs.org/dist/v${ver}/node-v${ver}-headers.tar.xz)
  set(url_ndlib https://nodejs.org/dist/v${ver}/${os}-${arch}/node.lib)
  set(opt_ndlib DOWNLOAD_NO_EXTRACT TRUE)
  foreach(dl hders ndlib)
    set(dlName ${PROJECT_NAME}_${dl})
    FetchContent_Declare(${dlName} URL ${url_${dl}} URL_HASH SHA256=${SHA256_${dl}} ${opt_${dl}})
    FetchContent_MakeAvailable(${dlName})
    set(pth_${dl} ${FETCHCONTENT_BASE_DIR}/${dlName}-src)
  endforeach()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  set(bin "bin/")
  set(ext tar.xz)
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    set(arch x64)
    set(sha ${SHA256_Linux})
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64|ARM64)$")
    set(arch arm64)
    set(sha ${SHA256_arm64})
  else()
    message(FATAL_ERROR "Unsupported architecture: ${CMAKE_SYSTEM_PROCESSOR}")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  set(bin "bin/")
  set(ext tar.xz)
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64|ARM64)$")
    set(arch arm64)
    set(sha ${SHA256_macos})
  else()
    message(FATAL_ERROR "Unsupported architecture: ${CMAKE_SYSTEM_PROCESSOR}")
  endif()
else()
  message(FATAL_ERROR "Unsupported OS: ${CMAKE_SYSTEM_NAME}")
endif()
set(url https://nodejs.org/dist/v${ver}/node-v${ver}-${os}-${arch}.${ext})
set(fcName ${PROJECT_NAME}_fc)
FetchContent_Declare(${fcName} URL ${url} URL_HASH SHA256=${sha})
FetchContent_MakeAvailable(${fcName})
string(TOLOWER ${fcName} fcName)
set(pth ${FETCHCONTENT_BASE_DIR}/${fcName}-src)
if(DEFINED XP_NAMESPACE)
  set(nameSpace ${XP_NAMESPACE}::)
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  string(JOIN "\n" EXT3
    "if(NOT TARGET ${nameSpace}node)"
    "  add_library(${nameSpace}node STATIC IMPORTED)"
    "  set_property(TARGET ${nameSpace}node APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)"
    "  set_target_properties(${nameSpace}node PROPERTIES"
    "    IMPORTED_LINK_INTERFACE_LANGUAGES_RELEASE \"C;CXX\""
    "    IMPORTED_LOCATION_RELEASE \"\${PKG_ROOTDIR}/lib/node\${CMAKE_STATIC_LIBRARY_SUFFIX}\""
    "    )"
    "endif()"
    ""
    )
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  string(JOIN "\n" EXT3
    "if(NOT TARGET ${nameSpace}node)"
    "  add_library(${nameSpace}node INTERFACE IMPORTED)"
    "endif()"
    ""
    )
endif()
string(JOIN "\n" EXT4
  "set(incDirs \${PKG_ROOTDIR}/include \${PKG_ROOTDIR}/include/node)"
  "set_target_properties(${nameSpace}node PROPERTIES"
  "  INTERFACE_INCLUDE_DIRECTORIES \"\${incDirs}\""
  "  )"
  ""
  )
if(DEFINED XP_NAMESPACE)
  xpExternPackage(NAMESPACE ${XP_NAMESPACE}
    EXE_PATH ${bin}node LIBRARIES node
    BASE v0 XPDIFF "bin"
    WEB "https://nodejs.org/en/blog/release/v22.19.0/"
    DESC "node/npm development platform and runtime executable bundled as externpro devel package to build addons"
    LICENSE "[MIT](https://raw.githubusercontent.com/nodejs/node/v22.19.0/LICENSE 'MIT License')"
    )
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  set(root_programs
    ${pth}/node${CMAKE_EXECUTABLE_SUFFIX}
    ${pth}/npm
    ${pth}/npx
    )
  set(root_files
    ${pth}/nodevars.bat
    ${pth}/npm.cmd
    ${pth}/npx.cmd
    )
  install(PROGRAMS ${root_programs} DESTINATION .)
  install(FILES ${root_files} DESTINATION .)
  install(DIRECTORY ${pth}/node_modules DESTINATION . USE_SOURCE_PERMISSIONS)
  install(FILES ${pth_ndlib}/node${CMAKE_STATIC_LIBRARY_SUFFIX} DESTINATION ${CMAKE_INSTALL_LIBDIR})
  set(hdrs ${pth_hders}/include/)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  install(DIRECTORY ${pth}/bin DESTINATION . USE_SOURCE_PERMISSIONS)
  install(DIRECTORY ${pth}/lib DESTINATION . USE_SOURCE_PERMISSIONS)
  set(hdrs ${pth}/include/)
endif()
install(DIRECTORY ${hdrs} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  PATTERN "openssl" EXCLUDE PATTERN "zlib.h" EXCLUDE PATTERN "zconf.h" EXCLUDE
  )
install(FILES ${pth}/CHANGELOG.md ${pth}/LICENSE ${pth}/README.md DESTINATION ${CMAKE_INSTALL_DATADIR})
