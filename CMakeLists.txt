# My template for compiling STM32 projects using CMake
# Modified, but most credit goes to docs.jasony.org (jasonyang) 

# Define CMake version

cmake_minimum_required(VERSION 3.22)


# Setup cmake module path and compiler settings
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

# Message of the current build printed to the console
message("Build type: "      $(CMAKE_BUILD_TYPE))


# Setup C and C++ version

# C:
set(CMAKE_C_STANDARD            	11)
set(CMAKE_C_STANDARD_REQUIRED   	ON)
set(CMAKE_C_EXTENSIONS          	ON)
# C++: 
set(CMAKE_CXX_STANDARD          	17)
set(CMAKE_CXX_STANDARD_REQUIRED		ON)
set(CMAKE_CXX_EXTENSIONS			ON)

set(CMAKE_EXPORT_COMPILE_COMMANDS	ON)

# Define path for shorter reference 
set(PROJ_PATH 				${CMAKE_CURRENT_SOURCE_DIR}) 

# Define .cmake module for toolchain compile flags that hold true on all ARM projects
# Path is defined by the list() function (below) 
set(CMAKE_TOOLCHAIN_FILE		gcc-arm-none-eabi)



# Project Name - EDIT ACCORDING TO YOUR PROJECT 
project(TRIGGER_TEST) 

enable_language(C CXX ASM)


# Linker file - EDIT ACCORDING TO YOUR PROJECT 
set(linker_script_SRC			${PROJ_PATH}/STM32G431RBTX_FLASH.ld)

# Use project name when compiling into binary files (binary names) 
set(EXECUTABLE				${CMAKE_PROJECT_NAME}) 


# MCU Specific settings - EDIT ACCORDING TO YOUR PROJECT
# Path defined in list() function above

set(CPU_PARAMETERS ${CPU_PARAMETERS}	
	-mthumb

	-mcpu=cortex-m4		# Modifed
	-mfpu=fpv4-sp-d16	# Modifed
	-mfloat-abi=hard	# Modified

	-fdata-sections
	-ffunction-sections 
)

set(compiler_define ${compiler_define}
	"USE_HAL_DRIVER"
	"STM32F4xx"
)


# .cmake module generated using .\getIncludeList.sh and .\getSourceList.sh 
# These paths contain ALL the project source file list and include list 
# Defined in the list() function above 
include(SourceList) 
include(IncludeList)

# Executable files (binaries) 
add_executable(${EXECUTABLE} ${source_list})

# Inc paths 
target_include_directories(${EXECUTABLE} PRIVATE ${include_list})

# Project symbols 
target_compile_definitions(${EXECUTABLE} PRIVATE ${compiler_define}) 

# Compiler options 
target_compile_options(${EXECUTABLE} PRIVATE
	-T${CPU_PARAMETERS}
	-Wall
	-Wpedantic
	-Wno-unused-parameter
	# Debug configuration
	-Og -g3 -ggdb
)


# Linker options 
target_link_options(${EXECUTABLE} PRIVATE 
	-T${linker_script_SRC}
	${CPU_PARAMETERS}
	-Wl,-Map=${CMAKE_PROJECT_NAME}.map 
	--specs=nosys.specs
	# -u _print_float - STDIO float formatting support 
	-Wl,--start-group 
	-lc
	-lm
	-lstdc++
	-lsupc++
	-Wl,--end-group
	-Wl,--print-memory-usage
)

# Executes post build to print the build size 
add_custom_command(TARGET ${EXECUTABLE} POST_BUILD 
	COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${EXECUTABLE}>
)

# Convert the output to hex and binary 
add_custom_command(TARGET ${EXECUTABLE} POST_BUILD 
	COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${EXECUTABLE}> ${EXECUTABLE}.hex 
) 

# Convert to bin files - should add conditional checks(e.g. if build filed, but 
# some object files were already created - they are most probably corrupt so should remove) 
add_custom_command(TARGET ${EXECUTABLE} POST_BUILD 
	COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${EXECUTABLE}> ${EXECUTABLE}.bin
)