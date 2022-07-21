# cmake 语法 
官网： https://cmake.org/cmake/help/latest/command/target_link_libraries.html

https://zhuanlan.zhihu.com/p/82244559

## target_include_directories
编译项目的时候，指定目标 include 头文件目录
```
target_include_directories(<target> [SYSTEM] [BEFORE]
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```



## target_link_libraries
编译项目的时候，指定目标 链接的库
```
target_link_libraries(<target> ... <item>... ...)
```


## target_compile_options()
指定目标的编译选项。官方文档

# 目标
目标 由 add_library() 或 add_executable() 生成。


# 例子

# 针对出现中文注释编译报错，使用utf-8字符集
add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")


target_include_directories(${PLUGIN_NAME} INTERFACE
"${CMAKE_CURRENT_SOURCE_DIR}/include"
"${CMAKE_CURRENT_SOURCE_DIR}/imsdk/include"
)
message(DEBUG "path: ${CMAKE_CURRENT_SOURCE_DIR}")
#  //静态链接库
target_link_libraries(${PLUGIN_NAME} PRIVATE flutter flutter_wrapper_plugin
"${CMAKE_CURRENT_SOURCE_DIR}/imsdk/lib/Win64/ImSDK.lib"
)

# List of absolute paths to libraries that should be bundled with the plugin
set(frog_zim_plugin_bundled_libraries
"${CMAKE_CURRENT_SOURCE_DIR}/imsdk/lib/Win64/"
PARENT_SCOPE
)

