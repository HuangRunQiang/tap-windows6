TAP-Windows驱动程序（NDIS 6）
===========================

这是TAP-Windows驱动程序的NDIS 6.20/6.30实现，被OpenVPN和其他应用程序使用。NDIS 6.20驱动程序可以在Windows 7或更高版本上运行，但在ARM64台式系统上，由于该平台依赖其驱动程序中的下一代电源管理，需要使用NDIS 6.30。

构建
-----

构建所需的先决条件包括：

- Python 2.7
- Microsoft Windows 10 EWDK（企业Windows驱动程序工具包）
    - Visual Studio+Windows驱动程序工具包也可以。确保从“Visual Studio命令提示符”中工作，并使用“--sdk=wdk”调用buildtap.py。
- 来自`Windows-driver-samples <https://github.com/OpenVPN/Windows-driver-samples>`_（可选）的devcon源代码目录（setup/devcon）
    - 如果使用`upstream <https://github.com/Microsoft/Windows-driver-samples>`_的repo，请记得包含我们对devcon.vcxproj的补丁，以确保devcon.exe是静态链接的。
- Windows代码签名证书
- Git（不是强制要求，但在使用捆绑的bash shell运行命令时很有用）
- MakeNSIS（可选）
- 预构建的tapinstall.exe二进制文件（可选）
- Visual Studio 2019和WiX Toolset用于MSM打包（可选）

确保将Python的安装目录（通常为c:\\python27）添加到PATH环境变量中。

在Windows 10和Windows Server 2016上，使用CMD.exe、Powershell和Git Bash成功构建了tap-windows6。

查看构建脚本选项：

  $ python buildtap.py
  Usage: buildtap.py [options]

  Options:
    -h, --help           显示此帮助信息并退出
    -s SRC, --src=SRC    TAP-Windows顶级目录，默认=<CWD>
    --ti=TAPINSTALL      tapinstall（即devcon）目录（可选）
    -d, --debug          启用调试构建
    --hlk                为HLK测试构建（测试签名，无调试）
    -c, --clean          在构建之前进行nmake clean
    -b, --build          构建TAP-Windows和可能的tapinstall（在构建之前添加-c进行清理）
    --sdk=SDK            用于构建的SDK：ewdk或wdk，默认=ewdk
    --sign               对驱动程序文件进行签名
    -p, --package        从编译文件生成NSIS安装程序
    -m, --package-msm    从编译文件生成MSM安装程序
    --cert=CERT          代码签名证书的通用名称，默认为openvpn
    --certfile=CERTFILE  代码签名证书的路径
    --certpw=CERTPW      代码签名证书/密钥的密码（可选）
    --crosscert=CERT     要使用的交叉证书文件，默认为MSCV-VSClass3.cer
    --timestamp=URL      要使用的时间戳URL，默认为http://timestamp.verisign.com/scripts/timstamp.dll
    --versionoverride=FILE
                         版本覆盖文件的路径

根据需要编辑**version.m4**和**paths.py**，然后构建：

  $ python buildtap.py -b

成功完成后，所有构建产品将放置在“dist”目录中，以及tap6.tar.gz。NSIS安装程序包将放置在构建根目录中。

构建tapinstall（可选）
------------------------------

构建tapinstall最简单的方法是克隆Microsoft驱动程序示例并将devcon.exe的源代码复制到tap-windows6树中。使用PowerShell：

  $ git clone https://github.com/OpenVPN/Windows-driver-samples.git
  $ Copy-Item -Recurse Windows-driver-samples/setup/devcon tap-windows6
  $ cd tap-windows6
  $ python.exe buildtap.py -b --ti=devcon

构建系统还支持重用预构建的tapinstall.exe可执行文件。为确保构建系统找到这些可执行文件，请在tap-windows6目录下创建以下目录结构：

  devcon
  ├── Release
  │   └── devcon.exe
  ├── x64
  │   └── Release
  │       └── devcon.exe
  └── ARM64
      └── Release
          └── devcon.exe

此结构与构建tapinstall将创建的结构相同。然后使用“--ti=devcon”调用buildtap.py。将“Release”替换为您的构建配置；例如，使用--Hlk时使用“Hlk”。

请注意，如果您没有可用的tapinstall.exe，则NSIS打包（-p）步骤将失败。此外，不要使用“-c”标志，否则上述目录将在MakeNSIS能够找到它们之前被清除。

开发者模式：安装、卸载和替换驱动程序
-------------------------------------------------

可以使用命令行工具tapinstall.exe来安装TAP-Windows NDIS 6驱动程序，该工具已与OpenVPN和tap-windows安装程序捆绑在一起。请按照以下步骤安装、更新或删除tap-windows NDIS 6驱动程序：

- 将tapinstall.exe/devcon.exe放置到您的PATH中
- 打开管理员shell
- 切换到**dist**目录
- 切换到根据您的系统处理器架构的**amd64**、**i386**或**arm64**目录。

如果您正在积极开发驱动程序（例如：编辑、编译、调试、循环...），您可能不会每次都对驱动程序进行签名，因此您需要了解以下附加事项。

禁用安全启动：

未签名的驱动程序需要禁用安全启动。

- 安全启动：根据PC制造商和/或测试机器上的BIOS设置而有所不同。
- https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/disabling-secure-boot
- VMWare（一个示例）：https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.vm_admin.doc/GUID-898217D4-689D-4EB5-866C-888353FE241C.html
- Virtual Box：Virtual Box不支持SecureBoot
- Parallels（MacOS）https://kb.parallels.com/en/124242 [使用Parallels 15，默认启用，使用0禁用]

启用Windows测试模式：

还需要启用测试模式。

- 通过BCEDIT启用Windows测试模式
- 详细信息：https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdedit-command-line-options
- 具体来说，``bcdedit /set testsigning off``或``bcdedit /set testsigning on``
- 结果应该是窗口屏幕右下角显示``Test Mode``。

驱动程序安装：

注意事项

- 命令``tapinstall install OemVista.inf TAP0901``安装驱动程序
- 由于您的驱动程序未签名，“tapinstall install”步骤将弹出“大胆的未签名驱动程序警告”，您需要点击“确定”。
- 结果，驱动程序将被复制到Windows驱动程序存储中。

更新驱动程序和Windows驱动程序存储：

在某个时候，您将构建一个全新的驱动程序并需要进行测试。

- 命令``tapinstall remove TAP0901`` - 删除驱动程序
- 但是，先前批准的驱动程序仍然在Windows驱动程序存储中
- 现在键入``tapinstall install ...``，只会重新安装复制到驱动程序存储中的旧驱动程序。

关键步骤：还需要从驱动程序存储中删除驱动程序。

- 详细信息：https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc730875(v=ws.11)

有一个脚本可以做到这一点，但只有在您的驱动程序包中的文本字符串未更改时才有效。

- 脚本位置：https://github.com/mattock/tap-windows-scripts

手动步骤如下：

- 步骤1 - 通过命令``pnputil -e``获取已安装驱动程序的列表，这将列出驱动程序存储中的所有``oemNUMBER.inf``文件。
- 步骤2 - 在该列表中找到您的驱动程序，它将是某个``oem<NUMBER>.inf``文件
- 步骤3 - 要删除，请使用``pnputil.exe /d oemNUMBER.inf``

最后使用``tapinstall install OemVista.inf TAP0901``安装您的驱动程序

重要提示：

如果您没有看到“大胆的未签名驱动程序警告”，Windows将使用旧（而不是新）驱动程序。

故障排除：

检查SetupAPI日志文件有助于排查问题，查看``C:\Windows\INF\setupapi.dev.log``。

为HLK测试构建
-------------------

HLK测试应使用测试签名版本的tap-windows6驱动程序。建议的步骤是使用预构建的交叉签名的devcon.exe，并使用WDK生成的密钥对驱动程序进行签名。

首先按照上述说明设置带有预构建的devcon的目录。然后使用--hlk选项运行构建：

  $ python.exe buildtap.py -c -b --ti=devcon-prebuilt --hlk

发布过程和签名
---------------------------

在过去几年中，Microsoft的驱动程序签名要求已经大大收紧。因此，默认情况下，此构建系统不再尝试在构建时签名文件。如果您想在构建时对文件进行签名，请使用--sign选项。"sign"目录包含几个PowerShell脚本，可帮助生成已发布签名的tap-windows6软件包：

- *Cross-Sign*：交叉签名tap-windows6驱动程序文件和tapinstall.exe
- *Create-DriverSubmission*：创建特定架构的证明签名提交文件
- *Extract-DriverSubmission*：提取证明签名的zip文件
- *Sign-File*：签名文件（例如tap-windows6安装程序或驱动程序提交文件）
- *Sign-tap6.conf.ps1*：上述所有脚本的配置文件
- *Prepare-Msm.ps1*：使用Win7和Win10签名的“dist”目录生成MSM打包可以使用的“dist”目录

这些脚本中的大多数直接在tap-windows6构建系统生成的“dist”目录上操作。以下假定构建和签名是在同一台计算机上进行的。

首先为（Windows 7/8/8.1/Server 2012r2）生成交叉签名的驱动程序：

  $ python.exe buildtap.py -c -b --ti=devcon
  $ sign\Cross-Sign.ps1 -SourceDir dist -Force

请注意，Cross-Sign.ps1的"-Force"选项是*必需的*，除非您要附加签名。

接下来，为证明签名创建驱动程序提交文件：

  $ sign\Create-DriverSubmission.ps1
  $ Get-ChildItem -Path disk1|sign\Sign-File.ps1

将创建三个特定架构（i386、amd64、arm64）的提交文件。将这些提交文件提交给Windows Dev Center进行证明签名。请注意，未签名的提交文件将自动被拒绝。

在将驱动程序提交给Microsoft时，请务必仅请求适用于每个架构的签名。

此时将交叉签名的“dist”目录移开：

  $ Move-Item dist dist.win7

下载证明签名的驱动程序作为zip文件，并将其放入临时目录中（例如tap-windows6\tempdir）。然后运行Extract-DriverSubmission.ps1：

  $ Get-ChildItem -Path tempdir -Filter "*.zip"|sign\Extract-DriverSubmission.ps1

这将把驱动程序提取到“dist”目录中。将该目录移动到dist.win10：

  $ Move-Item dist dist.win10

完成后，您可以开始创建安装程序和/或MSM软件包。

如果要创建NSIS软件包，请执行以下操作：

  $ Move-Item dist.win7 dist
  $ python.exe buildtap.py -p --ti=devcon
  $ Move-Item dist dist.win7

然后执行以下操作：

  $ Move-Item dist.win10 dist
  $ python.exe buildtap.py -p --ti=devcon
  $ Move-Item dist dist.win10

最后对两个安装程序进行签名：

  $ Get-Item tap-windows*.exe|sign\Sign-File.ps1

另一方面，如果要创建MSM软件包，请执行以下操作：

  $ sign\Prepare-Msm.ps1
  $ python buildtap.py -m --sdk=wdk
  $ Get-Item tap-windows*.msm|sign\Sign-File.ps1

有关更多说明和背景信息，请参阅OpenVPN社区维基上的`此文章 <https://community.openvpn.net/openvpn/wiki/BuildingTapWindows6>`_。

覆盖version.m4中定义的设置
----------------------------------------

可以使用--versionoverride <file>选项覆盖version.m4文件中的一个或多个设置。覆盖文件中给出的任何设置优先于version.m4中的设置。

这在构建多个具有不同组件ID的tap-windows6驱动程序时非常有用，例如。

关于代理的注意事项
----------------

可以在没有与Internet连接的情况下构建tap-windows6，但任何尝试为驱动程序加上时间戳的操作都将失败。因此，在开始构建之前，请配置出站代理服务器。请注意，命令提示符还需要重新启动以使用新的代理设置。

MSM打包
-------------

为了构建MSM软件包，请先构建并签名驱动程序：

- 使用buildtap.py和“-b”标志构建TAP驱动程序。
- 对驱动程序进行EV签名
- 对驱动程序进行WHQL/Attestation签名

将已签名的驱动程序放置在tap-windows6目录下的目录结构中。每个平台目录应包含带有“win10”子目录的EV签名驱动程序，该子目录包含该平台的WHQL/Attestation签名驱动程序：

  dist
  ├── amd64
  │   ├── win10
  │   │   ├── OemVista.inf
  │   │   ├── tap0901.cat
  │   │   └── tap0901.sys
  │   ├── OemVista.inf
  │   ├── tap0901.cat
  │   └── tap0901.sys
  ├── arm64
  │   ├── win10
  │   │   ├── OemVista.inf
  │   │   ├── tap0901.cat
  │   │   └── tap0901.sys
  │   └── （注意：arm64的EV签名驱动程序未使用。）
  ├── include
  │   └── tap-windows.h
  └── i386
      ├── win10
      │   ├── OemVista.inf
      │   ├── tap0901.cat
      │   └── tap0901.sys
      ├── OemVista.inf
      ├── tap0901.cat
      └── tap0901.sys

构建MSM软件包需要安装Visual Studio 2019（EWDK不足够）和WiX Toolset。在Visual Studio 2019的开发人员命令提示符中，运行：

  $ python buildtap.py -m --sdk=wdk

这将使用嵌入驱动程序的installer.dll文件进行编译，并将其打包为特定于平台的tap-windows-<version>-<platform>.msm文件。

由于WiX Toolset尚不支持arm64平台，因此只构建了amd64和i386的MSM文件。

可选：在部署之前考虑对MSM软件包进行EV签名。但是，当将MSM合并到MSI软件包中时，MSM签名将被忽略，用户可以选择手动验证下载的MSM软件包的完整性。

许可证
-------

请参阅文件`COPYING <COPYING>`_。