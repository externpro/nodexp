cmake_minimum_required(VERSION 3.31.7)
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/.devcontainer/cmake)
include(preproject)
project(nodexp VERSION 14.21.3.0)
# https://nodejs.org/dist/v14.21.3/SHASUMS256.txt
set(SHA256_Linux 05c08a107c50572ab39ce9e8663a2a2d696b5d262d5bd6f98d84b997ce932d9a)
set(SHA256_win64 47cfb919bb86ab681369636a9cb925e2bd61991aad1638b2e38e61ec956796a6)
set(SHA256_arm64 f06642bfcf0b8cc50231624629bec58b183954641b638e38ed6f94cd39e8a6ef)
set(SHA256_hders b5393b510a579e9bc48bce01dbc637f578a3bbc5cb50c757d8fc2873e06ac7ae)
set(SHA256_ndlib 5226634dfd3f9a99dd40ca374fe654b6ed4fdb1890c629a3426e34e212fb011c)
set(webpro_REV NONE)
include(flags)
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
  set(arch x64)
  set(sha ${SHA256_Linux})
endif()
if(DEFINED ENV{XPX} AND "$ENV{XPX}" STREQUAL "JetPack")
  set(arch arm64)
  set(sha ${SHA256_arm64})
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
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
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
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  install(DIRECTORY ${pth}/bin DESTINATION . USE_SOURCE_PERMISSIONS)
  install(DIRECTORY ${pth}/lib DESTINATION . USE_SOURCE_PERMISSIONS)
  set(hdrs ${pth}/include/)
endif()
install(DIRECTORY ${hdrs} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  PATTERN "openssl" EXCLUDE PATTERN "zlib.h" EXCLUDE PATTERN "zconf.h" EXCLUDE
  )
install(FILES ${pth}/CHANGELOG.md ${pth}/LICENSE ${pth}/README.md DESTINATION ${CMAKE_INSTALL_DATADIR})
