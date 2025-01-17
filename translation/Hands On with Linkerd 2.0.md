# [上手Linkerd2.0](<https://kubernetes.io/blog/2018/09/18/hands-on-with-linkerd-2.0/>)

**作者**：Thomas Rampelberg

最近，Linkerd 2.0宣布为普遍可用（GA），表明它已经准备投入生产使用。在本教程中，我们将指导您如何在几秒钟内在Kubernetes集群上启动和运行Linkerd 2.0，如何在几秒钟内利用Kubernetes集群启动和运行Linkerd 2.0。

但是，我们需要首先了解一下，什么是Linkerd 2.0？它值得我们关注吗?Linkerd是一个能够增强Kubernetes服务的服务边车，它可以提供零配置仪表板和UNIX风格的CLI工具，用于运行时调试和诊断以及可靠性的验证。Linkerd也是一个服务网格，应用于集群中的多个（或所有）服务，包括统一的遥测、安全性和控制层。 

Linkerd的工作原理是将超轻型的代理安装到服务的每个pod中。这些代理将遥测数据报告给控制层并从控制层接收信号。这意味着使您用Linkerd时不需要任何代码的更改，您甚至可以在正在运行的服务上实时安装它。Linkerd是完全开源的，它由Cloud Native Computing Foundation托管（就像Kubernetes本身一样！），并且获得了Apache v2许可。

话不多说，让我们来看看在Kubernetes集群上运行Linkerd的速度有多快吧！在本教程中，我们将向您介绍在任何Kubernetes 1.9+集群上部署Linkerd以及如何使用它来调试示例gRPC应用程序中的故障。

## 第1步：安装演示应用程序🚀

在我们安装Linkerd之前，让我们首先在Kubernetes集群上安装一个名为Emojivoto的基本gRPC演示程序。要安装Emojivoto，请运行：

```
curl https://run.linkerd.io/emojivoto.yml | kubectl apply -f -
```

此命令下载Emojivoto的Kubernetes清单，并将kubectl应用于您的Kubernetes集群。Emojivoto由几个在“emojivoto”命名空间中运行的服务组成。您可以通过运行以下代码来查看服务：

```
kubectl get -n emojivoto deployments
```

您还可以通过运行一下代码查看应用程序

```
minikube -n emojivoto service web-svc --url # if you’re on minikube
```

… 或者：

```
kubectl get svc web-svc -n emojivoto -o jsonpath="{.status.loadBalancer.ingress[0].*}" #
```

......如果你在别的地方

点击左右。您可能会注意到这个程序的某些部分其实已经损坏！如果您要检查您的手工本地Kubernetes仪表板，这是令人厌烦的 - 对于Kubernetes而言，显示程序运行正常非常普遍。因为Kubernetes能够检测您的pod是否正在运行，但无法确定它们能否正常响应。

下面，我们将向您介绍如何使用Linkerd来诊断问题。

## 第2步：安装Linkerd的CLI

首先，我们将Linkerd的命令行界面（CLI）安装到本地计算机上。访问[Linkerd发布页面](https://github.com/linkerd/linkerd2/releases/)，或者运行：

```
curl -sL https://run.linkerd.io/install | sh
```

安装后，将`linkerd`命令添加到路径中：

```
export PATH=$PATH:$HOME/.linkerd2/bin
```

您现在应该能够运行该命令`linkerd version`，该命令应显示：

```
Client version: v2.0
Server version: unavailable
```

“服务器版本：不可用”意味着我们需要将Linkerd的控制平面添加到集群，我们接下来会这样做。不过先让我们运行以下命令验证您的群集是否为Linkerd做好了准备：

```
linkerd check --pre
```

这个方便的命令将报告任何会影响您安装Linkerd的问题。希望一切都没问题，你已经准备好继续下一步了。

## 步骤3：将Linkerd的控制平面安装到群集上

在这一步中，我们将Linkerd的轻量级控制平面安装到集群中自己的命名空间（“linkerd”）中。请运行：

```
linkerd install | kubectl apply -f -
```

此命令生成Kubernetes清单并使用`kubectl`命令将其应用于Kubernetes集群。（在申请之前，请检查清单。）

（注意：如果您的Kubernetes群集在启用了RBAC的GKE上，则需要额外的步骤：您必须首先向您的Google Cloud帐户授予ClusterRole群集管理员权限，以便在控制平面中安装某些遥测功能。为此，运行：`kubectl create clusterrolebinding cluster-admin-binding-$USER --clusterrole=cluster-admin --user=$(gcloud config get-value account)`。）

根据您的互联网连接速度，Kubernetes群集可能需要一两分钟才能获取Linkerd图像。在发生这种情况时，我们可以通过运行来验证是否存在疏漏：

```
linkerd check
```

此命令将耐心等待Linkerd安装并运行。

最后，我们准备好查看Linkerd的仪表板了！运行：

```
linkerd dashboard
```

如果您看到类似下面的内容，则Linkerd现在正在您的群集上运行。🎉



![0722](https://d33wubrfki0l68.cloudfront.net/0b1a1f9993f9ccbc091285011600b2a63e0ef8fe/3eb1d/images/blog/2018-09-18-2018-linkerd-2.0/1-dashboard.png)



## 第4步：将Linkerd添加到Web服务

此时，我们在“linkerd”命名空间中安装了Linkerd控制平面，并且在“emojivoto”命名空间中安装了我们的emojivoto演示应用程序。但我们还没有将Linkerd添加到我们的服务中。所以，让我们这样做：

在这个例子中，让我们假装我们是“网络”服务的所有者。其他服务，如“表情符号”和“投票”，属于其他团队 - 我们尽可能不触及它们。

将Linkerd添加到我们的服务中有几种方法。这里我们使用最简单的方法，具体的操作如：

```
kubectl get -n emojivoto deploy/web -o yaml | linkerd inject - | kubectl apply -f -
```

此命令从Kubernetes检索“web”服务的清单，运行此清单`linkerd inject`，最后将其重新应用于Kubernetes集群。该`linkerd inject`命令将清单扩充为包含Linkerd的数据平面代理。`linkerd install`，`linkerd inject`是纯文本操作，这意味着您可以检查之前的输入和输出正确与否。由于“web”是一个部署，Kubernetes可以足够缓慢地将服务推送到一个平台 - 这意味着当我们将Linkerd添加到它时，“web”可以提供流量服务！

我们现在拥有一个服务边车在“网络”服务上运行了！

## 第5步：调试fun和for Profit

恭喜！您现在通过在“Web”服务上安装了Linkerd，在Kubernetes集群上成功运行了一个完整的gRPC应用程序。当然，当你使用它时，应用程序出现了问题- 所以现在让我们使用Linkerd来追踪这些错误。

如果您浏览Linkerd仪表板（`linkerd dashboard`命令），您应该会看到“emojivoto”命名空间中的所有服务都显示出来。由于“web”上安装了Linkerd服务边车，您还会看到成功率，每秒请求数和延迟百分位数。



![0722](https://d33wubrfki0l68.cloudfront.net/4568f67a862883f5edc6eab23e17fc013c161c73/49354/images/blog/2018-09-18-2018-linkerd-2.0/2-web-overview.png)



这很漂亮，但你可能首先注意到的是成功率远低于100％！点击“网络”，让我们深入挖掘。

您现在应该查看Web服务的“部署”页面。您将在这里看到的第一件事是Web正在从投票机器人（Emojivoto清单中包含的服务，以持续生成低水平的实时流量）中获取流量，并且具有两个外发依赖项，表情符号和投票。



![0722](https://d33wubrfki0l68.cloudfront.net/230de289f5759d221f1423a8fd3cf6cf4210fb4a/91c8f/images/blog/2018-09-18-2018-linkerd-2.0/3-web-detail.png)



表情符号服务的运行率为100％，但投票服务失败！Web返回的错误的原因可能正是依赖服务中的故障造成的。

让我们在页面上滚动一点，我们将看到“web”正在接收的所有流量端点的实时列表。有意思的是：



![0722](https://d33wubrfki0l68.cloudfront.net/5cb45bea9fea0cbeda4b09e4d8be1bb30391cff7/085c8/images/blog/2018-09-18-2018-linkerd-2.0/4-web-top.png)



有两个不是100％的调用：第一个是投票机器人调用“/ api / vote”端点。第二个是从Web服务到投票服务的“VotePoop”调用。很有意思！由于/ api / vote是一个来电，而“/ VotePoop”是一个外拨电话。这是一个很好的线索，投票服务的VotePoop端点失败是造成问题的原因！

最后，如果我们点击最右侧列表中的“点击”图标，我们将会看到与该端点匹配的实时列表。这使得我们确认请求失败（它们都具有[gRPC状态代码2](https://godoc.org/google.golang.org/grpc/codes#Code)，表示错误）。



![0722](https://d33wubrfki0l68.cloudfront.net/83f4729b0db4a9e4f15bebc3ccf3a50c15d1db49/5b199/images/blog/2018-09-18-2018-linkerd-2.0/5-web-tap.png)



在这一点上，我们需要与“投票”服务的所有者交谈的依据。我们已经在他们的服务上确定了一个始终返回错误的端点，并且在系统中没有发现其他明显的故障源。

我们希望您能享受通过Linkerd 2.0发现问题的这段旅程。还有很多值得探索的地。例如，我们在上面使用Web UI所做的一切，也可以通过纯粹的CLI命令，如完成`linkerd top`，`linkerd stat`和`linkerd tap`。

另外，你注意到我们看到的第一页上的小Grafana图标了吗？Linkerd为所有这些指标提供了自动Grafana仪表板，允许您以时间序列格式查看您在Linkerd仪表板中看到的所有内容。看看这个！



![0722](https://d33wubrfki0l68.cloudfront.net/02e67f209ddf00822f1af3fb73cfadf59958fcfa/51869/images/blog/2018-09-18-2018-linkerd-2.0/6-grafana.png)



## 意犹未尽？

在本教程中，我们向您展示了如何在群集上安装Linkerd，将其作为服务边车添加到一个服务 - 而服务正在接收实时流量！ - 并使用它来调试运行时问题。但这只是冰山一角，我们甚至没有触及任何Linkerd的可靠性或安全性功能！

Linkerd拥有一个充满活力的使用者和贡献者社区，我们希望你能够成为其中一员。想要了解更多相关信息，请查看[docs](https://linkerd.io/docs)和[GitHub](https://github.com/linkerd/linkerd) repo，加入[Linkerd Slack](https://slack.linkerd.io/)和邮件列表（[用户](https://lists.cncf.io/g/cncf-linkerd-users)，[开发人员](https://lists.cncf.io/g/cncf-linkerd-dev)，[宣布](https://lists.cncf.io/g/cncf-linkerd-announce)）。当然，你也[可以](https://twitter.com/linkerd)在Twitter上关注[@linkerd](https://twitter.com/linkerd)！我们非常期待你的加入！

