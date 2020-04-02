---
title: 【note】Terraform
date: 2020-04-02 20:55:43
categories: note
tags:
    - devops
---

# Terraform

###  Terraform 是什么

Terraform 是一种安全有效地构建、更改和版本控制基础设施的工具(基础架构自动化的编排工具)。它的目标是 "Write, Plan, and create Infrastructure as Code", 基础架构即代码。

通俗的讲，Terraform是一个运行在客户端的开源的用于资源编排的自动化运维工具。通过代码的形式将所要管理的资源定义在模板中，解析并执行模板来自动化完成资源的创建、变更和管理，进而达到自动化运维的目标。

Terraform是由Hashicorp公司推出的一个开源项目，使用Go语言编写。目前主流的云厂商如AWS、GCP、腾讯云、阿里云等都支持使用Terraform进行云上资源的自动编排管理。

### Terraform核心功能

#### 基础设施即代码(IaC, Infrastructure as Code)

Terraform 基于一种特定的配置语言（HCL, Hashicorp Configuration Language）来描述基础设施资源。由此，可以像对待任何其他代码一样，实现对所描述的解决方案或者基础架构的版本控制和管理。

#### 执行计划(Execution Plans)

Terraform 在执行模板前，可以执行plan步骤(terraform plan)解析模板生成一个可执行的计划，展示当前模板所要创建或变更的资源及其属性。操作人员可以预览这个计划j进行检查，以免操作真正的基础结构时发生一些超预期的问题。

#### 资源图(Resource Graph)

Terraform 会根据模板中的定义，构建所有资源的图形，并且以并行的方式创建和修改任何没有相互依赖的资源。因此，Terraform 可以高效地构建基础设施，操作人员也可以通过图表深入地解其基础设施中的依赖关系。

#### 自动化变更（Change Automation）

当模板中定义的资源内容发生变更时，Terraform 都会基于新的资源拓扑图将变更的内容plan 出来，从而避免了人为操作带来的错误。





### 配置介绍

Terraform使用文本文件来描述基础设施和设置变量。这些文件称为Terraform 配置。配置文件的格式可以有两种格式：Terraform格式和JSON。

Terraform格式更加人性化，支持注释，推荐使用该格式。Terraform格式后缀名以.tf结尾，JSON格式后缀名以.tf.json结尾。

在调用加载Terraform配置的任何命令时，Terraform将按字母顺序加载指定目录中的所有配置文件。加载文件的后缀名必须是.tf或.tf.json。否则，文件将被忽略。

Terraform配置是声明式的，因此对其他资源和变量的引用不依赖于它们定义的顺序。

### 配置语法

Terraform配置的语法称为HashiCorp配置语言（HCL）。示例如下:

```json
provider "tencentcloud" {
  secret_id  = "your_secret_id"
  secret_key = "your_secret_key"
  region     = "region"
}
```





### 资源

基础设施和服务统称为资源，如私有网络、子网、物理机、虚拟机、镜像、NAT网关等等都可以称之为资源，也是开发和运维人员经常要打交道要维护的东西。Terraform把资源大致分为resurce和data source。

#### resource

```json
resource "资源类名" "映射到本地的唯一资源名" {
  参数 = 值
  ...
}
```

这类资源一般是抽象的真正的云服务资源，支持增删改，如私有网络、NAT网关、虚拟机实例。

```json
resource "tencentcloud_instance" "web" {
  instance_name              = "web example"
  availability_zone          = "ap-guangzhou-3"
  image_id                   = "img-9qabwvbn"
  instance_type              = "S1.SMALL1"
  system_disk_type           = "CLOUD_PREMIUM"
  password                   = "your_ssh_password"
  internet_max_bandwidth_out = 10
  count                      = 1
}
```



#### data source

```json
data "资源类名" "映射到本地的唯一资源名" {
  参数 = 值
  ...
}
```

这类资源一般是固定的一些可读资源，如可用区列表、镜像列表。大部分情况下，resource资源也会封装一个data source方法，用于资源查询。

```json
data "tencentcloud_images" "centos" {
  image_type = ["PUBLIC_IMAGE"]
  os_name    = "centos 7.5"
}
```



# 使用演示

### 环境

#### 安装Terraform

```
$ terraform -h
Usage: terraform [-version] [-help] <command> [args]

Common commands:
    apply              Builds or changes infrastructure
    console            Interactive console for Terraform interpolations
    destroy            Destroy Terraform-managed infrastructure
    env                Workspace management
    fmt                Rewrites config files to canonical format
    get                Download and install modules for the configuration
    graph              Create a visual graph of Terraform resources
    import             Import existing infrastructure into Terraform
    init               Initialize a Terraform working directory
    login              Obtain and save credentials for a remote host
    logout             Remove locally-stored credentials for a remote host
    output             Read an output from a state file
    plan               Generate and show an execution plan
    providers          Prints a tree of the providers used in the configuration
    refresh            Update local state file against real resources
    show               Inspect Terraform state or plan
    taint              Manually mark a resource for recreation
    untaint            Manually unmark a resource as tainted
    validate           Validates the Terraform files
    version            Prints the Terraform version
    workspace          Workspace management
```



#### 获取 AccessKey/SecretKey

腾讯云控制台 访问管理->访问秘钥->API秘钥管理，查看或新建秘钥，得到`SecretId`和`SecretKey`





### 创建配置文件

Terraform通过Provider来扩展对新的基础架构的支持，几乎支持所有的云服务平台，tencentcloud只是Terraform内建 Providers 中的一种。

```json
provider "tencentcloud" {
  secret_id  = "your_secret_id"
  secret_key = "your_secret_key"
  region     = "ap-guangzhou-3"
}
```

tencentcloud凭据配置支持静态凭据和环境变量两种方式。

* 静态凭据

  ```json
  provider "tencentcloud" {
    secret_id  = "your_secret_id"
    secret_key = "your_secret_key"
    region     = "ap-guangzhou"
  }
  ```

  

* 环境变量

  ```
  $ export TENCENTCLOUD_SECRET_ID="your_fancy_accesskey"
  $ export TENCENTCLOUD_SECRET_KEY="your_fancy_secretkey"
  $ export TENCENTCLOUD_REGION="ap-guangzhou"
  
  provider "tencentcloud" {}
  ```





#### 资源文件

```json
# cvm.tf
provider "tencentcloud" {}

 // Create a cvm
resource "tencentcloud_instance" "cvm_test" {
    instance_name = "cvm-test"
    availability_zone = "ap-guangzhou-3"
    image_id = "img-9qabwvbn"
    instance_type = "S3.SMALL1"
    
    # disk
    system_disk_type = "CLOUD_PREMIUM"
    
    # auth
    password = "your_ssh_login_password"
    
    # security groups
    security_groups = [
        tencentcloud_security_group.sg_ssh.id
    ]

    # network 
    allocate_public_ip = true
    internet_max_bandwidth_out = 1  // Mbps

    # count
    count = 1
}

resource "tencentcloud_security_group" "sg_ssh" {
  name        = "ssh accessibility"
}

resource "tencentcloud_security_group_rule" "ssh" {
  security_group_id = tencentcloud_security_group.sg_ssh.id
  type              = "ingress"
  cidr_ip           = "0.0.0.0/0"
  ip_protocol       = "tcp"
  port_range        = "22"
  policy            = "ACCEPT"
}
```





### 初始化工作目录

```
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Checking for available provider plugins...
- Downloading plugin for provider "tencentcloud" (terraform-providers/tencentcloud) 1.30.7...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.tencentcloud: version = "~> 1.30"
```





### plan预览

```
$ terraform plan

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # tencentcloud_instance.cvm_test[0] will be created
  + resource "tencentcloud_instance" "cvm_test" {
      + allocate_public_ip         = true
      + availability_zone          = "ap-gruangzhou"
      + create_time                = (known after apply)
      + disable_monitor_service    = false
      + disable_security_service   = false
      + expired_time               = (known after apply)
      + id                         = (known after apply)
      + image_id                   = "img-9qabwvbn"
      + instance_charge_type       = "POSTPAID_BY_HOUR"
      + instance_name              = "cvm-test"
      + instance_status            = (known after apply)
      + instance_type              = "S3.SMALL1"
      + internet_charge_type       = "TRAFFIC_POSTPAID_BY_HOUR"
      + internet_max_bandwidth_out = 1
      + key_name                   = (known after apply)
      + password                   = (sensitive value)
      + private_ip                 = (known after apply)
      + project_id                 = 0
      + public_ip                  = (known after apply)
      + running_flag               = true
      + security_groups            = (known after apply)
      + subnet_id                  = (known after apply)
      + system_disk_id             = (known after apply)
      + system_disk_size           = 50
      + system_disk_type           = "CLOUD_PREMIUM"
      + vpc_id                     = (known after apply)

      + data_disks {
          + data_disk_id         = (known after apply)
          + data_disk_size       = (known after apply)
          + data_disk_type       = (known after apply)
          + delete_with_instance = (known after apply)
        }
    }

  # tencentcloud_security_group.sg_ssh will be created
  + resource "tencentcloud_security_group" "sg_ssh" {
      + id         = (known after apply)
      + name       = "ssh accessibility"
      + project_id = (known after apply)
    }

  # tencentcloud_security_group_rule.ssh will be created
  + resource "tencentcloud_security_group_rule" "ssh" {
      + cidr_ip           = "0.0.0.0/0"
      + description       = (known after apply)
      + id                = (known after apply)
      + ip_protocol       = "tcp"
      + policy            = "ACCEPT"
      + port_range        = "22"
      + security_group_id = (known after apply)
      + source_sgid       = (known after apply)
      + type              = "ingress"
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```





### 资源apply

```
$ terraform apply
...
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

tencentcloud_instance.cvm_test[0]: Creating...
tencentcloud_instance.cvm_test[0]: Still creating... [10s elapsed]
tencentcloud_instance.cvm_test[0]: Creation complete after 13s [id=ins-13zz24b0]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

{% asset_img apply.png %}



### 查看状态

```
$ terraform show
# tencentcloud_instance.cvm_test[0]:
resource "tencentcloud_instance" "cvm_test" {
    allocate_public_ip         = true
    availability_zone          = "ap-guangzhou-3"
    create_time                = "2020-03-31T13:05:10Z"
    disable_monitor_service    = false
    disable_security_service   = false
    id                         = "ins-13zz24b0"
    image_id                   = "img-9qabwvbn"
    instance_charge_type       = "POSTPAID_BY_HOUR"
    instance_name              = "cvm-test"
    instance_status            = "RUNNING"
    instance_type              = "S3.SMALL1"
    internet_charge_type       = "TRAFFIC_POSTPAID_BY_HOUR"
    internet_max_bandwidth_out = 1
    password                   = (sensitive value)
    private_ip                 = "172.16.0.2"
    project_id                 = 0
    public_ip                  = "203.195.206.81"
    running_flag               = true
    security_groups            = [
        "sg-f8y2yhyj",
    ]
    subnet_id                  = "subnet-itv02446"
    system_disk_id             = "disk-gr11n1oq"
    system_disk_size           = 50
    system_disk_type           = "CLOUD_PREMIUM"
    vpc_id                     = "vpc-1a17zjtx"
}

# tencentcloud_security_group.sg_ssh:
resource "tencentcloud_security_group" "sg_ssh" {
    id         = "sg-f8y2yhyj"
    name       = "ssh accessibility"
    project_id = 0
    tags       = {}
}

# tencentcloud_security_group_rule.ssh:
resource "tencentcloud_security_group_rule" "ssh" {
    cidr_ip           = "0.0.0.0/0"
    id                = "eyJzZ19pZCI6InNnLWY4eTJ5aHlqIiwicG9saWN5X3R5cGUiOiJpbmdyZXNzIiwiY2lkcl9pcCI6IjAuMC4wLjAvMCIsInByb3RvY29sIjoidGNwIiwicG9ydF9yYW5nZSI6IjIyIiwiYWN0aW9uIjoiQUNDRVBUIiwic291cmNlX3NnX2lkIjoiIiwiZGVzY3JpcHRpb24iOiIifQ=="
    ip_protocol       = "tcp"
    policy            = "ACCEPT"
    port_range        = "22"
    security_group_id = "sg-f8y2yhyj"
    type              = "ingress"
}
```





### 查看依赖图

```
terraform graph | dot -Tsvg > graph.svg
```

{% asset_img graph.png %}



### 资源销毁

```
$ terraform destroy

...
Plan: 0 to add, 0 to change, 3 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
tencentcloud_security_group_rule.ssh: Destroying... [id=eyJzZ19pZCI6InNnLWY4eTJ5aHlqIiwicG9saWN5X3R5cGUiOiJpbmdyZXNzIiwiY2lkcl9pcCI6IjAuMC4wLjAvMCIsInByb3RvY29sIjoidGNwIiwicG9ydF9yYW5nZSI6IjIyIiwiYWN0aW9uIjoiQUNDRVBUIiwic291cmNlX3NnX2lkIjoiIiwiZGVzY3JpcHRpb24iOiIifQ==]
tencentcloud_instance.cvm_test[0]: Destroying... [id=ins-13zz24b0]
tencentcloud_security_group_rule.ssh: Destruction complete after 1s
tencentcloud_instance.cvm_test[0]: Still destroying... [id=ins-13zz24b0, 10s elapsed]
tencentcloud_instance.cvm_test[0]: Still destroying... [id=ins-13zz24b0, 20s elapsed]
tencentcloud_instance.cvm_test[0]: Still destroying... [id=ins-13zz24b0, 30s elapsed]
tencentcloud_instance.cvm_test[0]: Still destroying... [id=ins-13zz24b0, 40s elapsed]
tencentcloud_instance.cvm_test[0]: Destruction complete after 50s
tencentcloud_security_group.sg_ssh: Destroying... [id=sg-f8y2yhyj]
tencentcloud_security_group.sg_ssh: Destruction complete after 0s

Destroy complete! Resources: 3 destroyed.  

```



# 扩展

### 常见规范

Terraform 运行时会读取工作目录中所有的 *.tf, *.tfvars文件，不必把所有的东西都写在单个文件中去，应按职责分列在不同的文件中，例如

```
provider.tf             ### provider 配置
terraform.tfvars        ### 配置 provider 要用到的变量
varable.tf              ### 通用变量
resource.tf             ### 资源定义
data.tf                 ### 包文件定义
output.tf               ### 输出
```





### 使用ssh key

```json
variable "tf_ssh_key" {
  default="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDgEzN0r6XJ7nYcZ4ba98S1Su1a0WnBrA0KHcs/rzCusshjjQrLBbnxzAkiwIaRq7x2M4NsXhg/G4zQB2nkYFwIaLD7Kw8j2n5AgCkqrgTp3mBUnBy9lLQ4pMeScenaR8DSbchFCjEumycuqn91Q/iiGv+R0B1xmuz7l6wnDjF4fQucfpytFKWZL/lX4dsJ/a6IWkTNoMXd8u/XyrfBt+fuYF4SlFsUldTxUzFOiy/r/DSXWB1jSwpO8s228p57Ax67TY59oBLfF4qyTUyWtvNYO3mOjRXPsRBa+W6/asqRKras+kCGRsgw76O/jW8fNmaM4tZ0O7GXNpKEJ8SJGVk7 ethan@loc.com"
}

resource "tencentcloud_key_pair" "bees_ssh_key" {
  key_name   = "from_terraform_public_key"
  public_key = var.tf_ssh_key
}

resource "tencentcloud_instance" "cvm_test" {
    instance_name = "cvm-test"
		...
    # auth
    key_name = tencentcloud_key_pair.ksane_ssh_key.id
}
```


### AWS

#### provider

AWS Provider被用来与AWS支持的许多资源进行交互。在使用该提供程序之前, 需要使用适当的Credentials进行配置。https://www.terraform.io/docs/providers/aws/index.html

* 静态凭据

  ```
  provider "aws" {
  	region = "us-west-2"
  	access_key = "anaccesskey"
  	secret_key = "ansecret_key"
  }
  ```

* 环境变量

  ```
  provider "aws" {}
  
  $ export AWS_ACCESS_KEY_ID="anaccesskey"
  $ export AWS_SECRET_ACCESS_KEY="ansecret_key"
  $ export AWS_DEFAULT_REGION="us-west-2"
  ```

* 共享凭据文件

  ```json
  provider "aws" {
   region = "us-west-2"
   shared_credentials_file = "/USers/tf_user/.aws/creds"
   profile = "customprofile"
  }
  ```

  aws 共享凭据文件，默认路径为

  ```
  OSX and Linux: $HOME/.aws/credentials
  Windows: "%USERPROFILE%\.aws\credentials"
  ```



# REF

* [Terraform：简介](https://www.cnblogs.com/sparkdev/p/10052310.html)

* [terraform: TencentCloud Provider document](https://www.terraform.io/docs/providers/tencentcloud/index.html)

* [腾讯云支持 Terraform 开发实践](https://cloud.tencent.com/developer/article/1067230)
* [github: 腾讯云应用指南](https://github.com/tencentyun/terraform-provider-tencentcloud/blob/master/website/docs/index.html.markdown) 
* [使用 Terraform 管理 COS](https://cloud.tencent.com/document/product/436/37271)


