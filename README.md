

> 在日常开发时，如果有文件上传下载的需求（比如用户头像），但是又不想使用对象存储，那么自己搭建一个 MinIO 服务器是一个比较简单的解决方案。


MinIO 是一个基于 Apache License v2\.0 开源协议的对象存储服务。它兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从 几kb 到最大 5T 不等。


MinIO是一个非常轻量的服务,可以很简单的和其他应用的结合，类似 NodeJS, Redis 或者 MySQL。




---


MinIO 中文网站：[https://www.minio.org.cn/docs/minio/linux/operations/installation.html](https://github.com)


## 部署 MinIO 并修改 minioadmin 的账户密码


本文主要介绍在 Ububtu 上单节点单硬盘部署 MinIO，步骤如下：
下载 MinIO 服务器



```
wget https://dl.min.io/server/minio/release/linux-amd64/minio

```

为 MinIO 二进制文件添加执行权限



```
chmod +x minio

```

在合适的位置创建一个文件夹，用于存储 MinIO 上传的文件数据



```
# 在根目录创建 minio-data 文件夹，存储 MinIO 上传的文件数据
mkdir ~/minio-data

```

安装 MinIO。将 MinIO 的二进制文件移到 `/usr/local/bin/`目录下，以使其全局可用



```
sudo mv minio /usr/local/bin/

```

使用持久化环境变量作为 MinIO console 的登录账户和密码。


编辑 `.bashrc`文件，这里使用 nano。



```
nano ~/.bashrc

```

在文件的最后加上环境变量



```
export MINIO_ROOT_USER=newrootuser
export MINIO_ROOT_PASSWORD=newrootpassword

```

重新加载`.bashrc`以使更改生效



```
source ~/.bashrc

```

启动 MinIO



```
nohup minio server --secure ~/minio-data

```

这里使用`nohup`确保会话关闭之后 MinIO 不会停止，也可以使用`screen`等会话技术，或者将 MinIO 作为一个服务启动。这里不做过多介绍。


启动 MinIO 之后，在浏览器访问 ip\+9000 端口即可访问 MinIO 的 Web 控制台。如果访问不了，**请先检查 MinIO 的两个端口是否已经开放**，一个是 MinIO 的 WebUI 端口，这个是随机的；一个是 MinIO 的 API 端口，这个固定是 9000。


在浏览器访问 ip\+9000 实际会跳转到 WebUI 的端口。但是在使用 API 上传下载文件需要使用 9000 端口。




---


## 给 MinIO 配置域名


如果不想直接暴露 MinIO 的地址和端口，则可以使用 Nginx 给 MinIO 配置域名访问。
在此之前，您需要先准备一个已备案的域名并解析到当前服务器。


步骤如下：


先安装 Nginx （如果没有的话）



```
sudo apt-get update
sudo apt-get install nginx

```

一般不建议直接更改位于`/etc/nginx/nginx.conf`的 nginx 主配置文件。采用如下的配置方式：


在`/etc/nginx/sites-available/`创建新的配置文件，可以直接以当前配置的域名未文件名。我这里由于需要配置多个域名，文件名叫做`minio.conf`



```
cd /etc/nginx/sites-available/
touch minio.conf

```

书写配置文件



```
nano minio.conf

```

配置文件示例



```
# WebUI 配置
server {
    listen 80;
    server_name yourdomain.com; # 替换为您的域名

    location / {
        proxy_pass http://localhost:44366;  # 替换为实际的端口
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# API 配置
server {
    listen 80;
    server_name api.yourdomain.com; # 替换为您的API域名

    location / {
        proxy_pass http://localhost:9000;   # 替换为实际的端口
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

启用新创建的配置文件。将`/etc/nginx/sites-available/minio.conf`链接到`/etc/nginx/sites-enabled/`目录



```
sudo ln -s /etc/nginx/sites-available/minio.conf /etc/nginx/sites-enabled/

```

可以先检查一下 nginx，确认没有语法错误



```
nginx -t

```

重启 nginx 服务



```
sudo systemctl restart nginx

```

此时，在本机浏览器上应该可以用域名访问 Minio console 了。




---


## MinIO 调用示例


### 在 SpringBoot 中调用


先添加依赖

```
<dependencies>
    <dependency>
        <groupId>io.miniogroupId>
        <artifactId>minioartifactId>
        <version>8.4.3version>
    dependency>
dependencies>

```

然后创建一个服务类来实现文件的上传下载



```
import io.minio.*;
import io.minio.messages.Item;

import java.io.InputStream;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.UUID;

@Service
public class MinIOService {

    private final MinioClient minioClient;

    public MinIOService() {
        try {
            // 配置更改成自己的，建议写在配置文件中
            this.minioClient = MinioClient.builder()
                    .endpoint("http://localhost:9000")
                    .credentials("minioadmin", "minioadmin")
                    .build();
        } catch (MinioException e) {
            throw new RuntimeException("Failed to create MinioClient", e);
        }
    }

    public String uploadFile(String bucketName, String objectName, InputStream stream, long size) {
        try {
            minioClient.putObject(bucketName, objectName, stream, size);
            return "File uploaded successfully.";
        } catch (MinioException | IOException e) {
            throw new RuntimeException("File upload failed.", e);
        }
    }

    public InputStream downloadFile(String bucketName, String objectName) {
        try {
            return minioClient.getObject(bucketName, objectName);
        } catch (MinioException | IOException e) {
            throw new RuntimeException("File download failed.", e);
        }
    }
}

```

并为其书写对应的 Controller



```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.InputStreamResource;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;

@RestController
@RequestMapping("/minio")
public class MinIOController {

    @Autowired
    private MinIOService minIOService;

    @PostMapping("/upload")
    public String uploadFile(@RequestParam("file") MultipartFile file) {
        String bucketName = "my-bucket";
        String objectName = UUID.randomUUID().toString();
        try (InputStream stream = file.getInputStream()) {
            return minIOService.uploadFile(bucketName, objectName, stream, file.getSize());
        } catch (IOException e) {
            return "Failed to upload file.";
        }
    }

    @GetMapping("/download/{objectName}")
    public ResponseEntity downloadFile(@PathVariable String objectName) {
        String bucketName = "my-bucket";
        InputStream stream = minIOService.downloadFile(bucketName, objectName);
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + objectName + "\"")
                .body(new InputStreamResource(stream));
    }
}

```

### 在 Flask 中调用


下载依赖

```
pip install minio

```

然后创建一个 Flask 来集成 MinIO



```
from flask import Flask, request, send_file, jsonify
from minio import Minio
from minio.error import S3Error

app = Flask(__name__)

# MinIO配置
minio_client = Minio(
    # 更改成自己的
    "localhost:9000",
    access_key="minioadmin",
    secret_key="minioadmin",
    secure=False
)

@app.route('/upload', methods=['POST'])
def upload_file():
    file = request.files['file']
    bucket_name = 'my-bucket'
    object_name = file.filename
    try:
        with open('/tmp/' + object_name, 'wb') as f:
            f.write(file.read())
        minio_client.fput_object(bucket_name, object_name, '/tmp/' + object_name)
        return jsonify({'message': 'File uploaded successfully'}), 200
    except S3Error as exc:
        return jsonify({'error': str(exc)}), 500

@app.route('/download/', methods=['GET'])
def download_file(object_name):
    bucket_name = 'my-bucket'
    try:
        response = minio_client.get_object(bucket_name, object_name)
        return send_file(
            response.stream,
            as_attachment=True,
            attachment_filename=object_name,
            mimetype=response.headers['content-type']
        )
    except S3Error as exc:
        return jsonify({'error': str(exc)}), 500

if __name__ == '__main__':
    app.run(debug=True)

```

## 推荐阅读


MinIO Plus：https://mp.weixin.qq.com/s/kSkC3X\-SQqo5GzXt\-H66xw

 本博客参考[FlowerCloud机场](https://yunbeijia.com)。转载请注明出处！
