cmake_minimum_required(VERSION 3.31)
set(CMAKE_PROJECT_TOP_LEVEL_INCLUDES .devcontainer/cmake/xproinc.cmake)
project(nodexp VERSION 22.18.0)
# https://nodejs.org/dist/v22.18.0/SHASUMS256.txt
set(SHA256_win64 c95d8a7e1c99e669cc08c9f1176e068c1f50847c37908fcb8c35b62482366511) # node-v22.18.0-win-x64.zip
set(SHA256_hders 68a214508678fc6119bf1ad391fbf89fb624ab000d85693ef7c8a764b6622351) # node-v22.18.0-headers.tar.xz
set(SHA256_ndlib 4af0951712fb05a686a03e0592880e195ba53e5eb70e224d7ea7b8b76f2a3e86) # win-x64/node.lib
set(SHA256_Linux c1bfeecf1d7404fa74728f9db72e697decbd8119ccc6f5a294d795756dfcfca7) # node-v22.18.0-linux-x64.tar.xz
set(SHA256_arm64 04fca1b9afecf375f26b41d65d52aa1703a621abea5a8948c7d1e351e85edade) # node-v22.18.0-linux-arm64.tar.xz
set(SHA256_macos 6616f388e127c858989fc7fa92879cdb20d2a5d446adbfdca6ee4feb385bfa8a) # node-v22.18.0-darwin-arm64.tar.xz
include(xpflags)
include(FetchContent)
cmake_policy(SET CMP0168 OLD) # don't want to see download progress
include(GNUInstallDirs)
set(XP_INSTALL_CMAKEDIR ${CMAKE_INSTALL_DATADIR}/cmake) # xpPackageDevel
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
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  string(JOIN "\n" EXT3
    "if(NOT TARGET xpro::node)"
    "  add_library(xpro::node STATIC IMPORTED)"
    "  set_property(TARGET xpro::node APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)"
    "  set_target_properties(xpro::node PROPERTIES"
    "    IMPORTED_LINK_INTERFACE_LANGUAGES_RELEASE \"C;CXX\""
    "    IMPORTED_LOCATION_RELEASE \"\${PKG_ROOTDIR}/lib/node${CMAKE_STATIC_LIBRARY_SUFFIX}\""
    "    )"
    "endif()"
    ""
    )
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  string(JOIN "\n" EXT3
    "if(NOT TARGET xpro::node)"
    "  add_library(xpro::node INTERFACE IMPORTED)"
    "endif()"
    ""
    )
endif()
string(JOIN "\n" EXT4
  "set(incDirs \${PKG_ROOTDIR}/include \${PKG_ROOTDIR}/include/node)"
  "set_target_properties(xpro::node PROPERTIES"
  "  INTERFACE_INCLUDE_DIRECTORIES \"\${incDirs}\""
  "  )"
  ""
  )
xpPackageDevel(EXE_PATH ${bin}node LIBRARIES xpro::node)
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
