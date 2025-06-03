cmake_minimum_required(VERSION 3.10)
# set the project name
project(cdt CXX)

# Set the target architecture.
# All modern x86/x64 processors support AVX2.
# Older x86/x64 processors may support SSE2 but not AVX2.
# Very old x86/x64 processors, or non x86/x64 
# processors, do not support any of the two.
#set(ENABLE_SSE2 True)
#set(ENABLE_AVX2 True)
include(CheckCXXCompilerFlag)
check_cxx_compiler_flag(-mavx2 ENABLE_AVX2)
check_cxx_compiler_flag(-msse2 ENABLE_SSE2)

#####################

# Use CMake -DLGPL=ON if you opt for the LGPL license
option(LGPL "LGPL" OFF)


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# add the executable
set(SOURCES
    src/giftWrap.cpp
    src/PLC.cpp
    src/delaunay.cpp
    #src/main.cpp
)
add_library(${PROJECT_NAME}_lib
    ${SOURCES}
)

if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
# optimize for speed
	target_compile_options(${PROJECT_NAME}_lib PUBLIC /Ot)
# grant IEEE 754 compliance
	target_compile_options(${PROJECT_NAME}_lib PUBLIC "/fp:strict")
# use intrinsic functions
	target_compile_options(${PROJECT_NAME}_lib PUBLIC "/Oi")
# turn off annoying warnings
	target_compile_options(${PROJECT_NAME}_lib PUBLIC "/D _CRT_SECURE_NO_WARNINGS")
	# set target architecture
	if(ENABLE_AVX2)
		target_compile_options(${PROJECT_NAME}_lib PUBLIC "/arch:AVX2")
	elseif(ENABLE_SSE2)
		target_compile_options(${PROJECT_NAME}_lib PUBLIC "/arch:SSE2")
	endif()
# reserve enough stack size
	target_link_options(${PROJECT_NAME}_lib PUBLIC "/STACK:8421376")
	if(NOT LGPL)
		target_compile_options(${PROJECT_NAME}_lib PUBLIC "/D USE_MAROTS_METHOD")
	endif()
else()
# set standard optimization level
	target_compile_options(${PROJECT_NAME}_lib PUBLIC -O2)
# grant IEEE 754 compliance
	target_compile_options(${PROJECT_NAME}_lib PUBLIC -frounding-math)
	# set target architecture
	if(ENABLE_AVX2)
		target_compile_options(${PROJECT_NAME}_lib PUBLIC "-mavx2")
                target_compile_options(${PROJECT_NAME}_lib PUBLIC "-mfma")
	elseif(ENABLE_SSE2)
		target_compile_options(${PROJECT_NAME}_lib PUBLIC "-msse2")
	endif()
# reserve enough stack size	
	target_compile_options(${PROJECT_NAME}_lib PUBLIC -Wl,-z,stacksize=8421376)
	if(NOT LGPL)
		target_compile_options(${PROJECT_NAME}_lib PUBLIC "-DUSE_MAROTS_METHOD")
	endif()
endif()

include(gmp)
# Public include directory
target_include_directories(${PROJECT_NAME}_lib PUBLIC
	${CMAKE_CURRENT_SOURCE_DIR}/include
)

target_link_libraries(${PROJECT_NAME}_lib PUBLIC gmp::gmp)
target_compile_features(${PROJECT_NAME}_lib PUBLIC cxx_std_20)

add_library(CDT::CDT ALIAS ${PROJECT_NAME}_lib)


# add the executable
get_directory_property(HAS_PARENT PARENT_DIRECTORY)
if(NOT HAS_PARENT)
	add_executable(${PROJECT_NAME}
    	src/main.cpp
	)
	target_link_libraries(${PROJECT_NAME} PUBLIC CDT::CDT)
endif()

# target_compile_definitions(${PROJECT_NAME}_lib PUBLIC USE_GNU_GMP_CLASSES)
