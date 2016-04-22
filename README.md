# Docker 源码编译

------

Docker源码编译的目的是加入调试符号，便于动态执行Docker的程序时获取数据流信息，帮助分析者了解Docker的执行过程。编译过程比较复杂，主要由 make build和make install组成。
###1.make build
make build的目的是创建docker所需的运行环境，便于后续生成docker的可执行文件。它分为4个步骤：
> * 1.docker build -t docker .
> * 2.docker run -v \`pwd\`:/go/src/github.com/docker/docker --privileged -i -t docker bash
> * 3.docker run --privileged docker hack/make.sh test
> * 4.docker run --privileged -e AWS_S3_BUCKET=baz -e AWS_ACCESS_KEY=foo -e AWS_SECRET_KEY=bar -e GPG_PASSPHRASE=gloubiboulga docker hack/release.sh

####1.1 docker build

首先，使用docker build创建新的镜像，tag为docker，这个镜像中包含很多docker安装过程中所需运行环境，同时也是编译过程中最为耗时和耗磁盘的环节，如果直接运行该指令，可能会因为网络问题导致中途执行失败。有两种解决方法：

第一，定位到出错的容器，然后找到该容器使用的基础镜像image0，手动运行images0，然后执行出错的RUN指令，当该指令完成后再保存为新的image1。最后，退回到docker host，修改Dockerfile，主要为删除出错位置的指令及其前面的指令，并修改FROM 字段为image1，再执行docker build命令。

第二，定位到出错的容器，然后找到该容器使用的基础镜像image0，手动运行images0，然后执行出错位置和之后的所有指令，保证其执行成功，然后退出保存为新的image2，即是doker。
