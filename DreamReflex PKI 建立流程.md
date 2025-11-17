# DreamReflex PKI 建立流程

该流程依赖Cloudflare的自动化PKI工具。

## 初始化根CA

1. 创建根CA的CSR模板：`root-ca-csr.json`

    ```json
    {
        "CN": "DreamReflex Root CA",
        "key": {
            "algo": "rsa",
            "size": 4096
        },
        "ca": {
            "expiry": "87600h"
        },
        "names": [
            {
            "C":  "CN",
            "ST": "Beijing",
            "L":  "Beijing",
            "O":  "DreamReflex PKI",
            "OU": "Root CA"
            }
        ]
    }
    ```

2. 创建`root-ca-config.json`并填写必要信息

    ```json
    {
        "signing": {
            "default": {
            "expiry": "87600h",
            "usages": [
                "signing",
                "key encipherment",
                "cert sign",
                "crl sign"
            ]
            },
            "profiles": {
            "intermediate": {
                "expiry": "43800h",
                "usages": [
                "signing",
                "key encipherment",
                "cert sign",
                "crl sign"
                ],
                "ca_constraint": {
                "is_ca": true,
                "max_path_len": 0,
                "max_path_len_zero": true
                }
            }
            }
        }
    }
    ```

    - 根 CA 自己有效期 10 年（87600h）
    - 中间 CA 有效期 5 年（43800h）
    - ca_constraint.is_ca=true & usages 里有 cert sign / crl sign → 这是 CA 证书

3. 生成 Root CA（自签）

    ```bash
    cfssl gencert -initca root-ca-csr.json | cfssljson -bare root-ca
    ```

    根据根CA的JSON来自签发根CA证书
    生成

    1. `root-ca.pem` —— 根 CA 证书
    2. `root-ca.csr`—— 根 CA CSR 用于自签
    3. `root-ca-key.pem` —— 根 CA 私钥

    检查证书：

    ```bash
    cfssl certinfo -cert root-ca.pem
    ```

## 签发中间CA

1. 创建`intermediate-app-csr.json`并填写必要信息

    ```json
    {
    "CN": "DreamReflex Application Intermediate CA",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
        "C":  "CN",
        "ST": "Beijing",
        "L":  "Beijing",
        "O":  "DreamReflex",
        "OU": "Application Intermediate CA"
        }
    ]
    }
    ```

2. 开始生成CSR和私钥

    ```bash
    cfssl genkey intermediate-app-csr.json | cfssljson -bare intermediate-app
    ```

    得到：
    - `intermediate-app-key.pem` —— 中间 CA1 私钥（要保护）
    - `intermediate-app.csr` —— 中间 CA1 的 CSR

3. 使用 Root CA 签发中间 CA1 证书

    ```bash
    cfssl sign ^
    -ca ../../root-ca.pem ^
    -ca-key ../../root-ca-key.pem ^
    -config ../../root-ca-config.json ^
    -profile intermediate ^
    intermediate-app.csr | cfssljson -bare intermediate-app
    ```

    获得：
    - `intermediate-app.pem` —— 中间 CA1 证书（由 Root CA 签发

    检查可用性：

    ```abash
    openssl x509 -in intermediate-app.pem -noout -text | grep -A3 "Basic Constraints"
    openssl x509 -in intermediate-app.pem -noout -text | grep -A3 "Key Usage"
    ```

4. 构建中间CA链

    ```bash
    cat intermediate-app.pem ../../root-ca.pem > intermediate-app-chain.pem
    ```
