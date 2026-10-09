# DarkHelp - C++ helper class for Darknet's C API.
# Copyright 2019-2024 Stephane Charette <stephanecharette@gmail.com>
# MIT license applies.  See "license.txt" for details.


ADD_EXECUTABLE			( cli DarkHelpCli.cpp DarkHelp.rc )
SET_TARGET_PROPERTIES	( cli PROPERTIES OUTPUT_NAME "DarkHelp" )
TARGET_LINK_LIBRARIES	( cli PRIVATE Threads::Threads dh ${Darknet} ${OpenCV_LIBS} ${Magic} ${StdCppFS} )


ADD_EXECUTABLE			( server DarkHelpServer.cpp DarkHelp.rc )
SET_TARGET_PROPERTIES	( server PROPERTIES OUTPUT_NAME "DarkHelpServer" )
TARGET_LINK_LIBRARIES	( server PRIVATE Threads::Threads dh ${Darknet} ${OpenCV_LIBS} ${Magic} ${StdCppFS} )


ADD_EXECUTABLE			( combine DarkHelpCombine.cpp DarkHelp.rc )
SET_TARGET_PROPERTIES	( combine PROPERTIES OUTPUT_NAME "DarkHelpCombine" )
TARGET_LINK_LIBRARIES	( combine PRIVATE Threads::Threads dh ${Darknet} ${OpenCV_LIBS} ${StdCppFS} )


IF (UNIX)
	# non-Windows installation is very simple
	INSTALL (TARGETS cli server combine DESTINATION bin)
ELSE ()
	INSTALL ( FILES DarkHelp.ico DESTINATION bin )
	# more complicated install for Windows so we also get the .DLL files copied over from vcpkg
	# https://stackoverflow.com/a/72088102/13022
	#
	# vcpkg copies the 3rd-party DLLs next to each .exe, but not darknet.dll or the DLLs that only darknet.dll needs
	# (e.g. openblas.dll in CPU-only builds).  Copy those next to each .exe too, so every runtime dependency is found
	# in a single directory.  Searching several directories (such as src-lib/Release) fails with "conflicting paths"
	# since vcpkg puts identical copies of the same DLLs (e.g. abseil_dll.dll) in each of them.
	GET_FILENAME_COMPONENT (DARKNET_BIN_DIR "${DARKNET_PARENT_DIR}/../bin" ABSOLUTE)
	FOREACH (target cli server combine)
		FOREACH (dll darknet.dll openblas.dll)
			IF (EXISTS "${DARKNET_BIN_DIR}/${dll}")
				ADD_CUSTOM_COMMAND (TARGET ${target} POST_BUILD
					COMMAND ${CMAKE_COMMAND} -E copy_if_different "${DARKNET_BIN_DIR}/${dll}" "$<TARGET_FILE_DIR:${target}>")
			ENDIF ()
		ENDFOREACH ()
	ENDFOREACH ()

	INSTALL (TARGETS cli server combine DESTINATION bin
		RUNTIME_DEPENDENCIES
			# darkhelp.dll is installed by src-lib
			PRE_EXCLUDE_REGEXES "api-ms-" "ext-ms-" "wpaxholder" "HvsiFileTrust" "PdmUtilities" "[Dd]ark[Hh]elp\\.dll"
			POST_EXCLUDE_REGEXES ".*system32/.*\\.dll"
		RUNTIME DESTINATION bin
		)
ENDIF ()
