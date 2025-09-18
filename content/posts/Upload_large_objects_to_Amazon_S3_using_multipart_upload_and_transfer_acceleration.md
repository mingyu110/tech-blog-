---
title: "使用分段上传和传输加速向Amazon S3上传大对象"
date: 2025-09-18
draft: false
tags: ["AWS", "S3", "大文件传输", "技术架构"]
categories: ["Cloud Architect"]
math: true
---



# 使用分段上传和传输加速向Amazon S3上传大对象

## 1. 引言

Web和移动应用程序常常需要将大文件对象（如高清照片、视频文件）上传到云端。例如，[Amazon Photos](https://www.amazon.com/Amazon-Photos/b?ie=UTF8&node=13234696011) 这类服务就允许用户从浏览器或移动端上传大型媒体文件。

[Amazon S3](https://aws.amazon.com/s3/) 是存储大对象的理想选择，它不仅支持高达5TB的单个对象，还提供了分段上传（Multipart Upload）和传输加速（Transfer Acceleration）功能，以显著缩短上传时间。

## 2. 应用场景与技术挑战

### 2.1. 应用场景

当开发者需要从Web或移动应用向云端上传大文件时，例如：
- 社交媒体平台的用户上传高清视频。
- 在线教育平台的教师上传课程视频。
- 企业网盘用户上传大型设计文件或备份数据。
- 医疗影像系统上传DICOM等大型医疗影像文件。

### 2.2. 技术挑战

在此类场景下，开发者通常面临以下挑战：
- **网络吞吐量限制**：单个HTTP连接受限于底层的TCP吞吐量，无法充分利用所有可用带宽，导致上传速度缓慢。
- **网络延迟与不稳定性**：广域网的延迟和网络质量波动，可能导致用户体验不佳或上传中断。
- **安全性问题**：直接在客户端暴露长期有效的AWS访问凭证，会带来严重的安全风险。
- **大文件上传失败的恢复成本**：如果一个巨大的文件在上传过程中因网络问题中断，从头开始重传会浪费大量时间和带宽。

## 3. 解决方案与技术选型

为了应对上述挑战，我们采用以Amazon S3为核心的解决方案，结合使用其**预签名URL（Presigned URLs）**、**分段上传（Multipart Upload）**和**传输加速（Transfer Acceleration）**功能。该方案能够安全地提升上传吞吐量，最小化网络错误导致的重传，并通过全球边缘节点降低网络延迟，为全球用户提供一致的高性能体验。

下方是该解决方案的参考架构图。

[![架构概览](/images/multipartupload_S3_Acceleration.png)](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/multipartupload.png)

**解决方案流程解析:**

1.  **发起上传请求**：Web或移动应用（前端）通过[Amazon API Gateway](https://aws.amazon.com/api-gateway/)与云端后端通信，以启动和完成分段上传流程。
2.  **处理后端逻辑**：[AWS Lambda](https://aws.amazon.com/lambda/)函数代表前端应用调用S3 API，处理如创建分段上传、生成预签名URL等需要安全凭证的操作。
3.  **执行文件上传**：前端应用使用获取到的预签名URL，通过S3传输加速节点，将大文件的各个部分直接上传到S3。
4.  **边缘加速**：文件上传流量被路由到距离用户最近的AWS边缘站点（Edge Location），从而有效降低网络延迟，随后通过AWS优化的内部网络传输到目标S3存储桶。

### 3.1. 使用S3分段上传（Multipart Upload）提升效率

分段上传允许应用程序将一个大对象拆分成一组较小的部分并行上传。所有部分上传完成后，S3会将它们在云端组合成原始的大对象。

**分段上传的优势:**
- **提高吞吐量**：通过并行上传多个部分，可以更好地利用网络带宽。
- **增强恢复能力**：当网络错误导致某个部分上传失败时，只需重新上传该部分，而无需从头开始，提高了上传的健壮性。

**分段上传的核心三步流程:**
1.  **启动 (Initiate)**：调用`CreateMultipartUpload` API来启动一个分段上传任务，S3会返回一个全局唯一的`UploadId`。
2.  **上传分片 (Upload Parts)**：将大文件分割成多个部分。为每个部分调用`UploadPart` API（通常结合预签名URL）来并行上传。
3.  **完成 (Complete)**：所有部分上传成功后，调用`CompleteMultipartUpload` API，并提供所有部分的编号和ETag列表，S3将根据这些信息在服务器端完成对象的合并。

当与预签名URL结合使用时，分段上传允许应用程序以一种安全、有时限的方式上传大对象，而无需在客户端存储或暴露私有的存储桶凭证。

以下Lambda函数示例展示了如何代表Web或移动应用启动一个分段上传：

```javascript
// 使用AWS SDK for JavaScript v2
// 调用S3的createMultipartUpload方法来初始化一个分段上传任务
const multipartUpload = await s3.createMultipartUpload(multipartParams).promise();

// 成功后，返回一个包含fileId (即UploadId) 和 fileKey (即对象键) 的响应
// 这些信息将由前端用于后续的分片上传
return {
    statusCode: 200,
    body: JSON.stringify({
        fileId: multipartUpload.UploadId, // 分段上传的唯一标识符
        fileKey: multipartUpload.Key,       // 在S3中存储的对象键名
      }),
    headers: {
      'Access-Control-Allow-Origin': '*' // 设置CORS头，允许跨域访问
    }
};
```

`UploadId`是后续上传每个分片和完成整个上传过程所必需的关键标识。

### 3.2. 使用S3预签名URL（Presigned URLs）保障上传安全

Web或移动应用需要写权限才能将对象上传到S3存储桶。传统方式可能是在应用中存储和使用AWS凭证，但这存在安全风险。

**预签名URL**提供了一种更安全的机制。它允许您在不共享或存储凭证的情况下，授予对S3存储桶的临时访问权限。预签名URL是时间限制的（默认为15分钟），符合安全最佳实践。

**工作流程:**
1.  Web应用调用后端API。
2.  后端服务（如Lambda）使用其拥有的AWS凭证调用S3 API，生成一个有时限的预签名URL。
3.  后端将此URL返回给Web应用。
4.  Web应用在URL过期前，使用该URL直接将对象（或对象的一部分）上传到S3。它无需拥有对S3存储桶的显式写入权限。
5.  一旦预签名URL过期，它将自动失效，无法再被使用。

当与分段上传结合时，可以为每个待上传的部分都生成一个独立的预签名URL。

以下示例展示了如何为指定数量（`parts`）的分片生成一组预签名URL：

```javascript
// multipartParams包含了存储桶名称、对象键名和UploadId
const multipartParams = {
    Bucket: bucket_name,
    Key: fileKey,
    UploadId: fileId,
};

// 创建一个Promise数组，用于并行生成所有分片的预签名URL
const promises = [];

for (let index = 0; index < parts; index++) {
    promises.push(
        // 调用getSignedUrlPromise为每个分片生成一个预签名URL
        s3.getSignedUrlPromise("uploadPart", {
            ...multipartParams,
            PartNumber: index + 1, // 分片编号，从1开始
            Expires: parseInt(url_expiration) // URL的有效期（秒）
        }),
    );
}

// 等待所有Promise完成，获取所有预签名URL
const signedUrls = await Promise.all(promises);
```
在调用`getSignedUrlPromise`之前，客户端必须已经通过`CreateMultipartUpload`获取了`UploadId`。更多信息请参阅[生成预签名URL以共享对象](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)。

### 3.3. 使用传输加速（Transfer Acceleration）降低延迟

通过启用S3传输加速，应用程序可以利用[Amazon CloudFront](https://aws.amazon.com/cloudfront/)的全球分布式边缘站点网络。当与分段上传结合时，每个文件部分都可以自动上传到离用户最近的边缘站点，从而显著减少网络延迟，加快上传速度。

**使用前提:**
- 必须在S3存储桶上启用传输加速功能。
- 启用后，使用特定的加速终端节点（endpoint）进行访问，格式为 `bucketname.s3-acceleration.amazonaws.com` 或 `bucketname.s3-accelerate.dualstack.amazonaws.com`（用于IPv6）。

您可以使用AWS官方的[速度对比工具](http://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html)来测试从您所在位置使用传输加速所带来的效益。

以下示例展示了如何使用AWS CDK和TypeScript创建一个启用了传输加速的S3存储桶：

```typescript
// 使用AWS CDK (Cloud Development Kit) 定义S3存储桶
const s3Bucket = new s3.Bucket(this, "document-upload-bucket", {
      bucketName: "BUCKET-NAME", // 存储桶名称
      encryption: BucketEncryption.S3_MANAGED, // 启用S3管理的服务器端加密
      enforceSSL: true, // 强制使用SSL/TLS传输
      transferAcceleration: true, // 关键：在此处启用传输加速
      removalPolicy: cdk.RemovalPolicy.DESTROY // 定义当CDK栈被删除时，存储桶的移除策略
    });
```

在S3存储桶上激活传输加速后，后端应用程序可以通过在初始化S3 SDK时设置相应选项，来生成启用了传输加速的预签名URL：

```javascript
// 初始化S3服务客户端时，设置useAccelerateEndpoint为true
// 这样所有后续通过此客户端生成的URL都将是加速终端节点
s3 = new AWS.S3({useAccelerateEndpoint: true});
```

Web或移动应用随后便可使用这些生成的预签名URL来上传文件分片，自动享受加速效果。更多信息请参阅[S3传输加速](https://aws.amazon.com/s3/transfer-acceleration/)。

## 4. 生产化部署与测试

### 4.1. 部署测试解决方案

要搭建和运行本博客中概述的测试，您需要：
- 一个AWS账户。
- 安装并配置好AWS CLI。
- 安装并引导（bootstrap）好AWS CDK。
- 从此Git[仓库](https://github.com/aws-samples/amazon-s3-multipart-upload-transfer-acceleration)部署后端和前端解决方案。
- 一个足够大的测试文件，建议至少100MB。

**部署后端:**

1.  将[仓库](https://github.com/aws-samples/amazon-s3-multipart-upload-transfer-acceleration)克隆到本地。
2.  进入`backendv2`目录，运行以下命令安装所有依赖：
    ```bash
    npm install
    ```
3.  使用CDK将后端部署到AWS：
    ```bash
    cdk deploy --context env="randnumber" --context whitelistip="xx.xx.xxx.xxx"
    ```
    - 您可以使用一个额外的上下文变量`urlExpiry`来设置S3预签名URL的特定过期时间（秒），默认值为300秒。
    - 一个名为`document-upload-bucket-randnumber`的新S3存储桶将被创建。
    - `whitelistip`值将API Gateway的访问权限限制在指定的IP地址。

部署完成后，请记下输出的API Gateway终端节点URL。

**部署前端:**

1.  进入`frontend`目录，安装依赖：
    ```bash
    npm install
    ```
2.  要从浏览器启动前端应用，运行：
    ```bash
    npm run start
    ```

### 4.2. 测试应用程序

[![测试应用程序界面](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-1.png)](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-1.png)

**测试步骤:**

1.  在`frontend`目录中运行`npm run start`启动用户界面。
2.  在**API URL**文本框中输入之前记下的API Gateway地址。
3.  **Part Size**: 选择每个上传分片的最大尺寸（最小为5MB）。
4.  **Concurrency**: 选择并行上传的数量。
    - **注意**: 最佳分片大小取决于您的可用带宽、TCP窗口大小和重试时间要求。同时，Web浏览器对同一服务器的并发连接数有限制，指定过大的并发数可能会在浏览器端导致阻塞。
5.  **Use Transfer Acceleration**: 勾选此项以启用传输加速。
6.  **Choose File**: 选择一个用于上传的测试文件。
7.  在**Monitor**部分观察上传测试文件所需的总时间。

您可以尝试不同的分片大小、并行上传数、是否使用传输加速以及不同大小的测试文件，来观察它们对总上传时间的影响。您也可以使用浏览器的开发者工具来获取更深入的洞察。

### 4.3. 测试结果

以下测试具有以下共同特征：
- S3存储桶位于美东（弗吉尼亚北部）区域 (us-east-1)。
- 客户端的平均上传速度为79 Mbps。
- 使用Firefox浏览器上传一个485MB的文件。

#### 测试1 – 单部分上传，不使用传输加速

为了建立一个基准，我们首先不使用传输加速，并以单一部分上传整个文件。这模拟了不使用分段上传优化的大文件上传。基准测试结果为**72秒**。

[![单部分上传，不使用传输加速](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-2.png)](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-2.png)

#### 测试2 – 单部分上传，使用传输加速

下一个测试在保持单部分上传（无分段上传优势）的同时，启用了传输加速。结果为**43秒**（比基准快40%）。

[![单部分上传，使用传输加速](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-3.png)](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-3.png)

#### 测试3 – 分段上传，不使用传输加速

此测试使用分段上传，将测试文件分割成5MB的分片，并设置最多6个并行上传。传输加速被禁用。结果为**45秒**（比基准快38%）。

[![分段上传，不使用传输加速](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-4.png)](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-4.png)

#### 测试4 – 分段上传，并使用传输加速

在此测试中，我们将文件分割成5MB的分片，最多6个并行上传，并为每次上传启用传输加速。结果为**28秒**（比基准快61%）。

[![分段上传，并使用传输加速](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/02/22/s3-5.png)](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d4s3-5.png)

#### 结果汇总

下表总结了所有测试结果。

| 是否使用分段上传 | 是否使用传输加速 | 上传时间 | 性能提升（对比基准） |
| :--------------- | :----------------- | :------- | :------------------- |
| 否                 | 否                 | 72秒     | 0%                   |
| 否                 | 是                 | 43秒     | 40%                  |
| 是                 | 否                 | 45秒     | 38%                  |
| **是**             | **是**             | **28秒** | **61%**              |

## 5. 结论与问答

### 5.1. 结论

本文展示了Web和移动应用如何通过结合使用**预签名URL**和**分段上传**，以一种安全且高效的方式将大对象上传到Amazon S3。

此外，开发者可以利用**传输加速**来进一步降低网络延迟并加速对象上传。当分段上传与传输加速结合使用时，上传时间最多可减少**61%**，极大地改善了用户体验。

### 5.2. 常见问答 (Q&A)

**Q1: 分片大小（Part Size）应该如何选择？**
A1: 最佳分片大小没有固定答案，它取决于多种因素，包括可用网络带宽、网络稳定性以及文件总大小。

- **网络好、文件大**: 可以选择较大的分片大小（如100MB或更大），以减少API调用总数。
- **网络差、易中断**: 选择较小的分片大小（如5MB到20MB），可以使失败后的重传成本更低。
- **经验法则**: 从16MB或32MB开始测试，然后根据实际性能进行调整。S3分段上传要求除最后一个分片外，其他分片最小为5MB。

**Q2: 为什么不直接在客户端使用AWS SDK上传，而是通过API Gateway和Lambda？**
A2: 主要出于安全考虑。直接在客户端（浏览器或移动App）使用AWS凭证会将其暴露给最终用户，极易泄露。通过API Gateway + Lambda的后端代理模式，可以将AWS凭证安全地存储在后端，前端仅通过调用API获取临时的、权限受限的预签名URL来完成上传，遵循了最小权限原则。

**Q3: 传输加速是否总是能提升速度？**
A3: 传输加速对于地理位置远离S3存储桶所在区域的用户效果最明显。它通过将数据上传到就近的AWS边缘节点来减少公网传输距离。如果您的用户和S3存储桶在同一地理区域，或者网络条件本身已经非常好，那么加速效果可能不明显，甚至在极少数情况下会因为额外的路由步骤而略微变慢。建议使用官方的速度对比工具进行评估。

## 6. 参考文档

- [Amazon S3 Multipart Upload Overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Using presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Amazon S3 Transfer Acceleration](https://aws.amazon.com/s3/transfer-acceleration/)
- [Best practices design patterns: optimizing Amazon S3 performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [Cross-Region Access with S3 Transfer Acceleration](https://dev.to/imsushant12/cross-region-access-with-s3-transfer-acceleration-7ng)
- [Sample Application GitHub Repository](https://github.com/aws-samples/amazon-s3-multipart-upload-transfer-acceleration)
