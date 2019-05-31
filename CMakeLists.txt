#
# Copyright 2019, Matthew Renodin
#
# This software may be distributed and modified according to the terms of
# the MIT license. Note that NO WARRANTY is provided.
# See "LICENSE" for details.
#
#

cmake_minimum_required(VERSION 3.7.2)

project(sel4training C)

add_executable(sel4training src/main.c)

target_link_libraries(sel4training sel4 muslc utils sel4muslcsys sel4platsupport sel4utils sel4debug)

DeclareRootserver(sel4training)

include(${KERNEL_HELPERS_PATH})

if(KernelSel4ArchX86_64)
    SetSimulationScriptProperty(MEM_SIZE "3G")
endif()

GenerateSimulateScript()

set(Sel4testAllowSettingsOverride OFF CACHE BOOL "Allow user to override configuration settings")
set(SIMULATION OFF CACHE BOOL "Include only simulation compatible tests")
set(RELEASE OFF CACHE BOOL "Performance optimized build")
set(VERIFICATION OFF CACHE BOOL "Only verification friendly kernel features")
set(BAMBOO OFF CACHE BOOL "Enable machine parseable output")
set(DOMAINS OFF CACHE BOOL "Test multiple domains")
set(SMP OFF CACHE BOOL "(if supported) Test SMP kernel")
set(PLATFORM "x86_64" CACHE STRING "Platform to test")
set_property(CACHE PLATFORM PROPERTY STRINGS ${KernelX86Sel4Arch_all_strings} ${KernelARMPlatform_all_strings} ${KernelRiscVPlatform_all_strings})
set(ARM_HYP OFF CACHE BOOL "Hyp mode for ARM platforms")

# We use 'FORCE' when settings these values instead of 'INTERNAL' so that they still appear
# in the cmake-gui to prevent excessively confusing users
if (NOT Sel4testAllowSettingsOverride)
    # Determine the platform/architecture
    if (${PLATFORM} IN_LIST KernelX86Sel4Arch_all_strings)
        set(KernelArch x86 CACHE STRING "" FORCE)
        set(KernelX86Sel4Arch ${PLATFORM} CACHE STRING "" FORCE)
    elseif(${PLATFORM} IN_LIST KernelRiscVPlatform_all_strings)
        set(KernelArch riscv CACHE STRING "" FORCE)
        set(KernelRiscVPlatform ${PLATFORM} CACHE STRING "" FORCE)
        if(RISCV64)
            set(KernelRiscVSel4Arch "riscv64" CACHE STRING "" FORCE)
        else()
            set(KernelRiscVSel4Arch "riscv32" CACHE STRING "" FORCE)
        endif()
    else()
        if (NOT ${PLATFORM} IN_LIST KernelARMPlatform_all_strings)
            kernel_platforms_string(plat_names_error)
            message(FATAL_ERROR "Unknown PLATFORM. Valid platforms:\n  ${plat_names_error}")
        endif()
        set(KernelArch arm CACHE STRING "" FORCE)
        set(KernelARMPlatform ${PLATFORM} CACHE STRING "" FORCE)

        if(ARM_HYP)
            set(KernelArmHypervisorSupport ON CACHE BOOL "" FORCE)
        endif()

        if(AARCH64)
            set(KernelArmSel4Arch "aarch64" CACHE STRING "" FORCE)
        elseif(AARCH32)
            set(KernelArmSel4Arch "aarch32" CACHE STRING "" FORCE)
            if(ARM_HYP)
                set(KernelArmSel4Arch "arm_hyp" CACHE STRING "" FORCE)
            endif()
        endif()

        # Elfloader settings that correspond to how Data61 sets its boards up.
        ApplyData61ElfLoaderSettings(${KernelARMPlatform} ${KernelArmSel4Arch})
    endif()

    if(SIMULATION)
        ApplyCommonSimulationSettings(${KernelArch})
    else()
        if(KernelArchX86)
            set(KernelIOMMU ON CACHE BOOL "" FORCE)
        endif()
    endif()


    # sel4test specific config settings.

    # sel4test creates processes from libsel4utils that require relatively large cspaces
    set(LibSel4UtilsCSpaceSizeBits 17 CACHE STRING "" FORCE)

    if (SIMULATION)
        set(Sel4testHaveCache OFF CACHE BOOL "" FORCE)
    else()
        set(Sel4testHaveCache ON CACHE BOOL "" FORCE)
    endif()
    if (KernelArchRiscV OR (SIMULATION AND KernelArchARM))
        set(Sel4testHaveTimer OFF CACHE BOOL "" FORCE)
    else()
        set(Sel4testHaveTimer ON CACHE BOOL "" FORCE)
    endif()

    # Check the hardware debug API non simulated (except for ia32, which can be simulated),
    # skipping any aarch64 platform, as this does not yet support the debug API, and a
    # few other miscelaneous platforms that do not support it
    if (((NOT SIMULATION) OR KernelSel4ArchIA32) AND (NOT KernelSel4ArchAarch64)
            AND (NOT KernelArchRiscV)
            AND (NOT ("${KernelARMPlatform}" STREQUAL "exynos5250"))
            AND (NOT ("${KernelARMPlatform}" STREQUAL "am335x"))
            AND (NOT ("${KernelARMPlatform}" STREQUAL "omap3"))
            AND (NOT ("${KernelARMPlatform}" STREQUAL "kzm"))
            AND (NOT ("${KernelARMPlatform}" STREQUAL "exynos5410"))
            )
        set(HardwareDebugAPI ON CACHE BOOL "" FORCE)
    else()
        set(HardwareDebugAPI OFF CACHE BOOL "" FORCE)
    endif()

    ApplyCommonReleaseVerificationSettings(${RELEASE} ${VERIFICATION})

    # Need to disable GC sections as it causes our tests to be stripped sometimes
    set(UserLinkerGCSections OFF CACHE BOOL "" FORCE)

    if (BAMBOO)
        set(LibSel4TestPrintXML ON CACHE BOOL "" FORCE)
        set(LibSel4BufferOutput ON CACHE BOOL "" FORCE)
        set(KernelIRQReporting OFF CACHE BOOL "" FORCE)
    else()
        set(LibSel4TestPrintXML OFF CACHE BOOL "" FORCE)
        set(LibSel4BufferOutput OFF CACHE BOOL "" FORCE)
        set(KernelIRQReporting ON CACHE BOOL "" FORCE)
    endif()

    if (DOMAINS)
        set(KernelNumDomains 16 CACHE STRING "" FORCE)
    else()
        set(KernelNumDomains 1 CACHE STRING "" FORCE)
    endif()

    if (SMP)
        set(KernelMaxNumNodes 4 CACHE STRING "" FORCE)
    else()
        set(KernelMaxNumNodes 1 CACHE STRING "" FORCE)
    endif()
endif()
