cmake_minimum_required (VERSION 3.4)
project (nano-core)

set (CPACK_PACKAGE_VERSION_MAJOR "18")
set (CPACK_PACKAGE_VERSION_MINOR "0")
set (CPACK_PACKAGE_VERSION_PATCH "0")
if (DEFINED GIT_COMMIT)
	set (CPACK_PACKAGE_VERSION_PATCH "GIT-${GIT_COMMIT}")
endif ()
set (CPACK_PACKAGE_VENDOR "Nano Currency")

set(CMAKE_INSTALL_RPATH "@executable_path/../Frameworks")
# Create all libraries and executables in the root binary dir
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

set (NANO_GUI OFF CACHE BOOL "")
set (NANO_TEST OFF CACHE BOOL "")
set (NANO_SECURE_RPC OFF CACHE BOOL "")

option(NANO_ASAN_INT "Enable ASan+UBSan+Integer overflow" OFF)
option(NANO_ASAN "Enable ASan+UBSan" OFF)
option(NANO_SIMD_OPTIMIZATIONS "Enable CPU-specific SIMD optimizations (SSE/AVX or NEON, e.g.)" OFF)

SET (ACTIVE_NETWORK nano_live_network CACHE STRING "Selects which network parameters are used")
set_property (CACHE ACTIVE_NETWORK PROPERTY STRINGS nano_test_network nano_beta_network nano_live_network)

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# TODO check if any library requires these
#find_package(Threads REQUIRED)
#set (PLATFORM_LIBS ${CMAKE_THREAD_LIBS_INIT} ${CMAKE_DL_LIBS})

if (WIN32)
	add_definitions(-D_WIN32_WINNT=0x0600
					-DWINVER=0x0600
					-DWIN32_LEAN_AND_MEAN
					-DMINIUPNP_STATICLIB)
else ()
	add_compile_options(-Werror=switch)
	if (NANO_ASAN OR RAIBLOCKS_ASAN)
		add_compile_options(-fsanitize=address,undefined)
		add_definitions(-DED25519_NO_INLINE_ASM)
	elseif (NANO_ASAN_INT OR RAIBLOCKS_ASAN_INT)
		add_compile_options(-fsanitize=address,undefined,integer)
		add_definitions(-DED25519_NO_INLINE_ASM)
	elseif (NANO_TSAN OR RAIBLOCKS_TSAN)
		add_compile_options(-fsanitize=thread)
		add_definitions(-DED25519_NO_INLINE_ASM)
	endif()

	IF (CMAKE_SYSTEM_PROCESSOR MATCHES "^(i.86|x86(_64)?)$")
		if (NANO_SIMD_OPTIMIZATIONS OR RAIBLOCKS_SIMD_OPTIMIZATIONS)
			add_compile_options(-msse4)
		elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
			add_compile_options(-msse2)
		endif()

		if (ENABLE_AVX2)
			add_compile_options(-mavx2 -mbmi -mbmi2)
			if (PERMUTE_WITH_GATHER)
				add_definitions(-DPERMUTE_WITH_GATHER)
			elseif (PERMUTE_WITH_SHUFFLES)
				add_definitions(-DPERMUTE_WITH_SHUFFLES)
			endif()
		endif()
	endif()
endif ()

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
#set(CMAKE_C_EXTENSIONS OFF)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
#set(CMAKE_CXX_EXTENSIONS OFF)

if (WIN32)
	add_definitions(/bigobj)
endif ()

if (APPLE)
	set (PLATFORM_LINK_FLAGS "-framework Foundation -framework OpenCL")
elseif (WIN32)
	set (PLATFORM_LINK_FLAGS "") 
else ()
	set (PLATFORM_LINK_FLAGS "-static-libgcc -static-libstdc++")
	if (NANO_ASAN OR RAIBLOCKS_ASAN)
		set (PLATFORM_LINK_FLAGS "${PLATFORM_LINK_FLAGS} -fsanitize=address,undefined")
	elseif (NANO_ASAN_INT OR RAIBLOCKS_ASAN_INT)
		set (PLATFORM_LINK_FLAGS "${PLATFORM_LINK_FLAGS} -fsanitize=address,undefined,integer")
	elseif (NANO_TSAN OR RAIBLOCKS_TSAN)
		set (PLATFORM_LINK_FLAGS "${PLATFORM_LINK_FLAGS} -fsanitize=thread")
	endif()
endif ()

SET( CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${PLATFORM_LINK_FLAGS}" )

if (NANO_SECURE_RPC OR RAIBLOCKS_SECURE_RPC)
	find_package (OpenSSL 1.0 REQUIRED)
	include_directories(${OPENSSL_INCLUDE_DIR})
	add_definitions (-DNANO_SECURE_RPC)
	message("OpenSSL include dir: ${OPENSSL_INCLUDE_DIR}")
	message("OpenSSL libraries: ${OPENSSL_LIBRARIES}")
	message("OpenSSL lib: ${OPENSSL_SSL_LIBRARY}")
	message("Crypto lib: ${OPENSSL_CRYPTO_LIBRARY}")
else ()
	set (OPENSSL_LIBRARIES "")
endif ()

include_directories (${CMAKE_SOURCE_DIR})

set(Boost_USE_STATIC_LIBS        ON)
set(Boost_USE_MULTITHREADED      ON)

if (BOOST_ROOT)
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules")
endif ()

find_package (Boost 1.66.0 REQUIRED COMPONENTS filesystem log thread program_options)

add_subdirectory(ed25519-donna)

set (UPNPC_BUILD_SHARED OFF CACHE BOOL "")
add_subdirectory (miniupnp/miniupnpc)
# FIXME: This fixes miniupnpc include directories without modifying miniupnpc's
# CMakeLists.txt but should be set there
set_target_properties(libminiupnpc-static PROPERTIES INTERFACE_INCLUDE_DIRECTORIES
	"${CMAKE_SOURCE_DIR}/miniupnp/miniupnpc")

set (BUILD_SHARED OFF CACHE BOOL "")
set (BUILD_TESTING OFF CACHE BOOL "")
set (USE_INTERMEDIATE_OBJECTS_TARGET OFF CACHE BOOL "")
if (CRYPTOPP_CUSTOM OR NOT (NANO_SIMD_OPTIMIZATIONS OR RAIBLOCKS_SIMD_OPTIMIZATIONS))
	set (CRYPTOPP_LIBRARY cryptopp)
	add_library (cryptopp
		cryptopp/algparam.cpp
		cryptopp/asn.cpp
		cryptopp/basecode.cpp
		cryptopp/cpu.cpp
		cryptopp/cryptlib.cpp
		cryptopp/default.cpp
		cryptopp/des.cpp
		cryptopp/dessp.cpp
		cryptopp/dll.cpp
		cryptopp/ec2n.cpp
		cryptopp/ecp.cpp
		cryptopp/filters.cpp
		cryptopp/fips140.cpp
		cryptopp/gcm.cpp
		cryptopp/gf2n.cpp
		cryptopp/gfpcrypt.cpp
		cryptopp/hex.cpp
		cryptopp/hmac.cpp
		cryptopp/hrtimer.cpp
		cryptopp/integer.cpp
		cryptopp/iterhash.cpp
		cryptopp/misc.cpp
		cryptopp/modes.cpp
		cryptopp/mqueue.cpp
		cryptopp/nbtheory.cpp
		cryptopp/oaep.cpp
		cryptopp/osrng.cpp
		cryptopp/pubkey.cpp
		cryptopp/queue.cpp
		cryptopp/randpool.cpp
		cryptopp/rdtables.cpp
		cryptopp/rijndael.cpp
		cryptopp/rng.cpp
		cryptopp/sha.cpp
		cryptopp/simple.cpp
		cryptopp/winpipes.cpp)
else ()
	set (CRYPTOPP_LIBRARY cryptopp-static)
	add_subdirectory (cryptopp)
endif ()

if (CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
	set (ARGON_CORE phc-winner-argon2/src/opt.c)
else ()
	set (ARGON_CORE phc-winner-argon2/src/ref.c)
endif ()

add_library (argon2
	phc-winner-argon2/src/argon2.c
	phc-winner-argon2/include/argon2.h
	phc-winner-argon2/src/core.c
	phc-winner-argon2/src/thread.c
	phc-winner-argon2/src/encoding.c
	${ARGON_CORE})

target_include_directories(argon2 PUBLIC phc-winner-argon2/include)
target_include_directories(argon2 PUBLIC phc-winner-argon2/src)
target_include_directories(argon2 PUBLIC blake2)

add_library (xxhash
	xxhash/xxhash.c
	xxhash/xxhash.h)

add_library (lmdb
	lmdb/libraries/liblmdb/lmdb.h
	lmdb/libraries/liblmdb/mdb.c
	lmdb/libraries/liblmdb/midl.c)

if (WIN32)
	target_link_libraries(lmdb ntdll)
endif()

if (WIN32)
	set (BLAKE2_IMPLEMENTATION "blake2/blake2b.c")
else ()
	IF (CMAKE_SYSTEM_PROCESSOR MATCHES "^(i.86|x86(_64)?)$")
		set (BLAKE2_IMPLEMENTATION "blake2/blake2b.c")
	else()
		set (BLAKE2_IMPLEMENTATION "blake2/blake2b-ref.c")
	endif()
endif ()

add_library (blake2
	blake2/blake2-config.h
	blake2/blake2-impl.h
	blake2/blake2.h
	${BLAKE2_IMPLEMENTATION})

target_compile_definitions(blake2 PRIVATE -D__SSE2__)

add_subdirectory(nano/secure)
add_subdirectory(nano/lib)
add_subdirectory(nano/node)
add_subdirectory(nano/nano_node)

if (NANO_TEST OR RAIBLOCKS_TEST)
    if(WIN32)
        if(MSVC_VERSION)
            if(MSVC_VERSION GREATER_EQUAL 1910)
                add_definitions(-DGTEST_LANG_CXX11=1)
                add_definitions(-DGTEST_HAS_TR1_TUPLE=0)
            endif()
        endif()
        set (gtest_force_shared_crt ON)
    else ()
        set (gtest_force_shared_crt OFF)
    endif()

    add_subdirectory (gtest)
	# FIXME: This fixes gtest include directories without modifying gtest's
	# CMakeLists.txt. Ideally we should use GTest::GTest and GTest::Main as
	# dependencies but it requires building gtest differently
	set_target_properties(gtest PROPERTIES INTERFACE_INCLUDE_DIRECTORIES
		"${CMAKE_SOURCE_DIR}/gtest/include")

	add_subdirectory(nano/core_test)
	add_subdirectory(nano/slow_test)
endif ()

if (NANO_GUI OR RAIBLOCKS_GUI)
	if (WIN32)
		set (PLATFORM_QT_PACKAGES WinExtras)
	else ()
		set (PLATFORM_QT_PACKAGES)
	endif ()

	find_package (Qt5 COMPONENTS Core Gui Widgets Test ${PLATFORM_QT_PACKAGES})

	add_library (qt
		nano/qt/qt.cpp
		nano/qt/qt.hpp)

	target_link_libraries(qt
		secure nano_lib node libminiupnpc-static Qt5::Gui Qt5::Widgets)

	target_compile_definitions(qt
		PRIVATE
		-DNANO_VERSION_MAJOR=${CPACK_PACKAGE_VERSION_MAJOR}
		-DNANO_VERSION_MINOR=${CPACK_PACKAGE_VERSION_MINOR})

	if (WIN32)
		set (PLATFORM_GUI_TARGET_PROPERTIES WIN32)
	else ()
		set (PLATFORM_GUI_TARGET_PROPERTIES "")
	endif ()

	if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
		set (PLATFORM_WALLET_SOURCE nano/nano_wallet/plat/default/icon.cpp)
	elseif (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
		set (PLATFORM_WALLET_SOURCE nano/nano_wallet/plat/windows/icon.cpp Nano.rc)
	elseif (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
		set (PLATFORM_WALLET_SOURCE nano/nano_wallet/plat/default/icon.cpp)
	elseif (${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
		set (PLATFORM_WALLET_SOURCE nano/nano_wallet/plat/default/icon.cpp)
	else ()
		error ("Unknown platform: ${CMAKE_SYSTEM_NAME}")
	endif ()

	qt5_add_resources(RES resources.qrc)

	add_executable (nano_wallet ${PLATFORM_GUI_TARGET_PROPERTIES}
		${PLATFORM_WALLET_SOURCE}
		nano/nano_wallet/entry.cpp
		nano/nano_wallet/icon.hpp
		${RES})

	target_link_libraries (nano_wallet
		node
		qt)

	if (WIN32)
		target_link_libraries (nano_wallet Qt5::WinExtras)
	endif()

	add_executable (qt_system
		nano/qt_system/entry.cpp)

	target_link_libraries (qt_system qt node Qt5::Gui Qt5::Widgets)

	set_target_properties (qt nano_wallet qt_system PROPERTIES COMPILE_FLAGS "-DQT_NO_KEYWORDS -DBOOST_ASIO_HAS_STD_ARRAY=1")

	if (NANO_TEST OR RAIBLOCKS_TEST)
		add_executable (qt_test
			nano/qt_test/entry.cpp
			nano/qt_test/qt.cpp)

		target_link_libraries(qt_test gtest gtest_main qt Qt5::Test)

		set_target_properties (qt_test PROPERTIES COMPILE_FLAGS "-DQT_NO_KEYWORDS -DBOOST_ASIO_HAS_STD_ARRAY=1")
	endif ()

	if (APPLE)
		install (TARGETS nano_wallet DESTINATION Nano.app/Contents/MacOS)
		install (FILES Info.plist DESTINATION Nano.app/Contents)
		install (FILES qt.conf DESTINATION Nano.app/Contents/Resources)
		install (DIRECTORY ${Qt5_DIR}/../../QtCore.framework DESTINATION Nano.app/Contents/Frameworks)
		install (DIRECTORY ${Qt5_DIR}/../../QtGui.framework DESTINATION Nano.app/Contents/Frameworks)
		install (DIRECTORY ${Qt5_DIR}/../../QtPrintSupport.framework DESTINATION Nano.app/Contents/Frameworks)
		install (DIRECTORY ${Qt5_DIR}/../../QtTest.framework DESTINATION Nano.app/Contents/Frameworks)
		install (DIRECTORY ${Qt5_DIR}/../../QtWidgets.framework DESTINATION Nano.app/Contents/Frameworks)
		install (FILES "${Qt5_DIR}/../../../plugins/platforms/libqcocoa.dylib" DESTINATION Nano.app/Contents/PlugIns/platforms)
		install (FILES Nano.icns DESTINATION Nano.app/Contents/Resources)
	elseif (WIN32)
		if ("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
			set (WIN_REDIST vc_redist.x64.exe)
		else ()
			set (WIN_REDIST vc_redist.x86.exe)
		endif ()
		set (CPACK_NSIS_EXTRA_INSTALL_COMMANDS "ExecWait '\\\"$INSTDIR\\\\${WIN_REDIST}\\\" /quiet /norestart'")
		set (CPACK_NSIS_MUI_ICON ${CMAKE_SOURCE_DIR}/Nano.ico)
		set (CPACK_NSIS_DISPLAY_NAME "Nano")
		set (CPACK_PACKAGE_NAME "Nano_Installer")
		set (CPACK_NSIS_PACKAGE_NAME "Nano ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
		set (CPACK_NSIS_URL_INFO_ABOUT "https://nano.org")
		set (CPACK_NSIS_CONTACT "info@nano.org")
		set (CPACK_NSIS_MENU_LINKS "nano_wallet.exe" "Nano Wallet" "https://nano.org" "Nano website")
		set (CPACK_NSIS_ENABLE_UNINSTALL_BEFORE_INSTALL ON)
		get_target_property (Qt5WindowsPlugin Qt5::QWindowsIntegrationPlugin LOCATION)
		get_filename_component (Qt5_bin_DIR ${Qt5_DIR}/../../../bin ABSOLUTE)
		install (TARGETS nano_wallet DESTINATION .)
		install (FILES ${CMAKE_CURRENT_BINARY_DIR}/${WIN_REDIST} DESTINATION .)
		install (FILES ${Qt5_bin_DIR}/libGLESv2.dll DESTINATION .)
		install (FILES ${Qt5_bin_DIR}/Qt5Core.dll DESTINATION .)
		install (FILES ${Qt5_bin_DIR}/Qt5Gui.dll DESTINATION .)
		install (FILES ${Qt5_bin_DIR}/Qt5Widgets.dll DESTINATION .)
		install (FILES ${Qt5_bin_DIR}/Qt5WinExtras.dll DESTINATION .)
		install (FILES ${Qt5WindowsPlugin} DESTINATION platforms)
	else ()
		install(TARGETS nano_wallet
				RUNTIME DESTINATION bin
				LIBRARY DESTINATION lib
				ARCHIVE DESTINATION lib
		)
	endif ()
endif ()

set (CPACK_RESOURCE_FILE_LICENSE ${CMAKE_SOURCE_DIR}/LICENSE)

include (CPack)
