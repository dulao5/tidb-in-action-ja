---
title: AWS EKS に TiDB クラスターをデプロイする
category: how-to
---

# 1.2.3.1.1 AWS EKS に TiDB クラスターをデプロイする

ここでは、パソコン( Linux または macOS システム)を使って、AWS EKS(Elastic Kubernetes Service)上に TiDB クラスタをデプロイする方法を説明する。
## 1. 環境の準備

デプロイする前に、下記のソフトウェアがインストールされ、設定されていることを確認してください。：

* [awscli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) >= 1.16.73、 AWS のリソースをコントロールする

    AWSと連携するには、 [`awscli`を設定する](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)必要がある。 最速の方法は、`aws configure`コマンドの`awscli`を使うことである

    ``` shell
    aws configure
    ```

    以下の AWS Access Key ID と AWS Secret Access Key を入れ替えます。

    ```
    AWS Access Key ID [None]: IOSFODNN7EXAMPLE
    AWS Secret Access Key [None]: wJaXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    Default region name [None]: us-west-2
    Default output format [None]: json
    ```

    > **注意：**
    > 
    > アクセスキーには、少なくとも以下の権限が必要です:
    > VPCの作成、EBSの作成、EC2の作成、およびRoleの作成

* [terraform](https://learn.hashicorp.com/terraform/getting-started/install.html) >= 0.12
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl) >= 1.11
* [helm](https://helm.sh/docs/using_helm/#installing-the-helm-client) >= 2.11.0 且 < 3.0.0
* [jq](https://stedolan.github.io/jq/download/)
* [aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)、AWS Privilege Authentication ツールは、 `PATH` パスの下にインストールされていることを確認してください。。

    最も簡単なインストール方法は、以下のようにコンパイルされたバイナリファイル `aws-iam-authenticator` をダウンロードすることである。

    Linux ユーザー：

    ``` shell
    curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.12.7/2019-03-27/bin/linux/amd64/aws-iam-authenticator
    ```

    macOS ユーザー：

    ``` shell
    curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.12.7/2019-03-27/bin/darwin/amd64/aws-iam-authenticator
    ```

    バイナリファイルをダウンロードしたら、次のようにします。：

    ``` shell
    chmod +x ./aws-iam-authenticator && \
    sudo mv ./aws-iam-authenticator /usr/local/bin/aws-iam-authenticator
    ```

## 2. クラスターをデプロイする

デフォルトのデプロイメントでは、新しい VPC 、踏み台としての t2.micro インスタンス、およびワークノードとして次の ec2 インスタンスを含む EKS クラスタが作成されます。

* m5.xlarge X 3台 : PD
* c5d.4xlarge X 3台: TiKV
* c5.4xlarge X 2 : 、TiDB
* c5.2xlarge １台 : ト監視コンポーネント

下記のコマンドでクラスターをデプロイする

Github からコードをクローンして、指定したパスに移動する:

``` shell
git clone --depth=1 https://github.com/pingcap/tidb-operator && \
cd tidb-operator/deploy/aws
```

`terraform` コマンドで初期化して、クラスターをデプロイする：

``` shell
terraform init
```

``` shell
terraform apply
```

> **注意：**
>
> 継続するには、`terraform apply`が実行しているの間に "yes "を入力しなければならない。

全体の処理は最低でも10分はかかる。 terraform apply`の実行が成功すると、コンソールには以下のようなメッセージが出力される。

```
Apply complete! Resources: 67 added，0 changed，0 destroyed.

Outputs:

bastion_ip = [
  "34.219.204.217",
]
default-cluster_monitor-dns = a82db513ba84511e9af170283460e413-1838961480.us-west-2.elb.amazonaws.com
default-cluster_tidb-dns = a82df6d13a84511e9af170283460e413-d3ce3b9335901d8c.elb.us-west-2.amazonaws.com
eks_endpoint = https://9A9A5ABB8303DDD35C0C2835A1801723.yl4.us-west-2.eks.amazonaws.com
eks_version = 1.12
kubeconfig_filename = credentials/kubeconfig_my-cluster
region = us-west-21
```

上記の出力情報は、`terraform output`コマンドを再度利用することで取得できる。

> **注意：**
>
> バージョン 1.14 より前の EKS は、アベイラビリティゾーン間の自動 NLB 負荷分散をサポートしていないため、デフォルトでは TiDB インスタンス間で不均衡な量の圧力が発生する。 本番環境については、[AWS公式ドキュメント](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/enable-disable-crosszone-lb.html#enable-)を参照して、アベイラビリティゾーン間自動NLB負荷分散を手動で有効にすることを強くお勧めします。

## 3. データベースをアクセスする

`terraform apply`が完了したら、まず `ssh` を使って踏み台に接続し、MySQLクライアントを使って TiDB クラスタにアクセスすることができる。

コマンドは以下の通りです（`<>`の部分を上記の出力メッセージに置き換えてください）。

```shell
ssh -i credentials/<eks_name>.pem centos@<bastion_ip>
```

```shell
mysql -h <default-cluster_tidb-dns> -P 4000 -u root
```

`eks_name` のデフォルト値は `my-cluster` です。 DNS名が解決しない場合は、数分お待ちください。

また、`kubectl` コマンドと `helm` コマンドで `kubeconfig` ファイル `credentials/kubeconfig_<eks_name>` を使用して、 EKS クラスタと対話することもできる。下記の２つの方式がある。

- `--kubeconfig` パラメーターを指定する：

    ```shell
    kubectl --kubeconfig credentials/kubeconfig_<eks_name> get po -n <default_cluster_name>
    ```

    ```shell
    helm --kubeconfig credentials/kubeconfig_<eks_name> ls
    ```

- `KUBECONFIG` 環境変数を設定する

    ```shell
    export KUBECONFIG=$PWD/credentials/kubeconfig_<eks_name>
    ```

    ```shell
    kubectl get po -n <default_cluster_name>
    ```

    ```shell
    helm ls
    ```

## 4. Grafana 監視

ブラウザで `<monitor-dns>:3000` のURLで Grafana をアクセスすることができる。

Grafana の初期ユーザー情報(初回ログインの後パスワード変更の画面が出る)：

- ユーザー名：admin
- パスワード：admin 

## 5. TiDB クラスターをアップグレードする

TiDB クラスターをアップグレードしたい場合、 `terraform.tfvars` ファイルに `default_cluster_version` 変数を新バージョンコードに変更して、 `terraform apply`を実行してださい。

例、 TiDB クラスターを 4.0.1 にアップグレードする場合、 `default_cluster_version` を `v4.0.1` に：

```
default_cluster_version= "v4.0.1"
```

> **注意点：**
>
> アップグレードには時間がかかるので、 `kubectl --kubeconfig credentials/kubeconfig_<eks_name> get po -n <default_cluster_name> --watch` コマンドでアップグレードの進捗状況を継続的に確認することができる。

## 6. TiDBクラスターのスケーリング

TiDBクラスターをスケーリングしたい場合、 `terraform.tfvars` ファイルに `default_cluster_tikv_count` か `default_cluster_tidb_count` 変数かを変更して、 `terraform apply` を実行してください。

例、 `default_cluster_tidb_count` を 2 から 4 に変更し、TiDB をスケーリングする：

```
default_cluster_tidb_count = 4
```

> **注意：**
>
> - 由于缩容过程中无法确定会缩掉哪个节点，目前还不支持 TiDB 集群的缩容。
> - 扩容过程会持续几分钟，可以通过 `kubectl --kubeconfig credentials/kubeconfig_<eks_name> get po -n <default_cluster_name> --watch` 命令持续观察进度。

## 7. 自定义

可以按需在 `terraform.tfvars` 文件中设置各个变量，例如集群名称和镜像版本等。

### 自定义 AWS 相关的资源

由于 TiDB 服务通过 [Internal Elastic Load Balancer](https://aws.amazon.com/blogs/aws/internal-elastic-load-balancers/) 暴露，默认情况下，会创建一个 Amazon EC2 实例作为堡垒机，访问创建的 TiDB 集群。堡垒机上预装了 MySQL 和 Sysbench，所以可以通过 SSH 方式登陆到堡垒机后通过 ELB 访问 TiDB。如果的 VPC 中已经有了类似的 EC2 实例，可以通过设置 `create_bastion` 为 `false` 禁掉堡垒机的创建。

TiDB 版本和组件数量也可以在 `terraform.tfvars` 中修改，可以按照自己的需求配置。

### 自定义 TiDB 参数配置

Terraform 脚本中为运行在 EKS 上的 TiDB 集群提供了合理的默认配置。有自定义需求时，可以在 `clusters.tf` 中通过 `override_values` 参数为每个 TiDB 集群指定一个 `values.yaml` 文件来自定义集群参数配置。

作为例子，默认集群使用了 `./default-cluster.yaml` 作为 `values.yaml` 配置文件，并在配置中打开了"配置文件滚动更新"特性。

值得注意的是，在 EKS 上部分配置项无法在 `values.yaml` 中进行修改，包括集群版本、副本数、`NodeSelector` 以及 `Tolerations`。`NodeSelector` 和 `Tolerations` 由 Terraform 直接管理以确保基础设施与 TiDB 集群之间的一致性。集群版本和副本数可以通过 `cluster.tf` 文件中的 `tidb-cluster` module 参数来修改。

> **注意：**
>
> 自定义 `values.yaml` 配置文件中，不建议包含如下配置（`tidb-cluster` module 默认固定配置）：

```
pd:
  storageClassName: ebs-gp2
tikv:
  stroageClassName: local-storage
tidb:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-internal: '0.0.0.0/0'
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: >'true'
  separateSlowLog: true
monitor:
  storage: 100Gi
  storageClassName: ebs-gp2
  persistent: true
  grafana:
    config:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
    service:
      type: LoadBalancer
```

### 自定义 TiDB Operator

可以通过在 `terraform.tfvars` 中设置 `operator_values` 参数传入自定义的 `values.yaml` 内容来配置 TiDB Operator。示例如下：

```
operator_values = "./operator_values.yaml"
}
```

## 8. 管理多个 TiDB 集群

一个 `tidb-cluster` 模块的实例对应一个 TiDB 集群，可以通过编辑 `clusters.tf` 添加新的 `tidb-cluster` 模块实例来新增 TiDB 集群，示例如下：

```hcl
module example-cluster {
  source = "../modules/aws/tidb-cluster"
  
  # The target EKS, required
  eks = local.eks
  # The subnets of node pools of this TiDB cluster, required
  subnets = local.subnets
  # TiDB cluster name, required
  cluster_name    = "example-cluster"
  
  # Helm values file
  override_values = file("example-cluster.yaml")
  # TiDB cluster version
  cluster_version               = "v3.0.0"
  # SSH key of cluster nodes
  ssh_key_name                  = module.key-pair.key_name
  # PD replica number
  pd_count                      = 3
  # TiKV instance type
  pd_instance_type              = "t2.xlarge"
  # TiKV replica number
  tikv_count                    = 3
  # TiKV instance type
  tikv_instance_type            = "t2.xlarge"
  # The storage class used by TiKV, if the TiKV instance type do not have local SSD, you should change it to storage class
  # TiDB replica number
  tidb_count                    = 2
  # TiDB instance type
  tidb_instance_type            = "t2.xlarge"
  # Monitor instance type
  monitor_instance_type         = "t2.xlarge"
  # The version of tidb-cluster helm chart
  tidb_cluster_chart_version    = "v1.0.0"
  # Decides whether or not to create the tidb-cluster helm release.
  # If this variable is set to false, you have to
  # install the helm release manually
  create_tidb_cluster_release   = true
}
```

> **注意：**
>
> `cluster_name` 必须是唯一的。

可以通过 `kubectl` 获取新集群的监控系统地址与 TiDB 地址。假如希望让 Terraform 脚本输出这些地址，可以通过在 `outputs.tf` 中增加相关的输出项实现：

```hcl
output "example-cluster_tidb-hostname" {
  value = module.example-cluster.tidb_hostname
}

output "example-cluster_monitor-hostname" {
  value = module.example-cluster.monitor_hostname
}
```

修改完成后，执行 `terraform init` 和 `terraform apply` 创建集群。

最后，只要移除 `tidb-cluster` 模块调用，对应的 TiDB 集群就会被销毁，EC2 资源也会随之释放。

## 9. 仅管理基础设施

通过调整配置，可以控制 Terraform 脚本只创建 Kubernetes 集群和 TiDB Operator。操作步骤如下：

* 修改 `clusters.tf` 中 TiDB 集群的 `create_tidb_cluster_release` 配置项：

  ```hcl
  module "default-cluster" {
    ...
    create_tidb_cluster_release = false
  }
  ```

  如上所示，当 `create_tidb_cluster_release` 设置为 `false` 时，Terraform 脚本不会创建和修改 TiDB 集群，但仍会创建 TiDB 集群所需的计算和存储资源。此时，可以使用 Helm 等工具来独立管理集群。

> **注意：**
>
> 在已经部署的集群上将 `create_tidb_cluster_release` 调整为 `false` 会导致已安装的 TiDB 集群被删除，对应的 TiDB 集群对象也会随之被删除。

## 10. 销毁集群

可以通过如下命令销毁集群：

```shell
terraform destroy
```

> **注意：**
>
> * 该操作会销毁 EKS 集群以及部署在该 EKS 集群上的所有 TiDB 集群。
> * 如果不再需要存储卷中的数据，在执行 `terraform destroy` 后，需要在 AWS 控制台手动删除 EBS 卷。

## 11. 管理多个 Kubernetes 集群

本节详细介绍了如何管理多个 Kubernetes 集群（EKS），并在每个集群上部署一个或更多 TiDB 集群。

上述文档中介绍的 Terraform 脚本组合了多个 Terraform 模块：

- `tidb-operator` 模块，用于创建 EKS 集群并在 EKS 集群上安装配置 [TiDB Operator](/tidb-in-kubernetes/deploy/tidb-operator.md)。
- `tidb-cluster` 模块，用于创建 TiDB 集群所需的资源池并部署 TiDB 集群。
- EKS 上的 TiDB 集群专用的 `vpc` 模块、`key-pair`模块和`bastion` 模块

管理多个 Kubernetes 集群的最佳实践是为每个 Kubernetes 集群创建一个单独的目录，并在新目录中自行组合上述 Terraform 模块。这种方式能够保证多个集群间的 Terraform 状态不会互相影响，也便于自由定制和扩展。下面是一个例子：

```shell
mkdir -p deploy/aws-staging
vim deploy/aws-staging/main.tf
```

`deploy/aws-staging/main.tf` 的内容可以是：

```hcl
provider "aws" {
  region = "us-west-1"
}

# 创建一个 ssh key，用于登录堡垒机和 Kubernetes 节点
module "key-pair" {
  source = "../modules/aws/key-pair"

  name = "another-eks-cluster"
  path = "${path.cwd}/credentials/"
}

# 创建一个新的 VPC
module "vpc" {
  source = "../modules/aws/vpc"

  vpc_name = "another-eks-cluster"
}

# 在上面的 VPC 中创建一个 EKS 并部署 tidb-operator
module "tidb-operator" {
  source = "../modules/aws/tidb-operator"

  eks_name           = "another-eks-cluster"
  config_output_path = "credentials/"
  subnets            = module.vpc.private_subnets
  vpc_id             = module.vpc.vpc_id
  ssh_key_name       = module.key-pair.key_name
}

# 特殊处理，确保 helm 操作在 EKS 创建完毕后进行
resource "local_file" "kubeconfig" {
  depends_on        = [module.tidb-operator.eks]
  sensitive_content = module.tidb-operator.eks.kubeconfig
  filename          = module.tidb-operator.eks.kubeconfig_filename
}
provider "helm" {
  alias    = "eks"
  insecure = true
  install_tiller = false
  kubernetes {
    config_path = local_file.kubeconfig.filename
  }
}

# 在上面的 EKS 集群上创建一个 TiDB 集群
module "tidb-cluster-a" {
  source = "../modules/aws/tidb-cluster"
  providers = {
    helm = "helm.eks"
  }

  cluster_name = "tidb-cluster-a"
  eks          = module.tidb-operator.eks
  ssh_key_name = module.key-pair.key_name
  subnets      = module.vpc.private_subnets
}

# 在上面的 EKS 集群上创建另一个 TiDB 集群
module "tidb-cluster-b" {
  source = "../modules/aws/tidb-cluster"
  providers = {
    helm = "helm.eks"
  }
  
  cluster_name = "tidb-cluster-b"
  eks          = module.tidb-operator.eks
  ssh_key_name = module.key-pair.key_name
  subnets      = module.vpc.private_subnets
}

# 创建一台堡垒机
module "bastion" {
  source = "../modules/aws/bastion"

  bastion_name             = "another-eks-cluster-bastion"
  key_name                 = module.key-pair.key_name
  public_subnets           = module.vpc.public_subnets
  vpc_id                   = module.vpc.vpc_id
  target_security_group_id = module.tidb-operator.eks.worker_security_group_id
  enable_ssh_to_workers    = true
}

# 输出 tidb-cluster-a 的 TiDB 服务地址
output "cluster-a_tidb-dns" {
  description = "tidb service endpoints"
  value       = module.tidb-cluster-a.tidb_hostname
}

# 输出 tidb-cluster-b 的监控地址
output "cluster-b_monitor-dns" {
  description = "tidb service endpoint"
  value       = module.tidb-cluster-b.monitor_hostname
}

# 输出堡垒机 IP
output "bastion_ip" {
  description = "Bastion IP address"
  value       = module.bastion.bastion_ip
}
```

上面的例子很容易进行定制，比如，假如不需要堡垒机，便可以删去对 `bastion` 模块的调用。同时，项目中提供的 Terraform 模块均设置了合理的默认值，因此在调用这些 Terraform 模块时，可以略去大部分的参数。

可以参考默认的 Terraform 脚本来定制每个模块的参数，也可以参考每个模块的 `variables.tf` 文件来了解所有可配置的参数。

另外，这些 Terraform 模块可以很容易地集成到自己的 Terraform 工作流中。假如对 Terraform 非常熟悉，这也是我们推荐的一种使用方式。

> **注意：**
>
> * 由于 Terraform 本身的限制（[hashicorp/terraform#2430](https://github.com/hashicorp/terraform/issues/2430#issuecomment-370685911)），在自己的 Terraform 脚本中，也需要保留上述例子中对 `helm provider` 的特殊处理。
> * 创建新目录时，需要注意与 Terraform 模块之间的相对路径，这会影响调用模块时的 `source` 参数。
> * 假如想在 tidb-operator 项目之外使用这些模块，需要确保 `modules` 目录中的所有模块的相对路径保持不变。

假如不想自己写 Terraform 代码，也可以直接拷贝 `deploy/aws` 目录来创建新的 Kubernetes 集群。但要注意不能拷贝已经运行过 `terraform apply` 的目录（已经有 Terraform 的本地状态）。这种情况下，推荐在拷贝前克隆一个新的仓库。
