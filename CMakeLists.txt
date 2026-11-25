project(libcxxabi)

# DARLING: libcxxabi 19.1.7, the same release as libcxx. The sources are C++23
# (as upstream builds them) and use libcxx's private headers from libcxx/src
# ("include/overridable_function.h", "include/atomic_support.h").
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -nostdinc -std=c++23")
set(CMAKE_SHARED_LINKER_FLAGS "-nostdlib")

include_directories(BEFORE
	${CMAKE_CURRENT_SOURCE_DIR}/include
	${CMAKE_CURRENT_SOURCE_DIR}/../libcxx/include
	${CMAKE_CURRENT_SOURCE_DIR}/../libcxx/src
)

add_definitions(-D__DARWIN_NON_CANCELABLE=1
	-D_LIBCXXABI_BUILDING_LIBRARY=1
	-D_LIBCPP_BUILDING_LIBRARY=1
)

set(cxxabi_sources
	src/abort_message.cpp
	src/cxa_aux_runtime.cpp
	src/cxa_default_handlers.cpp
	src/cxa_demangle.cpp
	src/cxa_exception.cpp
	src/cxa_exception_storage.cpp
	src/cxa_guard.cpp
	src/cxa_handlers.cpp
	src/cxa_personality.cpp
	# not built for Apple upstream; kept because Darling's libc++abi has always exported
	# __cxa_thread_atexit
	src/cxa_thread_atexit.cpp
	src/cxa_vector.cpp
	src/cxa_virtual.cpp
	src/fallback_malloc.cpp
	src/private_typeinfo.cpp
	src/stdlib_exception.cpp
	src/stdlib_new_delete.cpp
	src/stdlib_stdexcept.cpp
	src/stdlib_typeinfo.cpp
)

# As for Apple's libc++abi (libcxx/cmake/caches/Apple.cmake): libc++abi defines every
# operator new/delete (libc++ re-exports them) and dynamic_cast is forgiving.
add_darling_object_library(cxxabi_obj ${cxxabi_sources})
target_compile_definitions(cxxabi_obj PRIVATE _LIBCXXABI_FORGIVING_DYNAMIC_CAST=1)
make_fat(cxxabi_obj)

# The static library is linked into dyld, which runs without libSystem: it has
# no aligned_alloc(), syslog() or dlopen(). Without aligned allocation the
# exception buffers come from malloc(), which dyld aligns to 16 bytes; a strict
# dynamic_cast has nothing to log; and dyld never shares exceptions with
# libstdc++, so the check for its empty string is left out.
add_darling_object_library(cxxabi_static_obj ${cxxabi_sources})
target_compile_definitions(cxxabi_static_obj PRIVATE
	_LIBCPP_HAS_NO_LIBRARY_ALIGNED_ALLOCATION=1
	_LIBCPP_DARLING_NO_GCC_EMPTY_STRING_CHECK=1
)
make_fat(cxxabi_static_obj)

set(DYLIB_INSTALL_NAME "/usr/lib/libc++abi.dylib")
set(DYLIB_COMPAT_VERSION "1.0.0")
set(DYLIB_CURRENT_VERSION "1.0.0")

add_darling_static_library(cxxabi_static SOURCES $<TARGET_OBJECTS:cxxabi_static_obj>)
make_fat(cxxabi_static)

add_darling_library(cxxabi SHARED $<TARGET_OBJECTS:cxxabi_obj>)
target_link_libraries(cxxabi system)
set_target_properties(cxxabi PROPERTIES OUTPUT_NAME "c++abi")
#set_property(TARGET cxxabi APPEND_STRING PROPERTY LINK_FLAGS " -Wl,-undefined,dynamic_lookup") # Needed for weakly imported symbols
make_fat(cxxabi)

install(TARGETS cxxabi DESTINATION libexec/darling/usr/lib)

