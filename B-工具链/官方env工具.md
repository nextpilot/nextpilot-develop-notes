# RTT官方的ENV工具



## ConEmu

### 启动

### 启动流程

启动ConEmu之后，运行`CmdInit.cmd`脚本，

- 设置ENV_ROOT

  ```shell
  set ENV_ROOT=%~dp0..\..\..
  ```

  - 对于安装RT-Thread Studio的ENV来说：`ENV_ROOT=D:\RT-ThreadStudio\platform\env_released\env`
  - 对于直接安装ENV工具来说：`ENV_ROOT=C:\env-windows-v1.3.5`

- 设置PYTHONPATH、PYTHONHOME

  ```shell
  if /i "%processor_architecture%"=="x86" (
          set PYTHONPATH=%ENV_ROOT%\tools\Python27_32
          set PYTHONHOME=%ENV_ROOT%\tools\Python27_32
  ) else if /i "%processor_architecture%"=="amd64" (
      if defined processor_architew6432 (
          set PYTHONPATH=%ENV_ROOT%\tools\Python27_32
          set PYTHONHOME=%ENV_ROOT%\tools\Python27_32
      ) else (
          set PYTHONPATH=%ENV_ROOT%\tools\Python27
          set PYTHONHOME=%ENV_ROOT%\tools\Python27
      )
  )
  ```

  - PYTHONPATH：python查找模块和包的路径，python解释器将在这些路径下查找包。可以通过`sys.path`查看路径有哪些。

- 设置各工具相关环境变量

  ```shell
  set RTT_EXEC_PATH=%ENV_ROOT%\tools\gnu_gcc\arm_gcc\mingw\bin
  set RTT_CC=gcc
  set PKGS_ROOT=%ENV_ROOT%\packages
  set SCONS=%PYTHONPATH%\Scripts
  ```

  - RTT_EXEC_PATH：gcc编译工具路径；
  - PKGS_ROOT：RT-Thread的各类工具包存放路径，例如LVGL等；
  - SCONS：scons编译工具路径（相当于cmake，最终生成makefile）；

- 设置环境变量PATH

  ```shell
  set PATH=%ENV_ROOT%\tools\MinGit-2.25.1-32-bit\cmd;%PATH%
  set PATH=%ENV_ROOT%\tools\bin;%PATH%
  set PATH=%RTT_EXEC_PATH%;%PATH%
  set PATH=%PYTHONHOME%;%PATH%
  set PATH=%PYTHONPATH%;%PATH%
  set PATH=%SCONS%;%PATH%
  set PATH=%ENV_ROOT%\tools\qemu\qemu32;%PATH%
  ```

  > 启动ENV后，在Windows查看环境变量：
  >
  > 1）`set`：查看所有环境变量
  >
  > 2）`set PATH`：查看环境变量path
  >
  > 3）`echo %path%`：查看环境变量path

- 打印RTT版本

  ```shell
  chcp 65001 > nul
  echo RT-Thread Env Tool for Windows (V1.3.5)
  echo  ^\ ^| /
  echo - RT -     Thread Operating System
  echo  / ^| ^\
  echo 2006 - 2022 Copyright by RT-Thread team
  ```

  

## python

安装的包都会放到路径：`C:\env-windows-v1.3.5\tools\Python27\Lib\site-packages`下面。



## scons

### 简介

scons作为python的安装包，放到了在site-packages，例如其具体路径为：

`C:\env-windows-v1.3.5\tools\Python27\Lib\site-packages\scons\SCons\Script`

> 这个路径在使用scons命令进行编译时会添加到sys.path中，这样才能够正常import相关包。

其中SConscript.py提供了如下函数：

- Import()
- Export()
- GetOption()
- Import()
- SConscript()
- SetOption()

### 相关原理

使用scons进行编译时，需要在python脚本中引用scons：

```python
from SCons.Script import *
```



执行`scons`命令进行工程编译时，会自动运行相关脚本将scons相关的包添加到python解释器搜索路径下，这样在python脚本下就能够通过import导入scons相关包。应该是通过`sys.path.append()`函数添加的路径，主要有如下路径：

```shell
 '.', # scons命令所在的路径
 'd:\\rt-threadstudio\\platform\\env_released\\env\\tools\\python27\\lib\\site-packages\\scons-3.1.2',
 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\ConEmu\\ConEmu\\..\\..\\..\\tools\\Python27\\scons-3.1.2',
 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\ConEmu\\ConEmu\\..\\..\\..\\tools\\Python27\\Lib\\site-packages\\scons-3.1.2',
 'd:\\rt-threadstudio\\platform\\env_released\\env\\tools\\python27\\lib\\site-packages\\scons',
 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\ConEmu\\ConEmu\\..\\..\\..\\tools\\Python27\\scons',
 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\ConEmu\\ConEmu\\..\\..\\..\\tools\\Python27\\Lib\\site-packages\\scons',
```

> 注意：
>
> 如果直接在ENV终端运行python，默认的路径不会包含scons相关包，python启动后会默认添加的路径只包括如下：
>
> ```shell
> 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27',
> 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27\\python27.zip',
> 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27\\DLLs',
> 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27\\lib',
> 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27\\lib\\plat-win',
> 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27\\lib\\lib-tk',
> 'D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27\\lib\\site-packages']
> ```
>
> 故这时候无法导入scons相关包。



### scons编译流程

building.py文件：`rtos\rt-thread\tools\building.py`



### vscode配置

#### 智能提示

为了增加对scons等相关的智能提示，需要在Pylance插件进行一些设置。

打开Pylance插件配置，在`Python>Analysis: Extra Pahts`增加路径如下：

```shell
# scons相关包
D:\\RT-ThreadStudio\\platform\\env_released\\env\\tools\\Python27\\Lib\\site-packages\\scons
# rt-thread源码工具
D:\D2-nextpilot\nextpilot-firmware\rtos\rt-thread\tools
```



## 附录

### cmd脚本说明

获取当前盘符和路径

命令为：`%~dp0`

示例：

```shell
##
echo 当前盘符和路径：%~dp0
##
set PRJ_ROOT=%~dp0..
```