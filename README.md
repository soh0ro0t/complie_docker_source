# Docker 源码编译

------

Docker源码编译的目的是加入调试符号，便于动态执行Docker的程序时获取数据流信息，帮助分析者了解Docker的执行过程。编译过程比较复杂，主要由 make build和make binary组成。
###1.make build【docker container中编译docker源码】
make build的目的是创建docker所需的运行环境，生成docker-dev的image，然后在docker-dev中编译源码，不建议使用这种方法：
> * docker build -t docker .

1.1 使用docker build创建新的镜像，tag为docker，这个镜像中包含很多docker安装过程中所需运行环境，同时也是编译过程中最为耗时和耗磁盘的环节，如果直接运行该指令，可能会因为网络问题导致中途执行失败。有两种解决方法：

第一，定位到出错的容器，然后找到该容器使用的基础镜像image0，手动运行images0，然后执行出错的RUN指令，当该指令完成后再保存为新的image1。最后，退回到docker host，修改Dockerfile，主要为删除出错位置的指令及其前面的指令，并修改FROM 字段为image1，再执行docker build命令。

第二，定位到出错的容器，然后找到该容器使用的基础镜像image0，手动运行images0，然后执行出错位置和之后的所有指令，保证其执行成功，然后退出保存为新的image2，即是doker。

1.2 但是，上述两种方案各有优劣，第一种占用磁盘空间太多，使用Dockerfile成功执行一条RUN指令后会生成中间态镜像层。执行到后面阶段时，每个镜像层有GB级，特别耗磁盘空间，而且上层依赖下层，删除时只能从最上层删除，直到创建这些镜像的Dockerfile中的FROM字段的基础镜像；第二种磁盘耗损较少，但是需对Dockerfile文件中的指令和指令间的关联性了解清楚，因为bash执行和docker执行存在差别。个人建议，两种方法结合使用，尽量使用第二种方法，如遇不确定的指令时保存镜像，然后使用docker build执行，直到遇到下个错误。

###2.make binary【主机中直接编译docker源码】
> * docker run -v \`pwd\`:/go/src/github.com/docker/docker --privileged -i -t docker bash
> * docker run --privileged docker hack/make.sh test
> * docker run --privileged -e AWS_S3_BUCKET=baz -e AWS_ACCESS_KEY=foo -e AWS_SECRET_KEY=bar -e GPG_PASSPHRASE=gloubiboulga docker hack/release.sh

make binary的目的是创建docker的二进制文件，实质是执行hack/make/xx的shell脚本文件，所以只需将该文件中LDFLAGS的"-w"或"-s"选项去除即可。实际上，直接在主机上编译源码中的hack/make目录下的脚本文件即可。完全没必要再docker中创建源码的编译环境，然后再编译docker源码的方法。此外，由于自动执行时的Makefile文件存在多个步骤，手动执行时需要自己完成。譬如，编译过程中需要设置GOPATH目录，docker_src_code在该目录下被编译生成可执行文件，还需安装brctl-tools等等。

####***2.1 基础工具集安装***
具体安装的工具需在编译时报错得知，至少包含 btrfs-progs、llvm、golang。

2.1.1 安装下面工具：
[btrfs-progs](https://github.com/kdave/btrfs-progs.git)
[llvm](https://mirrors.kernel.org/sourceware/lvm2/LVM2.2.02.103.tgz)

2.1.2 安装golang，替换系统默认(老版本编译时诸多语法问题过不去)
> * curl -fsSL "https://storage.googleapis.com/golang/go1.5.4.linux-amd64.tar.gz"
> * tar -xvf -C /usr/local go1.5.4.linux-amd64.tar.gz
> * mv /usr/bin/go /usr/bin/go.old
> * ln -s /usr/local/go/bin/go /usr/bin/go

####***2.2 增加编译文件中符号信息***
编译源码最重要就是添加符号信息，否则调试看不到源码。Go build的编译选项是ldflags，正确设置改值即可。

2.2.1 注释掉Makefile中的make build选项

2.2.2 去除hack/make.sh文件中LDFLAGS中"-w"选项；
    去除hack/make/binary文件中LDFLAGS中"-s"选项

####***2.3 分部编译docker的工具集***
2.3.1 创建GOPATH
> * mkdir -p /home/thebeeman/zgo/src/github.com/docker

2.3.2 安装containerd
> * cd /home/thebeeman/zgo/src/github.com/docker && git clone github.com/docker/docker/containerd

设置： export GOPATH=/home/thebeeman/zgo
编译： make，生成contaienrd的二进制工具

2.3.3 安装runc
> * mkdir -p /home/thebeeman/zgo/src/github.com/opencontainers/runc 
> * cd /home/thebeeman/zgo/src/github.com/opencontainers && git clone https://github.com/opencontainers/runc.git

安装：apt-get install libseccom*
编译：make，生成runc

2.3.4 安装docker client与dockerd
> * cd /home/thebeeman/zgo/src/github.com/docker && git clone github.com/docker/docker

设置：export GOPATH=/home/thebeeman/zgo:/home/thebeeman/zgo/src/github.com/docker/docker/ventor
运行：hack/make.sh or hack/make.sh binary
如报错提示“useragent.go : 18 Version undefined...”之类的错误，进行如下操作：
> * cp $(find /home/thebeeman/zgo  "version_autogen.go") /home/thebeeman/zgo/src/github.com/docker/docker/dockerversion/

