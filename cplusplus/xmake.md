# 前言

1. xmake语法基于Lua语言，并使用xmake.lua文件描述构建过程。
2. xmake提出了中心仓库+自建仓库的依赖管理方式。添加第三方依赖只需要一行 add_requires 语句即可完成。
3. 设计哲学是与其他工具共存而不是取代，因此xmake支持引入使用包括cmake在内的任何构建工具的第三方库，也支持导出pkg-config/cmake的配置文件。



# shell

```sh
# 更新
xmake update

# 启用自动补全和虚拟环境功能
xmake update --integrate

# 卸载
xmake update --uninstall

# 图形化菜单配置
xmake f --menu

# 生成cmake文件
xmake project -k cmake

# 生成makefile
xmake project -k make
```



## 项目相关

1. 中间缓存被存储在 .xmake 目录中
2. 构建生成的中间文件和目标文件存放在 build 目录。

```sh
# 创建项目
xmake create project_name
xmake create -l c -P ./project_name

# 构建项目
xmake
xmake build
xmake build -v # 显示编译指令
xmake build -r # 重新编译整个项目

# 
cd project_name/

# 切换编译目标平台为mingw
xmake config -p mingw
xmake f -p mingw # f是config简写

# 更改默认工具链
xmake config --toolchain=gcc/clang

# 清理项目
xmake clean 	# 清理中间文件和目标文件
xmake clean a 	# 连同xmake缓存一同清理
xmake config -c # 仅清理xmake缓存并重新生成，不清理中间文件和目标文件 

# 运行项目
xmake run
xmake run -d target # 启动调试器并调试指定的target
```



# xmake.lua

## 一般选项

```lua
-- 设置语言及版本
-- 当前编译器支持的最新c++标准
set_languages("cxxlatest") 

-- 添加c/c++编译选项
add_cxflags()
-- 添加c++编译选项
add_cxxflags()
-- 添加c编译选项
add_cflags()
-- 增加链接选项
add_ldflags()
-- 添加静态库生成参数
add_arflags()
-- 添加动态库生成参数
add_shflags()

-- 增加包含目录, 使用模块, 无需设置该选项
add_includedirs()

-- 添加链接目录和链接库, 放在末尾
add_linkdirs() -- 查找第三方链接库的目录
add_links() -- 第三方链接库
add_syslinks("pthread", "m") -- 系统库

-- 添加预定义宏
add_defines("MYMACRO=hello")
add_defines("MYMACRO=\"hello\"")

-- 设置warning等级, 设置编译器警告等级
set_warnings("all", "error") -- -Wall -Werror

-- 描述target之间的依赖关系, 会继承其public的属性
add_deps()
-- 例如
target("mod")
	add_includedirs("include", {public=true})
	...

target("test")
	add_deps("mod") -- 会继承add_includedirs("include")
	...

-- 不链接库文件, 一般配合static使用
target("test")
	add_deps("mod", {inherit=false}) -- 不链接对应的库文件


-- 远程依赖
add_requires("package_name version-range", {<option>, configs={<>configs-options}})
add_requires("mylib >=1.0.0 <1.2.0", {<option>, configs={<>configs-options}})

target("test")
	add_packages("mylib") -- 将引入的第三方库导入对应的target作为依赖

if is_mode("debug") then
    set_symbols("debug") -- 设置生成符号信息
    set_optimize("none") -- 设置优化等级
elseif is_mode("release") then
    set_symbols("release")
    set_optimize("fastest")
    set_strip("all") -- 链接时去掉所有符号
end
```



## 选择语句

```lua
-- 选择语句
-- and/or
if <condition1> then
    <task1>
elseif <condition2> then
    <task2>
else
    <task3>
end
-- 例如: windows和linux定义不同宏
-- set_allowedplats: 限定支持的平台
-- is_plat: 判断编译的目标平台
-- is_host: 判断编译器的宿主平台
-- is_arch: 判断编译的目标架构
set_allowedplats("windows", "linux")
if is_plat("windows") then 
    add_defines("PLAT_WINDOWS")
elseif is_plat("linux") then
    add_defines("PLAT_LINUX")
end
```



## 循环语句

```lua
-- while, for, repeat

-- 声明变量
-- 数组下标从1开始
local a = 0 -- local: 局部变量
local b = a + 1
local v = {a, b, b+2}
local m = {first = a, second = b}

-- 不等判断
a ~= b
```



## 模块编译（子目录）

```lua
-- 项目目录
-- - src
-- -- main
-- --- main.cpp
-- -- mod
-- --- mod.ixx
-- --- mod.cpp
-- --- xmake.lua
-- - xmake.lua

-- ./src/mod/xmake.lua
target("mod")
	set_kind("shared")
	add_files("./mod.ixx", {public=true})
	add_files("./mod.cpp")

-- ./xmake.lua
add_rules("mode.debug", "mode.release")
set_languages("c++23")

includes("src/mod")

target("xmake-module")
	set_kind("binary")
	add_files("src/main/main.cpp")
	add_deps("mod")
```



# 包管理

1. xmake默认安装的第三方库路径： `~/.xmake/`， 可以通过环境变量 `XMAKE_PKG_CACHEDIR` 和 `XMAKE_PKG_INSTALLDIR` 更改。

2. 包管理命令：xrepo

```sh
xrepo search <package>

xrepo info <package>

xrepo scan # 查看已安装的包

xrepo install <package>

xrepo remove <package>
```



## 内网

```sh
xmake g --network=private
```



`xmake.lua`: 

```lua
set_policy("package.precompiled", false)
```





# Issue

```sh
# 警告:
warning: std and std.compat modules not found ! disabling them for the build, maybe try to add --sdk=<PATH/TO/LLVM>
# 修复
xmake f --toolchain=llvm --runtimes=c++_shared -c
```

