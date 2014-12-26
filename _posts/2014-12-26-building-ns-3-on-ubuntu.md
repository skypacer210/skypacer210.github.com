---
layout: post
title: "Building NS 3 on Ubuntu"
categories:
- technology
tags:
- NS-3
- Python


---


#1 概述  

本文记录了NS-3模拟器在Ubuntu12.04桌面版上的完整安装记录。  

#2 安装

NS-3的安装方法有两种，一种是利用bake工具自动化安装，另一种是手动安装，安装包的下载可以通过Mercurial或则是直接下载tar包。  
由于Bake工具安装陷入超时，因此直接采用基于Mercurial的手动安装。

##2.1 依赖软件包安装  

Bake本身提供依赖包检测，这里直接将所需包安装成功。  
<pre><code>
apt-get install mercurial
apt-get install qt4-dev-tools
apt-get install gdb valgrind
apt-get install gsl-bin libgsl0-dev libgsl0ldbl
apt-get install flex bison libfl-dev
apt-get install tcpdump
apt-get install sqlite sqlite3 libsqlite3-dev
apt-get install libxml2 libxml2-dev
apt-get install libgtk2.0-0 libgtk2.0-dev
apt-get install vtun lxc
apt-get install uncrustify
apt-get install doxygen graphviz imagemagick
apt-get install python-sphinx dia
apt-get install python-pygraphviz python-kiwi python-pygoocanvas libgoocanvas-dev
apt-get install libboost-signals-dev libboost-filesystem-dev
apt-get install openmpi-bin openmpi-common openmpi-doc libopenmpi-dev
</code></pre>


##2.2 建立工作目录    

<pre><code>
mkdir workspace
cd workspace
mkdir repos
cd repos
hg clone http://code.nsnam.org/ns-3-allinone
</code></pre>

过程：
<pre><code>
requesting all changes
adding changesets
adding manifests
adding file changes
</code></pre>

最终生成ns-3-allinone目录。

##2.3 利用Mercurial下载ns-3-dev  

<pre><code>
./download.py -n ns-3-dev
</code></pre>

下载过程：  
<pre><code>
    #
    # Get NS-3
    #
    
Cloning ns-3 branch
 =>  hg clone http://code.nsnam.org/ns-3-dev ns-3-dev
requesting all changes
adding changesets
adding manifests
adding file changes
...
added 11117 changesets with 52385 changes to 7390 files
updating to branch default
2998 files updated, 0 files merged, 0 files removed, 0 files unresolved

    #
    # Get PyBindGen
    #
    
Required pybindgen version:  0.17.0.886
Trying to fetch pybindgen; this will fail if no network connection is available.  Hit Ctrl-C to skip.
 =>  bzr checkout -rrevno:886 https://launchpad.net/pybindgen pybindgen
Fetch was successful.                                                                                         

    #
    # Get NetAnim
    #
    
Required NetAnim version:  netanim-3.105
Retrieving NetAnim from http://code.nsnam.org/netanim
 =>  hg clone http://code.nsnam.org/netanim netanim
requesting all changes
adding changesets
adding manifests
adding file changes
added 275 changesets with 1533 changes to 228 files
updating to branch default
196 files updated, 0 files merged, 0 files removed, 0 files unresolved

    #
    # Get bake
    #
    
Retrieving bake from http://code.nsnam.org/bake
 =>  hg clone http://code.nsnam.org/bake
destination directory: bake
requesting all changes
adding changesets
adding manifests
adding file changes
added 333 changesets with 790 changes to 63 files
updating to branch default
45 files updated, 0 files merged, 0 files removed, 0 files unresol
</code></pre>

##2.4 编译

NS-3的编译借助build.py脚本工具。

<pre><code>
./build.py
</code></pre>

但是会碰到如下错误：  
<pre><code>
[1459/1770] cxx: src/lte/model/lte-ffr-sap.cc -> build/src/lte/model/lte-ffr-sap.cc.1.o
[1460/1770] cxx: src/lte/model/lte-fr-no-op-algorithm.cc -> build/src/lte/model/lte-fr-no-op-algorithm.cc.1.o
[1461/1770] cxx: src/lte/model/lte-ffr-soft-algorithm.cc -> build/src/lte/model/lte-ffr-soft-algorithm.cc.1.o
[1462/1770] cxx: src/lte/model/lte-ue-power-control.cc -> build/src/lte/model/lte-ue-power-control.cc.1.o
[1463/1770] cxx: build/src/lte/bindings/ns3module.cc -> build/src/lte/bindings/ns3module.cc.7.o
g++: internal compiler error: Killed (program cc1plus)
Please submit a full bug report,
with preprocessed source if appropriate.
See <file:///usr/share/doc/gcc-4.6/README.Bugs> for instructions.
Waf: Leaving directory `/home/fyang/project/workspace/repos/ns-3-allinone/ns-3-dev/build'
Build failed
 -> task in 'ns3module_lte' failed (exit status 4): 
	{task 159048492: cxx ns3module.cc -> ns3module.cc.7.o}
['/usr/bin/g++', '-O0', '-ggdb', '-g3', '-Wall', '-Werror', '-Wno-error=deprecated-declarations', '-fstrict-alid', '-fno-strict-aliasing', '-fwrapv', '-fstack-protector', '-fno-strict-aliasing', '-fvisibility=hidden', '-Wn, '-I..', '-Isrc/lte/bindings', '-I../src/lte/bindings', '-I/usr/include/python2.7', '-I/usr/include/gtk-2.0', -I/usr/include/atk-1.0', '-I/usr/include/cairo', '-I/usr/include/gdk-pixbuf-2.0', '-I/usr/include/pango-1.0', 'glib-2.0', '-I/usr/lib/i386-linux-gnu/glib-2.0/include', '-I/usr/include/pixman-1', '-I/usr/include/freetype2',bxml2', '-DNS3_ASSERT_ENABLE', '-DNS3_LOG_ENABLE', '-DHAVE_SYS_IOCTL_H=1', '-DHAVE_IF_NETS_H=1', '-DHAVE_NET_ETTE3=1', '-DHAVE_IF_TUN_H=1', '-DHAVE_GSL=1', '-DNS_DEPRECATED=', '-DNS3_DEPRECATED_H', '-DNDEBUG', 'src/lte/binings/ns3module.cc.7.o']
Traceback (most recent call last):
  File "./build.py", line 170, in <module>
    sys.exit(main(sys.argv))
  File "./build.py", line 161, in main
    build_ns3(config, build_examples, build_tests, args, build_options)
  File "./build.py", line 81, in build_ns3
    run_command([sys.executable, "waf", "build"] + build_options)
  File "/home/fyang/project/workspace/repos/ns-3-allinone/util.py", line 24, in run_command
    raise CommandError("Command %r exited with code %i" % (argv, retval))
util.CommandError: Command ['/usr/bin/python', 'waf', 'build'] exited with code 1
</code></pre>

似乎是g++编译器自身的问题，解决办法是利用swap:  
<pre><code>
sudo dd if=/dev/zero of=/swapfile bs=64M count=16
sudo mkswap /swapfile
sudo swapon /swapfile
</code></pre>

编译完成后删除：  
<pre><code>
sudo swapoff /swapfile
sudo rm /swapfile
</code></pre>

如果成功，则显示：  
<pre><code>
Waf: Leaving directory `/home/fyang/project/workspace/repos/ns-3-allinone/ns-3-dev/build'
'build' finished successfully (7m31.696s)
Modules built:
antenna                   aodv                      applications             
bridge                    buildings                 config-store             
core                      csma                      csma-layout              
dsdv                      dsr                       energy                   
fd-net-device             flow-monitor              internet                 
lr-wpan                   lte                       mesh                     
mobility                  mpi                       netanim (no Python)      
network                   nix-vector-routing        olsr                     
point-to-point            point-to-point-layout     propagation              
sixlowpan                 spectrum                  stats                    
tap-bridge                test (no Python)          topology-read            
uan                       virtual-net-device        visualizer               
wave                      wifi                      wimax                    

Modules not built (see ns-3 tutorial for explanation):
brite                     click                     openflow                 

Leaving directory `./ns-3-dev
</code></pre>

#3 测试  

##3.1 添加实例程序    
Waf工具用于脚本运行，并保证共享库在运行时处在正确位置。 通过修改Waf程序配置，添加实例程序：
<pre><code>
$ ./waf clean
$ ./waf configure --enable-examples --enable-tests --enable-modules=core
$ ./waf build
</code></pre>

##3.2 单元测试  

NS-3自带单元测试功能，有了上述实程序之后，进行验证：    
<pre><code>
fyang@fyang-virtual-machine:~/project/workspace/repos/ns-3-allinone/ns-3-dev$ ./test.py 
Waf: Entering directory `/home/fyang/project/workspace/repos/ns-3-allinone/ns-3-dev/build'
Waf: Leaving directory `/home/fyang/project/workspace/repos/ns-3-allinone/ns-3-dev/build'
'build' finished successfully (0.210s)

Modules built:
core                     

Modules not built (see ns-3 tutorial for explanation):
brite                     click                     openflow                 

PASS: TestSuite attributes
PASS: TestSuite callback
PASS: TestSuite command-line
PASS: TestSuite config
PASS: TestSuite global-value
PASS: TestSuite int64x64
PASS: TestSuite object-name-service
PASS: TestSuite object
PASS: TestSuite ptr
PASS: TestSuite event-garbage-collector
PASS: TestSuite sample
PASS: TestSuite simulator
PASS: TestSuite time
PASS: TestSuite timer
PASS: TestSuite traced-callback
PASS: TestSuite type-traits
PASS: TestSuite watchdog
PASS: TestSuite hash
PASS: TestSuite type-id
PASS: TestSuite threaded-simulator
PASS: TestSuite random-number-generators
PASS: TestSuite random-variable-stream-generators
PASS: Example examples/tutorial/hello-simulator
PASS: Example examples/tutorial/fourth
PASS: Example src/core/examples/main-callback
PASS: Example src/core/examples/sample-simulator
PASS: Example src/core/examples/main-ptr
PASS: Example src/core/examples/sample-random-variable
PASS: Example src/core/examples/sample-simulator.py
</code></pre>  

##3.3 添加实例程序  

运行模拟器：  
<pre><code>
./waf --run hello-simulator
</code></pre>

