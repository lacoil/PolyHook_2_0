cmake_minimum_required(VERSION 3.15)

project(PolyHook_2)

include(CMakePackageConfigHelpers)

#
# Options
#

option(POLYHOOK_BUILD_SHARED_LIBS "Build polyhook as shared libary" OFF)
option(POLYHOOK_BUILD_DLL "Build dll & lib instead of tests" ON)
if(MSVC)
	option(POLYHOOK_BUILD_STATIC_RUNTIME "Use static runtime" ON)
endif()

option(POLYHOOK_DISASM_CAPSTONE "Enable capstone support" ON)
option(POLYHOOK_DISASM_CAPSTONE_FULL "Build all features of capstone." OFF)
option(POLYHOOK_DISASM_ZYDIS "Enable Zydis support." ON)

option(POLYHOOK_FEATURE_EXCEPTION "Implement all exception hooking functionality" ON)
option(POLYHOOK_FEATURE_DETOURS "Implement detour functionality" ON)
option(POLYHOOK_FEATURE_INLINENTD "Support inline hooks without specifying typedefs by generating callback stubs at runtime with AsmJit" ON)
option(POLYHOOK_FEATURE_PE "Implement all win pe hooking functionality" ON)
option(POLYHOOK_FEATURE_VIRTUALS "Implement all virtual table hooking functionality" ON)

#
# ASMJIT
#

if(POLYHOOK_FEATURE_INLINENTD)

	if(POLYHOOK_BUILD_SHARED_LIBS)
		set(ASMJIT_STATIC OFF CACHE BOOL "")
	else()
		set(ASMJIT_STATIC ON CACHE BOOL "")
	endif()

	add_subdirectory(asmjit)
	if(MSVC)
		if(POLYHOOK_BUILD_STATIC_RUNTIME)
			set_target_properties(asmjit PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
		else()
			set_target_properties(asmjit PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
		endif()
	endif()
endif()

#
# Capstone
#

if(POLYHOOK_DISASM_CAPSTONE)
	set(CAPSTONE_BUILD_STATIC_RUNTIME ${POLYHOOK_BUILD_STATIC_RUNTIME} CACHE BOOL "")
	if(POLYHOOK_BUILD_SHARED_LIBS)
		set(CAPSTONE_BUILD_SHARED ON CACHE BOOL "")
		set(CAPSTONE_BUILD_STATIC OFF CACHE BOOL "")
		set(CAPSTONE_LIBRARY_NAME "capstone-shared")
	else()
		set(CAPSTONE_BUILD_SHARED OFF CACHE BOOL "")
		set(CAPSTONE_BUILD_STATIC ON CACHE BOOL "")
		set(CAPSTONE_LIBRARY_NAME "capstone-static")
	endif()
	set(CAPSTONE_BUILD_TESTS OFF CACHE BOOL "")
	set(CAPSTONE_BUILD_CSTOOL OFF CACHE BOOL "")
	if(NOT ${POLYHOOK_DISASM_CAPSTONE_FULL})
		set(CAPSTONE_ARM_SUPPORT OFF CACHE BOOL "ARM support") 
		set(CAPSTONE_ARM64_SUPPORT OFF CACHE BOOL "ARM64 support") 
		set(CAPSTONE_M680X_SUPPORT OFF CACHE BOOL "M680X support") 
		set(CAPSTONE_M68K_SUPPORT OFF CACHE BOOL "M68K support") 
		set(CAPSTONE_MIPS_SUPPORT OFF CACHE BOOL "MIPS support") 
		set(CAPSTONE_PPC_SUPPORT OFF CACHE BOOL "PPC support") 
		set(CAPSTONE_SPARC_SUPPORT OFF CACHE BOOL "SPARC support")
		set(CAPSTONE_SYSZ_SUPPORT OFF CACHE BOOL "SYSZ support")
		set(CAPSTONE_XCORE_SUPPORT OFF CACHE BOOL "XCORE support")
		set(CAPSTONE_TMS320C64X_SUPPORT OFF CACHE BOOL "TMS320C64X support")
		set(CAPSTONE_M680X_SUPPORT OFF CACHE BOOL "M680X support")
		set(CAPSTONE_EVM_SUPPORT OFF CACHE BOOL "EVM support")
	endif()

	add_subdirectory(capstone)

	if(MSVC)
		if(POLYHOOK_BUILD_STATIC_RUNTIME)
			set_target_properties(${CAPSTONE_LIBRARY_NAME} PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
		else()
			set_target_properties(${CAPSTONE_LIBRARY_NAME} PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
		endif()
	endif()

endif()



#
# Zydis
#

if(POLYHOOK_DISASM_ZYDIS)
    set(ZYDIS_BUILD_SHARED_LIB ${POLYHOOK_BUILD_SHARED_LIBS} CACHE BOOL "")
	set(ZYCORE_BUILD_SHARED_LIB ${POLYHOOK_BUILD_SHARED_LIBS} CACHE BOOL "")
	set(ZYDIS_BUILD_TOOLS OFF CACHE BOOL "")
	set(ZYDIS_BUILD_EXAMPLES OFF CACHE BOOL "")

	add_subdirectory(zydis/dependencies/zycore)
	add_subdirectory(zydis)

	if(MSVC)
		if(POLYHOOK_BUILD_STATIC_RUNTIME)
			set_target_properties(Zycore PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
			set_target_properties(Zydis PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
		else()
			set_target_properties(Zycore PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
			set_target_properties(Zydis PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
		endif()
	endif()
endif()

#
# Polyhook
#

if(POLYHOOK_BUILD_DLL)
	if(POLYHOOK_BUILD_SHARED_LIBS)
		add_library(${PROJECT_NAME} SHARED)
	else()
		add_library(${PROJECT_NAME} STATIC)
	endif()
else()
	add_executable(${PROJECT_NAME})
endif()

set_target_properties(${PROJECT_NAME} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD 17)
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD_REQUIRED ON)

if(MSVC)
    if(POLYHOOK_BUILD_STATIC_RUNTIME)
		set_target_properties(${PROJECT_NAME} PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
	else()
		set_target_properties(${PROJECT_NAME} PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
	endif()

	if(MSVC)
		  set(COMPILE_FLAGS_PLH "/W4 /WX /Z7")
		  if(NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang")
			set(COMPILE_FLAGS_PLH "/MP ${COMPILE_FLAGS_PLH}")
		  endif()

		  set_target_properties(${PROJECT_NAME} PROPERTIES COMPILE_FLAGS ${COMPILE_FLAGS_PLH})
		  target_link_libraries(${PROJECT_NAME} PRIVATE optimized -DEBUG) # mhhm ya pdbs
	endif()

endif()

target_include_directories(${PROJECT_NAME}
    PRIVATE
		$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
    INTERFACE
		$<INSTALL_INTERFACE:include>
)

#Core
set(POLYHOOK_CORE_HEADERS ${PROJECT_SOURCE_DIR}/polyhook2/ADisassembler.hpp
        ${PROJECT_SOURCE_DIR}/polyhook2/Enums.hpp
        ${PROJECT_SOURCE_DIR}/polyhook2/IHook.hpp
        ${PROJECT_SOURCE_DIR}/polyhook2/Instruction.hpp
        ${PROJECT_SOURCE_DIR}/polyhook2/Misc.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/UID.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/ErrorLog.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/MemProtector.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/PageAllocator.hpp)
install(FILES ${POLYHOOK_CORE_HEADERS} DESTINATION include/polyhook2)


target_sources(${PROJECT_NAME} PRIVATE
	${PROJECT_SOURCE_DIR}/sources/MemProtector.cpp
	${PROJECT_SOURCE_DIR}/sources/TestEffectTracker.cpp
	${PROJECT_SOURCE_DIR}/sources/PageAllocator.cpp)

#DisAsm/Capstone
if(POLYHOOK_DISASM_CAPSTONE)
	target_link_libraries(${PROJECT_NAME} PRIVATE $<BUILD_INTERFACE:${CAPSTONE_LIBRARY_NAME}>)
	install(TARGETS ${CAPSTONE_LIBRARY_NAME} EXPORT exporthack)

	target_sources(${PROJECT_NAME} PRIVATE "${PROJECT_SOURCE_DIR}/sources/CapstoneDisassembler.cpp")

	target_include_directories(${PROJECT_NAME} PRIVATE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/capstone/include>)

	install(FILES ${PROJECT_SOURCE_DIR}/polyhook2/CapstoneDisassembler.hpp DESTINATION include/polyhook2)
endif()

#DisAsm/Zydis
if(POLYHOOK_DISASM_ZYDIS)
	target_link_libraries(${PROJECT_NAME} PRIVATE $<BUILD_INTERFACE:Zydis>)

	target_sources(${PROJECT_NAME} PRIVATE "${PROJECT_SOURCE_DIR}/sources/ZydisDisassembler.cpp")

	target_include_directories(${PROJECT_NAME} PRIVATE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/zydis/include>)
	target_include_directories(${PROJECT_NAME} PRIVATE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/zydis/dependencies/zycore/include>)
	target_include_directories(${PROJECT_NAME} PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/zydis>)
	
	install(FILES ${PROJECT_SOURCE_DIR}/polyhook2/ZydisDisassembler.hpp DESTINATION include/polyhook2)
endif()

#Feature/Detours
if(POLYHOOK_FEATURE_DETOURS)
	set(POLYHOOK_DETOUR_HEADERS
		${PROJECT_SOURCE_DIR}/polyhook2/Detour/ADetour.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/Detour/x64Detour.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/Detour/x86Detour.hpp)
	install(FILES ${POLYHOOK_DETOUR_HEADERS} DESTINATION include/polyhook2/Detour)

	target_sources(${PROJECT_NAME} PRIVATE
		${PROJECT_SOURCE_DIR}/sources/ADetour.cpp
		${PROJECT_SOURCE_DIR}/sources/x64Detour.cpp
		${PROJECT_SOURCE_DIR}/sources/x86Detour.cpp)

	# only build tests if making exe
	if(NOT POLYHOOK_BUILD_DLL)
		if(CMAKE_SIZEOF_VOID_P EQUAL 8) #64-bit
			target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/UnitTests/TestDetourx64.cpp)
		elseif(CMAKE_SIZEOF_VOID_P EQUAL 4) #32-bit
			target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/UnitTests/TestDetourx86.cpp)
		endif()
	endif()
endif()

#Feature/Exception
if(POLYHOOK_FEATURE_EXCEPTION)
	set(POLYHOOK_EXCEPTION_HEADERS
		${PROJECT_SOURCE_DIR}/polyhook2/Exceptions/AVehHook.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/Exceptions/BreakPointHook.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/Exceptions/HWBreakPointHook.hpp)
	install(FILES ${POLYHOOK_EXCEPTION_HEADERS} DESTINATION include/polyhook2/Exceptions)

	target_sources(${PROJECT_NAME} PRIVATE
		${PROJECT_SOURCE_DIR}/sources/AVehHook.cpp
		${PROJECT_SOURCE_DIR}/sources/BreakPointHook.cpp
		${PROJECT_SOURCE_DIR}/sources/HWBreakPointHook.cpp)

	# only build tests if making exe
	if(NOT POLYHOOK_BUILD_DLL)
		target_sources(${PROJECT_NAME} PRIVATE
			${PROJECT_SOURCE_DIR}/UnitTests/TestBreakpointHook.cpp
			${PROJECT_SOURCE_DIR}/UnitTests/TestHWBreakpointHook.cpp)
	endif()
endif()


#Feature/Inlinentd
if(POLYHOOK_FEATURE_INLINENTD)
	target_link_libraries(${PROJECT_NAME} PRIVATE $<BUILD_INTERFACE:asmjit>)

	install(FILES ${PROJECT_SOURCE_DIR}/polyhook2/Detour/ILCallback.hpp DESTINATION include/polyhook2/Detour)

	target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/sources/ILCallback.cpp)

	# only build tests if making exe
	if(NOT POLYHOOK_BUILD_DLL)
		if(CMAKE_SIZEOF_VOID_P EQUAL 8)
			target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/UnitTests/TestDetourNoTDx64.cpp)
		elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
			target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/UnitTests/TestDetourNoTDx86.cpp)
		endif()
	endif()
endif()


#Feature/PE
if(POLYHOOK_FEATURE_PE)
	set(POLYHOOK_PE_HEADERS
		${PROJECT_SOURCE_DIR}/polyhook2/PE/EatHook.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/PE/IatHook.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/PE/PEB.hpp)
	install(FILES ${POLYHOOK_PE_HEADERS} DESTINATION include/polyhook2/PE)

	target_sources(${PROJECT_NAME} PRIVATE
		${PROJECT_SOURCE_DIR}/sources/EatHook.cpp
		${PROJECT_SOURCE_DIR}/sources/IatHook.cpp)

	# only build tests if making exe
	if(NOT POLYHOOK_BUILD_DLL)
		target_sources(${PROJECT_NAME} PRIVATE
			${PROJECT_SOURCE_DIR}/UnitTests/TestEatHook.cpp
			${PROJECT_SOURCE_DIR}/UnitTests/TestIatHook.cpp)
	endif()
endif()


#Feature/Virtuals
if(POLYHOOK_FEATURE_VIRTUALS)
	set(POLYHOOK_VIRTUAL_HEADERS 
		${PROJECT_SOURCE_DIR}/polyhook2/Virtuals/VTableSwapHook.hpp
		${PROJECT_SOURCE_DIR}/polyhook2/Virtuals/VFuncSwapHook.hpp)
	install(FILES ${POLYHOOK_VIRTUAL_HEADERS} DESTINATION include/polyhook2/Virtuals)

	target_sources(${PROJECT_NAME} PRIVATE
		${PROJECT_SOURCE_DIR}/sources/VTableSwapHook.cpp
		${PROJECT_SOURCE_DIR}/sources/VFuncSwapHook.cpp)

	# only build tests if making exe
	if(NOT POLYHOOK_BUILD_DLL)
		target_sources(${PROJECT_NAME} PRIVATE
			${PROJECT_SOURCE_DIR}/UnitTests/TestVTableSwapHook.cpp
			${PROJECT_SOURCE_DIR}/UnitTests/TestVFuncSwapHook.cpp)
	endif()
endif()


#Tests
if(NOT POLYHOOK_BUILD_DLL)
	target_sources(${PROJECT_NAME} PRIVATE
		${PROJECT_SOURCE_DIR}/MainTests.cpp
		${PROJECT_SOURCE_DIR}/UnitTests/TestDisassembler.cpp
		${PROJECT_SOURCE_DIR}/UnitTests/TestMemProtector.cpp)
endif()


#
# Install
#

configure_package_config_file(
		"${PROJECT_NAME}-config.cmake.in"
		"${PROJECT_NAME}-config.cmake"
	INSTALL_DESTINATION
        "lib/${PROJECT_NAME}"
)

install(
    FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
    DESTINATION
        "lib/${PROJECT_NAME}"
)

install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}-targets
    RUNTIME DESTINATION "bin"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
)

install(
    EXPORT
        ${PROJECT_NAME}-targets
    NAMESPACE
        ${PROJECT_NAME}::
    DESTINATION
        "lib/${PROJECT_NAME}"
)
