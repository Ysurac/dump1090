cmake_minimum_required (VERSION 2.8.9)
project (Dump1090-FlightAirMap)
set (${PROJECT_NAME}_VERSION_MAJOR 1)
set (${PROJECT_NAME}_VERSION_MINOR 0)

set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
set (CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Wall -Werror -ggdb -D_GLIBCXX_DEBUG -D_GLIBCXX_DEBUG_PEDANTIC")

# Set default build type
IF( NOT CMAKE_BUILD_TYPE )
  SET( CMAKE_BUILD_TYPE "release" )
ENDIF()

list (APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)
set (EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)


if (WIN32)
    set(MATH_LIB "")
    set(RT_LIBRARIES "")
    INCLUDE(ExternalProject)
    
    message(STATUS "Configuring Curl")
    set(CURL_FLAGS -DBUILD_CURL_EXE=OFF -DBUILD_CURL_TESTS=OFF -DCURL_STATICLIB=ON -DCMAKE_USE_OPENSSL=OFF -DCURL_ZLIB=OFF -DHTTP_ONLY=ON)
    ExternalProject_Add(
	curl_src
	GIT_REPOSITORY https://github.com/bagder/curl.git
	#SOURCE_DIR ${CMAKE_SOURCE_DIR}/curl/curl_src
	UPDATE_COMMAND ""
	#BUILD_COMMAND ${MAKE} -f Makefile.m32 CROSSPREFIX=i686-pc-mingw32- CFG=-winssl-ipv6
	COMMAND cp COPYING ${CMAKE_BINARY_DIR}/bin/COPYING.curl
	CMAKE_ARGS -DBUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR} -DCMAKE_TOOLCHAIN_FILE=${TOOLCHAIN_CONF} ${CURL_FLAGS}
    )

#    message(STATUS "Configuring WinPhtreads")
#    ExternalProject_Add(
#	winpthreads_src
#	DOWNLOAD_COMMAND ""
#	SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/compat/windows/winpthreads/src
#	CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_BINARY_DIR} --host=i686-pc-mingw32 --target=i686-pc-mingw32 --enable-static
#	BUILD_COMMAND make
#	INSTALL_COMMAND make install
#    )
    message(STATUS "Configuring WinPthreads")
    ExternalProject_Add(
	winpthreads_src
	URL "http://downloads.sourceforge.net/project/mingw-w64/mingw-w64/mingw-w64-release/mingw-w64-v4.0.4.tar.bz2"
#	SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/compat/windows/winpthreads/src
#	CONFIGURE_COMMAND mingw-w64-libraries/winpthreads/configure --prefix=${CMAKE_BINARY_DIR} --host=i686-pc-mingw32 --target=i686-pc-mingw32 --enable-static
#	SOURCE_DIR mingw-w64-libraries/winpthreads
	PATCH_COMMAND patch -p0 < ${CMAKE_SOURCE_DIR}/cmake/winpthreads.patch
	CONFIGURE_COMMAND cd mingw-w64-libraries/winpthreads && ./configure --prefix=${CMAKE_BINARY_DIR} --host=i686-pc-mingw32 --target=i686-pc-mingw32 --enable-static
#	CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_BINARY_DIR} --host=${TARGET_ARCH} --target=${TARGET_ARCH} --enable-static --disable-shared
	BUILD_IN_SOURCE 1
	COMMAND cp COPYING ${CMAKE_BINARY_DIR}/bin/COPYING.WinPthreads

	BUILD_COMMAND cd mingw-w64-libraries/winpthreads && make
	INSTALL_COMMAND cd mingw-w64-libraries/winpthreads && make install
    )

    message(STATUS "Configuring LibUSB")
    ExternalProject_Add(
	libusb_src
	GIT_REPOSITORY https://github.com/libusb/libusb.git
	BUILD_IN_SOURCE 1
	#SOURCE_DIR ${CMAKE_SOURCE_DIR}/rtlsdr/libusb_src
	UPDATE_COMMAND ""
	CONFIGURE_COMMAND ./bootstrap.sh
		  COMMAND ./configure --prefix=${CMAKE_BINARY_DIR} --host=${TARGET_ARCH} --target=${TARGET_ARCH} --enable-static
	COMMAND cp COPYING ${CMAKE_BINARY_DIR}/bin/COPYING.LibUSB
	BUILD_COMMAND make
	INSTALL_COMMAND make install
    )    

    message(STATUS "Configuring RTLSDR")
    ExternalProject_Add(
	rtlsdr_src
	DEPENDS libusb_src winpthreads_src
	GIT_REPOSITORY https://github.com/keenerd/rtl-sdr.git
#	SOURCE_DIR ${CMAKE_SOURCE_DIR}/rtlsdr/rtlsdr_src
#	BUILD_IN_SOURCE 1
	UPDATE_COMMAND ""
	COMMAND cp COPYING ${CMAKE_BINARY_DIR}/bin/COPYING.RTLSDR
	CMAKE_ARGS 
	    -DBUILD_SHARED_LIBS=OFF
	    -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}
	    -DCMAKE_TOOLCHAIN_FILE=${TOOLCHAIN_CONF}
	    -DLIBUSB_INCLUDE_DIR=${CMAKE_BINARY_DIR}/include/libusb-1.0
	    -DLIBUSB_LIBRARIES=${CMAKE_BINARY_DIR}/lib/libusb-1.0.a
	    -DTHREADS_PTHREADS_INCLUDE_DIR=${CMAKE_BINARY_DIR}/include
	    -DTHREADS_PTHREADS_WIN32_LIBRARY=${CMAKE_BINARY_DIR}/lib/libwinpthread.a
    )    
#    include_directories(.)
    include_directories(compat)
    include_directories(compat/endian)
#    include_directories(compat/windows/winpthreads)
#    link_directories(compat/windows/winpthreads)
    include_directories(${CMAKE_BINARY_DIR}/include)
    link_directories(${CMAKE_BINARY_DIR}/lib)
#    link_directories(rtlsdr)
#    link_directories(curl/bin/lib)
    set(THREAD_LIB pthread)
    set(RTLSDR_LIBRARY rtlsdr)
    set(LIBUSB_1_LIBRARIES usb-1.0)
    set(COMPAT wsock32)
    set(CURL_LIBRARIES curl ws2_32)
else(WIN32)
    find_package(CURL REQUIRED)
    find_package(Threads REQUIRED)
    find_package(LibUSB REQUIRED)
    find_package(RTLSDR REQUIRED)

    set(MATH_LIB "m")
    include_directories(${RT_INCLUDES})
    include_directories(${THREADS_PTHREADS_INCLUDE_DIR})
    set(THREAD_LIB ${CMAKE_THREAD_LIBS_INIT})
    include_directories(${LIBUSB_1_INCLUDE_DIRS})
    include_directories(${CURL_INCLUDE_DIRS})
    include_directories(${RTLSDR_INCLUDE_DIR})
    set(COMPAT "")
endif()

set (DUMP1090_SRC dump1090.c anet.c interactive.c mode_ac.c mode_s.c net_io.c crc.c demod_2000.c demod_2400.c stats.c cpr.c icao_filter.c track.c util.c convert.c)
set (VIEW1090_SRC view1090.c anet.c interactive.c mode_ac.c mode_s.c net_io.c crc.c stats.c cpr.c icao_filter.c track.c util.c)
set (FAUP1090_SRC faup1090.c anet.c mode_ac.c mode_s.c net_io.c crc.c stats.c cpr.c icao_filter.c track.c util.c)
set (ZFAMUP1090_SRC zfamup1090.c anet.c mode_ac.c mode_s.c net_io.c crc.c stats.c cpr.c icao_filter.c track.c util.c)

add_executable(dump1090 ${DUMP1090_SRC})
add_executable(view1090 ${VIEW1090_SRC})
add_executable(faup1090 ${FAUP1090_SRC})
add_executable(zfamup1090 ${ZFAMUP1090_SRC})

target_link_libraries(dump1090 ${THREAD_LIB} ${MATH_LIB} ${RT_LIBRARIES} ${RTLSDR_LIBRARY} ${LIBUSB_1_LIBRARIES} ${COMPAT})
target_link_libraries(view1090 ${THREAD_LIB} ${MATH_LIB} ${COMPAT})
target_link_libraries(faup1090 ${THREAD_LIB} ${MATH_LIB} ${COMPAT})
target_link_libraries(zfamup1090 ${THREAD_LIB} ${MATH_LIB} ${CURL_LIBRARIES} ${COMPAT})

if(WIN32)
add_dependencies(dump1090 rtlsdr_src winpthreads_src)
add_dependencies(zfamup1090 curl_src winpthreads_src)
endif()

if(WIN32)
    set(CPACK_GENERATOR "ZIP")
    set(CPACK_MONOLITHIC_INSTALL ON)
    set(CPACK_ZIP_COMPONENT_INSTALL OFF)
    set(CPACK_INCLUDE_TOPLEVEL_DIRECTORY ON)
    set(CPACK_COMPONENT_INCLUDE_TOPLEVEL_DIRECTORY ON)
    set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
#    install(TARGETS dump1090 RUNTIME DESTINATION .)
    install(DIRECTORY ${CMAKE_BINARY_DIR}/bin/ DESTINATION . PATTERN "rtl*" EXCLUDE PATTERN "curl-config" EXCLUDE)
    include(CPack)
endif()

#set(CPACK_DEBIAN_PACKAGE_NAME Dump1090-FlightAirMap)
#set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE amd64)
#    configure_file(
#        ${PROJECT_SOURCE_DIR}/etc/Linux/postrm.in
#        ${PROJECT_BINARY_DIR}/etc/Linux/postrm
#        @ONLY)
#    set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA
#        ${PROJECT_BINARY_DIR}/etc/Linux/postinst
#        ${PROJECT_BINARY_DIR}/etc/Linux/postrm)
#set(CPACK_PACKAGE_VERSION_MAJOR ${djvVersionMajor})
#set(CPACK_PACKAGE_VERSION_MINOR ${djvVersionMinor})
#set(CPACK_PACKAGE_VERSION_PATCH ${djvVersionPatch})
#set(CPACK_PACKAGE_CONTACT http://djv.sourceforge.net)
#set(CPACK_PACKAGE_DESCRIPTION_FILE ${PROJECT_SOURCE_DIR}/README)
#set(CPACK_PACKAGE_DESCRIPTION_SUMMARY
#    "DJV Imaging provides professional movie playback software for use in film, VFX, and computer animation.")
#set(CPACK_PACKAGE_EXECUTABLES djv_view djv_view)
#set(CPACK_RESOURCE_FILE_LICENSE ${PROJECT_SOURCE_DIR}/LICENSE)
#set(CPACK_PACKAGE_FILE_NAME ${djvPackageName})
