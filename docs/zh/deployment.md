# 部署

## 📦 先决条件

### 硬件要求

| 硬件 | 最低要求 | 推荐要求 |
|---|---|---|
| CPU | X86_64或ARM64 CPU，2核或更多 | 4核或更多 |
| 内存 | 4GB或更多 | 8GB或更多 |
| 存储 | 10GB或更多用于库、模型和数据 | 50GB或更多，推荐使用SSD |
| GPU | 不需要 | 支持CUDA的GPU用于加速，4GB或更多显存 |

### 软件要求

- 本地部署：Python 3.10 ~ Python 3.12，并已安装 [uv包管理器](https://docs.astral.sh/uv/getting-started/installation/)。
- Docker部署：Docker和Docker Compose（对于CUDA用户，需要`nvidia-container-runtime`）或等效的容器运行时。

## 🖥️ 本地部署

### 选择元数据存储方式

#### Qdrant数据库（推荐）

在大多数情况下，我们建议使用Qdrant数据库来存储元数据。Qdrant数据库提供高效的检索性能、灵活的可扩展性和更好的数据安全性。

请根据[Qdrant文档](https://qdrant.tech/documentation/quick-start/)部署Qdrant数据库。建议使用Docker进行部署。

如果您不想自己部署Qdrant，可以使用[Qdrant提供的在线服务](https://qdrant.tech/documentation/cloud/)。

#### 本地文件存储

本地文件存储直接将图像元数据（包括特征向量等）存储在本地SQLite数据库中。仅建议用于小规模部署或开发部署。

本地文件存储不需要额外的数据库部署过程，但有以下缺点：

- 本地存储不对向量进行索引和优化，因此所有搜索的时间复杂度均为`O(n)`。因此，如果数据规模较大，搜索和索引的性能会下降。
- 使用本地文件存储会使NekoImageGallery变为有状态，从而失去水平可扩展性。
- 当您想迁移到Qdrant数据库进行存储时，已索引的元数据可能难以直接迁移。

### 部署NekoImageGallery

> [!NOTE]
> 本教程适用于NekoImageGallery v1.4.0及更高版本，其中我们切换到`uv`作为包管理器。如果您使用的是早期版本，请参阅相应版本标签中的README文件。

1. 将项目目录克隆到您自己的PC或服务器上，然后切换到特定的版本标签（如`v1.4.0`）。
2. 安装所需的依赖项：

    ```shell
    uv sync --no-dev --extra cpu # 仅CPU部署
    
    uv sync --no-dev --extra cu124 # CUDA v12.4部署
    
    uv sync --no-dev --extra cu118 # CUDA v11.8部署
    ```

> [!NOTE]
>
> - 需要指定`--extra`选项来安装正确的依赖项。如果您不指定`--extra`选项，PyTorch及其相关依赖项将不会被安装。
> - 如果您想使用CUDA进行加速推理，请务必在此步骤中选择启用CUDA的extra变体（我们推荐`cu124`，除非您的平台不支持cuda12+）。安装后，您可以使用`torch.cuda.is_available()`来确认CUDA是否可用。
> - 如果您正在开发或测试，可以在不带`--no-dev`开关的情况下同步，以安装开发、测试和代码检查所需的依赖项。

3. 根据需要修改`config`目录中的配置文件。您可以直接修改`default.env`，但建议创建一个名为`local.env`的文件来覆盖`default.env`中的配置。
4. （可选）启用内置前端：
   NekoImageGallery v1.5.0+具有基于[NekoImageGallery.App](https://github.com/hv0905/NekoImageGallery.App)的内置前端应用程序。
   要启用它，请在您的配置文件中设置`APP_WITH_FRONTEND=True`。
   > [!WARNING]
   > 启用内置前端后，所有API将自动挂载到`/api`子路径下。例如，原来的`/docs`将变为`/api/docs`。
   > 这可能会影响您现有的部署，请谨慎操作。
5. 运行应用程序：

    ```shell
    uv run main.py
    ```

   您可以使用`--host`指定要绑定的IP地址（默认为0.0.0.0），使用`--port`指定要绑定的端口（默认为8000）。
   您可以使用`uv run main.py --help`查看所有可用的命令和选项。
6. （可选）部署前端应用程序：如果您不想使用内置前端，或者想独立部署前端，可以参考[NekoImageGallery.App](https://github.com/hv0905/NekoImageGallery.App)的[部署文档](https://github.com/hv0905/NekoImageGallery.App)。

## 🐋 Docker部署

### 关于Docker镜像

NekoImageGallery的docker镜像在Docker Hub上构建和发布，包括几个变体：

| 标签 | 描述 |
|---|---|
| `edgeneko/neko-image-gallery:<version>`<br>`edgeneko/neko-image-gallery:<version>-cuda`<br>`edgeneko/neko-image-gallery:<version>-cuda12.4` | 支持使用CUDA12.4进行GPU推理 |
| `edgeneko/neko-image-gallery:<version>-cuda11.8` | 支持使用CUDA11.8进行GPU推理 |
| `edgeneko/neko-image-gallery:<version>-cpu` | 支持CPU推理 |
| `edgeneko/neko-image-gallery:<version>-cpu-arm` | （Alpha）支持在ARM64（aarch64）设备上进行CPU推理 |

其中`<version>`是NekoImageGallery的版本号或版本别名，如下所示：

| 版本 | 描述 |
|---|---|
| `latest` | NekoImageGallery的最新稳定版本 |
| `v*.*.*` / `v*.*` | 特定的版本号（对应Git标签） |
| `edge` | NekoImageGallery的最新开发版本，可能包含不稳定功能和重大更改 |

在每个镜像中，我们都捆绑了必要的依赖项、`openai/clip-vit-large-patch14`模型权重、`bert-base-chinese`模型权重和`easy-paddle-ocr`模型，以提供一个完整且即用即得的镜像。

镜像使用`/opt/NekoImageGallery/static`作为卷来存储图像文件，如果需要本地存储，请将其挂载到您自己的卷或目录。

对于配置，我们建议使用环境变量来覆盖默认配置。机密信息（如API令牌）可以通过[docker secrets](https://docs.docker.com/engine/swarm/secrets/)提供。

> [!NOTE]
> 要启用内置前端，请设置环境变量`APP_WITH_FRONTEND=True`。
> 启用后，所有API将自动挂载到`/api`子路径下，请确保您的反向代理和其他配置正确。

### 准备`nvidia-container-runtime`

如果您想在推理过程中支持CUDA加速，请参阅[Docker GPU相关文档](https://docs.docker.com/config/containers/resource_constraints/#gpu)进行安装。

> 相关文档：
>
> 1. <https://docs.docker.com/config/containers/resource_constraints/#gpu>
> 2. <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker>
> 3. <https://nvidia.github.io/nvidia-container-runtime/>

### 运行服务器

1. 从存储库下载`docker-compose.yml`文件。

    ```shell
    # 对于cuda部署（默认）
    wget https://raw.githubusercontent.com/hv0905/NekoImageGallery/master/docker-compose.yml
    # 对于仅CPU部署
    wget https://raw.githubusercontent.com/hv0905/NekoImageGallery/master/docker-compose-cpu.yml && mv docker-compose-cpu.yml docker-compose.yml
    ```

2. 根据需要修改docker-compose.yml文件。
3. 运行以下命令以启动服务器：

    ```shell
    # 在前台启动
    docker compose up
    # 在后台（分离模式）启动
    docker compose up -d
    ```

### 上传图片到NekoImageGallery

有几种方法可以将图片上传到NekoImageGallery：

- 通过Web界面：您可以使用内置的Web界面或独立的[NekoImageGallery.App](https://github.com/hv0905/NekoImageGallery.App)将图片上传到服务器。请确保您已在配置文件中启用了**Admin API**并设置了**Admin Token**。
- 通过本地索引：这适用于本地部署或当您要上传的图片已在服务器上时。使用以下命令索引您的本地图片目录：

  ```shell
  python main.py local-index <path-to-your-image-directory>
  ```

  以上命令将递归上传指定目录及其子目录中的所有图片到服务器。您还可以为您上传的图片指定类别/星标，有关更多信息，请参阅`python main.py local-index --help`。
- 通过API：您可以使用NekoImageGallery提供的上传API来上传图片。通过这种方法，服务器可以避免在本地保存图像文件，而只存储它们的URL和元数据。
  请确保您已在配置文件中启用了**Admin API**并设置了**Admin Token**。此方法适用于自动上传图片或将NekoImageGallery与外部系统同步。有关更多信息，请查看[API文档](./api)。
