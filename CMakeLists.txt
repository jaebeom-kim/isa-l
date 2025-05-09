cmake_minimum_required(VERSION 3.14)
project(libisal NONE)   # no languages yet—will enable below

# 1) PROJECT + languages
enable_language(C CXX ASM)   # we will need C compiler and an assembler

# 2) Detect CPU type
string(TOLOWER "${CMAKE_SYSTEM_PROCESSOR}" _cpu)
if (_cpu MATCHES "x86_64|amd64")
  set(CPU "x86_64")
elseif (_cpu MATCHES "aarch64|arm64")
  set(CPU "aarch64")
elseif (_cpu MATCHES "ppc64le|powerpc64le")
  set(CPU "ppc64le")
elseif (_cpu MATCHES "riscv64")
  set(CPU "riscv64")
else()
  set(CPU "undefined")
endif()

# Expose as options if ever needed downstream
set(CPU ${CPU} CACHE STRING "Detected host CPU: x86_64, aarch64, ppc64le, riscv64, or undefined")

# 3) Expose debug toggle
option(ENABLE_DEBUG "Enable debug messages" OFF)
if (ENABLE_DEBUG)
  add_definitions(-DENABLE_DEBUG=1)
endif()

# 4) Test for RVV (only on riscv64)
set(HAVE_RVV OFF)
if (CPU STREQUAL "riscv64")
  include(CheckCSourceCompiles)
  check_c_source_compiles("
    int main() {
      __asm__(\".option arch, +v\\n\"
              \"vsetivli zero, 0, e8, m1, ta, ma\\n\");
      return 0;
    }
  " HAVE_RVV)
  if (HAVE_RVV)
    add_definitions(-DHAVE_RVV=1)
  endif()
endif()

# 5) Find NASM (for x86_64 only)
set(USING_NASM OFF)
if (CPU STREQUAL "x86_64")
  find_program(NASM_EXECUTABLE nasm)
  if (NASM_EXECUTABLE)
    # test for a modern instruction
    include(CheckAssembly)
    check_asm_compiles("
      vpcompressb zmm0, k1, zmm1
    " NASM_MODERN)
    if (NASM_MODERN)
      set(USING_NASM ON)
      set(ASM_COMPILER "${NASM_EXECUTABLE}")
    endif()
  endif()
endif()

# 6-1) Collect crc sources by CPU
set(ISAL_SRCS "")
if (CPU STREQUAL "aarch64")
  file(GLOB AARCH64_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/crc/aarch64/*.c"
  )
  list(APPEND ISAL_SRCS ${AARCH64_SRCS})
else()
  # generic portable C fallback
  file(GLOB GENERIC_CRC_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/crc/*.c"
  )
  list(APPEND ISAL_SRCS ${GENERIC_CRC_SRCS})
endif()

# 6-2) Collect igzip sources by CPU
if (CPU STREQUAL "aarch64")
  file(GLOB AARCH64_IGZIP_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/igzip/aarch64/*.c"
  )
  list(APPEND ISAL_SRCS ${AARCH64_IGZIP_SRCS})
elseif (CPU STREQUAL "riscv64")
  file(GLOB RISCV64_IGZIP_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/igzip/riscv64/*.c"
  )
  list(APPEND ISAL_SRCS ${RISCV64_IGZIP_SRCS})
else()
  # generic portable C fallback
  file(GLOB GENERIC_IGZIP_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/igzip/*.c"
  )
  list(APPEND ISAL_SRCS ${GENERIC_IGZIP_SRCS})
endif()

# 6-3) Collect raid sources by CPU
if (CPU STREQUAL "aarch64")
  file(GLOB AARCH64_RAID_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/raid/aarch64/*.c"
  )
  list(APPEND ISAL_SRCS ${AARCH64_RAID_SRCS})
elseif (CPU STREQUAL "riscv64")
  file(GLOB RISCV64_RAID_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/raid/riscv64/*.c"
  )
  list(APPEND ISAL_SRCS ${RISCV64_RAID_SRCS})
else()
  # generic portable C fallback
  file(GLOB GENERIC_RAID_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/raid/*.c"
  )
  list(APPEND ISAL_SRCS ${GENERIC_RAID_SRCS})
endif()

# 6-4) Collect erasure code sources by CPU
if (CPU STREQUAL "aarch64")
  file(GLOB AARCH64_ERASURE_CODE_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/erasure_code/aarch64/*.c"
  )
  list(APPEND ISAL_SRCS ${AARCH64_ERASURE_CODE_SRCS})
elseif (CPU STREQUAL "riscv64")
  file(GLOB RISCV64_ERASURE_CODE_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/erasure_code/riscv64/*.c"
  )
  list(APPEND ISAL_SRCS ${RISCV64_ERASURE_CODE_SRCS})
elseif (CPU STREQUAL "ppc64le")
  file(GLOB PPC64LE_ERASURE_CODE_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/erasure_code/ppc64le/*.c"
  )
  list(APPEND ISAL_SRCS ${PPC64LE_ERASURE_CODE_SRCS})
else()
  # generic portable C fallback
  file(GLOB GENERIC_ERASURE_CODE_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/erasure_code/*.c"
  )
  list(APPEND ISAL_SRCS ${GENERIC_ERASURE_CODE_SRCS})
endif()

# # plus the rest of the library
# list(APPEND ISAL_SRCS
#   ${CMAKE_CURRENT_SOURCE_DIR}/erasure_code/*.c
#   # … etc
# )

# 7) Create the library
add_library(isal STATIC ${ISAL_SRCS})

# 8) Public include dir
target_include_directories(isal
  PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# 9) If we’re using NASM, instruct CMake how to compile `.asm` files
if (USING_NASM)
  message(STATUS "Building ISA-L with NASM: ${ASM_COMPILER}")
  enable_language(ASM_NASM)
  set_source_files_properties(
    ${CMAKE_CURRENT_SOURCE_DIR}/crc/x86_64/*.asm
    PROPERTIES
      LANGUAGE ASM_NASM
      NASM_OUTPUT_FORMAT elf64
      NASM_EXECUTABLE  ${ASM_COMPILER}
  )
endif()

# 10) Linker flags (for x86_64 only)
if (CPU STREQUAL "x86_64")
  set_target_properties(isal PROPERTIES
    LINK_FLAGS "-Wl,-z,noexecstack -Wl,-z,relro -Wl,-z,now"
  )
endif()
