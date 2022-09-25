---
title: Charts模板指南
date: 2021-08-17 20:24:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - helm
---

# 一、Charts

## Helm chart的结构

```
wordpress/
  Chart.yaml          # 包含了chart信息的YAML文件
  LICENSE             # 可选: 包含chart许可证的纯文本文件
  README.md           # 可选: 可读的README文件
  values.yaml         # chart 默认的配置值
  values.schema.json  # 可选: 一个使用JSON结构的values.yaml文件
  charts/             # 包含chart依赖的其他chart
  crds/               # 自定义资源的定义
  templates/          # 模板目录， 当和values 结合时，可生成有效的Kubernetes manifest文件
  templates/NOTES.txt # 可选: 包含简要使用说明的纯文本文件
```

## 通过命令创建helm

```
$ helm create mychart
Creating mychart
```

 `mychart/templates/` 目录：

- `NOTES.txt`: chart的"帮助文本"。这会在你的用户执行`helm install`时展示给他们。
- `deployment.yaml`: 创建Kubernetes 工作负载的基本清单。
- `service.yaml`: 为你的工作负载创建一个 service终端基本清单。
- `_helpers.tpl`: 放置可以通过chart复用的模板辅助对象

为了方便学习，先删除templates所有文件：

```
$ rm -rf mychart/templates/*
```

## 手动创建模板

创建一个配置映射名为 `mychart/templates/configmap.yaml`的文件

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

安装helm chart

```
[root@k8s-master01 mychart]# helm install full-coral  ./mychart
NAME: full-coral
LAST DEPLOYED: Sat Jul 23 23:06:05 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

查看helm chart清单明细

```
[root@k8s-master01 mychart]# helm get manifest full-coral
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

`helm get manifest` 命令后跟一个发布名称(`full-coral`)然后打印出了所有已经上传到server的Kubernetes资源。 每个文件以`---`开头表示YAML文件的开头，然后是自动生成的注释行，表示哪个模板文件生成了这个YAML文档。

现在卸载发布： `helm uninstall full-coral`。

## 模板调用

通过命令卸载full-coral，然后修改configmap.yaml

```
$ helm uninstall full-coral
$ vi configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

执行结果：

```
$ helm install full-coral  ./mychart
$ helm get manifest full-coral
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: full-coral-configmap
data:
  myvalue: "Hello World"
```

模板命令要括在 `{{` 和 `}}` 之间，模板命令 `{{ .Release.Name }}` 将发布名称注入了模板。值作为一个 *命名空间对象* 传给了模板，用点(`.`)分隔每个命名空间的元素。

`Release`前面的点表示从作用域最顶层的命名空间开始（稍后会谈作用域）。这样`.Release.Name`就可解读为“通顶层命名空间开始查找 Release对象，然后在其中找Name对象”。`Release`是一个Helm的内置对象。

## 非安装渲染模板

当你想测试模板渲染的内容但又不想安装任何实际应用时，可以使用`helm install --debug --dry-run goodly-guppy ./mychart`。这样不会安装应用(chart)到你的kubenetes集群中，只会渲染模板内容到控制台（用于测试）。渲染后的模板如下：

```
$ helm install --debug --dry-run goodly-guppy ./mychart
```

使用`--dry-run`会让你变得更容易测试，但不能保证Kubernetes会接受你生成的模板。 最好不要仅仅因为`--dry-run`可以正常运行就觉得chart可以安装。

# 二、内置对象

对象可以通过模板引擎传递到模板中。对象可以是非常简单的:仅有一个值。或者可以包含其他对象或方法。比如，`Release`对象可以包含其他对象（比如：`Release.Name`）和`Files`对象有一组方法。

- Release： Release对象描述了版本发布本身。包含了以下对象：
  - Release.Name： release名称
  - Release.Namespace： 版本中包含的命名空间(如果manifest没有覆盖的话)
  - Release.IsUpgrade： 如果当前操作是升级或回滚的话，该值将被设置为true
  - Release.IsInstall： 如果当前操作是安装的话，该值将被设置为true
  - Release.Revision： 此次修订的版本号。安装时是1，每次升级或回滚都会自增
  - Release.Service： 该service用来渲染当前模板。Helm里始终Helm
- Values： Values对象是从values.yaml文件和用户提供的文件传进模板的。默认为空
- Chart： Chart.yaml文件内容。 Chart.yaml里的所有数据在这里都可以可访问的。
- Files： 在chart中提供访问所有的非特殊文件的对象。你不能使用它访问Template对象，只能访问其他文件。
  - Files.Get 通过文件名获取文件的方法。 （.Files.Getconfig.ini）
  - Files.GetBytes 用字节数组代替字符串获取文件内容的方法。 对图片之类的文件很有用
  - Files.Glob 用给定的shell glob模式匹配文件名返回文件列表的方法
  - Files.Lines 逐行读取文件内容的方法。迭代文件中每一行时很有用
  - Files.AsSecrets 使用Base 64编码字符串返回文件体的方法
  - Files.AsConfig 使用YAML格式返回文件体的方法
- Capabilities： 提供关于Kubernetes集群支持功能的信息
  - Capabilities.APIVersions 是一个版本列表
  - Capabilities.APIVersions.Has $version 说明集群中的版本 (比如,batch/v1) 或是资源 (比如, apps/v1/Deployment) 是否可用
  - Capabilities.KubeVersion 和Capabilities.KubeVersion.Version 是Kubernetes的版本号
  - Capabilities.KubeVersion.Major Kubernetes的主版本
  - Capabilities.KubeVersion.Minor Kubernetes的次版本
  - Capabilities.HelmVersion 包含Helm版本详细信息的对象，和 helm version 的输出一致
  - Capabilities.HelmVersion.Version 是当前Helm语义格式的版本
  - Capabilities.HelmVersion.GitCommit Helm的git sha1值
  - Capabilities.HelmVersion.GitTreeState 是Helm git树的状态
  - Capabilities.HelmVersion.GoVersion 是使用的Go编译器版本
- Template： 包含当前被执行的当前模板信息
  - Template.Name: 当前模板的命名空间文件路径 (e.g. mychart/templates/mytemplate.yaml)
  - Template.BasePath: 当前chart模板目录的路径 (e.g. mychart/templates)

内置的值都是以大写字母开始。 这是符合Go的命名惯例。

# 三、Values 文件

其内容来自于多个位置：

- chart中的`values.yaml`文件
- 如果是子chart，就是父chart中的`values.yaml`文件
- 使用`-f`参数(`helm install -f myvals.yaml ./mychart`)传递到 `helm install` 或 `helm upgrade`的values文件
- 使用`--set` (比如`helm install --set foo=bar ./mychart`)传递的单个参数

以上列表有明确顺序：默认使用`values.yaml`，可以被父chart的`values.yaml`覆盖，继而被用户提供values文件覆盖， 最后会被`--set`参数覆盖，优先级为`values.yaml`最低，`--set`参数最高。

删除`values.yaml`中的默认内容，仅设置一个参数：

```
favoriteDrink: coffee
```

在模板中使用它：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

渲染结果：

```
$ helm install geared-marsupi ./mychart --dry-run --debug
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: geared-marsupi-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

调用`helm install`时设置`--set`，很容易就能覆盖这个值。

```
$ helm install solid-vulture ./mychart --dry-run --debug --set favoriteDrink=slurm
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: slurm
```

由于`--set`比默认的`values.yaml`文件优先级更高，模板就生成了`drink: slurm`。

values文件更多结构化的内容：

```
favorite:
  drink: coffee
  food: pizza
```

现在需要稍微修改一些模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

建议构建更加平坦的浅层树。以后想要给子chart赋值时，会看到如何使用树结构给value命名。

**删除默认的key**

如果需要从默认的value中删除key，可以将key设置为`null`，Helm将在覆盖的value合并时删除这个key。

比如，稳定的Drupal允许在配置自定义镜像时配置活动探针。默认值为`httpget`：

```
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

如果你想替换掉`httpGet`用`exec`重写活动探针，使用`--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt]`， Helm会把默认的key和重写的key合并在一起，从而生成以下YAML：

```
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  exec:
    command:
    - cat
    - docroot/CHANGELOG.txt
  initialDelaySeconds: 120
```

因为Kubernetes中不能声明多个活动探针句柄，从而会应用发布会失败。为了解决这个问题，Helm可以指定通过设定null来删除`livenessProbe.httpGet`：

```
helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

# 四、模板函数和流水线

## `quote`函数 

可以通过调用模板指令中的`quote`函数把`.Values`对象中的字符串属性用引号引起来，然后放到模板中。 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

模板函数的语法是 `functionName arg1 arg2...`。在上面的代码片段中，`quote .Values.favorite.drink`调用了`quote`函数并传递了一个参数(`.Values.favorite.drink`)。 

## 管道符

管道符是将一系列的模板语言紧凑地将多个流式处理结果合并的工具。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

在这个示例中，并不是调用`quote 参数`，而是倒置了命令。使用管道符(`|`)将参数“发送”给函数： `.Values.favorite.drink | quote`。使用管道符可以将很多函数链接在一起：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

运行结果：

```
[root@k8s-master01 mychart]# helm install solid-vulture ../mychart --dry-run --debug
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

第一个表达式的结果(`.Values.favorite.drink | upper` 的结果) 作为了`quote`的最后一个参数。 



`repeat COUNT STRING`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

`repeat`函数会返回给定参数特定的次数，则可以得到以下结果：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: melting-porcup-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

## 使用`default`函数

模板中频繁使用的一个函数是`default`： `default DEFAULT_VALUE GIVEN_VALUE`。 这个函数允许你在模板中指定一个默认值，以防这个值被忽略。

```
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

如果运行，会得到 `coffee`:

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: virtuous-mink-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

现在，从`values.yaml`中移除设置：

```
favorite:
  #drink: coffee
  food: pizza
```

现在重新运行 `helm install --dry-run --debug fair-worm ./mychart` 会生成如下内容：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

在实际的chart中，所有的静态默认值应该设置在 `values.yaml` 文件中，且不应该重复使用 `default` 命令 (否则会出现冗余)。然而这个`default` 命令很适合计算值，其不能声明在`values.yaml`文件中，比如：

```
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```

## 使用`lookup`函数

`lookup` 函数可以用于在运行的集群中 *查找* 资源。lookup函数简述为查找 `apiVersion, kind, namespace,name -> 资源或者资源列表`。

| parameter  | type   |
| ---------- | ------ |
| apiVersion | string |
| kind       | string |
| namespace  | string |
| name       | string |

`name` 和 `namespace` 都是选填的，且可以传空字符串(`""`)作为空。

以下是可能的参数组合：

| 命令                                   | Lookup 函数                                |
| -------------------------------------- | ------------------------------------------ |
| `kubectl get pod mypod -n mynamespace` | `lookup "v1" "Pod" "mynamespace" "mypod"`  |
| `kubectl get pods -n mynamespace`      | `lookup "v1" "Pod" "mynamespace" ""`       |
| `kubectl get pods --all-namespaces`    | `lookup "v1" "Pod" "" ""`                  |
| `kubectl get namespace mynamespace`    | `lookup "v1" "Namespace" "" "mynamespace"` |
| `kubectl get namespaces`               | `lookup "v1" "Namespace" "" ""`            |

当`lookup`返回一个对象，它会返回一个字典。这个字典可以进一步被引导以获取特定值。

下面的例子将返回`mynamespace`对象的annotations属性：

```
(lookup "v1" "Namespace" "" "mynamespace").metadata.annotations
```

当`lookup`返回一个对象列表时，可以通过`items`字段访问对象列表：

```
{{ range $index, $service := (lookup "v1" "Service" "mynamespace" "").items }}
    {{/* do something with each service */}}
{{ end }}
```

当对象未找到时，会返回空值。可以用来检测对象是否存在。

`lookup`函数使用Helm已有的Kubernetes连接配置查询Kubernetes。当与调用API服务交互时返回了错误 （比如缺少资源访问的权限），helm 的模板操作会失败。

请记住，Helm在`helm template`或者`helm install|update|delete|rollback --dry-run`时， 不应该请求Kubernetes API服务。由此，`lookup`函数在该案例中会返回空列表（即字典）。

## 运算符也是函数

对于模板来说，运算符(`eq`, `ne`, `lt`, `gt`, `and`, `or`等等) 都是作为函数来实现的。 在管道符中，操作可以按照圆括号分组。

现在我们可以从函数和管道符返回到条件控制流，循环和范围修饰符。

# 五、模板函数

[Helm | 模板函数列表](https://helm.sh/zh/docs/chart_template_guide/function_list/) 

# 六、流控制

Helm的模板语言提供了以下控制结构：

- `if`/`else`， 用来创建条件语句
- `with`， 用来指定范围
- `range`， 提供"for each"类型的循环

除了这些之外，还提供了一些声明和使用命名模板的关键字：

- `define` 在模板中声明一个新的命名模板
- `template` 导入一个命名模板
- `block` 声明一种特殊的可填充的模板块

## If/Else

基本的条件结构看起来像这样：

```
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

如果是以下值时，管道会被设置为 *false*：

- 布尔false
- 数字0
- 空字符串
- `nil` (空或null)
- 空集合(`map`, `slice`, `tuple`, `dict`, `array`)

在所有其他条件下，条件都为true。

让我们先在配置映射中添加一个简单的条件。如果饮品是coffee会添加另一个配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}mug: "true"{{ end }}
```

结果：

```
[root@k8s-master01 mychart]# helm install solid-vulture ../mychart --dry-run --debug
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: "true"
```

在values.yaml文件中注释favorite.drink

```
root@k8s-master01 mychart]# helm install solid-vulture ../mychart --dry-run --debug
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

## 控制空格

查看条件时，我们需要快速了解一下模板中控制空白的方式，格式化之前的例子，使其更易于阅读：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
    mug: "true"
  {{ end }}
```

初始情况下，看起来没问题。但是如果通过模板引擎运行时，我们将得到一个不幸的结果：

```
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/helm.sh/helm/_scratch/mychart
Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key
```

发生了啥？因为空格导致生成了错误的YAML。

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
    mug: "true"
```

`mug`的缩进是不对的。取消缩进重新执行一下：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
  mug: "true"
  {{ end }}
```

这个就得到了合法的YAML，但是看起来还是有点滑稽：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telling-chimp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: "true"
```

注意在YAML中有一个空行，为什么？当模板引擎运行时，它 *移除了* `{{` 和 `}}` 里面的内容，但是留下的空白完全保持原样。

YAML认为空白是有意义的，因此管理空白变得很重要。幸运的是，Helm模板有些工具可以处理此类问题。

首先，模板声明的大括号语法可以通过特殊的字符修改，并通知模板引擎取消空白。`{{- `(包括添加的横杠和空格)表示向左删除空白， 而` -}}`表示右边的空格应该被去掉。 *一定注意空格就是换行*

> 要确保`-`和其他命令之间有一个空格。 `{{- 3 }}` 表示“删除左边空格并打印3”，而`{{-3 }}`表示“打印-3”。

使用这个语法，我们就可修改我们的模板，去掉新加的空白行：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" }}
  mug: "true"
  {{- end }}
```

注意这个删除字符的更改，很容易意外地出现情况：

```
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" -}}
  mug: "true"
  {{- end -}}
```

这样会变成`food: "PIZZA"mug:"true"`，因为这把两边的新行都删除了。

最终，有时这更容易告诉模板系统如何缩进，而不是试图控制模板指令间的间距。因此，您有时会发现使用`indent`方法(`{{ indent 2 "mug:true" }}`)会很有用。

## 修改使用`with`的范围

`with`的语法与`if`语句类似：

```
{{ with PIPELINE }}
  # restricted scope
{{ end }}
```

作用域可以被改变。`with`允许你为特定对象设定当前作用域(`.`)。比如，我们已经在使用`.Values.favorite`。 修改配置映射中的`.`的作用域指向`.Values.favorite`：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

 注意现在可以引用`.drink`和`.food`了，而不必限定他们。因为`with`语句设置了`.`指向`.Values.favorite`。 `.`被重置为`{{ end }}`之后的上一个作用域。

但是这里有个注意事项，在限定的作用域内，无法使用`.`访问父作用域的对象。错误示例如下：

```
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ .Release.Name }}
  {{- end }}
```

这样会报错因为`Release.Name`不在`.`限定的作用域内。但是如果对调最后两行就是正常的， 因为在`{{ end }}`之后作用域被重置了。

```
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  release: {{ .Release.Name }}
```

或者，我们可以使用`$`从父作用域中访问`Release.Name`对象。当模板开始执行后`$`会被映射到根作用域，且执行过程中不会更改。 下面这种方式也可以正常工作：

```
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $.Release.Name }}
  {{- end }}
```

## 使用`range`操作循环

在Helm的模板语言中，在一个集合中迭代的方式是使用`range`操作符。

开始之前，我们先在`values.yaml`文件添加一个披萨的配料列表：

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

现在我们有了一个`pizzaToppings`列表（模板中称为切片）。修改模板把这个列表打印到配置映射中：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}    
```

我可以使用`$`从父作用域访问`Values.pizzaToppings`列表。当模板开始执行后`$`会被映射到根作用域， 且执行过程中不会更改。下面这种方式也可以正常工作：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  toppings: |-
    {{- range $.Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}    
  {{- end }}
```

让我们仔细看看`toppings:`列表。`range`方法“涵盖”（迭代）`pizzaToppings`列表。但现在发生了有意思的事情。 就像`with`设置了`.`的作用域，`range`操作符也做了同样的事。每一次循环，`.`都会设置为当前的披萨配料。 也就是说，第一次`.`设置成了`mushrooms`，第二次迭代设置成了`cheese`，等等。

我们可以直接发送`.`的值给管道，因此当我们执行`{{ . | title | quote }}`时，它会发送`.`到`title`然后发送到`quote`。 如果执行这个模板，输出是这样的：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-dragonfly-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"    
```

现在，我们已经处理了一些棘手的事情。`toppings: |-`行是声明的多行字符串。所以这个配料列表实际上不是YAML列表， 是个大字符串。为什么要这样做？因为在配置映射`data`中的数据是由键值对组成，key和value都是简单的字符串。 

> 正如例子中所示，`|-`标识在YAML中是指多行字符串。这在清单列表中嵌入大块数据是很有用的技术。

有时能在模板中快速创建列表然后迭代很有用，Helm模板的`tuple`可以很容易实现该功能。在计算机科学中， 元组表示一个有固定大小的类似列表的集合，但可以是任意数据类型。这大致表达了`tuple`的用法。

```
  sizes: |-
    {{- range tuple "small" "medium" "large" }}
    - {{ . }}
    {{- end }}    
```

上述模板会生成以下内容：

```
  sizes: |-
    - small
    - medium
    - large    
```

除了列表和元组，`range`可被用于迭代有键值对的集合（像`map`或`dict`）。我们会在下一部分介绍模板变量是看到它是如何应用的。

# 七、变量

在`with`块开始之前，赋值`$relname := .Release.Name`。 现在在`with`块中，`$relname`变量仍会执行版本名称。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

运行之后会生成以下内容：

```
[root@k8s-master01 mychart]# helm install solid-vulture ../mychart --dry-run --debug
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  release: solid-vulture
```

变量在`range`循环中特别有用。可以用于类似列表的对象，以捕获索引和值：

```
  toppings: |-
    {{- range $index, $topping := .Values.pizzaToppings }}
      {{ $index }}: {{ $topping }}
    {{- end }}    
```

注意先是`range`，然后是变量，然后是赋值运算符，然后是列表。会将整型索引（从0开始）赋值给`$index`并将值赋值给`$topping`。 执行会生成：

```
  toppings: |-
      0: mushrooms
      1: cheese
      2: peppers
      3: onions      
```

对于数据结构有key和value，可以使用`range`获取key和value。比如，可以通过`.Values.favorite`进行循环：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

第一次迭代，`$key`会是`drink`且`$val`会是`coffee`，第二次迭代`$key`会是`food`且`$val`会是`pizza`。 运行之后会生成：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eager-rabbit-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

变量一般不是"全局的"。作用域是其声明所在的块。上面我们在模板的顶层赋值了`$relname`。变量的作用域会是整个模板。 但在最后一个例子中`$key`和`$val`作用域会在`{{ range... }}{{ end }}`块内。

但有个变量一直是全局的 - `$` - 这个变量一直是指向根的上下文。当在一个范围内循环时会很有用，同时你要知道chart的版本名称。

# 八、命名模板

**模板名称是全局的**。如果您想声明两个相同名称的模板，哪个最后加载就使用哪个。 因为在子chart中的模板和顶层模板一起编译，命名时要注意 *chart特定名称*。

常见命名惯例是用chart名称作为模板前缀：`{{ define "mychart.labels" }}`。使用特定chart名称作为前缀可以避免可能因为 两个不同chart使用了相同名称的模板而引起的冲突。

## 局部的和`_`文件

在编写模板细节之前，文件的命名惯例需要注意：

- `templates/`中的大多数文件被视为包含Kubernetes清单
- `NOTES.txt`是个例外
- 命名以下划线(`_`)开始的文件则假定 *没有* 包含清单内容。这些文件不会渲染为Kubernetes对象定义，但在其他chart模板中都可用。

这些文件用来存储局部和辅助对象，实际上当我们第一次创建`mychart`时，会看到一个名为`_helpers.tpl`的文件，这个文件是模板局部的默认位置。

## 用`define`和`template`声明和使用模板

`define`操作允许我们在模板文件中创建一个命名模板，语法如下：

```
{{- define "MY.NAME" }}
  # body of template here
{{- end }}
```

比如我们可以定义一个模板封装Kubernetes的标签：

```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

现在我们将模板嵌入到了已有的配置映射中，然后使用`template`包含进来：

```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

当模板引擎读取该文件时，它会存储`mychart.labels`的引用直到`template "mychart.labels"`被调用。 然后会按行渲染模板，因此结果类似这样：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: running-panda-configmap
  labels:
    generator: helm
    date: 2016-11-02
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

注意：`define`不会有输出，除非像本示例一样用模板调用它。

按照惯例，Helm chart将这些模板放置在局部文件中，一般是`_helpers.tpl`。把这个方法移到那里：

```
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

按照惯例`define`方法会有个简单的文档块(`{{/* ... */}}`)来描述要做的事。

尽管这个定义是在`_helpers.tpl`中，但它仍能访问`configmap.yaml`：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

如上所述，**模板名称是全局的**。因此，如果两个模板使用相同名字声明，会使用最后出现的那个。由于子chart中的模板和顶层模板一起编译， 最好用 *chart特定名称* 命名你的模板。常用的命名规则是用chart的名字作为模板的前缀： `{{ define "mychart.labels" }}`。

## 设置模板范围

在上面定义的模板中，我们没有使用任何对象，仅仅使用了方法。修改定义好的模板让其包含chart名称和版本号：

```
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
```

如果渲染这个，会得到以下错误：

```
$ helm install --dry-run moldy-jaguar ./mychart
Error: unable to build kubernetes objects from release manifest: error validating "": error validating data: [unknown object type "nil" in ConfigMap.metadata.labels.chart, unknown object type "nil" in ConfigMap.metadata.labels.version]
```

要查看渲染了什么，可以用`--disable-openapi-validation`参数重新执行： `helm install --dry-run --disable-openapi-validation moldy-jaguar ./mychart`。 结果并不是我们想要的：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moldy-jaguar-configmap
  labels:
    generator: helm
    date: 2021-03-06
    chart:
    version:
```

名字和版本号怎么了？没有出现在我们定义的模板中。当一个（使用`define`创建的）命名模板被渲染时，会接收被`template`调用传入的内容。 在我们的示例中，包含模板如下：

```
{{- template "mychart.labels" }}
```

没有内容传入，所以模板中无法用`.`访问任何内容。但这个很容易解决，只需要传递一个范围给模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
```

注意这个在`template`调用末尾传入的`.`，我们可以简单传入`.Values`或`.Values.favorite`或其他需要的范围。但一定要是顶层范围。

现在我们可以用`helm install --dry-run --debug plinking-anaco ./mychart`执行模板，然后得到：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plinking-anaco-configmap
  labels:
    generator: helm
    date: 2021-03-06
    chart: mychart
    version: 0.1.0
```

现在`{{ .Chart.Name }}`解析为`mychart`，`{{ .Chart.Version }}`解析为`0.1.0`。

## `include`方法

假设定义了一个简单模板如下：

```
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}"
{{- end -}}
```

现在假设我想把这个插入到模板的`labels:`部分和`data:`部分：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart.app" . }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ template "mychart.app" . }}
```

如果渲染这个，会得到以下错误：

```
$ helm install --dry-run measly-whippet ./mychart
Error: unable to build kubernetes objects from release manifest: error validating "": error validating data: [ValidationError(ConfigMap): unknown field "app_name" in io.k8s.api.core.v1.ConfigMap, ValidationError(ConfigMap): unknown field "app_version" in io.k8s.api.core.v1.ConfigMap]
```

要查看渲染了什么，可以用`--disable-openapi-validation`参数重新执行： `helm install --dry-run --disable-openapi-validation measly-whippet ./mychart`。 输入不是我们想要的：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: measly-whippet-configmap
  labels:
    app_name: mychart
app_version: "0.1.0"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
app_name: mychart
app_version: "0.1.0"
```

注意两处的`app_version`缩进都不对，为啥？因为被替换的模板中文本是左对齐的。由于`template`是一个行为，不是方法，无法将 `template`调用的输出传给其他方法，数据只是简单地按行插入。

为了处理这个问题，Helm提供了一个`template`的可选项，可以将模板内容导入当前管道，然后传递给管道中的其他方法。

下面这个示例，使用`indent`正确地缩进了`mychart.app`模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.app" . | indent 2 }}
```

现在生成的YAML每一部分都可以正确缩进了：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-mole-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0"
```

# 九、在模板内部访问文件

Helm 提供了通过.Files对象访问文件的方法。不过，在我们使用模板示例之前，有些事情需要注意：

可以添加额外的文件到chart中。虽然这些文件会被绑定。但是要小心，由于Kubernetes对象的限制，Chart必须小于1M。
通常处于安全考虑，一些文件无法通过.Files对象访问：

- 无法访问templates/中的文件
- 无法访问使用.helmignore排除的文件

**Basic example**

直接放到mychart/myexample目录中。

`config1.toml`:

```
message = Hello from config 1
```

`config2.toml`:

```
message = This is config 2
```

`config3.toml`:

```
message = Goodbye from config 3
```

使用`range`功能遍历它们并将它们的内容注入到我们的ConfigMap中。 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- $files := .Files }}
  {{- range tuple "myexample/config1.tom" "myexample/config2.tom" "myexample/config3.tom" }}
  {{ . }}: |-
        {{ $files.Get . }}
  {{- end }}
```

结果：

```
[root@k8s-master01 mychart]# helm install solid-vulture ../mychart --dry-run --debug                      
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myexample/config1.tom: |-
        message = Hello from config 1

  myexample/config2.tom: |-
        message = This is config 2

  myexample/config3.tom: |-
        message = This is config 3
```

## Glob patterns

当你的chart不断变大时，你会发现你强烈需要组织你的文件，所以我们提供了一个 `Files.Glob(pattern string)`方法来使用 [全局模式](https://godoc.org/github.com/gobwas/glob)的灵活性读取特定文件。

`.Glob`返回一个`Files`类型，因此你可以在返回对象上调用任意的`Files`方法。

比如，假设有这样的目录结构：

```
foo/:
  foo.txt foo.yaml
bar/:
  bar.go bar.conf baz.yaml
```

全局模式下您有多种选择：

```
{{ $currentScope := .}}
{{ range $path, $_ :=  .Files.Glob  "**.yaml" }}
    {{- with $currentScope}}
        {{ .Files.Get $path }}
    {{- end }}
{{ end }}
```

Or

```
{{ range $path, $_ :=  .Files.Glob  "**.yaml" }}
      {{ $.Files.Get $path }}
{{ end }}
```

## ConfigMap and Secrets utility functions

（在Helm 2.0.2及后续版本可用）

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: conf
data:
{{ (.Files.Glob "foo/*").AsConfig | indent 2 }}
---
apiVersion: v1
kind: Secret
metadata:
  name: very-secret
type: Opaque
data:
{{ (.Files.Glob "bar/*").AsSecrets | indent 2 }}
```

## Encoding

您可以导入一个文件并使用模板的base-64方式对其进行编码来保证成功传输：

```
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  token: |-
        {{ .Files.Get "config1.toml" | b64enc }}
```

上面的内容使用我们之前使用的相同的`config1.toml`文件进行编码：

```
# Source: mychart/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lucky-turkey-secret
type: Opaque
data:
  token: |-
        bWVzc2FnZSA9IEhlbGxvIGZyb20gY29uZmlnIDEK
```

## Lines

有时需要访问模板中的文件的每一行。我们提供了一个方便的`Lines`方法。

你可以使用`range`方法遍历`Lines`：

```
data:
  some-file.txt: {{ range .Files.Lines "foo/bar.txt" }}
    {{ . }}{{ end }}
```

在`helm install`过程中无法将文件传递到chart外。因此如果你想请求用户提供数据，必须使用`helm install -f`或`helm install --set`加载。

# 十、创建一个NOTES.txt文件

该部分会介绍为chart用户提供说明的Helm工具。在`helm install` 或 `helm upgrade`命令的最后，Helm会打印出对用户有用的信息。 使用模板可以高度自定义这部分信息。

要在chart添加安装说明，只需创建`templates/NOTES.txt`文件即可。该文件是纯文本，但会像模板一样处理， 所有正常的模板函数和对象都是可用的。

让我们创建一个简单的`NOTES.txt`文件：

```
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To learn more about the release, try:

  $ helm status {{ .Release.Name }}
  $ helm get all {{ .Release.Name }}

```

执行会在底部看到：

```
[root@k8s-master01 mychart]# helm install solid-vulture ../mychart --dry-run --debug
NOTES:
Thank you for installing mychart.

Your release is named solid-vulture.

To learn more about the release, try:

  $ helm status solid-vulture
  $ helm get all solid-vulture
```

使用`NOTES.txt`这种方式是给用户提供关于如何使用新安装的chart细节信息的好方法。尽管并不是必需的，强烈建议创建一个`NOTES.txt`文件。

# 十一、子chart和全局值

在深入研究代码之前，需要了解一些子chart的重要细节：

1. 子chart被认为是“独立的”，意味着子chart从来不会显示依赖它的父chart。
2. 因此，子chart无法访问父chart的值。
3. 父chart可以覆盖子chart的值。
4. Helm有一个 *全局值* 的概念，所有的chart都可以访问。

## 创建子chart

为了做这些练习，我们可以从本指南开始时创建的`mychart/`开始，并在其中添加一个新的chart。

```
$ cd mychart/charts
$ helm create mysubchart
Creating mysubchart
$ rm -rf mysubchart/templates/*
```

## 在子chart中添加值和模板

下一步，为`mysubchart`创建一个简单的模板和values文件。`mychart/charts/mysubchart`应该已经有一个`values.yaml`。 设置如下：

```
dessert: cake

```

下一步，在`mychart/charts/mysubchart/templates/configmap.yaml`中创建一个新的配置映射模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
```

因为每个子chart都是 *独立的chart*，可以单独测试`mysubchart`：

```
[root@k8s-master01 charts]# helm install mysubchart  mysubchart --dry-run --debug 
MANIFEST:
---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysubchart-cfgmap2
data:
  dessert: cake
```

父chart测试mychart

```
[root@k8s-master01 charts]# helm install mychart  ../../mychart --dry-run --debug 
MANIFEST:
---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysubchart-cfgmap2
data:
  dessert: cake
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysubchart-configmap
data:
  myexample/config1.tom: |-
```

## 用父chart的值来覆盖

原始chart，`mychart`现在是`mysubchart`的 *父*。这种关系是基于`mysubchart`在`mychart/charts`中这一事实。

因为`mychart`是父级，可以在`mychart`指定配置并将配置推送到`mysubchart`。比如可以修改`mychart/values.yaml`如下：

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream
```

注意最后两行，在`mysubchart`中的所有指令会被发送到`mysubchart`chart中。因此如果运行`helm install --dry-run --debug mychart`，会看到一项`mysubchart`的配置：

```
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: unhinged-bee-cfgmap2
data:
  dessert: ice cream
```

现在，子chart的值已经被顶层的值覆盖了。

这里需要注意个重要细节。我们不会改变`mychart/charts/mysubchart/templates/configmap.yaml`模板到 `.Values.mysubchart.dessert`的指向。从模板的角度来看，值依然是在`.Values.dessert`。当模板引擎传递值时，会设置范围。 因此对于`mysubchart`模板，`.Values`中只提供专门用于`mysubchart`的值。

但是有时确实希望某些值对所有模板都可用。这是使用全局chart值完成的。

## 全局Chart值

全局值是使用完全一样的名字在所有的chart及子chart中都能访问的值。全局变量需要显示声明。不能将现有的非全局值作为全局值使用。

这些值数据类型有个保留部分叫`Values.global`，可以用来设置全局值。在`mychart/values.yaml`文件中设置一个值如下：

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream

global:
  salad: caesar
```

因为全局的工作方式，`mychart/templates/configmap.yaml`和`mysubchart/templates/configmap.yaml` 应该都能以`{{ .Values.global.salad }}`进行访问。

`mychart/templates/configmap.yaml`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  salad: {{ .Values.global.salad }}
```

`mysubchart/templates/configmap.yaml`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
  salad: {{ .Values.global.salad }}
```

现在如果预安装，两个输出会看到相同的值：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-configmap
data:
  salad: caesar

---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-cfgmap2
data:
  dessert: ice cream
  salad: caesar
```

全局值在类似这样传递信息时很有用，不过要确保使用全局值配置正确的模板，确实需要一些计划。

## 与子chart共享模板

父chart和子chart可以共享模板。在任意chart中定义的块在其他chart中也是可用的。

比如，我们可以这样定义一个简单的模板：

```
{{- define "labels" }}from: mychart{{ end }}
```

回想一下模板标签时如何 *全局共享的*。因此，`标签`chart可以包含在任何其他chart中。

当chart开发者在`include` 和 `template` 之间选择时，使用`include`的一个优势是`include`可以动态引用模板：

```
{{ include $mytemplate }}
```

上述会取消对`$mytemplate`的引用，相反，`template`函数只接受字符串字符。

## 避免使用块

Go 模板语言提供了一个 `block` 关键字允许开发者提供一个稍后会被重写的默认实现。在Helm chart中， 块并不是用于覆盖的最好工具，因为如果提供了同一个块的多个实现，无法预测哪个会被选定。

建议改为使用`include`。

# 十二、.helmignore 文件

`.helmignore` 文件用来指定你不想包含在你的helm chart中的文件。

如果该文件存在，`helm package` 命令会在打包应用时忽略所有在`.helmignore`文件中匹配的文件。

这有助于避免不需要的或敏感文件及目录添加到你的helm chart中。

`.helmignore` 文件支持Unix shell的全局匹配，相对路径匹配，以及反向匹配（以！作为前缀）。每行只考虑一种模式。

这里是一个`.helmignore`文件示例：

```
.git
*.txt
mydir/
/*.txt
/foo.txt
a[b-d].txt
*/temp*
*/*/temp*
temp?
```

一些值得注意的和.gitignore不同之处：

- 不支持'**'语法。
- globbing库是Go的 'filepath.Match'，不是fnmatch(3)
- 末尾空格总会被忽略(不支持转义序列)
- 不支持'!'作为特殊的引导序列

