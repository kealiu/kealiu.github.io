# 使用说明

[Github Repo](https://github.com/kealiu/aws-genomics-workflow-cromwell-nwcd)

## 代码 S3 Bucket

上传代码到s3 bucket: 因为cloudformation yaml模版以及自定义的文件、script需要能够下载，所以，我们需要讲本repo代码上传到一个s3 bucket。可以新建或者用已有的，确保当前使用的`aws iam user` 有对应的bucket访问权限。将 `nwcdcromwell` 目录上传到你的S3 `Bucket`。 
    1. 通过web UI 方式上传：
        ![code-s3-bucket-upload-folder](https://github.com/kealiu/aws-genomics-workflow-cromwell-nwcd/raw/master/docs/images/code-s3-bucket-upload-folder.png)
    1. 通过 `aws cli` 方式上传: `aws s3 cp -r nwcdcromwell s3://<your-bucket>/`

## Cromwell 环境部署

### 创建Cromwell 运行环境 S3 Bucket

Cromwell运行需要使用到s3，可以新建或者用已有的，确保当前使用的`aws iam user` 有对应的bucket访问权限。

### 运行Cloudformation 模版

1. 进入到Cloudformation 服务，新建stack。如图，复制 `nwcdcromwell\cn-gwfcore-root.template.yaml` 的URL
    ![code-s3-bucket-root-template](https://github.com/kealiu/aws-genomics-workflow-cromwell-nwcd/raw/master/docs/images/code-s3-bucket-root-template.png)
1. 新建Cloudformation Stack，填入 `nwcdcromwell\cn-gwfcore-root.template.yaml` 的URL 
    ![cf-create-stack](https://github.com/kealiu/aws-genomics-workflow-cromwell-nwcd/raw/master/docs/images/cf-create-stack.png)
1. 填入stack名字（英文）
1. 填写参数，具体说明如下：
    1. S3 Bucket Name: Cromwell 用的S3 Bucket
    1. Existing Bucket?: Bucket是否存在。由于s3 bucket名字需要AWS全局唯一，所以使用以及建好的Bucket
    1. Template Root URL: cloudformation子模块的目录。等于 `nwcdcromwell\cn-gwfcore-root.template.yaml` 的URL去掉最后的`\cn-gwfcore-root.template.yaml`。如 `https://<code-bucket-name>.s3.cn-northwest-1.amazonaws.com.cn/nwcdcromwell`
    1. S3 Pathname of Artifacts： 一些script以及辅助文件存储位置。一般为 `s3://<code-bucket-name>/nwcdcromwell/artifacts`
    1. VPC ID / VPC Subnet IDs: 运行的网络环境
    1. Keypaire For Login Cromwell Server: 服务器的登陆 key。后面要用该 key 登陆服务器。
    1. Database Username: 数据库用户名
    1. Database Password: 数据库密码
    1. Namespace: 可选，服务名字
    1. Default Min vCPU: 默认任务最小CPU，可以不修改，或按实际情况填写。
    1. Default Max vCPU: 默认任务最大CPU，可以不修改，或按实际情况填写。
    1. High Priority Min vCPU: 高优先级任务最小CPU，可以不修改，或按实际情况填写。
    1. High Priority Max vCPU: 高优先级任务最大CPU，可以不修改，或按实际情况填写。
    1. Artifact S3 Bucket Name / Artifact S3 Prefix: 公共文件下载处。不建议修改
    1. The Cromwell Server Instance Type: Cromwell Server的配置。建议使用默认
    1. DockerStorageVolumeSize: 运行任务的磁盘空间。建议根据分析文件的数据大小填写。注意：如太小会导致任务因为磁盘空间不够而失败。
1. 下一步，根据情况修改
1. 创建前Review。 注意：最下面的两个 check box 一定要勾选。 然后创建。等待环境搭建完成
1. 记录下创建完成后的`PublicIp`。该IP为Cromwell Server的运行IP
    ![cf-stack-output](https://github.com/kealiu/aws-genomics-workflow-cromwell-nwcd/raw/master/docs/images/cf-stack-output.png)

## 执行Hello World任务

通过 `ssh -i <keypaire> ec2-user@<cromwell-public-ip>` 登陆服务器:

```
cat << EOF > simple-hello.wdl
task echoHello{
    command {
        echo "Hello AWS!"
    }
    runtime {
        docker: "ubuntu:latest"
    }

}

workflow printHelloAndGoodbye {
    call echoHello
}
EOF
curl -X POST "http://localhost:8000/api/workflows/v1" \
    -H  "accept: application/json" \
    -F "workflowSource=@simple-hello.wdl"
```

### 通过AWS Batch Dashboard 检查运行结果

![batch-dashboard](https://github.com/kealiu/aws-genomics-workflow-cromwell-nwcd/raw/master/docs/images/batch-dashboard.png)

## 运行大型任务GATK HaplotypeCaller

### 准备数据

将数据同步到自己的Bucket： `aws s3 sync s3://gatk-test-data s3://yourbucket/ --region cn-northwest-1`

### 修改配置文件

修改 `tests/HaplotypeCaller/HaplotypeCaller.aws.json`，将里面的 `yourbucket` 替换为刚刚的gatk 数据bucket名字

### 登陆 Cromwell 服务器，提交任务

上传 `tests/HaplotypeCaller/HaplotypeCaller.aws.json`, `tests/HaplotypeCaller/HaplotypeCaller.aws.wdl` 文件，然后:

```
curl -X POST "http://localhost:8000/api/workflows/v1" \
    -H  "accept: application/json" \
    -F "workflowSource=@HaplotypeCaller.aws.wdl" \
    -F "workflowInputs=@HaplotypeCaller.aws.json"
```

记录下命令的返回id，通过AWS Batch Dashboard等待任务完成，大概需要 25 分钟左右。然后到 `s3://<your-cromwell-bucket>/cromwell-execution/HaplotypeCallerGvcf_GATK4/<task-id>/call-HaplotypeCaller/` 接收运行结果。

## 日志说明

所有日志都会存储到 `cloudwatch logs`。在 `Cloudwatch` -> `Logs` -> `Log groups` 里面查找。

### Cromwell Server 运行logs

在名为 `cromwell-server` 的logs group内，根据 `cromwell server` 的`instances id` 查看相应的日志。任务提交失败或者不运行，可以通过该logs调查。

### 单个 AWS Batch Jobs logs

在 各个 Jobs 详情页面，`Log stream name` 标记了logs位置。点击前往。一般任务 FAILED 时，可以通过该log调查。
![batch-job-detail](https://github.com/kealiu/aws-genomics-workflow-cromwell-nwcd/raw/master/docs/images/batch-job-detail.png)

# 参考
- https://github.com/Iwillsky/cromwellcn
- https://docs.opendata.aws/genomics-workflows/orchestration/orchestration-intro.html
