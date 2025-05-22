cmake_minimum_required(VERSION 3.31.7)
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/.devcontainer/cmake)
include(preproject)
project(nodexp VERSION 14.17.6.0)
# https://nodejs.org/dist/v14.17.6/SHASUMS256.txt
set(SHA256_Linux 3bbe4faf356738d88b45be222bf5e858330541ff16bd0d4cfad36540c331461b)
set(SHA256_win64 b83e9ce542fda7fc519cec6eb24a2575a84862ea4227dedc171a8e0b5b614ac0)
set(SHA256_arm64 9c4f3a651e03cd9b5bddd33a80e8be6a6eb15e518513e410bb0852a658699156)
set(SHA256_hders 9429d544af0507c92f309fb1a0b7c2082b52846bdff3cd8aa039ada0461c95b9)
set(SHA256_ndlib e456fef578acd36eb661055445b4a06c81c47b2cfe0066619e7faa135137d4f6)
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
