```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 管理 Kubernetes operator

# 课程目标

- 解释 operator 模式以及安装和更新 Kubernetes operator 的不同方法。​

- 使用 Web 控制台和 operator 生命周期管理器安装和更新 operator。​

- 使用 operator 生命周期管理器 API 安装和更新 operator。​

# operator和operator OLM

## Operator 安装与普通 YAML 安装的区别

在 OpenShift 中，有两种常见的资源部署方式：使用 Operator 和普通 YAML 文件。虽然它们都能够完成资源的部署，但在功能、管理和自动化方面存在一些显著的区别。

### Operator 安装

**Operator** 是一种基于 Kubernetes 的应用管理模式，通过捕捉和自动化应用生命周期的复杂性来扩展 Kubernetes API。Operator 通常由社区或第三方开发，用于管理特定类型的应用或服务。

#### 优点

1. **自动化管理**：Operator 能自动执行诸如安装、升级、备份和恢复等操作，减少手动干预。

2. **一致性**：通过定义一致的操作流程，确保集群中的所有应用实例保持一致的状态。

3. **自愈能力**：Operator 可以监控应用状态，并在检测到问题时自动采取修复措施。

4. **扩展功能**：Operator 能扩展 Kubernetes 的功能，提供更高级的管理能力。

#### 缺点

1. **复杂性**：编写和维护 Operator 可能比较复杂，尤其是对于没有经验的开发者。

2. **依赖性**：依赖于第三方开发和维护的 Operator 可能存在版本不兼容或不及时更新的问题。

### 普通 YAML 安装

使用 **YAML 文件** 进行资源定义和部署是 Kubernetes 和 OpenShift 中最基本和常见的操作方法。用户通过编写 YAML 文件来定义所需资源的配置，然后使用 `kubectl` 或 `oc` 命令将这些配置应用到集群中。

#### 优点

1. **简单直接**：编写 YAML 文件简单直接，适合快速部署简单的应用。

2. **灵活性高**：用户可以灵活地定义各种资源配置，没有额外的约束。

3. **透明性**：所有的配置都清晰地展示在 YAML 文件中，易于理解和调试。

#### 缺点

1. **手动管理**：需要手动管理应用的整个生命周期，如升级、备份和故障恢复等。

2. **一致性难以保证**：在多个环境中手动应用配置文件时，容易出现配置不一致的问题。

3. **缺乏高级功能**：不具备自动化和高级管理功能，适合简单的应用场景。

### 总结

- <mark>**Operator 安装** 适合复杂、需要自动化管理的应用场景。它提供了自动化操作、自愈能力和一致性保证，但也带来了更多的复杂性和第三方依赖。</mark>

- <mark>**普通 YAML 安装** 则更适合简单、灵活需求的应用场景。它操作简便、透明，但需要手动管理应用的生命周期，缺乏自动化和高级功能。</mark>

## Operator的核心概念

1. **Custom Resource (自定义资源)**
   
   - 自定义资源是一种扩展 Kubernetes API 的方式，允许用户定义新的资源类型。每个自定义资源由一个 CRD（Custom Resource Definition）定义。例如，一个数据库 Operator 可能会定义一个名为 `Database` 的自定义资源。

2. **Custom Resource Definition (CRD，自定义资源定义)**
   
   - CRD 是 Kubernetes API 的一种扩展方式，用来定义新的资源类型及其行为。通过 CRD，可以创建、读取、更新和删除自定义资源。

3. **Controller（控制器）**
   
   - 控制器是负责监控 Kubernetes 中资源状态并进行相应操作的程序。Operator 中的控制器会监控自定义资源的状态，并根据需要执行操作（如部署、更新、扩缩等）。

4. **Operator Lifecycle Manager (OLM)**
   
   - OLM 是 OpenShift 提供的用于安装、升级和管理 Operator 的组件。它帮助用户简化 Operator 的部署和管理，并确保 Operator 的生命周期操作（如安装、更新和退回）更加可靠。

5. **Cluster Service Version (CSV)**
   
   - CSV 是描述 Operator 版本的 YAML 文件，包含了 Operator 的安装信息、功能、所需权限等。CSV 帮助 OLM 管理和调度 Operator。

6. **CatalogSource**
   
   - CatalogSource 是一个描述 Operator Catalog（Operator 的集合）的资源，用于存储可用 Operator 的列表。用户可以通过 CatalogSource 查找和安装所需的 Operator。

7. **InstallPlan**
   
   - InstallPlan 是描述 Operator 安装过程的资源，包含了 Operator 安装所需的步骤和依赖项。InstallPlan 由 OLM 创建和管理。

8. **Subscription**
   
   - Subscription 是用于指定要跟踪的 Operator 及其版本的资源。通过 Subscription，用户可以定义是否自动更新 Operator 以及更新频率等。

9. **Cluster Version Operator (CVO)** 
   
   - OpenShift 集群中的一个关键组件，负责管理集群的版本升级和配置管理

## Operator 分类

1. **基础 Operator**
   
   - 基础 Operator 主要用于管理特定的单一应用或服务，例如数据库 Operator 或消息队列 Operator。

2. **应用 Operator**
   
   - 应用 Operator 管理整个应用栈，包括依赖的多个组件和服务。例如，典型的应用 Operator 可能管理 Web 应用的前端、后端和数据库。

3. **集群 Operator**
   
   - 平台 Operator 负责管理 Kubernetes 集群本身的组件和功能，例如网络插件、存储解决方案等。

# 使用 Web 控制台安装 operator

打开控制台，并演示安装：File Integrity Operator

[https://console-openshift-console.apps.ocp4.example.com](https://gitee.com/link?target=https%3A%2F%2Fconsole-openshift-console.apps.ocp4.example.com)



**OpenShift 以Web界面友好著称，如需命令行模式安装，请课下自行按照课后练习体验**


