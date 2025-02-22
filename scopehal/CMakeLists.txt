include_directories(SYSTEM ${GTKMM_INCLUDE_DIRS} ${SIGCXX_INCLUDE_DIRS})
link_directories(${GTKMM_LIBRARY_DIRS} ${SIGCXX_LIBRARY_DIRS})

find_library(LXI_LIB lxi)
find_library(TIRPC_LIB tirpc)
find_library(LINUXGPIB_LIB gpib)
find_package(glfw3 REQUIRED)


# Additional Windows/Linux libraries
if(WIN32)
	set(WIN_LIBS shlwapi)
	# The packaging for yaml-cpp on mingw64 seems to be broken, not providing a
	# libyaml-cpp.dll.a to be found by find_library inside of find_package(YAML)
	# when using FindYAML.cmake, just set the library directly
	set(YAML_LIBRARIES yaml-cpp)
else()
	set(LIN_LIBS dl)
	find_package(YAML REQUIRED)
endif()

if(LINUX AND CMAKE_SYSTEM_VERSION MATCHES "\.fc[0-9]+\.")
	set(FEDORA TRUE)
endif()

if(WIN32 OR FEDORA)
	set(SPIRV_LIBRARIES SPIRV SPIRV-Tools-opt SPIRV-Tools)
else()
	set(SPIRV_LIBRARIES "")
endif()

# Apple has their own idea about stat structs
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Dst_mtim=st_mtimespec")
endif()

if(LXI_LIB AND (NOT FEDORA OR TIRPC_LIB))
	set(HAS_LXI true)
	if (FEDORA)
		# On Fedora, tirpc is (still?) not linked into lxi
		set(LXI_LIBRARIES ${LXI_LIB} ${TIRPC_LIB})
	else()
		set(LXI_LIBRARIES ${LXI_LIB})
	endif()
else()
	set(HAS_LXI false)
	set(LXI_LIBRARIES "")
endif()

if(LINUXGPIB_LIB)
	set(HAS_LINUXGPIB true)
	set(LINUXGPIB_LIBRARIES ${LINUXGPIB_LIB})
else()
	set(HAS_LINUXGPIB false)
	set(LINUXGPIB_LIBRARIES "")
endif()

set(SCOPEHAL_SOURCES
	base64.cpp
	scopehal.cpp
	ScopehalUtil.cpp
	avx_mathfun.cpp
	VulkanInit.cpp

	FileSystem.cpp
	Unit.cpp
	Waveform.cpp
	DensityFunctionWaveform.cpp
	EyeMask.cpp
	EyeWaveform.cpp

	SCPITransport.cpp
	SCPISocketTransport.cpp
	SCPITwinLanTransport.cpp
	VICPSocketTransport.cpp
	SCPILinuxGPIBTransport.cpp
	SCPILxiTransport.cpp
	SCPINullTransport.cpp
	SCPIUARTTransport.cpp
	SCPIDevice.cpp

	IBISParser.cpp
	SParameters.cpp
	TouchstoneParser.cpp

	FlowGraphNode.cpp
	Trigger.cpp
	CDRTrigger.cpp
	CDR8B10BTrigger.cpp
	CDRNRZPatternTrigger.cpp
	DCAEdgeTrigger.cpp
	DropoutTrigger.cpp
	EdgeTrigger.cpp
	GlitchTrigger.cpp
	NthEdgeBurstTrigger.cpp
	PulseWidthTrigger.cpp
	RuntTrigger.cpp
	SerialTrigger.cpp
	SlewRateTrigger.cpp
	TwoLevelTrigger.cpp
	UartTrigger.cpp
	WindowTrigger.cpp

	Instrument.cpp
	InstrumentChannel.cpp
	BERT.cpp
	BERTInputChannel.cpp
	BERTOutputChannel.cpp
	FunctionGenerator.cpp
	FunctionGeneratorChannel.cpp
	Load.cpp
	LoadChannel.cpp
	Multimeter.cpp
	MultimeterChannel.cpp
	Oscilloscope.cpp
	OscilloscopeChannel.cpp
	PowerSupply.cpp
	PowerSupplyChannel.cpp
	RFSignalGenerator.cpp
	RFSignalGeneratorChannel.cpp
	SCPIBERT.cpp
	SCPIFunctionGenerator.cpp
	SCPIInstrument.cpp
	SCPILoad.cpp
	SCPIMiscInstrument.cpp
	SCPIMultimeter.cpp
	SCPIOscilloscope.cpp
	SCPIPowerSupply.cpp
	SCPIRFSignalGenerator.cpp
	SCPIVNA.cpp
	RemoteBridgeOscilloscope.cpp

	AgilentOscilloscope.cpp
	AntikernelLabsOscilloscope.cpp
	AntikernelLogicAnalyzer.cpp
	CopperMountainVNA.cpp
	CSVStreamInstrument.cpp
	DemoOscilloscope.cpp
	DemoPowerSupply.cpp
	DigilentOscilloscope.cpp
	DSLabsOscilloscope.cpp
	GWInstekGPDX303SPowerSupply.cpp
	KeysightDCA.cpp
	LeCroyOscilloscope.cpp
	LeCroyFWPOscilloscope.cpp
	MockOscilloscope.cpp
	MultiLaneBERT.cpp
	PicoOscilloscope.cpp
	PicoVNA.cpp
	RigolFunctionGenerator.cpp
	RigolOscilloscope.cpp
	SiglentLoad.cpp
	SiglentPowerSupply.cpp
	SiglentSCPIOscilloscope.cpp
	SiglentVectorSignalGenerator.cpp
	TektronixOscilloscope.cpp
	ThunderScopeOscilloscope.cpp
	RigolDP8xxPowerSupply.cpp
	RohdeSchwarzOscilloscope.cpp
	RSRTO6Oscilloscope.cpp
	RohdeSchwarzHMC8012Multimeter.cpp
	RohdeSchwarzHMC804xPowerSupply.cpp

	StandardColors.cpp
	Filter.cpp
	ActionProvider.cpp
	FilterParameter.cpp
	ImportFilter.cpp
	PacketDecoder.cpp
	PausableFilter.cpp
	PeakDetectionFilter.cpp
	SpectrumChannel.cpp
	SParameterSourceFilter.cpp
	SParameterFilter.cpp

	TestWaveformSource.cpp

	ComputePipeline.cpp
	FilterGraphExecutor.cpp
	PipelineCacheManager.cpp
	VulkanFFTPlan.cpp
	QueueManager.cpp
	)

if(NOT WIN32 AND NOT APPLE)
	list(APPEND SCOPEHAL_SOURCES
		# TMC is only supported on Linux for now.
		# https://github.com/glscopeclient/scopehal/issues/519
		SCPITMCTransport.cpp
	)
endif()

configure_file(config.h.in config.h)

add_library(scopehal SHARED
	${SCOPEHAL_SOURCES})

if(WIN32)
# mingw64 build using
# https://sdk.lunarg.com/sdk/download/1.3.224.1/windows/VulkanSDK-1.3.224.1-Installer.exe
# https://github.com/KhronosGroup/glslang.git (tags/sdk-1.3.224.1) static lib (SPIRV* glslang OGLCompiler GenericCodeGen MachineIndependent OSDependent)
target_link_libraries(scopehal
	${SIGCXX_LIBRARIES}
	${GTKMM_LIBRARIES}
	xptools
	log
	${YAML_LIBRARIES}
	${LXI_LIBRARIES}
	${LINUXGPIB_LIBRARIES}
	${WIN_LIBS}
	${LIN_LIBS}
	${LIBFFTS_LIBRARIES}
	${OpenMP_CXX_LIBRARIES}
	${Vulkan_LIBRARIES}
	${SPIRV_LIBRARIES}
	glslang
	Vulkan::Vulkan
	-Wl,--start-group
	OGLCompiler
	GenericCodeGen
	MachineIndependent
	-Wl,--end-group
	OSDependent
	glfw
	)
elseif(APPLE_SILICON)
target_link_libraries(scopehal
	${SIGCXX_LIBRARIES}
	${GTKMM_LIBRARIES}
	xptools
	log
	${YAML_LIBRARIES}
	${LXI_LIBRARIES}
	${LINUXGPIB_LIBRARIES}
	${WIN_LIBS}
	${LIN_LIBS}
	${OpenMP_CXX_LIBRARIES}
	${Vulkan_LIBRARIES}
	${SPIRV_LIBRARIES}
	Vulkan::Vulkan
	Vulkan::glslang
	Vulkan::shaderc_combined
	glfw
	)
else()
target_link_libraries(scopehal
	${SIGCXX_LIBRARIES}
	${GTKMM_LIBRARIES}
	xptools
	log
	${YAML_LIBRARIES}
	${LXI_LIBRARIES}
	${LINUXGPIB_LIBRARIES}
	${WIN_LIBS}
	${LIN_LIBS}
	${LIBFFTS_LIBRARIES}
	${OpenMP_CXX_LIBRARIES}
	${Vulkan_LIBRARIES}
	${SPIRV_LIBRARIES}
	Vulkan::Vulkan
	Vulkan::glslang
	Vulkan::shaderc_combined
	glfw
	)
endif()

if(APPLE_SILICON)
	target_include_directories(scopehal
	PUBLIC
		${CMAKE_CURRENT_SOURCE_DIR}
		${CMAKE_CURRENT_BINARY_DIR}
		${YAML_INCLUDES}

		# TODO: this needs to come from FindPackage etc
		/usr/include/glslang/Include
		# for macOS:
		/usr/local/include/glslang/Include
		)
else()
	target_include_directories(scopehal
	PUBLIC
		${CMAKE_CURRENT_SOURCE_DIR}
		${CMAKE_CURRENT_BINARY_DIR}
		${LIBFFTS_INCLUDE_DIR}
		${YAML_INCLUDES}

		# TODO: this needs to come from FindPackage etc
		/usr/include/glslang/Include
		# for macOS:
		/usr/local/include/glslang/Include
		)
endif()

target_include_directories(scopehal
PUBLIC SYSTEM
	${CMAKE_CURRENT_SOURCE_DIR}/../VkFFT/vkFFT
	)

target_include_directories(scopehal
PUBLIC SYSTEM
	${OpenMP_CXX_INCLUDE_DIR}
	)

# Don't include yaml-cpp in Windows PCH because this seems to give errors about path resolution
# TODO: fix this for faster builds
if(${WIN32})
	set(YAML_PCH_PATH "")
else()
	set(YAML_PCH_PATH "${YAML_INCLUDES}/yaml-cpp/yaml.h")
endif()

target_precompile_headers(scopehal
PUBLIC

	# libstdc++ headers used by vulkan and/or very widely throughout the project
	${CMAKE_CURRENT_SOURCE_DIR}/scopehal-pch.h

	${Vulkan_INCLUDE_DIR}/vulkan/vulkan_raii.hpp
	${CMAKE_CURRENT_SOURCE_DIR}/../VkFFT/vkFFT/vkFFT.h
	${YAML_PCH_PATH}
	)

if(${HAS_LXI})
	target_compile_definitions(scopehal PUBLIC HAS_LXI)
endif()

if(${HAS_LINUXGPIB})
	target_compile_definitions(scopehal PUBLIC HAS_LINUXGPIB)
endif()

install(TARGETS scopehal LIBRARY)

add_subdirectory(shaders)
