##### 指定cmake的最小版本

```
cmake_minimum_required(VERSION 3.4.1)
```

##### 设置项目名称

```
project(demo)
```



##### 设置编译类型

```
add_executable(demo demo.cpp) # 生成可执行文件
add_library(common STATIC util.cpp) # 生成静态库
add_library(common SHARED util.cpp) # 生成动态库或共享库
```

###### add_executable(<name> source1 source2 … sourceN)

用于指定从一组源文件 source1 source2 … sourceN 编译出一个**可执行文件**且命名为 name

###### add_library(<name> [STATIC | SHARED ]  source1source2 … sourceN)

用于指定从一组源文件 source1 source2 … sourceN 编译出一个静态或动态库文件且命名为 name



##### 指定包含的源文件

###### aux_source_directory( 路径  变量)

发现一个目录下所有的源代码文件并将列表存储在一个变量中

```
aux_source_directory(. SRC_LIST) # 搜索当前目录下的所有.cpp文件
add_library(demo ${SRC_LIST})
```

------

###### file(GLOB variable [RELATIVE path] [globbing expressions]...)

此命令提供了丰富的文件和目录的相关操作

1. GLOB 用于产生一个文件（目录）路径列表并保存在variable 中
2. 文件路径列表中的每个文件的文件名都能匹配globbing expressions（非正则表达式，但是类似）
3. 如果指定了 RELATIVE 路径，那么返回的文件路径列表中的路径为相对于 RELATIVE 的路径

```
file(GLOB ALL_FILE_PATH ./*)  获取当前目录下的所有的文件（目录）的路径并保存到 ALL_FILE_PATH 变量中


file(GLOB SRC_LIST "*.cpp" "protocol/*.cpp")
```



##### 设置包含的目录

###### include_directories(dir1 dir2 …)

用于设定目录，这些设定的目录将被编译器用来查找 include 文件

```
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

###### find_libraries

查找到指定的预编译库，并将它的路径存储在变量中。
默认的搜索路径为 cmake 包含的系统库，因此如果是 NDK 的公共库只需要指定库的 name 即可

```
find_library( # Sets the name of the path variable.
              log-lib
 
              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

```



###### target_link_libraries设置target需要连接的库

```
target_link_libraries( # 目标库
                       demo
                   # 目标库需要链接的库
                   # log-lib 是上面 find_library 指定的变量名
                   ${log-lib} )
```



##### 设置

###### set(<variable> <value> ）

用于设定变量 variable 的值为 value

```
 set(ENV{Name} value) # 这里没有“$”符号
```



###### add_definitions

用于添加编译器命令行标志

主要选项开关

1. **BUILD_SHARED_LIBS**：这个开关用来控制默认的库编译方式，如果不进行设置，使用 add_library 又没有指定库类型的情况下，默认编译生成的库都是静态库。如果 set(BUILD_SHARED_LIBS ON) 后，默认生成的为动态库
2. **CMAKE_C_FLAGS**：设置 C 编译选项，也可以通过指令 add_definitions() 添加
3. **CMAKE_CXX_FLAGS**：设置 C++ 编译选项，也可以通过指令 add_definitions() 添加

如果想要生成的可执行文件拥有符号表，可以gdb调试，就直接加上这句

add_definitions("-Wall -g")

###### message

message("hello world")



##### 链接第三方库

 在CMake的使用中，可以通过add_library依赖第三方库 

```
add_library(avcodec SHARED IMPORTED)  
set_target_properties(avcodec PROPERTIES IMPORTED_LOCATION  ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libavcodec-57.so)  
```

 上面代码avcodec 其实就是依赖库的名字，可以随便取，但是下面set_target_properties的第一个参数一定要和上面统一 。后面的参数就是你依赖的so库的存放路径 

### 预定义的变量

- PROJECT_SOURCE_DIR：工程的根目录
- PROJECT_BINARY_DIR：运行 cmake 命令的目录，通常是 ${PROJECT_SOURCE_DIR}/build
- PROJECT_NAME：返回通过 project 命令定义的项目名称
- CMAKE_CURRENT_SOURCE_DIR：当前处理的 CMakeLists.txt 所在的路径
- CMAKE_CURRENT_BINARY_DIR：target 编译目录
- CMAKE_CURRENT_LIST_DIR：CMakeLists.txt 的完整路径
- CMAKE_CURRENT_LIST_LINE：当前所在的行
- CMAKE_SOURCE_DIR : 其实就是CMakeLists.txt文件所在文件夹 