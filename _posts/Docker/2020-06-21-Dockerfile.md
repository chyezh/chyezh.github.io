# Dockerfile

`docker build`命令用于按照*dockerfile*以及*context*，生成*docker image*。需要注意的点如下：

- 上下文环境通过*url*以及本地*path*直接指定；*dockerfile*可以使用*docker build -f*参数指定，默认为*context*根目录下的*Dockerfile*文件。
- 实际构建镜像的工作是由*docker daemon*执行，而不是*docker cli*，完整的*context*会传送给*docker daemon*。因此*context*应当尽量精简，可以通过<em>.dockerimgnore</em>文件忽略一些目录和文件。
- *dockerfile*中的指令是相互独立的，因此类似于`RUN cd /tmp`不会影响其他指令。
- `docker build`会使用一些中间镜像，从而加速镜像构建流程。

## Dockerfile instruction

### FROM

`FROM`指令用于指定`docker build`的初始环境以及为后续的指令设置*base image*。一个合法的镜像构建过程必须要以一个`FROM`指令开始。

- 可以从公开仓库拉取并应用镜像。
- `ARG`是唯一一个可以在`FROM`之前出现的指令，在`FROM`之前出现的`ARG`指令不在生成阶段中，因此无法在`FROM`后使用；若要使用，则需要显示声明`ARG arg_name`。
- `FROM`可以在单个*Dockerfile*出现多次。每个`FROM`都会开启一个新镜像生成流程，并且镜像之间可以实现生成依赖。

### RUN

`RUN`指令用于在生成流程中，运行目标程序，达到镜像内容修改、消息提示等操作。

- `RUN`指令有两种形式：
    - `RUN <shell command>`：*shell*形式，在*linux*下默认使用`/bin/sh -c`运行指令，可以由`SHELL`指令更改。
    - `RUN ["excutable", "param1", "param2", ...]`：*exec*形式，直接通过系统调用创建新进程实现，需要注意正常的*shell*流程不会被执行。
- `RUN`指令是会在当前镜像上生成新的镜像层并提交生成新的中间镜像，后续指令会在新的中间镜像上执行。
- `RUN`指令生成的中间镜像缓存默认自动使用，如果存在某些不应该使用缓存的指令，如`RUN apt-get dist-upgrade -y`，应该通过命令`docker build --no-cache`显式指示不使用缓存。

### CMD

`CMD`指令是指示容器实例的默认属性，可以是可执行命令，也可以是`ENTRYPOINT`指令的默认参数。与`RUN`指令不同，`CMD`的对应程序或参数在镜像被用于创建实例`docker run`时运行或者生效；而`RUN`指令的对应程序在生成镜像时`docker build`生效。

- `CMD`指令有三种形式：
    - `CMD ["excutable", "param1", "param2", ...]`：*exec*形式，直接通过系统调用创建新进程，正常的*shell*流程不会执行。
    - `CMD ["param1", "param2", ...]`：*exec*形式，但没有指定可执行文件，用于指示`ENTRYPOINT`的默认参数。这种形式下，`CMD`和`ENTRYPOINT`都需要使用*exec*形式。如果镜像总是运行指定的可执行文件，则应该考虑使用这种形式。
    - `CMD <shell param>`：*shell*形式。
- 在*Dockerfile*中，只有最后一个`CMD`指令生效。
- 如果`docker run`指定了执行参数，则`CMD`将会被覆盖。

### LABEL

`LABEL`指令用于给镜像添加元信息。

- 镜像元信息是可继承的，最新层的镜像会覆盖*key*值冲突的*label*。
- 可以通过`docker image inspect`命令查看镜像元信息。

### EXPOSE

`EXPOSE`指令表明容器在运行时需要暴露哪些网络端口，可以是*udp*或者*tcp*，默认协议为*tcp*。`EXPOSE`的作用更多是一种镜像创建者与镜像使用者的沟通文档，并非强制指定运行时行为，当`docker run -P`时，默认发布容器的所有暴露端口，并且映射到临时端口。

- `EXPOSE`一般用于指示容器与宿主机直接的端口映射，当容器之间需要实现网络通信时，可以借助`docker network`，而不需要每个容器都暴露端口。

### ENV

`ENV`用于设置后续指令的运行的环境变量。

- `ENV`设置的环境变量会持久化，并在容器运行时仍然存在，可以使用`docker inspect`查看，使用`docker run --env <key>=<val>`改变。
- 由于对后续所有指令生效，`ENV`设置的环境变量可能带来意想不到的问题，可以使用`RUN <key>=<val> <command>`实现单指令的环境变量。

### ADD

`ADD`可以向镜像特定路径中添加文件，并且可以通过`ADD --chown=<uid>:<gid>`指定UID+GID。

### COPY
