if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	option(SKIP_POSTPROCESS_BUNDLE "Skip postprocessing bundle for redistributability" OFF)
endif()

set(LIBS	core
			${LZO}
			discio
			bdisasm
			inputcommon
			common
			audiocommon
			z
			sfml-network
			${GTK2_LIBRARIES})

if((NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin") AND (NOT ANDROID))
	set(LIBS ${LIBS} rt)
endif()

if(NOT ANDROID)
	if(USE_X11)
	  set(LIBS  ${LIBS} ${X11_LIBRARIES}
              ${XINPUT2_LIBRARIES}
	      ${XRANDR_LIBRARIES})
	endif()
	if(USE_WAYLAND)
	  set(LIBS  ${LIBS} ${WAYLAND_LIBRARIES}
		${XKBCOMMON_LIBRARIES})
	endif()

	if(SDL2_FOUND)
		# Using shared SDL2
		set(LIBS	${LIBS} ${SDL2_LIBRARY})
	else(SDL2_FOUND)
		if(SDL_FOUND)
			# Using shared SDL
			set(LIBS	${LIBS} ${SDL_LIBRARY})
		else(SDL_FOUND)
			# Using static SDL from Externals
			set(LIBS	${LIBS} SDL)
		endif()
	endif()
else()
	set(LIBS ${LIBS} png iconv)
endif()

if(LIBAV_FOUND)
	set(LIBS	${LIBS} ${LIBAV_LIBRARIES})
endif()

if(wxWidgets_FOUND)
	set(SRCS 
				ARCodeAddEdit.cpp
				AboutDolphin.cpp
				CheatsWindow.cpp
				ConfigMain.cpp
				Debugger/BreakpointDlg.cpp
				Debugger/BreakpointView.cpp
				Debugger/BreakpointWindow.cpp
				Debugger/CodeView.cpp
				Debugger/CodeWindow.cpp
				Debugger/CodeWindowFunctions.cpp
				Debugger/DSPDebugWindow.cpp
				Debugger/DSPRegisterView.cpp
				Debugger/DebuggerPanel.cpp
				Debugger/DebuggerUIUtil.cpp
				Debugger/JitWindow.cpp
				Debugger/MemoryCheckDlg.cpp
				Debugger/MemoryView.cpp
				Debugger/MemoryWindow.cpp
				Debugger/RegisterView.cpp
				Debugger/RegisterWindow.cpp
				FifoPlayerDlg.cpp
				Frame.cpp
				FrameAui.cpp
				FrameTools.cpp
				GameListCtrl.cpp
				GeckoCodeDiag.cpp
				HotkeyDlg.cpp
				ISOFile.cpp
				ISOProperties.cpp
				InputConfigDiag.cpp
				InputConfigDiagBitmaps.cpp
				LogConfigWindow.cpp
				LogWindow.cpp
				Main.cpp
				MemcardManager.cpp
				MemoryCards/WiiSaveCrypted.cpp
				NetWindow.cpp
				PHackSettings.cpp
				PatchAddEdit.cpp
				TASInputDlg.cpp
				UDPConfigDiag.cpp
				VideoConfigDiag.cpp
				WXInputBase.cpp
				WiimoteConfigDiag.cpp
				WxUtils.cpp)

	set(WXLIBS ${wxWidgets_LIBRARIES} dl)
else()
	if(ANDROID)
		set(SRCS Android/ButtonManager.cpp
			 MainAndroid.cpp)
	else()
		set(SRCS MainNoGUI.cpp)
	endif()
endif()

if(USE_EGL)
	set(SRCS ${SRCS} GLInterface/Platform.cpp
		GLInterface/EGL.cpp)	
	if(USE_WAYLAND)
		set(SRCS ${SRCS} GLInterface/Wayland_Util.cpp)
	endif()
	if(USE_X11)
		set(SRCS ${SRCS} GLInterface/X11_Util.cpp)
	endif()
else()
	if(WIN32)
		set(SRCS ${SRCS} GLInterface/WGL.cpp)
	elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
		set(SRCS ${SRCS} GLInterface/AGL.cpp)
	else()
		set(SRCS ${SRCS} GLInterface/GLX.cpp
			GLInterface/X11_Util.cpp)

	endif()
endif()

if(WIN32)
	set(SRCS ${SRCS} stdafx.cpp)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	# Link against OS X system frameworks.
	list(APPEND LIBS
		${APPKIT_LIBRARY}
		${AU_LIBRARY}
		${COREAUDIO_LIBRARY}
		${COREFUND_LIBRARY}
		${CORESERV_LIBRARY}
		${IOB_LIBRARY}
		${IOK_LIBRARY}
		)
	if(wxWidgets_FOUND)
		list(APPEND LIBS
			${APPSERV_LIBRARY}
			${COCOA_LIBRARY}
			)
	endif()
	# Add resource files to application bundle.
	set(RESOURCES resources/Dolphin.icns)
	list(APPEND SRCS ${RESOURCES})
	set_source_files_properties(${RESOURCES} PROPERTIES
		MACOSX_PACKAGE_LOCATION Resources)
elseif(USE_X11)
	set(SRCS ${SRCS} X11Utils.cpp)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD" OR
        ${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
	set(LIBS ${LIBS} usbhid)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	set(DOLPHIN_EXE_BASE Dolphin)
else()
	set(DOLPHIN_EXE_BASE dolphin-emu)
endif()
if(wxWidgets_FOUND)
	set(DOLPHIN_EXE ${DOLPHIN_EXE_BASE})
else()
	set(DOLPHIN_EXE ${DOLPHIN_EXE_BASE}-nogui)
endif()

if(USE_UPNP)
  set(LIBS  ${LIBS} miniupnpc)
endif()

include(FindGettext)
if(GETTEXT_MSGMERGE_EXECUTABLE AND GETTEXT_MSGFMT_EXECUTABLE AND wxWidgets_FOUND)
	file(GLOB LINGUAS ${CMAKE_SOURCE_DIR}/Languages/po/*.po)
	add_custom_target(translations ALL)
	GETTEXT_CREATE_TRANSLATIONS(${CMAKE_SOURCE_DIR}/Languages/po/dolphin-emu.pot ${LINGUAS})
endif()

if(ANDROID)
	set(DOLPHIN_EXE main)
	add_library(${DOLPHIN_EXE} SHARED ${SRCS})
	target_link_libraries(${DOLPHIN_EXE} 
	log
	android
	"-Wl,--no-warn-mismatch"
	"-Wl,--whole-archive"
	${LIBS}
	"-Wl,--no-whole-archive"
	)
	add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
		COMMAND mkdir ARGS -p ${CMAKE_SOURCE_DIR}/Source/Android/libs/${ANDROID_NDK_ABI_NAME}
	)
	add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
		COMMAND cp ARGS ${LIBRARY_OUTPUT_PATH_ROOT}/libs/${ANDROID_NDK_ABI_NAME}/lib${DOLPHIN_EXE}.so ${CMAKE_SOURCE_DIR}/Source/Android/libs/${ANDROID_NDK_ABI_NAME}/
	)
	add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
		COMMAND cp ARGS ${CMAKE_SOURCE_DIR}/Data/Sys/GC/* ${CMAKE_SOURCE_DIR}/Source/Android/assets/
	)
else()
	add_executable(${DOLPHIN_EXE} ${SRCS})
	target_link_libraries(${DOLPHIN_EXE} ${LIBS} ${WXLIBS})
	if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
		include(BundleUtilities)
		set(BUNDLE_PATH ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${DOLPHIN_EXE}.app)

		# Ask for an application bundle.
		set_target_properties(${DOLPHIN_EXE} PROPERTIES
			MACOSX_BUNDLE true
			MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in
			)

		# get rid of any old copies
		file (REMOVE_RECURSE ${BUNDLE_PATH}/Contents/Resources/Sys)
		if(NOT SKIP_POSTPROCESS_BUNDLE)
			# Fix up the bundle after it is finished.
			# There does not seem to be an easy way to run CMake commands post-build,
			# so we invoke CMake again on a generated script.
			file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/postprocess_bundle.cmake "
				include(BundleUtilities)
				message(\"Fixing up application bundle: ${BUNDLE_PATH}\")
				message(\"(Note: This is only necessary to produce a redistributable binary.\")
				message(\"To skip, pass -DSKIP_POSTPROCESS_BUNDLE=1 to cmake.)\")
				set(BU_CHMOD_BUNDLE_ITEMS ON)
				fixup_bundle(\"${BUNDLE_PATH}\" \"\" \"\")
				file(INSTALL ${CMAKE_SOURCE_DIR}/Data/Sys
					DESTINATION ${BUNDLE_PATH}/Contents/Resources
					)
				")
			add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
				COMMAND ${CMAKE_COMMAND} -P postprocess_bundle.cmake
				)
		else()
			add_custom_command(OUTPUT ${BUNDLE_PATH}/Contents/Resources/Sys
				COMMAND ln -nfs ${CMAKE_SOURCE_DIR}/Data/Sys ${BUNDLE_PATH}/Contents/Resources/Sys
				VERBATIM
				)
			add_custom_target(CopyDataIntoBundle ALL
				DEPENDS ${BUNDLE_PATH}/Contents/Resources/Sys
				)
		endif()

		if(GETTEXT_MSGMERGE_EXECUTABLE AND GETTEXT_MSGFMT_EXECUTABLE AND wxWidgets_FOUND)
			file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/copy_translations_into_bundle.cmake "
				file(GLOB TRANSLATION_FILES RELATIVE ${CMAKE_CURRENT_BINARY_DIR}
					${CMAKE_CURRENT_BINARY_DIR}/*.gmo
					)
				foreach(TRANSLATION_FILE \${TRANSLATION_FILES})
					string(REPLACE \".gmo\" \".lproj\" TRANSLATION_DIR
						\${TRANSLATION_FILE}
						)
					# It would be better to copy to the new name as a single action,
					# but I can't figure out a way to let CMake do that.
					file(COPY ${CMAKE_CURRENT_BINARY_DIR}/\${TRANSLATION_FILE}
						DESTINATION ${BUNDLE_PATH}/Contents/Resources/\${TRANSLATION_DIR}
						NO_SOURCE_PERMISSIONS
						)
					file(RENAME
						${BUNDLE_PATH}/Contents/Resources/\${TRANSLATION_DIR}/\${TRANSLATION_FILE}
						${BUNDLE_PATH}/Contents/Resources/\${TRANSLATION_DIR}/dolphin-emu.mo
						)
				endforeach(TRANSLATION_FILE)
				")

			file(GLOB PO_FILES RELATIVE ${CMAKE_SOURCE_DIR}/Languages/po
				${CMAKE_SOURCE_DIR}/Languages/po/*.po
				)
			string(REPLACE .po .gmo GMO_FILES "${PO_FILES}")

			add_custom_command(OUTPUT ${BUNDLE_PATH}/Contents/Resources/en.lproj
				COMMAND ${CMAKE_COMMAND} -P copy_translations_into_bundle.cmake
				DEPENDS ${GMO_FILES}
				${CMAKE_SOURCE_DIR}/Data/Sys
				VERBATIM
				)
			add_custom_target(CopyTranslationsIntoBundle ALL
				DEPENDS ${BUNDLE_PATH}/Contents/Resources/en.lproj
				)
		endif()

		# Install bundle into systemwide /Applications directory.
		install(DIRECTORY ${BUNDLE_PATH} DESTINATION /Applications
			USE_SOURCE_PERMISSIONS
			)
	else()
		install(TARGETS ${DOLPHIN_EXE} RUNTIME DESTINATION ${bindir})
	endif()
endif()

set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} ${DOLPHIN_EXE})
