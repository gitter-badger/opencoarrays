if("${CMAKE_Fortran_COMPILER_ID}" STREQUAL "GNU")
  set(gfortran_compiler true)
endif()

if (NOT CMAKE_VERSION VERSION_LESS 3.3.1) 
  # Detect Fortran compiler version directly
  if(gfortran_compiler AND (NOT CMAKE_Fortran_COMPILER_VERSION VERSION_LESS 5))
    set(opencoarrays_aware_compiler true)
  else()
    set(opencoarrays_aware_compiler false)
  endif()
else()
  # Use the C compiler version as a proxy for the Fortran compiler version (won't work with NAG)
  if(gfortran_compiler AND (NOT CMAKE_C_COMPILER_VERSION VERSION_LESS 5))
    set(opencoarrays_aware_compiler true)
  else()
    set(opencoarrays_aware_compiler false)
  endif()
endif()

if("${gfortran_compiler}" AND (NOT "${opencoarrays_aware_compiler}"))
  # This applied to gfortran 4.9 and some earlier versions (TODO: find out which)
  add_definitions(-DCOMPILER_SUPPORTS_CAF_INTRINSICS)
endif()

function(generate_test_script base_name num_images)
  set(source ${base_name}.F90)
  configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/../collectives/${source}
    ${CMAKE_CURRENT_BINARY_DIR}/${source}
    COPYONLY
  )
  # Now we write the script that compiles and runs the test
  set(harness "${CMAKE_BINARY_DIR}/bin_staging/test-${base_name}-extension.sh")
  install(
      FILES "${harness}"
      PERMISSIONS WORLD_EXECUTE WORLD_READ WORLD_WRITE OWNER_EXECUTE OWNER_READ OWNER_WRITE GROUP_EXECUTE GROUP_READ GROUP_WRITE
      DESTINATION ${CMAKE_CURRENT_BINARY_DIR}
  )
  file(WRITE  "${harness}" "#!/bin/bash\n")
  file(APPEND "${harness}" "cd ${CMAKE_CURRENT_BINARY_DIR}\n")
  set(executable "${base_name}_extension")
  if (opencoarrays_aware_compiler) 
    # Explicitly include the directory containing the .mod file because the caf script won't pass it 
    # automatically when the script wraps an OpenCoarrays-aware compiler
    file(APPEND 
      "${harness}" 
      "FC=mpif90 ${CMAKE_INSTALL_PREFIX}/bin/caf ${source} -o ${executable} -I${CMAKE_BINARY_DIR}/mod -DUSE_EXTENSIONS\n"
    )
  else()
    file(APPEND "${harness}" "FC=mpif90 ${CMAKE_INSTALL_PREFIX}/bin/caf ${source} -o ${executable} -DUSE_EXTENSIONS\n")
  endif()
  file(APPEND "${harness}" "${CMAKE_INSTALL_PREFIX}/bin/cafrun -np ${num_images} ./${executable}\n")
endfunction(generate_test_script)

generate_test_script(co_sum 4)
generate_test_script(co_broadcast 4)
generate_test_script(co_min 4)
generate_test_script(co_max 4)
generate_test_script(co_reduce 4)
