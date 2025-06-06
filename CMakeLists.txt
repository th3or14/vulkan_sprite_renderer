cmake_minimum_required(VERSION 3.9.0)

project(VulkanTest
	VERSION 1.0
	DESCRIPTION "Vulkan Test App"
	LANGUAGES C)

# Function to find the latest Vulkan SDK version in a given directory - needed for good old Windows
function(find_latest_vulkan_sdk base_path output_var)
    file(GLOB SDK_VERSIONS "${base_path}/*")
    list(SORT SDK_VERSIONS ORDER DESCENDING)
    foreach(version IN LISTS SDK_VERSIONS)
        if(IS_DIRECTORY "${version}")
            set(${output_var} "${version}" PARENT_SCOPE)
            return()
        endif()
    endforeach()
    message(FATAL_ERROR "No Vulkan SDK version found in ${base_path}")
endfunction()

find_package(Vulkan REQUIRED)

include_directories(${PROJECT_SOURCE_DIR}/include/)

file(GLOB_RECURSE SOURCE_LIST "${PROJECT_SOURCE_DIR}/src/*.c")
file(GLOB_RECURSE HEADER_LIST "${PROJECT_SOURCE_DIR}/include/*.h")

add_executable(main ${SOURCE_LIST} ${HEADER_LIST})

# Library linking is OS dependent
if(UNIX)
	message(STATUS "Adding Linux dependencies")
	
	find_package(SDL3 REQUIRED)

	target_link_libraries(
	  main
	  PRIVATE
	  m
	  SDL3
	  vulkan
	)
	
	# On linux let's override the output directory
	set_target_properties(main PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/dist")

elseif(WIN32)
	message(STATUS "Adding Windows dependencies")

	# Set the base path for Vulkan SDK
	set(VULKAN_SDK_BASE_PATH "C:/VulkanSDK")

	# Find the latest Vulkan SDK version
	find_latest_vulkan_sdk(${VULKAN_SDK_BASE_PATH} VULKAN_SDK_PATH)

	message(STATUS "Found Vulkan SDK at: ${VULKAN_SDK_PATH}")
	
    # Add Vulkan include directory
    target_include_directories(main PUBLIC "${VULKAN_SDK_PATH}/Include")
	target_link_libraries(main PRIVATE "${VULKAN_SDK_PATH}/Lib/vulkan-1.lib")

	target_include_directories(main PUBLIC "${CMAKE_SOURCE_DIR}/win/include")
	target_link_libraries(main PRIVATE "${CMAKE_SOURCE_DIR}/win/lib/SDL3.lib")
	
	target_link_libraries(
		main
		PRIVATE
			msvcrt
	)

	# Copy SDL3.dll to target directory
	add_custom_command(
		TARGET main POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy_if_different
			"${PROJECT_SOURCE_DIR}/win/bin/SDL3.dll"
			$<TARGET_FILE_DIR:main>
	)

endif()

# Copy shaders to the output directory

# Find all .spv files in the shaders directory
file(GLOB SHADER_SPV_FILES "${PROJECT_SOURCE_DIR}/shaders/*.spv")

# Create the shaders directory in the output directory if it doesn't exist
add_custom_command(
	TARGET main POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E make_directory
		"$<TARGET_FILE_DIR:main>/shaders"
)

# Copy each .spv file to the shaders subdirectory in the output directory
foreach(SHADER_FILE ${SHADER_SPV_FILES})
	get_filename_component(FILE_NAME ${SHADER_FILE} NAME)
	add_custom_command(
		TARGET main POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy_if_different
			"${SHADER_FILE}"
			"$<TARGET_FILE_DIR:main>/shaders/${FILE_NAME}"
	)
endforeach()

# Copy textures to output directory
add_custom_command(
    TARGET main POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    "${PROJECT_SOURCE_DIR}/textures"
    $<TARGET_FILE_DIR:main>/textures
)

if(MSVC)
	# Visual Studio is a C++ compiler - let's not enable everything there!
	# target_compile_options(main PRIVATE /W4 /WX)
else()
	# With gcc enable all of the warnings
	target_compile_options(main PRIVATE -Wall -Wextra -Wpedantic -Werror)
endif()
