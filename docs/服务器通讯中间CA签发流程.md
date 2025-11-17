# 服务器通讯中间CA签发流程

在受控工控机（稳定设备）中，root账户具有利用`DreamReflex Server Intermediate CA`签发证书的能力

在其内部的/root/im-pki文件夹中，存储着

- `root-ca.pem` - 根CA的证书，用于创建认证链
- `intermediate-ca.pem` - 中间CA的证书，用于签发
- `intermediate-ca-key.pem.gpg` - 加密后的中间CA的私钥
- `intermediate-ca-config.json` - 中间CA的签发规则，如没有，则新建：

```json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "server": {
        "expiry": "8760h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth"
        ]
      },
      "client": {
        "expiry": "8760h",
        "usages": [
          "signing",
          "key encipherment",
          "client auth"
        ]
      }
    }
  }
}
```

## 签发流程

1. 解密中间CA的私钥

    ```bash
    gpg intermediate-ca-key.pem.gpg
    ```

    输入密码后得到私钥，随后重置GPG

    ```bash
    gpgconf --reload gpg-agent
    ```

2. 准备`server-csr.json`文件用来等价替换CSR

    ```json
    {
        "CN": "console.in.dreamreflex.com",
        "hosts": [
            "console.in.dreamreflex.com",
            "192.168.1.11"
        ],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "DreamReflex",
            "OU": "Server"
            }
        ]
    }

    ```

3. 签发服务器证书：

    ```bash
    cfssl gencert \
    -ca=intermediate-ca.pem \
    -ca-key=intermediate-ca-key.pem \
    -config=intermediate-ca-config.json \
    -profile=server \
    server-csr.json | cfssljson -bare server
    ```

    获得如下文件：
    1. server.csr
    2. server-csr.json
    3. server-key.pem
    4. server.pem

4. 构建 certificate bundle

    ```bash
    cat server.pem intermediate-ca.pem > server-bundle.pem
    ```

    获得如下文件
    - server-bundle.pem

5. 重新加密私钥

    ```bash
    gpg -c intermediate-ca-key.pem && rm intermediate-ca-key.pem
    ```

6. 下载相关文件后清理所有无用文件

    ```bash
    ./pack-server-certs.sh
    ```

## 注意事项

1. 服务器通讯中间CA只允许服务器端签发HTTPS所使用的SSL证书，不能为客户端和mTLS的两端签发证书。
2. 服务器的hosts项目的第一个为接入域名，必须与CommonName一致。

## 附录

1. 自动打包脚本（当脚本不存在时）

    ```bash
    cat > pack-server-certs.sh << 'EOF'
    #!/usr/bin/env bash
    set -e

    mkdir -p certs

    timestamp=$(date +%Y%m%d-%H%M%S)
    zipfile="certs/server-$timestamp.zip"

    files=(
    server.csr
    server-csr.json
    server-key.pem
    server.pem
    server-bundle.pem
    )

    existing=()
    for f in "${files[@]}"; do
    if [ -f "$f" ]; then
        existing+=("$f")
    fi
    done

    if [ "${#existing[@]}" -eq 0 ]; then
    echo "没有找到可以打包的 server 相关文件。"
    exit 1
    fi

    zip "$zipfile" "${existing[@]}"
    rm "${existing[@]}"

    echo "已生成归档：$zipfile"
    echo "已删除原始文件：${existing[*]}"
    EOF

    ```

    赋予执行权限

    ```bash
    chmod +x pack-server-certs.sh
    ```
