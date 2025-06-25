cmake_minimum_required(VERSION 3.5)

# --- 核心设置 ---
set(PROJ_NAME ALMRun)
project(${PROJ_NAME})
add_definitions(-DWIN32_LEAN_AND_MEAN) # 解决 Winsock 冲突
# 1. 通用架构检测 (适用于所有平台)
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(ARCH_NAME "x64")
    set(ARCH_BITS "64")
else()
    set(ARCH_NAME "x86")
    set(ARCH_BITS "32")
endif()
message(STATUS "Detected architecture: ${ARCH_NAME} (${ARCH_BITS}-bit)")

# 2. 平台特定设置
if(WIN32)
    # Windows 平台
    set(PLATFORM_NAME "Windows")
    # 设置 VLD 库的目录名 (Windows specific)
    if(ARCH_NAME STREQUAL "x64")
        set(VLD_ARCH_DIR "Win64")
    else()
        set(VLD_ARCH_DIR "Win32")
    endif()
    # 设置 wxWidgets 库目录的后缀
    set(WX_ARCH_SUFFIX ${ARCH_NAME})

elseif(APPLE)
    # macOS 平台
    set(PLATFORM_NAME "macOS")
    # macOS 上的 wxWidgets 通常不区分架构目录名，但我们可以预留
    # set(WX_ARCH_SUFFIX ${ARCH_NAME}) # 通常不需要

elseif(UNIX AND NOT APPLE)
    # Linux/Unix 平台
    set(PLATFORM_NAME "Linux")
    # Linux 上的 wxWidgets 库目录可能需要根据发行版和架构来定
    # 例如，在 /usr/lib/x86_64-linux-gnu/ 或 /usr/lib/i386-linux-gnu/
    # find_package(wxWidgets) 是更常用的方式
    # set(WX_ARCH_SUFFIX ${ARCH_NAME}) # 同样，find_package 通常会处理好

else()
    set(PLATFORM_NAME "Unknown")
    message(WARNING "Unsupported platform detected.")
endif()

message(STATUS "Detected platform: ${PLATFORM_NAME}")

 

# --- 1. 配置 wxWidgets (按你的要求保留) ---
# 你的 wxWidgets 路径
set(WXWIN "F:/Dev/wxWidgets-3.3.0" CACHE PATH "Path to wxWidgets")
if (NOT EXISTS "${WXWIN}/include/msvc/wx/setup.h")
    MESSAGE(FATAL_ERROR "\nwxWidgets 库路径设置不正确: ${WXWIN}\n请检查路径是否正确，或设置 WXWIN 环境变量。")
endif()
# 添加 wxWidgets 的头文件和库目录
# 根据架构选择正确的库目录
if(ARCH_NAME STREQUAL "x64")
	set(WX_LIB_DIR "${WXWIN}/lib/vc_${ARCH_NAME}_lib")
else()
	set(WX_LIB_DIR "${WXWIN}/lib/vc_lib")
endif()

message(STATUS "Using wxWidgets lib directory: ${WX_LIB_DIR}")
include_directories(${WXWIN}/include ${WX_LIB_DIR}/mswud) # Debug
include_directories(${WXWIN}/include ${WX_LIB_DIR}/mswu)  # Release
link_directories(${WX_LIB_DIR})

# --- 2. 配置第三方库 (更清晰、健壮的方式) ---
# 定义第三方库的根目录
set(THIRD_PARTY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/third_party)


# ** 2.1 添加 Lua (预编译库，需要区分平台) **
include_directories(${THIRD_PARTY_DIR}/LuaJIT-2.1/src)  

# ** 2.2 添加 Everything-SDK (预编译库，需要区分平台) **
include_directories(${THIRD_PARTY_DIR}/Everything-SDK/include) 
# 注意：你需要根据你的CPU架构选择正确的lib文件
# link_libraries(${THIRD_PARTY_DIR}/Everything-SDK/lib/Everything64.lib)

# ** 2.3 添加 VLD (Visual Leak Detector, 预编译库，需要区分平台) **
# VLD 只在 Debug 模式下链接才有意义
include_directories(${THIRD_PARTY_DIR}/vld/include)
# 同样，选择 64 位的库
#link_libraries(debug ${THIRD_PARTY_DIR}/vld/lib/Win64/vld.lib)


# --- 3. 配置主程序 (ALMRun) ---
# 找到所有源文件
aux_source_directory(src SRCS)
aux_source_directory(src/Dialog SRCS)

# 添加主程序的头文件目录
include_directories(src src/Dialog)

# 创建可执行文件
if(WIN32)
    add_executable(${PROJ_NAME} WIN32 ${SRCS} resource/merry.rc)
endif()

# --- 4. 链接所有库到主程序 ---
target_link_libraries(${PROJ_NAME}
    # 链接 Lua 库
	"${THIRD_PARTY_DIR}/LuaJIT-2.1/src/lua51.lib"
    
    # --- 链接 Windows 核心库 ---
    # 添加 Ws2_32 来解决所有网络函数链接错误
    Ws2_32
    comctl32
    rpcrt4
    Winmm
    Shell32 

    # 明确链接 Everything-SDK (区分平台)
    # 直接指定 .lib 文件的完整路径 
    "${THIRD_PARTY_DIR}/Everything-SDK/lib/Everything${ARCH_BITS}.lib"
    # 明确链接 VLD (只在 Debug 模式，并区分平台)
    debug "${THIRD_PARTY_DIR}/vld/lib/${VLD_ARCH_DIR}/debug/vld.lib"
    
    # 链接 wxWidgets 核心库 (会自动从 link_directories 中找到)
    # Debug
    debug wxbase33ud
    debug wxmsw33ud_core
    debug wxmsw33ud_adv
    
    # Release
    optimized wxbase33u
    optimized wxmsw33u_core
    optimized wxmsw33u_adv

    # --- 链接 wxWidgets 的附加库 (根据需要添加) ---
    # Debug
    debug wxbase33ud_net
    debug wxpngd
    debug wxjpegd
    debug wxzlibd
    
    # Release
    optimized wxbase33u_net
    optimized wxpng
    optimized wxjpeg
    optimized wxzlib
)

# --- 5. 最终编译器设置 ---
if(WIN32)
    # 如果使用 /MT 编译 (可选，一般用默认的 /MD)
    if (RTL STREQUAL MT)
        foreach(FLAG_VAR
            CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
            CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
            string(REGEX REPLACE "/MD" "/MT" ${FLAG_VAR} "${${FLAG_VAR}}")
        endforeach()
    endif()
    # 添加必要的编译宏，并抑制旧函数警告
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_UNICODE -DUNICODE /wd4996")
endif()

# --- 打印最终配置信息 ---
message(STATUS "wxWidgets path: ${WXWIN}")
message(STATUS "Third party path: ${THIRD_PARTY_DIR}")