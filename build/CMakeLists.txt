cmake_minimum_required(VERSION 3.4.0)
project (faust)

#######################################
# versions management
set (VERSION 2.5.23)
set (SOVERS 2)

get_filename_component(ROOT ${CMAKE_CURRENT_SOURCE_DIR} DIRECTORY)
set (SRCDIR ${ROOT}/compiler)
set (BINDIR ${CMAKE_CURRENT_SOURCE_DIR}/bin)
set (LIBDIR ${CMAKE_CURRENT_SOURCE_DIR}/lib)

set (CMAKE_CXX_STANDARD 11)


#######################################
# Output options
option ( UNIVERSAL 	"Compiles and combines i386 and x86_64 architectures [MacOS]"	off )
option ( IOS     	"Compiles for iOS"	off )
option ( LLVM_CONFIG    "Force use off llvm-config"	off )

#######################################
# Check output options
if (UNIVERSAL AND NOT APPLE)
	message(WARNING "UNIVERSAL option is only available on MacOS or iOS")
endif()

####################################
# source and headers files
file (GLOB SRC ${SRCDIR}/*.cpp ${SRCDIR}/*/*.cpp ${SRCDIR}/draw/*/*.cpp ${SRCDIR}/generator/fir/*.cpp)
file (GLOB HH  ${SRCDIR}/*.hh  ${SRCDIR}/*/*.hh  ${SRCDIR}/generator/fir/*.hh)

####################################
# include directories
set (FAUST_INC ${SRCDIR} 
	${SRCDIR}/boxes ${SRCDIR}/documentator ${SRCDIR}/draw 
	${SRCDIR}/errors ${SRCDIR}/evaluate ${SRCDIR}/extended 
	${SRCDIR}/generator 
	${SRCDIR}/generator/fir 
	${SRCDIR}/generator/interpreter 
	${SRCDIR}/normalize ${SRCDIR}/parallelize ${SRCDIR}/parser 
	${SRCDIR}/patternmatcher ${SRCDIR}/propagate ${SRCDIR}/signals 
	${SRCDIR}/tlib ${SRCDIR}/utils  
	${SRCDIR}/draw/device ${SRCDIR}/draw/schema 
	${SRCDIR}/../architecture /usr/local/include)


####################################
function(backend NAME DIR TARGET)
	message(STATUS "In target ${TARGET}: include ${NAME} backend")
	target_compile_definitions (${TARGET} PRIVATE -D${NAME}_BUILD)
	if (NOT "${DIR}" STREQUAL "")
		file (GLOB BACKEND_SRC	${SRCDIR}/generator/${DIR}/*.cpp ${SRCDIR}/generator/${DIR}/*.hh)
		target_sources (${TARGET} PRIVATE ${BACKEND_SRC})
		target_include_directories (${TARGET} PRIVATE ${SRCDIR}/generator/${DIR})
	endif()
endfunction()

####################################
function (scan_backends TARGET FLAG)
	string (FIND "${ASMJS_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (ASMJS asmjs ${TARGET})
	endif()
	string (FIND "${C_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (C c ${TARGET})
	endif()
	string (FIND "${CPP_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (CPP cpp ${TARGET})
	endif()
	string (FIND "${FIR_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (FIR "" ${TARGET})
	endif()
	string (FIND "${INTERP_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (INTERP interpreter ${TARGET})
	endif()
	string (FIND "${JAVA_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (JAVA java ${TARGET})
	endif()
	string (FIND "${JS_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (JS js ${TARGET})
	endif()
	string (FIND "${OLDCPP_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (OCPP "" ${TARGET})
	endif()
	string (FIND "${RUST_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (RUST rust ${TARGET})
	endif()
	string (FIND "${WASM_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (WASM wasm ${TARGET})
	endif()
	string (FIND "${LLVM_BACKEND}" ${FLAG} POS)
	if (${POS} GREATER -1)
		backend (LLVM llvm ${TARGET})
		target_compile_definitions (${TARGET} PRIVATE -D${LLVM_VERSION})
		target_include_directories (${TARGET} PRIVATE ${SRCDIR}/generator/c ${SRCDIR}/generator/cpp  ${LLVM_INCLUDE_DIRS})
	endif()
endfunction()

####################################
# Manual LLVM scan
macro (llvm_config)
	set (LLVM_CONFIG llvm-config)
	execute_process (COMMAND ${LLVM_CONFIG} --version OUTPUT_VARIABLE LLVM_VERSION)
	string ( STRIP ${LLVM_VERSION} LLVM_PACKAGE_VERSION )
	message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")

	execute_process (COMMAND ${LLVM_CONFIG} --includedir OUTPUT_VARIABLE LLVM_INCLUDE)
	string ( STRIP ${LLVM_INCLUDE} LLVM_INCLUDE_DIRS )
	execute_process (COMMAND ${LLVM_CONFIG} --ldflags OUTPUT_VARIABLE LLVM_LDFLAGS_TMP)
	string ( STRIP ${LLVM_LDFLAGS_TMP} LLVM_LD_FLAGS )
	execute_process (COMMAND ${LLVM_CONFIG}  --libs OUTPUT_VARIABLE LLVM_LIBS_TMP)
	string ( STRIP ${LLVM_LIBS_TMP} LLVM_LIBS )
	execute_process (COMMAND ${LLVM_CONFIG}  --system-libs OUTPUT_VARIABLE LLVM_SYS_LIBS_TMP)
	string ( STRIP ${LLVM_SYS_LIBS_TMP} LLVM_SYS_LIBS)
	set (LLVM_LIBS ${LLVM_LIBS} ${LLVM_SYS_LIBS})
endmacro()

####################################
# LLVM
if (LLVM_BACKEND )
	if ((WIN32 AND NOT ${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC))
		llvm_config()
	elseif(LLVM_CONFIG)
		llvm_config()
	else()
		find_package(LLVM REQUIRED CONFIG)
		if (COMMAND llvm_map_components_to_libnames)
			message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
			message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")
			# Find the libraries that correspond to the LLVM components that we wish to use
			llvm_map_components_to_libnames(LLVM_LIBS all)
			list(REMOVE_ITEM LLVM_LIBS LTO)
		else()
			llvm_config()
		endif()
	endif()

	string (REGEX REPLACE "([0-9])\\.([0-9]).*" "LLVM_\\1\\2" LLVM_VERSION ${LLVM_PACKAGE_VERSION})

	# the declarations below are redundant but necessary to cope with different cmake behaviors on different platforms
	set (TMP ${SRCDIR}/generator/llvm)
	set_property(SOURCE ${TMP}/llvm_dsp_aux.cpp ${TMP}/llvm_dynamic_dsp_aux.cpp APPEND_STRING PROPERTY COMPILE_FLAGS   "-fno-rtti ${LLVM_DEFINITIONS}")
	set_property(SOURCE ${TMP}/llvm_dsp_aux.cpp ${TMP}/llvm_dynamic_dsp_aux.cpp APPEND_STRING PROPERTY CMAKE_CXX_FLAGS "-fno-rtti ${LLVM_DEFINITIONS}")
	set (FAUST_LIBS ${FAUST_LIBS} ${LLVM_LIBS})
endif()

####################################
# platform dependent settings
if(APPLE)
	if(IOS)
		set (CMAKE_XCODE_EFFECTIVE_PLATFORMS "iphoneos")
		set (CMAKE_OSX_ARCHITECTURES "arm64 armv7 armv7s")
		set (CMAKE_OSX_SYSROOT "iphoneos")
		set (CMAKE_XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET 8.0) 
	else()
		set (MACSDK /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk)
		if (EXISTS ${MACSDK})
			set (CMAKE_OSX_SYSROOT ${MACSDK})
		else()
			message ("WARNING No SDK at ${MACSDK}")
		endif()
		if (UNIVERSAL)
			message (STATUS "Compiles an universal binary")
			set (CMAKE_OSX_ARCHITECTURES x86_64 i386)
		endif()
	endif()
elseif(WIN32)
	if (NOT ${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC)
		set (FAUST_DEFINITIONS ${FAUST_DEFINITIONS} -DWIN32 -D__MINGW32__)
	endif()
	set (FAUST_DEFINITIONS ${FAUST_DEFINITIONS} -D_CRT_SECURE_NO_WARNINGS)
	set (FAUST_LIBS ${FAUST_LIBS} Ws2_32.lib)
else()
	set (FAUST_LIBS -lpthread ${FAUST_LIBS})
endif()


####################################
# Add the different targets
####################################
add_executable(faust ${SRC} ${HH})
set_target_properties(faust  PROPERTIES 
	RUNTIME_OUTPUT_DIRECTORY_RELEASE ${BINDIR})

# static faust library
add_library(staticlib STATIC ${SRC} ${HH})
set_target_properties(staticlib PROPERTIES ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${LIBDIR})
if (APPLE AND IOS)
	set_target_properties(staticlib  PROPERTIES OUTPUT_NAME faust-ios)
endif()

# dynamic faust library
add_library(dynamiclib SHARED ${SRC} ${HH})
if (WIN32)
	enable_language(RC)
	set_target_properties(dynamiclib PROPERTIES 
		ARCHIVE_OUTPUT_DIRECTORY_RELEASE  ${LIBDIR}
		RUNTIME_OUTPUT_DIRECTORY_RELEASE  ${LIBDIR})
	if (MSVC)
		set_target_properties(staticlib  PROPERTIES OUTPUT_NAME libfaust)
	else()
		set_target_properties(staticlib  PROPERTIES OUTPUT_NAME faust)
	endif()
	set_target_properties(dynamiclib PROPERTIES OUTPUT_NAME faust)
	target_sources (faust PRIVATE ${ROOT}/windows/faust.rc)
	target_sources (dynamiclib PRIVATE ${ROOT}/windows/faustdll.rc)
else()
	if (IOS)
		set_target_properties(dynamiclib staticlib PROPERTIES OUTPUT_NAME faust.ios)
	else()
		set_target_properties(dynamiclib staticlib PROPERTIES OUTPUT_NAME faust)
	endif()	
	set_target_properties(dynamiclib PROPERTIES LIBRARY_OUTPUT_DIRECTORY_RELEASE ${LIBDIR})
	set_target_properties(dynamiclib PROPERTIES 
		LIBRARY_OUTPUT_DIRECTORY_RELEASE ${LIBDIR}
		VERSION ${VERSION}
		SOVERSION ${SOVERS})
endif()
set_target_properties(dynamiclib  staticlib PROPERTIES COMPILE_FLAGS  -DFAUST_LIB)

# shared properties
foreach (TARGET faust staticlib dynamiclib)
	target_compile_definitions (${TARGET} PRIVATE ${FAUST_DEFINITIONS})
	target_include_directories (${TARGET} PRIVATE ${FAUST_INC} ${LLVM_INC})
	target_link_libraries (${TARGET} PRIVATE ${FAUST_LIBS})
endforeach()

scan_backends (faust COMPILER)
scan_backends (staticlib STATIC)
scan_backends (dynamiclib DYNAMIC)

####################################
# add sub projects (emcc, osclib)
####################################
if (NOT (${CMAKE_GENERATOR} STREQUAL Xcode OR ${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC))
	add_subdirectory (emcc)
elseif(NOT IOS)
	message ("WARNING Targets asmjslib and wasmlib are not included with Xcode or Visual Studio")
endif()
add_subdirectory (osc)
add_subdirectory (http)


####################################
# install section
####################################
message (STATUS "Install location is ${CMAKE_INSTALL_PREFIX}")

set (ARCHDIR ${ROOT}/architecture)
set (ARCHINC ${ARCHDIR}/faust)
set (ARCHDIRS ${ARCHINC}/au ${ARCHINC}/audio ${ARCHINC}/dsp ${ARCHINC}/gui ${ARCHINC}/midi ${ARCHINC}/unity ${ARCHINC}/vst)

####################################
# install the faust compiler and libraries
install ( TARGETS faust staticlib dynamiclib 
	RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin 
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib 
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_PREFIX}/include/faust
)

####################################
# install headers
install (
    DIRECTORY ${ARCHDIRS} DESTINATION ${CMAKE_INSTALL_PREFIX}/include/faust
    FILE_PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
    DIRECTORY_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
    	GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
    PATTERN ".DS_Store" EXCLUDE
    PATTERN "*.o" EXCLUDE
)
install (
    FILES     ${ARCHINC}/misc.h ${ARCHINC}/sound-file.h DESTINATION ${CMAKE_INSTALL_PREFIX}/include/faust
    PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install (
    FILES     ${SRCDIR}/generator/libfaust.h ${SRCDIR}/generator/libfaust-c.h DESTINATION ${CMAKE_INSTALL_PREFIX}/include/faust/dsp
    PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)

####################################
# install faust2xxx tools
file (GLOB FAUST2XXX ${ROOT}/tools/faust2appls/faust2* ${ROOT}/tools/faust2appls/encoderunitypackage ${ROOT}/tools/faust2sc-1.0.0/faust2sc )
install (
    FILES     ${FAUST2XXX} DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_EXECUTE WORLD_READ
)
install (
    FILES     ${ROOT}/tools/faust2appls/faustoptflags ${ROOT}/tools/faust2appls/faustpath DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
    PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_EXECUTE
)


####################################
# install shared resources
install (
    FILES     ${ROOT}/faust.1 DESTINATION ${CMAKE_INSTALL_PREFIX}/share/man
    PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)

# install faust libraries
file (GLOB FAUSTDSPLIBS ${ROOT}/libraries/*.lib ${ROOT}/libraries/old/*.lib)
# install various files from the architecture root folder
file (GLOB FAUSTARCHS   ${ARCHDIR}/*.cpp ${ARCHDIR}/*.c 
						${ARCHDIR}/*.rs  ${ARCHDIR}/*.js 
						${ARCHDIR}/*.java ${ARCHDIR}/*.h 
						${ARCHDIR}/mathdoctexts-*.txt  ${ARCHDIR}/latexheader.tex)
install (
    FILES     ${FAUSTDSPLIBS} ${FAUSTARCHS} DESTINATION ${CMAKE_INSTALL_PREFIX}/share/faust
    PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)

# install folders
set (ARCHSRC android api AU iOS juce max-msp nodejs osclib smartKeyboard unity webaudio)
foreach (dir ${ARCHSRC})
	set(ARCHFOLDERS ${ARCHFOLDERS} ${ARCHDIR}/${dir})
endforeach()
set(ARCHFOLDERS ${ARCHFOLDERS} ${ARCHDIR}/httpdlib/html/js)

install (
    DIRECTORY ${ARCHFOLDERS} DESTINATION ${CMAKE_INSTALL_PREFIX}/share/faust
    FILE_PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
    DIRECTORY_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
    	GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
    PATTERN ".DS_Store" EXCLUDE
    PATTERN "*.o" EXCLUDE
)

####################################
# install ios lib
install (
    FILES     ${ARCHDIR}/ios-libsndfile.a DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)

