## 问题现象

在开发一个名为的 Operator 过程中，当我执行 （其底层是 ）来安装CustomResourceDefinition (CRD) 时，终端抛出了一个错误：

```
The CustomResourceDefinition "nova.batch.suknna" is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
make: *** [install] Error 1
```

这个错误信息非常明确：CRD 的 metadata.annotations 字段总大小超过了 262144 字节的硬性限制。

## 概念厘清：注解、CSA 与三路合并的来龙去脉

要理解这个问题，需要先弄清楚几个关键概念。

**1. annotations 是什么？**

在 Kubernetes 中，注解是与对象关联的键值对，用于存储**非标识性的元数据**。这些信息可以被工具、库或控制器读取，但 Kubernetes 自身不依赖它们来核心逻辑。

**2. last-applied-configuration**

当使用 kubectl apply 命令时，默认采用的是 **客户端应用（Client-Side Apply, CSA）** 模式。为了智能地计算用户下一次 apply 时究竟需要修改哪些字段（而不是盲目覆盖），kubectl 需要一个参照物。

它的解决方案是：将你上次通过 apply 提交的整个 YAML/JSON 文件内容，完整地保存在一个名为的注解里。

这个过程依赖于**三路合并**：

* **旧状态**： last-applied-configuration 注解中的内容。
* **当前状态**：从 Kubernetes API 服务器获取的资源当前状态。
* **新状态**：用户本次想要应用的 YAML 文件。
  kubectl 会对比这三者，精确计算出需要修改、添加或删除的字段。

![](https://cdn.nlark.com/yuque/0/2025/png/42497920/1761707143094-d5647911-4ed5-4609-8f9e-c0ffad2bc0de.png)

**3. 问题原因**

Kubebuilder 生成的 CRD 包含了非常详尽的 OpenAPI 验证规则（即 spec.versions[\*].schema）。这些规则本身就是一个极其庞大的 JSON 结构。当这个庞大的结构被整个塞进 last-applied-configuration 注解时，注解的大小就很容易触达 256KB 的天花板。

## 解决办法

既然问题的根源是 CSA 模式依赖于一个本地的、可能很大的注解，那么解决方案就是换用一种不依赖这个注解的模式。

**服务端应用（Server-Side Apply, SSA）** 正是为此而生。

**SSA 的核心思想：**

* **所有权转移**：SSA 将字段管理的职责从客户端转移到了 API 服务器。
* **字段管理器**：服务器会为每个字段记录一个“管理者”。当你声明一个字段时，你就成为了它的管理者。
* **冲突解决**：如果另一个管理者（比如另一个工程师或控制器）试图修改你管理的字段，默认情况下会产生冲突，需要明确指定 --force-conflicts 来覆盖。

**实施与效果：**
切换到 SSA 非常简单，只需在 kubectl apply 命令后加上 --server-side 标志。例如，修改你的 Makefile：

```
install: manifests kustomize ## Install CRDs into the K8s cluster specified in ~/.kube/config.
	$(KUSTOMIZE) build config/crd | $(KUBECTL) apply --server-side=true  -f -
```

执行此命令后：

1. API 服务器接管了字段合并的职责。
2. 不再需要生成和存储那个庞大的 last-applied-configuration 注解。
3. CRD 的元数据大小显著减小，256KB 的限制自然就不再是问题了。

本博客参考[FlowerCloud机场](https://hushicha.org)。转载请注明出处！
