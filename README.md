# 此项目主要实现了在fabric的构建过程，去除了编译make docker时候的网络依赖，使编译可以快速，正常的通过，无需qiang后才能下载。实现者为zeoio

# 环境准备：
需要安装docker-ce 17.05或者以上版本的docker，不然编译fabric-tools的docker file文件时，会无法识别 from * as *的语法，以及copy --from=*这样的语法。  
这种语法是17.05之后才支持的muti-stage的特性。  
在make docker时，需要下载fabric-baseos和fabric-baseimage这两个镜像，其他的镜像都是基于这两个镜像产生而来的。  

## Let's do it!
当我们执行make docker的时候， 会先构建gotools。它会从网络里下载一些二进制文件。然后包含到镜像中。 所以我们现在先来去掉这部分的网络依赖。  
这些需要下载的二进制文件，在gotools.mk中。可以从第12行到第19行，和从第29行到37行看出：  
# go tool->path mapping
go.fqp.counterfeiter := github.com/maxbrunsfeld/counterfeiter  
go.fqp.gocov         := github.com/axw/gocov/gocov  
go.fqp.gocov-xml     := github.com/AlekSi/gocov-xml  
go.fqp.goimports     := golang.org/x/tools/cmd/goimports  
go.fqp.golint        := golang.org/x/lint/golint  
go.fqp.manifest-tool := github.com/estesp/manifest-tool  
go.fqp.misspell      := github.com/client9/misspell/cmd/misspell  
go.fqp.mockery       := github.com/vektra/mockery/cmd/mocker  

# Special override for protoc-gen-go since we want to use the version vendored with the project
gotool.protoc-gen-go:
    @echo "Building github.com/golang/protobuf/protoc-gen-go -> protoc-gen-go"
    GOBIN=$(abspath $(GOTOOLS_BINDIR)) go install ./vendor/github.com/golang/protobuf/protoc-gen-go

# Special override for ginkgo since we want to use the version vendored with the project
gotool.ginkgo:
    @echo "Building github.com/onsi/ginkgo/ginkgo -> ginkgo"
    GOBIN=$(abspath $(GOTOOLS_BINDIR)) go install ./vendor/github.com/onsi/ginkgo/ginkgo

所以我们现在需要先手动的下载这些二进制文件：  
go get github.com/maxbrunsfeld/counterfeiter  
go get github.com/axw/gocov/gocov  
go get github.com/AlekSi/gocov-xml  
go get golang.org/x/tools/cmd/goimports  
go get golang.org/x/lint/golint  
go get github.com/estesp/manifest-tool  
go get github.com/client9/misspell/cmd/misspell  
go get github.com/vektra/mockery/cmd/mocker  
go get github.com/golang/protobuf/protoc-gen-go  
go get github.com/onsi/ginkgo/ginkgo  
注意在下载这些二进制文件的时候，如果出现网络错误，请先翻墙，在重新下载。下载好的文件，会默认的保存到$GOPATH/bin目录下。  
所以现在，我们先在代码目录下，新建一个build/bin目录，然后把这些二进制文件复制到此目录中。  
make  -p build/bin  

cp $GOPATH/bin/counterfeiter build/bin/  
cp $GOPATH/bin/gocov build/bin/  
...
cp $GOPATH/bin/protoc-gen-go build/bin/  
cp $GOPATH/bin/ginkgo build/bin/  
我记得有个简单的shell命令，可以一行的复制指定的文件，但是忘记了 : (  
我们弄好这些依赖的二进制文件之后， 就修改makefile文件，把它们都包含进来。在第255行添加下面的代码：  
## 原来的代码
$(BUILD_DIR)/docker/gotools: gotools.mk
    @echo "Building dockerized gotools"
    @mkdir -p $@/bin $@/obj
    @$(DRUN) \
        -v $(abspath $@):/opt/gotools \
        -w /opt/gopath/src/$(PKGNAME) \
        $(BASE_DOCKER_NS)/fabric-baseimage:$(BASE_DOCKER_TAG) \
        make -f gotools.mk GOTOOLS_BINDIR=/opt/gotools/bin GOTOOLS_GOPATH=/opt/gotools/obj

## 修改后的代码
$(BUILD_DIR)/docker/gotools: gotools.mk
    @echo "Building dockerized gotools"
    @mkdir -p $@/bin $@/obj
    @$(DRUN) \
        -v $(abspath $@):/opt/gotools \
        -v $(abspath build/bin):/opt/gotools/tmp \
        -w /opt/gopath/src/$(PKGNAME) \
        $(BASE_DOCKER_NS)/fabric-baseimage:$(BASE_DOCKER_TAG) \
        make -f gotools.mk GOTOOLS_BINDIR=/opt/gotools/bin GOTOOLS_GOPATH=/opt/gotools/obj
把我们准备好的二进制文件，挂载到docker容器中。然后复制这些文件，到指定的golang二进制目录中。所以我们现在来修改gotools.mk文件，执行此操作。注释掉第22行，并将其修改成如下：
## 修改之前的
.PHONY: gotools-install
gotools-install: $(patsubst %,$(GOTOOLS_BINDIR)/%, $(GOTOOLS))

## 修改之后的
.PHONY: gotools-install
gotools-install: 
    cp /opt/gotools/tmp/* $(GOTOOLS_BINDIR)

到现在为止，我们修改好了第一步。但是在makefile中，还有一处需要修改。在代码的第226行，我们先把需要的文件下载下来，并把它放到build目录下。
curl -fL https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/chaintool-1.1.3/hyperledger-fabric-chaintool-1.1.3.jar > build/chaintool
上面那个链接是怎么来的？ 直接运行make docker，就可以看到，懒得再分析makefile代码。之后，我们修改第226行的代码，使用下载好的文件。
## 原来的代码
$(BUILD_DIR)/%/chaintool: Makefile
    @echo "Installing chaintool"
    @mkdir -p $(@D)
    curl -fL $(CHAINTOOL_URL) > $@
    chmod +x $@

## 现在的代码
$(BUILD_DIR)/%/chaintool: Makefile
    @echo "Installing chaintool"
    @mkdir -p $(@D)
    cp build/chaintool $@
    chmod +x $@
到目前为止，我们修改了2处，还有最后一处需要修改。在文件images/tools/Dockerfile.in中。在代码第16行通过apt-get安装了jq，我们先手动的下载jq的deb包，然后在添加进来进行安装。jq包依赖一个libonig2的包。所以也需要把这个包下载下来。
我们先创建一个目录，来存放这些下载的包文件：
mkdir build/deb
然后从网上手动的下载那两个包文件，放到build/deb/目录下。
之后，我们修改这个dockerfile文件。在第16行处添加：
## 原来的代码
RUN apt-get update && apt-get install -y jq

## 现在的代码
ADD build/deb/*.deb /tmp/
RUN dpkg -i /tmp/libonig2.deb
RUN dpkg -i /tmp/jq.deb
一切都已经搞定了, 大家可以自己去测试一下。

## 注意：
如果要实现其他的fabric版本无网 make docker，只需要复制里面的Makefile和gotools.mk，images/tools/Dockerfile.in以及build目录的所有文件到fabric目录下，然后再make docker即可。
注：build目录的二进制文件只适用centos和redhat系统，如果是其他系统，可以自行在平台上go build出二进制文件再复制到build的bin目录。


