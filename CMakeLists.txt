cmake_minimum_required(VERSION 3.5)

project(libsrtp)

option(DEBUG "Enable debugging" OFF)
option(ERROR_TO_FILE "Report errors to file instead of stderr" OFF)
option(ENABLE_GENERIC_AESICM "Enable generic AES ICM" OFF)
option(BUILD_FOR_RISK "Build for RISK cpu type" OFF)
option(SYSLOG_LOGGING "Enable syslog logging" OFF)

option(LIBSRTP_WITH_OPENSSL "Enable building with openssl" ON)


# check if openssl exists
if(${LIBSRTP_WITH_OPENSSL})
	find_package(OpenSSL)
	if(NOT ${OPENSSL_FOUND})
		message(STATUS "OpenSSL was not found.. will continue WITHOUT_OPENSSL")
	endif()
endif()

# -- start of checks

include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckLibraryExists)
include(CheckTypeSize)
include(TestBigEndian)

TEST_BIG_ENDIAN(BIG_ENDIAN)

if (${ENABLE_GENERIC_AESICM})
	set(GENERIC_AESICM 1)
endif()

if (${DEBUG})
	set(ENABLE_DEBUGGING 1)
endif()

# TODO: determine if this is a X86 architecture
set(IS_X86 ON)
if (${IS_X86})
	set(HAVE_X86 1)
endif()

# errors to log file
if (${ERROR_TO_FILE})
	set(ERR_REPORTING_FILE "libsrtp_error.log")
	set(USE_ERR_REPORTING_FILE 1)
else()
	set(ERR_REPORTING_STDOUT 1)
endif()

if (${BUILD_FOR_RISK})
	set(CPU_RISC 1)
else()
	set(CPU_CISC 1)
endif()

if (${BIG_ENDIAN} EQUAL 0)
	set(WORDS_BIGENDIAN 0)
else()
	set(WORDS_BIGENDIAN 1)
endif()

if(EXISTS /dev/urandom)
	set(DEV_URANDOM "/dev/urandom")
endif()

check_include_files(arpa/inet.h HAVE_ARPA_INET_H)
check_include_files(byteswap.h HAVE_BYTESWAP_H)
check_function_exists(inet_aton HAVE_INET_ATON)
check_type_size(int16_t HAVE_INT16_T)
check_type_size(int32_t HAVE_INT32_T)
check_type_size(int8_t HAVE_INT8_T)
check_include_files(inttypes.h HAVE_INTTYPES_H)
check_library_exists(socket socket "" HAVE_LIBSOCKET)
check_include_files(machine/types.h HAVE_MACHINE_TYPES_H)
check_include_files(memory.h HAVE_MEMORY_H)
check_include_files(netinet/in.h HAVE_NETINET_IN_H)
check_function_exists(socket HAVE_SOCKET)
check_include_files(stdint.h HAVE_STDINT_H)
check_include_files(stdlib.h HAVE_STDLIB_H)
check_include_files(strings.h HAVE_STRINGS_H)
check_include_files(string.h HAVE_STRING_H)
check_include_files(syslog.h HAVE_SYSLOG_H)
check_include_files(sys/int_types.h HAVE_SYS_INT_TYPES_H)
check_include_files(sys/socket.h HAVE_SYS_SOCKET_H)
check_include_files(sys/stat.h HAVE_SYS_STAT_H)
check_include_files(sys/types.h HAVE_SYS_TYPES_H)
check_include_files(sys/uio.h HAVE_SYS_UIO_H)
check_type_size(uint16_t HAVE_UINT16_T)
check_type_size(uint32_t HAVE_UINT32_T)
check_type_size(uint64_t HAVE_UINT64_T)
check_type_size(uint8_t HAVE_UINT8_T)
check_include_files(unistd.h HAVE_UNISTD_H)
check_function_exists(usleep HAVE_USLEEP)
check_include_files(windows.h HAVE_WINDOWS_H)
check_include_files(winsock2.h HAVE_WINSOCK2_H)

check_type_size("unsigned long" SIZEOF_UNSIGNED_LONG)
check_type_size("unsigned long long" SIZEOF_UNSIGNED_LONG_LONG)

check_type_size(size_t SIZE_T_EXISTS)

if("${SIZE_T_EXISTS}" STREQUAL "")
	message(STATUS "SIZE_T missing")
	set(size_t 1)
	set(size_t_type "unsigned")
endif()

# check if openssl exists
if(${LIBSRTP_WITH_OPENSSL})
	find_package(OpenSSL)
	if(NOT ${OPENSSL_FOUND})
		message(STATUS "OpenSSL was not found.. will continue WITHOUT_OPENSSL")
	endif()
endif()

if(MSVC)
	set(inline 1)
	set(INLINE_KEYWORD "_inline")
endif()

if(NOT "${HAVE_SYSLOG_H}" STREQUAL "")
	if (${SYSLOG_LOGGING})
		set(USE_SYSLOG 1)
	endif()
 	message(STATUS "Has syslog!")
endif()

configure_file(configure.h.cmake.in
	"${CMAKE_CURRENT_SOURCE_DIR}/crypto/include/config.h")

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/crypto)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/crypto/include)

list( APPEND src
  "srtp/srtp.c"
  "srtp/ekt.c"
  "crypto/kernel/alloc.c"
  "crypto/kernel/err.c"
  "crypto/kernel/crypto_kernel.c"
  "crypto/kernel/key.c"
  "crypto/math/datatypes.c"
  "crypto/math/stat.c"
  "crypto/replay/rdbx.c"
  "crypto/replay/rdb.c"
  "crypto/replay/ut_sim.c"
  "crypto/cipher/cipher.c"
  "crypto/cipher/null_cipher.c"
  "crypto/cipher/aes.c"
  "crypto/hash/auth.c"
  "crypto/hash/null_auth.c"
  )
if(OPENSSL_FOUND)
list( APPEND src
    "crypto/cipher/aes_gcm_ossl.c"
    "crypto/cipher/aes_icm_ossl.c"
    "crypto/hash/hmac_ossl.c"
  )
  list(APPEND platformlibs ${OPENSSL_LIBRARIES})
  include_directories(${OPENSSL_INCLUDE_DIR})
  list(APPEND srtp_def "OPENSSL")
else()
list( APPEND src
      "crypto/cipher/aes_icm.c"
      "crypto/hash/sha1.c"
      "crypto/hash/hmac.c"
  )
endif(OPENSSL_FOUND)


if(WIN32)
	list(APPEND platformlibs "wsock32.lib" "ws2_32.lib")
endif()

set(PACKAGE_NAME "libsrtp")
set(PACKAGE_VERSION "2.2.0")
set(PACKAGE_STRING "${PACKAGE_NAME}_${VERSION}")

add_library(${PROJECT_NAME} STATIC ${src})
target_link_libraries(${PROJECT_NAME} ${platformlibs})
list(APPEND srtp_def "HAVE_CONFIG_H")
if(${HAVE_UINT64_T})
	list(APPEND srtp_def "HAVE_UINT64_T")
endif()
target_compile_definitions(${PROJECT_NAME} PUBLIC ${srtp_def})