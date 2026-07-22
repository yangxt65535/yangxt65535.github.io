---
title: "volcano custom plugin 编辑安装指南"
date: 2026-07-22T12:00:00+08:00
draft: false
tags: ["Volcano", "Kubernetes", "Scheduler"]
categories: ["技术笔记"]
---

# volcano custom plugin 编辑安装指南

volcano 支持接入自定义插件，需要根据特性方式编译暗转。

## 下载源码

这里使用volcano上游社区v1.9.0（即release-1.9分支）

```bash
git clone https://github.com/volcano-sh/volcano -b release-1.9
```

示例magic插件`example/custom_plugins/magic.go`，在每一个Volcano Session开启时打印一句日志

```go=
package main

import (
	"k8s.io/klog/v2"

	"volcano.sh/volcano/pkg/scheduler/framework"
)

const PluginName = "magic"

type magicPlugin struct{}

func (mp *magicPlugin) Name() string {
	return PluginName
}

// New is a PluginBuilder, remove the comment when used.
func New(arguments framework.Arguments) framework.Plugin {
	return &magicPlugin{}
}

func (mp *magicPlugin) OnSessionOpen(ssn *framework.Session) {
	klog.V(4).Info("Enter magic plugin ...")
}

func (mp *magicPlugin) OnSessionClose(ssn *framework.Session) {}
```

## 修改 Makefile

release-1.9分支的Makefile对vc-scheduler加载插件支持还不够完善，需要一些修改。

- Makefile修改支持自定义插件的vc-scheduler的构建命令

    ```bash
    # 添加MUSL_CC路径
    MUSL_CC ?= "/usr/local/musl/bin/musl-gcc"

    # 修改启用插件时的GCC为MUSL_CC
    if [ ${SUPPORT_PLUGINS} = "yes" ];then\
        CC=${MUSL_CC} CGO_ENABLED=1 go build -ldflags ${LD_FLAGS_CGO} -o ${BIN_DIR}/vc-scheduler ./cmd/scheduler;\
    else\
        ...
    ```

    > 支持插件加载需要启用CGO以支持动态链接；而volcano的基础镜像Alpine使用musl libc工具链，需要调用musl-gcc构建

- Makefile.def添加编译标签

    ```bash
    # 添加
    LD_FLAGS_CGO="\
        -linkmode=external \
        -X '${REPO_PATH}/pkg/version.GitSHA=${GitSHA}' \
        -X '${REPO_PATH}/pkg/version.Built=${Date}'   \
        -X '${REPO_PATH}/pkg/version.Version=${RELEASE_VER}'"
    ```

    > 使用外部链接模式

## 构建vc-shceduler镜像与插件动态库

release-1.9分支的示例Dockerfile对vc-scheduler加载插件支持同样不够完善，也需要一些修改。

- 创建Dockerfile.scheduler，并构建支持自定义插件的vc-scheduler镜像。

    ```dockerfile=
    FROM golang:1.21.8 AS builder
    
    WORKDIR /go/src/volcano.sh/
    
    RUN apt-get update && \
        apt-get install -y sudo
    RUN wget http://musl.libc.org/releases/musl-latest.tar.gz && \
        mkdir musl-latest && \
        tar -xf musl-latest.tar.gz -C musl-latest --strip-components=1 && \
        cd musl-latest && \
        ./configure && make && sudo make install
    
    COPY go.mod go.sum ./
    RUN go mod download
    ADD . volcano
    
    RUN cd volcano && SUPPORT_PLUGINS=yes make vc-scheduler

    FROM alpine:latest
    COPY --from=builder /go/src/volcano.sh/volcano/_output/bin/vc-scheduler /vc-scheduler
    ENTRYPOINT ["/vc-scheduler"]
    ```

    ```bash
    nerdctl -n k8s.io build -t vc-scheduler:plugins -f Dockerfile.scheduler .
    ```

- 创建Dockerfile.plugin，在镜像环境中编译插件动态库。构建并运行临时镜像，从容器中提取.so文件。

    ```dockerfile=
    FROM golang:1.21.8 AS builder
    
    WORKDIR /go/src/volcano.sh/
    
    RUN apt-get update && \
        apt-get install -y sudo
    RUN wget http://musl.libc.org/releases/musl-latest.tar.gz && \
        mkdir musl-latest && \
        tar -xf musl-latest.tar.gz -C musl-latest --strip-components=1 && \
        cd musl-latest && \
        ./configure && make && sudo make install
    
    COPY go.mod go.sum ./
    RUN go mod download
    ADD . volcano
    
    RUN cd volcano && CC=/usr/local/musl/bin/musl-gcc CGO_ENABLED=1 \
        go build -buildmode=plugin -ldflags '-linkmode=external' \
        -o example/custom-plugin/magic.so example/custom-plugin/magic.go
    
    FROM alpine:latest
    COPY --from=builder /go/src/volcano.sh/volcano/example/custom-plugin/magic.so /plugins/magic.so
    ```

    ```bash
    nerdctl -n k8s.io build -t magic-plugin:latest -f Dockerfile.plugin .
    nerdctl -n k8s.io run -d --name magic magic-plugin:latest
    nerdctl -n k8s.io cp magic:/plugins/magic.so /path/to/plugins
    nerdctl -n k8s.io stop magic && nerdctl -n k8s.io rm magic
    ```

## 插件挂载

- 修改`volcano-scheduler-config` ConfigMap，在第一个tier中添加magic插件。

    ```yaml
    tiers:
    - plugins:
      - name: magic  # 新增
      - name: priority
      - name: gang
      - name: numa-aware
        enablePreemptable: false
      - name: conformance
    ```

- 编辑`volcano-scheduler` Deployment

  - 修改vc-scheduler镜像为上面构建的镜像
  - 使用hostPath方式挂载插件文件
      ```yaml
      ...
        volumeMounts:
        - mountPath: /plugins
          name: volcano-plugins
      ...
      volumes：
      - hostPath:
          path: /path/to/plugins
          type: Directory
        name: volcano-plugins
      ...
      ```
  - 添加启动参数`--plugins-dir /plugins`，声明插件文件所在的目录
  - 为了显示`magic.go`中的日志，还需要修改启动参数`-v=4`.

- vc-shceduler启动日志中可见插件加载的日志

    ![image](https://hackmd.io/_uploads/HJQZGoirZl.png)

    > 修正："Successfully loaded Scheduler conf, ..." 日志中的插件名称只和configmap的数据有关，和是否成功加载无关

    在每一个Volcano Session开启时，可以看到插件的OnSessionOpen日志

    ![image](https://hackmd.io/_uploads/HkcN7jsS-x.png)


