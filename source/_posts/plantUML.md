---
layout: title
title: PlantUML使用姿势
date: 2021-11-03 19:11:48
tags:
  - UML
  - tools
categories:
  - 专业
  - 工具
  - 编辑器
  - PlantUML
---

发现一个在线画时序图的工具PlantUML，该工具提供vscode插件，可以使用代码编辑时序图逻辑，并实时预览。

<!-- more -->

### 1. vscode插件安装和使用方法

https://marketplace.visualstudio.com/items?itemName=okazuki.okazukiplantuml

- `PlantUML Preview` : Start PlantUML preview.
- `PlantUML Export ***(*** is format type)` : Export png, svg, eps, etc... to same directory.

### 2. 使用文档

PlantUML提供了丰富的关键词和语法，详见https://plantuml.com/zh/sequence-diagram 下面摘要出日常使用的几个基本功能：

#### 2.1 基本用例

序列`->` 用于绘制两个 参与者之间的信息。 参与者不必明确声明。

要有一个点状的箭头，就用`-->`

也可以用`<-` 和`<--` 。 这不会改变绘图，但可能提高可读性。 注意，这只适用于序列图，其他图的规则不同。

```
@startuml
Alice -> Bob: Authentication Request
Bob --> Alice: Authentication Response

Alice -> Bob: Another authentication Request
Alice <-- Bob: Another authentication Response
@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuNBCoKnELT2rKt3AJx9IS2mjoKZDAybCJYp9pCzJ24ejB4qjBk42oYde0jM05MDHLLoGdrUSoeLkM5u-K5sHGY9sGo6ARNHr2QY66kwGcfS2SZ00



#### 2.2 生命参与者

##### 2.2.1 participant

使用 `participant` 关键字来声明一个参与者可以使你对参与者做出更多控制。

关键字 `participant` 用于改变参与者的先后顺序。

你也可以使用下面这些关键字来声明参与者，这会**改变参与者的外观**：

- `actor`（角色）
- `boundary`（边界）
- `control`（控制）
- `entity`（实体）
- `database`（数据库）
- `collections`（集合）
- `queue`（队列）

```
@startuml
participant participant as Foo
actor       actor       as Foo1
boundary    boundary    as Foo2
control     control     as Foo3
entity      entity      as Foo4
database    database    as Foo5
collections collections as Foo6
queue       queue       as Foo7
Foo -> Foo1 : To actor 
Foo -> Foo2 : To boundary
Foo -> Foo3 : To control
Foo -> Foo4 : To entity
Foo -> Foo5 : To database
Foo -> Foo6 : To collections
Foo -> Foo7 : To queue
@enduml
```

http://www.plantuml.com/plantuml/uml/LP4n3iCW34Ltdu8BT6ZI95A7AbDFq0iuX069uXJCaDitmKdbCi3JVyBYYp4p9Yxl0CjsUkiNZ6mqOpPF8a3Bb8oiFwxw2XELE6DQzqop-0OiHKuKwXtDubjmaJslCbEp-1lCo3XXTdkcMNotpG_1MVOKiz4ileTFSSKTRzOiVe1jCvT7xMBkvaL7IPKtaf_bb6d57BMKk8cGFYhl93zxADEVetuDb1n4rkV3wEAy_ziN

##### 2.2.2 as

关键字 `as` 用于重命名参与者

你还可以使用[RGB](https://plantuml.com/zh/color)值或者颜色名修改 actor 或参与者的背景颜色。

```
@startuml
actor Bob #red
' actor 和 participant 只在外观上有区别
participant Alice
participant "很长很长很长\n的名字" as L #99FF99
/' 也可以这样声明：
   participant L as "很长很长很长\n的名字"  #99FF99
  '/

Alice->Bob: 认证请求
Bob->Alice: 认证响应
Bob->L: 记录事务日志
@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuKfCBialKd3AJr9GBafDuL9NW0WydTIZK01KafcSMP2OLwBmj7_LqpahdYwPzc9vqvCTNS_cT3xjsVMqOpKNiYB7dCpaL1GHfQVxEbvEtOzCnkGzdzNoT4BlqxNJbHGIYnLy59GjBTtSB2svzDKLdkoS_xH__PFTIr_id_bimVQdYpSycz7tViyiBWK55EVuW7GICXnki8A2kZuN5zXrkdP0hrefl5YtvCNwnXVhjp_RsCG55D6r0yl299vExdswQmf4mWSakE7ftgbFTdK_xLhuRFhIf_kdSpcavgM0WWy0

##### 2.2.3 order

您可以使用关键字 `order`自定义顺序来打印参与者。

```
@startuml
participant 最后 order 30
participant 中间 order 20
participant 首个 order 10
@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuIe0qfd9cGM9UIKAp-OqF9tGfv1Vb99Qf61iW2BtPCVQbzEtGEMCKELUBflsPCSgg9oX0PT3QbuAo6m0

#### 2.3 箭头样式

修改箭头样式的方式有以下几种:

- 表示一条丢失的消息：末尾加 `x`
- 让箭头只有上半部分或者下半部分：将`<`和`>`替换成`\`或者 `/`
- 细箭头：将箭头标记写两次 (如 `>>` 或 `//`)
- 虚线箭头：用 `--` 替代 `-`
- 箭头末尾加圈：`->o`
- 双向箭头：`<->`

```
@startuml
Bob ->x Alice
Bob -> Alice
Bob ->> Alice
Bob -\ Alice
Bob \\- Alice
Bob //-- Alice

Bob ->o Alice
Bob o\\-- Alice

Bob <-> Alice
Bob <->o Alice
@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuNBAJrBGjQjGSCp9J4w5yb0uABmO94vCZ2uIJrzV5yQ5Qin7aiq7AaQHja6nnGQXsY4rBmNaBW00

#### 2.4 对消息序列编号

##### 2.4.1 autonumber

关键字 `autonumber` 用于自动对消息编号。

语句 `autonumber //start//` 用于指定编号的初始值，而 `autonumber //start// //increment//` 可以同时指定编号的初始值和每次增加的值。

```
@startuml
autonumber
Bob -> Alice : Authentication Request
Bob <- Alice : Authentication Response

autonumber 15
Bob -> Alice : Another authentication Request
Bob <- Alice : Another authentication Response

autonumber 40 10
Bob -> Alice : Yet another authentication Request
Bob <- Alice : Yet another authentication Response

@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuKeiBSdFAyrDIYtYSifFKj2rKt3CoKnELR1IS2mjoKZDAybCJYp9pCzJ24ejB4qjBW6hij75hQgu83-lE9KBoM05GrCCi_FoWTgA51A9imENQYnscHWe61gWMnUPMgAGI9ALU7N0h7L8pKi1XI40

#### 2.5 分隔示意图

##### 2.5.1 newpage

关键字 `newpage` 用于把一张图分割成多张。

在 `newpage` 之后添加文字，作为新的示意图的标题。

这样就能很方便地在 *Word* 中将长图分几页打印。

```
@startuml

Alice -> Bob : message 1
Alice -> Bob : message 2

newpage

Alice -> Bob : message 3
Alice -> Bob : message 4

newpage A title for the\nlast page

Alice -> Bob : message 5
Alice -> Bob : message 6
@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuU9opCbCJbNGjLDmoazIi5B8JIqkJanFLJ349M74nPMNMbu0PEBKQunBmWIkLy5HeIIp92TL8Is_IA4a8pKcBoUnk4G1hx6ck2JCk1nIyr90lW40

#### 2.6 组合消息

我们可以通过以下关键词来组合消息：

- `alt/else`
- `opt`
- `loop`
- `par`
- `break`
- `critical`
- `group`, 后面紧跟着消息内容

关键词 `end` 用来结束分组。注意，分组可以嵌套使用。

```
@startuml
Alice -> Bob: 认证请求

alt 成功情况

    Bob -> Alice: 认证接受

else 某种失败情况

    Bob -> Alice: 认证失败
    group 我自己的标签
    Alice -> Log : 开始记录攻击日志
        loop 1000次
            Alice -> Bob: DNS 攻击
        end
    Alice -> Log : 结束记录攻击日志
    end

else 另一种失败

   Bob -> Alice: 请重复

end
@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuNBCoKnELT2rKt3AJx9IUB5koOlrZI_MRt-siOcBAp6dHE5PnuIdNVEVDRS-RTlAnQK014258FLWZJ0Tp_gMFksVpiMLcbESgl1i_eJdotkVBjduOijIGXeXgi3IwKNvfGL0-oQ-Q5_rTFl6vxDQdYreVxvs7rWIxaoV_7G5AuMdUngUBkz-iMx3qxrJdqtP_RHzzxFfIv_kdmvM2m8v-Va52eO61WRFrYo42w8O1FQlYr-m0aG_N55gNWes6v_ldlnixdmSDeBqGFp-j7_PanqDSE-3FOxcx_NRNxO3fNk1Ee3Q7804A1u0

#### 2.7 次级分组标签

对于`group`而言，在标头处的`[`和`]`之间可以显示次级文本或标签。

```
@startuml
Alice -> Bob: 认证请求
Bob -> Alice: 认证失败
group 我自己的标签 [我自己的标签2]
    Alice -> Log : 开始记录攻击日志
    loop 1000次
        Alice -> Bob: DNS攻击
    end
    Alice -> Log : 结束记录攻击日志
end
@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuNBCoKnELT2rKt3AJx9IUB5koOlrZI_MRt-siOaBA0AI0Ak0IJrTil75bgLSwKNvfGKAppeclcXVzNJxnkUpMfujQ7--Tfz2DAQOKIoN0X30BVB9JrUmKdYwf-7fykuNwpOytJlrsPJTJzjtFvk-zEd-wM2rEVdv1Gg61WO6pzOi10kW601sgOjVC4GRM3urBmMR9SztJtusTpuMIq3g7O04A0G0

#### 2.8 信息注释

可以使用`note left` 或`note right` 关键字*在信息后面*加上注释。

你可以使用`end note` 关键字有一个多行注释。

```
@startuml
Alice->Bob : hello
note left: this is a first note

Bob->Alice : ok
note right: this is another note

Bob->Bob : I am thinking
note left
a note
can also be defined
on several lines
end note
@enduml
```

http://www.plantuml.com/plantuml/uml/JOz13iCW30JlViL-81_88KfxwpESO0AAOoIWVN-X71h91sizevNNKZdNzwNqqBZBj3pJXXb1L1DPgW8LNsVK40lQC7pCfQAVY1eyBJ-nEUaSGev7k1ij39BlnkXuWQzEsHdj-7SH3tHd0sj9s0HEV3Hnb0n5Ff9PeIqe9EO6lRQjV_45

#### 2.9 分隔符

你可以通过使用`==`关键词来将你的图表分割成多个逻辑步骤。

```
@startuml

== 初始化 ==

Alice -> Bob: 认证请求
Bob --> Alice: 认证响应

== 重复 ==

Alice -> Bob: 认证请求
Alice <-- Bob: 认证响应

@enduml
```

http://www.plantuml.com/plantuml/uml/SoWkIImgAStDuUAojLLusZ7twVBkfptJ56njkRWSSpAJKnLqxHISyfEi55wiM_9YVUEBzTkVRMpY0eeew09bm4fWSaydzpxTDGLiqClstgTBUWcP0f6oqTL5beEPuf2Qbm9o5m00

#### 2.10 生命线的激活与撤销

关键字`activate`和`deactivate`用来表示参与者的生命活动。

一旦参与者被激活，它的生命线就会显示出来。

`activate`和`deactivate`适用于以上情形。

`destroy`表示一个参与者的生命线的终结。

```
@startuml
participant User

User -> A: DoWork
activate A

A -> B: << createRequest >>
activate B

B -> C: DoWork
activate C
C --> B: WorkDone
destroy C

B --> A: RequestCreated
deactivate B

A -> User: Done
deactivate A

@enduml
```

http://www.plantuml.com/plantuml/uml/POzH2iCW38RVSufSe1SeHQhs18EnPz4yo3RjD1tqzZDsiC9U4iZ_vykVkR8hl3qViBOUVLnTOhnMAW1ISL2eHrpoBPSxEC_nxPXG0sYHp8ZJXBvG6rxejL5bLEhdCm16VFOVOS7YS214M78Y26s_vBrfidQS_c9jln6QvGpl8IIdy3lW776c5EIr3m00

### 3. 练手环节

使用vscode PlantUML插件，画出Pod创建过程中，各组件之间的交互逻辑

```
@startuml

actor "用户" as user
participant "External Provisioner" as provisioner
participant "External Attacher" as attacher
participant "CSI插件" as csidriver #lightgreen
participant "API服务器" as apiserver #lightblue
participant "调度器" as scheduler #lightblue
participant "PV控制器" as pvcontroller
participant "AD控制器" as adcontroller
participant "kubelet" as kubelet #lightblue

== 调度 ==
autonumber
activate user
user -> apiserver: 用户创建Pod(with PVC)
activate apiserver
activate scheduler
scheduler -> apiserver: watch Pod，发起调度流程

== Provision == 
activate pvcontroller
pvcontroller -> apiserver: watch PVC
alt 集群有符合需求的PV
    pvcontroller -> apiserver: 绑定PVC&PV
else 集群没有符合需求的PV
    pvcontroller -> apiserver: do nothing
end
activate provisioner
provisioner -> apiserver: watch PVC
provisioner -> csidriver: 调用CSI插件的Provision相关函数
activate csidriver
csidriver -> csidriver:  创建存储卷
csidriver -> provisioner: 创建成功
provisioner -> apiserver: 绑定PVC&PV
scheduler -> apiserver: 绑定Pod与Node，调度结束(Immediate)

== Attach ==
activate adcontroller
adcontroller -> apiserver: watch Pod&PV，发起attach
adcontroller -> apiserver: 创建VolumeAttachment资源
attacher -> apiserver: watch到新的VolumeAttachment资源
attacher -> csidriver: 调用CSI插件的Attach相关函数
csidriver -> csidriver: 将存储卷附着到节点
csidriver -> apiserver: 将VolumeAttachment标记为Attached

== Mount ==
activate kubelet
kubelet -> csidriver: VolumeManager组件调用Mount相关函数
csidriver -> csidriver: 将存储卷挂载到全局目录和Pod目录
csidriver -> kubelet: 挂载成功
kubelet -> kubelet: 启动容器并使用挂载后的存储卷
kubelet -> apiserver: 容器启动成功，Pod Running
apiserver -> user: 用户开心使用挂载了存储卷的业务Pod

@enduml
```

画出来的效果虽然有点复古..但胜在好用且方便

![](/images/pod_plantuml.png)

