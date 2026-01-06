---
name: embedded-build-systems
description: >
  Build systems for embedded C/C++ including CMake cross-compilation, Make for embedded,
  and toolchain configuration. Use when setting up embedded build environments or CI/CD pipelines.
---

# Embedded Build Systems

Comprehensive guide to build systems for embedded C/C++ projects.

## When to Use

- Setting up cross-compilation for ARM/RISC-V
- Configuring CMake for embedded projects
- Creating Makefiles for MCU projects
- Setting up CI/CD for firmware builds
- Managing multiple target platforms

## CMake for Embedded

### Cross-Compilation Toolchain File

```cmake
# toolchain-arm-none-eabi.cmake

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR ARM)

# Toolchain paths
set(TOOLCHAIN_PREFIX arm-none-eabi-)

# Find toolchain
find_program(CMAKE_C_COMPILER ${TOOLCHAIN_PREFIX}gcc)
find_program(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}g++)
find_program(CMAKE_ASM_COMPILER ${TOOLCHAIN_PREFIX}gcc)
find_program(CMAKE_AR ${TOOLCHAIN_PREFIX}ar)
find_program(CMAKE_OBJCOPY ${TOOLCHAIN_PREFIX}objcopy)
find_program(CMAKE_OBJDUMP ${TOOLCHAIN_PREFIX}objdump)
find_program(CMAKE_SIZE ${TOOLCHAIN_PREFIX}size)

set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# MCU-specific flags (override per project)
set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")

set(CMAKE_C_FLAGS_INIT "${MCU_FLAGS}")
set(CMAKE_CXX_FLAGS_INIT "${MCU_FLAGS}")
set(CMAKE_ASM_FLAGS_INIT "${MCU_FLAGS}")

set(CMAKE_EXE_LINKER_FLAGS_INIT "-specs=nosys.specs -specs=nano.specs")

# Search paths
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

### CMakeLists.txt for STM32

```cmake
cmake_minimum_required(VERSION 3.20)

# Include toolchain file before project()
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_SOURCE_DIR}/cmake/toolchain-arm-none-eabi.cmake)

project(firmware VERSION 1.0.0 LANGUAGES C CXX ASM)

# Project configuration
set(TARGET_MCU STM32F407VG)
set(TARGET_CORE cortex-m4)
set(TARGET_FPU fpv4-sp-d16)
set(TARGET_FLOAT_ABI hard)

# MCU-specific definitions
add_compile_definitions(
    STM32F407xx
    USE_HAL_DRIVER
    HSE_VALUE=8000000
)

# Compiler flags
set(MCU_FLAGS "-mcpu=${TARGET_CORE} -mthumb")
if(TARGET_FPU)
    set(MCU_FLAGS "${MCU_FLAGS} -mfpu=${TARGET_FPU} -mfloat-abi=${TARGET_FLOAT_ABI}")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${MCU_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${MCU_FLAGS} -fno-rtti -fno-exceptions")
set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} ${MCU_FLAGS}")

# Warning flags
add_compile_options(
    -Wall
    -Wextra
    -Wshadow
    -Wdouble-promotion
    -Wformat=2
    -Wformat-truncation
    $<$<COMPILE_LANGUAGE:C>:-Wimplicit-function-declaration>
    $<$<COMPILE_LANGUAGE:CXX>:-Weffc++>
)

# Debug/Release configurations
set(CMAKE_C_FLAGS_DEBUG "-Og -g3 -DDEBUG")
set(CMAKE_C_FLAGS_RELEASE "-O2 -DNDEBUG")
set(CMAKE_C_FLAGS_MINSIZEREL "-Os -DNDEBUG")

# Linker script
set(LINKER_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/linker/STM32F407VGTx_FLASH.ld)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -T${LINKER_SCRIPT}")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-Map=${PROJECT_NAME}.map,--cref")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--gc-sections")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--print-memory-usage")

# Source files
file(GLOB_RECURSE SOURCES
    "src/*.c"
    "src/*.cpp"
    "startup/*.s"
)

# HAL sources (selective inclusion)
set(HAL_SOURCES
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal.c
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_cortex.c
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_gpio.c
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_rcc.c
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_uart.c
)

# Include directories
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/STM32F4xx_HAL_Driver/Inc
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx/Include
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Include
)

# Create executable
add_executable(${PROJECT_NAME}.elf ${SOURCES} ${HAL_SOURCES})

# Generate HEX and BIN files
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary ${PROJECT_NAME}.elf ${PROJECT_NAME}.bin
    COMMAND ${CMAKE_SIZE} ${PROJECT_NAME}.elf
    COMMENT "Building ${PROJECT_NAME}.hex and ${PROJECT_NAME}.bin"
)

# Flash target
add_custom_target(flash
    COMMAND openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
            -c "program ${PROJECT_NAME}.elf verify reset exit"
    DEPENDS ${PROJECT_NAME}.elf
    COMMENT "Flashing ${PROJECT_NAME}.elf"
)

# Debug target
add_custom_target(debug
    COMMAND openocd -f interface/stlink.cfg -f target/stm32f4x.cfg &
    COMMAND ${TOOLCHAIN_PREFIX}gdb ${PROJECT_NAME}.elf
            -ex "target remote localhost:3333"
            -ex "monitor reset halt"
    DEPENDS ${PROJECT_NAME}.elf
    COMMENT "Starting debug session"
)
```

### Multi-Platform CMake

```cmake
# CMakeLists.txt with platform support

# Platform selection
set(SUPPORTED_PLATFORMS stm32f4 stm32l4 nrf52 esp32)
set(PLATFORM "stm32f4" CACHE STRING "Target platform")
set_property(CACHE PLATFORM PROPERTY STRINGS ${SUPPORTED_PLATFORMS})

# Platform-specific configuration
if(PLATFORM STREQUAL "stm32f4")
    set(CMAKE_TOOLCHAIN_FILE cmake/toolchain-arm-none-eabi.cmake)
    set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")
    set(LINKER_SCRIPT linker/stm32f407.ld)
    add_compile_definitions(STM32F407xx PLATFORM_STM32)

elseif(PLATFORM STREQUAL "nrf52")
    set(CMAKE_TOOLCHAIN_FILE cmake/toolchain-arm-none-eabi.cmake)
    set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")
    set(LINKER_SCRIPT linker/nrf52840.ld)
    add_compile_definitions(NRF52840_XXAA PLATFORM_NRF)

elseif(PLATFORM STREQUAL "esp32")
    # ESP-IDF uses its own build system
    include($ENV{IDF_PATH}/tools/cmake/project.cmake)
endif()

# Common sources
set(COMMON_SOURCES
    src/app/main.c
    src/app/state_machine.c
    src/services/comm_service.c
)

# Platform-specific sources
set(PLATFORM_SOURCES_stm32f4
    src/platform/stm32/hal_impl.c
    src/platform/stm32/startup.s
)

set(PLATFORM_SOURCES_nrf52
    src/platform/nrf/hal_impl.c
    src/platform/nrf/startup.s
)

# Combine sources
set(ALL_SOURCES ${COMMON_SOURCES} ${PLATFORM_SOURCES_${PLATFORM}})
```

## Makefile for Embedded

### Complete Embedded Makefile

```makefile
# Makefile for STM32 project

# Project name
TARGET = firmware

# Build directory
BUILD_DIR = build

# Toolchain
PREFIX = arm-none-eabi-
CC = $(PREFIX)gcc
CXX = $(PREFIX)g++
AS = $(PREFIX)gcc -x assembler-with-cpp
OBJCOPY = $(PREFIX)objcopy
OBJDUMP = $(PREFIX)objdump
SIZE = $(PREFIX)size
GDB = $(PREFIX)gdb

# MCU configuration
CPU = -mcpu=cortex-m4
FPU = -mfpu=fpv4-sp-d16
FLOAT-ABI = -mfloat-abi=hard
MCU = $(CPU) -mthumb $(FPU) $(FLOAT-ABI)

# Sources
C_SOURCES = \
    src/main.c \
    src/stm32f4xx_it.c \
    src/system_stm32f4xx.c \
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal.c \
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_gpio.c

ASM_SOURCES = startup/startup_stm32f407xx.s

# Includes
C_INCLUDES = \
    -Iinclude \
    -IDrivers/STM32F4xx_HAL_Driver/Inc \
    -IDrivers/CMSIS/Device/ST/STM32F4xx/Include \
    -IDrivers/CMSIS/Include

# Defines
C_DEFS = \
    -DSTM32F407xx \
    -DUSE_HAL_DRIVER \
    -DHSE_VALUE=8000000

# Linker script
LDSCRIPT = linker/STM32F407VGTx_FLASH.ld

# Libraries
LIBS = -lc -lm -lnosys
LIBDIR =

# Compiler flags
CFLAGS = $(MCU) $(C_DEFS) $(C_INCLUDES) -Wall -Wextra -fdata-sections -ffunction-sections

# Debug vs Release
ifeq ($(DEBUG), 1)
    CFLAGS += -g3 -Og -DDEBUG
    BUILD_DIR := $(BUILD_DIR)/debug
else
    CFLAGS += -O2 -DNDEBUG
    BUILD_DIR := $(BUILD_DIR)/release
endif

# Linker flags
LDFLAGS = $(MCU) -specs=nano.specs -T$(LDSCRIPT) $(LIBDIR) $(LIBS)
LDFLAGS += -Wl,-Map=$(BUILD_DIR)/$(TARGET).map,--cref -Wl,--gc-sections
LDFLAGS += -Wl,--print-memory-usage

# Object files
OBJECTS = $(addprefix $(BUILD_DIR)/,$(notdir $(C_SOURCES:.c=.o)))
vpath %.c $(sort $(dir $(C_SOURCES)))

OBJECTS += $(addprefix $(BUILD_DIR)/,$(notdir $(ASM_SOURCES:.s=.o)))
vpath %.s $(sort $(dir $(ASM_SOURCES)))

# Default target
all: $(BUILD_DIR)/$(TARGET).elf $(BUILD_DIR)/$(TARGET).hex $(BUILD_DIR)/$(TARGET).bin size

# Create build directory
$(BUILD_DIR):
	mkdir -p $@

# Compile C files
$(BUILD_DIR)/%.o: %.c Makefile | $(BUILD_DIR)
	@echo "CC $<"
	@$(CC) -c $(CFLAGS) -MMD -MP -MF"$(@:%.o=%.d)" $< -o $@

# Compile ASM files
$(BUILD_DIR)/%.o: %.s Makefile | $(BUILD_DIR)
	@echo "AS $<"
	@$(AS) -c $(CFLAGS) $< -o $@

# Link
$(BUILD_DIR)/$(TARGET).elf: $(OBJECTS) Makefile
	@echo "LD $@"
	@$(CC) $(OBJECTS) $(LDFLAGS) -o $@

# Generate HEX
$(BUILD_DIR)/$(TARGET).hex: $(BUILD_DIR)/$(TARGET).elf
	@echo "HEX $@"
	@$(OBJCOPY) -O ihex $< $@

# Generate BIN
$(BUILD_DIR)/$(TARGET).bin: $(BUILD_DIR)/$(TARGET).elf
	@echo "BIN $@"
	@$(OBJCOPY) -O binary $< $@

# Print size
.PHONY: size
size: $(BUILD_DIR)/$(TARGET).elf
	@$(SIZE) $<

# Flash
.PHONY: flash
flash: $(BUILD_DIR)/$(TARGET).elf
	openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
		-c "program $< verify reset exit"

# Debug
.PHONY: debug
debug: $(BUILD_DIR)/$(TARGET).elf
	openocd -f interface/stlink.cfg -f target/stm32f4x.cfg &
	$(GDB) $< -ex "target remote localhost:3333" -ex "monitor reset halt"

# Clean
.PHONY: clean
clean:
	rm -rf build

# Dependencies
-include $(wildcard $(BUILD_DIR)/*.d)
```

## Linker Script Basics

### STM32 Linker Script

```ld
/* STM32F407VG Linker Script */

/* Memory regions */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
    RAM (rwx)   : ORIGIN = 0x20000000, LENGTH = 128K
    CCMRAM (rw) : ORIGIN = 0x10000000, LENGTH = 64K
}

/* Entry point */
ENTRY(Reset_Handler)

/* Stack and heap sizes */
_Min_Heap_Size = 0x2000;   /* 8KB heap */
_Min_Stack_Size = 0x1000;  /* 4KB stack */

SECTIONS
{
    /* Vector table - must be first */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } >FLASH

    /* Code */
    .text :
    {
        . = ALIGN(4);
        *(.text)
        *(.text*)
        *(.glue_7)
        *(.glue_7t)
        *(.eh_frame)

        KEEP(*(.init))
        KEEP(*(.fini))
        . = ALIGN(4);
        _etext = .;
    } >FLASH

    /* Read-only data */
    .rodata :
    {
        . = ALIGN(4);
        *(.rodata)
        *(.rodata*)
        . = ALIGN(4);
    } >FLASH

    /* ARM exception handling */
    .ARM.extab :
    {
        *(.ARM.extab* .gnu.linkonce.armextab.*)
    } >FLASH

    .ARM :
    {
        __exidx_start = .;
        *(.ARM.exidx*)
        __exidx_end = .;
    } >FLASH

    /* Used by startup to initialize data */
    _sidata = LOADADDR(.data);

    /* Initialized data */
    .data :
    {
        . = ALIGN(4);
        _sdata = .;
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;
    } >RAM AT> FLASH

    /* CCM RAM section (no DMA access!) */
    .ccmram :
    {
        . = ALIGN(4);
        _sccmram = .;
        *(.ccmram)
        *(.ccmram*)
        . = ALIGN(4);
        _eccmram = .;
    } >CCMRAM AT> FLASH

    /* Uninitialized data */
    .bss :
    {
        . = ALIGN(4);
        _sbss = .;
        __bss_start__ = _sbss;
        *(.bss)
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
        __bss_end__ = _ebss;
    } >RAM

    /* Heap */
    ._user_heap_stack :
    {
        . = ALIGN(8);
        PROVIDE(end = .);
        PROVIDE(_end = .);
        . = . + _Min_Heap_Size;
        . = . + _Min_Stack_Size;
        . = ALIGN(8);
    } >RAM

    /* Stack pointer initial value */
    _estack = ORIGIN(RAM) + LENGTH(RAM);

    /* Remove debug info from final binary */
    /DISCARD/ :
    {
        libc.a(*)
        libm.a(*)
        libgcc.a(*)
    }
}
```

## CI/CD for Firmware

### GitHub Actions Workflow

```yaml
# .github/workflows/firmware-build.yml

name: Firmware Build

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  ARM_TOOLCHAIN_VERSION: "12.2.rel1"

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        platform: [stm32f4, nrf52]
        build_type: [Debug, Release]

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Install ARM Toolchain
        run: |
          wget -q https://developer.arm.com/-/media/Files/downloads/gnu/${{ env.ARM_TOOLCHAIN_VERSION }}/binrel/arm-gnu-toolchain-${{ env.ARM_TOOLCHAIN_VERSION }}-x86_64-arm-none-eabi.tar.xz
          tar -xf arm-gnu-toolchain-*.tar.xz
          echo "$PWD/arm-gnu-toolchain-${{ env.ARM_TOOLCHAIN_VERSION }}-x86_64-arm-none-eabi/bin" >> $GITHUB_PATH

      - name: Install CMake
        uses: lukka/get-cmake@latest

      - name: Configure
        run: |
          cmake -B build \
            -DCMAKE_BUILD_TYPE=${{ matrix.build_type }} \
            -DPLATFORM=${{ matrix.platform }}

      - name: Build
        run: cmake --build build --parallel

      - name: Check Binary Size
        run: |
          arm-none-eabi-size build/firmware.elf
          # Fail if too large
          SIZE=$(arm-none-eabi-size -B build/firmware.elf | tail -1 | awk '{print $1}')
          if [ $SIZE -gt 512000 ]; then
            echo "Binary too large: $SIZE bytes"
            exit 1
          fi

      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: firmware-${{ matrix.platform }}-${{ matrix.build_type }}
          path: |
            build/firmware.elf
            build/firmware.hex
            build/firmware.bin
            build/firmware.map

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run cppcheck
        run: |
          sudo apt-get install -y cppcheck
          cppcheck --enable=all --error-exitcode=1 \
            --suppress=missingIncludeSystem \
            -I include/ src/

      - name: Run clang-tidy
        run: |
          sudo apt-get install -y clang-tidy
          cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
          clang-tidy -p build src/*.c
```

## Best Practices

### Project Organization
- Keep toolchain files separate and reusable
- Use out-of-source builds (build directory)
- Generate compile_commands.json for IDE integration
- Version control linker scripts and startup files

### Build Optimization
- Use `-ffunction-sections -fdata-sections` and `--gc-sections`
- Enable LTO for release builds (careful with debugging)
- Use nano.specs for smaller printf
- Strip debug symbols for production binaries

### Reproducibility
- Pin toolchain versions in CI
- Use submodules for vendor SDKs
- Document build environment requirements
- Generate and archive build artifacts

## Resources

- [CMake Documentation](https://cmake.org/documentation/)
- [GNU Make Manual](https://www.gnu.org/software/make/manual/)
- [ARM Toolchain Downloads](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)
- [Linker Script Guide](https://sourceware.org/binutils/docs/ld/Scripts.html)
