---
layout:     post
title:      Matter Test PKI Guide
subtitle:   Matter Test PKI Guide
date:       2024-01-15
author:     ShuHua
header-img: img/MatterSmartHome22_large.jpg
catalog: true
tags:
    - Matter
    - 开发技巧
---

> Matter认证需要提供Test Net上线的PAA以及颁发的CD和DAC之类的相关生成资料，下面的说明一下如何生成以及使用。
 

## 名词解释：

| 缩写 | 全称                                       |
|------|--------------------------------------------|
| DAC  | 设备认证证书（Device Authentication Certificate）  |
| PAI  | 产品认证证书（Product Authentication Certificate）  |
| PAA  | 产品认证机构证书（Product Authentication Agency Certificate） |
| CD   | 认证声明（Certification Declaration）         |



## 环境搭建

搭建matter的编译环境： https://project-chip.github.io/connectedhomeip-doc/QUICK_START.html

	cd ~/matter/connectedhomeip
	cp ./credentials/test/attestation/Chip-Test-PAA-NoVID-Cert.pem ./out/debug/
	cp ./credentials/test/attestation/Chip-Test-PAA-NoVID-Key.pem ./out/debug/

### 安装依赖库
	
	pip install --upgrade cryptography


### 生成spake2p
	
	cd src/credentials
	source ../../scripts/activate.sh
	cd connectedhomeip/
	gn gen out/host
	ninja -C out/host

	gn gen out/debug
	ninja -C out/debug


### 设置spake2p路径：第一个命令不需要修改，需要将第二个命令中的地址替换成自己的地址

	export PATH=$PATH:path/to/connectedhomeip/out/host
	export PATH=$PATH:~/shear/Matter/connectedhomeip/out/host
	export PATH=$PATH:/home/zeng/Desktop/dac_enc/connectedhomeip-1/out/host


## 注意事项
统一介绍：
	--valid-from：从什么时间开始。注意：下面所有开始时间都应该是同一个值。
	--lifetime：证书截止日期，以天为单位。4294967295这个值是没有定义证书截止日期。注意：下面所有截止日期都应该是同一个值。
	--subject-vid：产商ID。
	--subject-pid：产品ID。

subject-cn  "Matter Development PAA 01" 只是举个例子，建议有实际意义，比如说"subject-cn  "BouffaloLab Matter Development PAA"

VID以及PID必须是完整的16bit即使是0也要写全，举例子：VID 0xFFF1  PID 0x0100   PID是0100，不能缺省为100.以下生成的所有资料仅可用于设备认证和Test Net。Test PAA 放置以及Model 放置需要联系dcl-admin@csa-iot.org，一般来说在slack的csa-dcl-testnet-enrollment里面联系 JC Pacheco比较快速。

  

### 可用于生成产品认证机构 (PAA) 证书和私钥的示例命令(这条命令对生成证书无用)

	./chip-cert gen-att-cert --type a --subject-cn "Matter Development PAA 01" --valid-from "2022-05-28 14:23:43" --lifetime 4294967295 --out-key Chip-PAA-Key.pem --out Chip-PAA-Cert.pem


### 可以使用 PAA 证书/密钥 输出来生成产品证明(PAI) 证书和私钥：需要产商ID(VID)

	./chip-cert gen-att-cert --type i --subject-cn "Matter Development PAI 01" --subject-vid 0xFFF1 --valid-from "2022-05-28 14:23:43" --lifetime 4294967295 --ca-key Chip-Test-PAA-NoVID-Key.pem --ca-cert Chip-Test-PAA-NoVID-Cert.pem --out-key Chip-PAI-Key.pem --out Chip-PAI-Cert.pem


	./chip-cert gen-att-cert --type i --subject-cn "Matter Development PAI 01" --subject-vid  0xFFF1 --valid-from "2022-05-28 14:23:43" --lifetime 4294967295 --ca-key Chip-Test-PAA-NoVID-Key.pem --ca-cert Chip-Test-PAA-NoVID-Cert.pem --out-key Chip-PAI-Key.pem --out Chip-PAI-Cert.pem


### 生成的 PAI 证书/密钥可用于签署多个设备证明证书 (DAC)：需要产商ID（VID）和产品ID（PID）

	./chip-cert gen-att-cert --type d --subject-cn "Matter Development DAC 01" --subject-vid  0xFFF1  --subject-pid 0100--valid-from "2022-05-28 14:23:43" --lifetime 4294967295 --ca-key Chip-PAI-Key.pem --ca-cert Chip-PAI-Cert.pem --out-key Chip-DAC-Key.pem --out Chip-DAC-Cert.pem

	./chip-cert gen-att-cert --type d --subject-cn "Matter Development DAC 01" --subject-vid  0xFFF1 --subject-pid  0100 --valid-from "2022-05-28 14:23:43" --lifetime 4294967295 --ca-key Chip-PAI-Key.pem --ca-cert Chip-PAI-Cert.pem --out-key Chip-DAC-Key.pem --out Chip-DAC-Cert.pem

### 验证生成的证书是否符合规范

	./chip-cert validate-att-cert --dac Chip-DAC-Cert.pem --pai Chip-PAI-Cert.pem --paa Chip-Test-PAA-NoVID-Cert.pem

### 将生成的PAI、DAC的证书、私钥通过下述三条命令将pem格式转化为der格式

	./chip-cert convert-key Chip-DAC-Key.pem Chip-Test-DAC-0xFFF1 -0100-0008-Key.der --x509-der

### 生成CD(认证声明)文件 该文件需要产商ID（VID）和产品ID（PID）

	./chip-cert gen-cd --key ../../credentials/test/certification-declaration/Chip-Test-CD-Signing-Key.pem --cert ../../credentials/test/certification-declaration/Chip-Test-CD-Signing-Cert.pem  --out chip-CD-0xFFF1-0100.der --format-version 1 --vendor-id 0x130D --product-id 0x0100 --device-type-id  0x130D  --certificate-id  MAT20141ZB330001-24 --security-level 0 --security-info 0 --version-number 9876 --certification-type 1

### 查看CD(认证声明)里面的细节内容

	./src/credentials/out/chip-cert print-cd credentials/test/certification-declaration/Chip-Test-CD-${VID}-${PID}.der

# 可能用的openssl 命令

	$ openssl ec -noout -text -in credentials/test/attestation/test-DAC-${VID}-${PID}-key.pem
	read EC key
	Private-Key: (256 bit)
	priv:
		c9:f2:b3:04:b2:db:0d:6f:cd:c6:be:f3:7b:76:8d:
		8c:01:4e:0b:9e:ce:3e:72:49:3c:0e:35:63:7c:6c:
		6c:d6
	pub:
		04:4f:93:ba:3b:bf:63:90:73:98:76:1e:af:87:79:
		11:e6:77:e8:e2:df:a7:49:f1:7c:ac:a8:a6:91:76:
		08:5b:39:ce:6c:72:db:6d:9a:92:b3:ba:05:b0:e8:
		31:a0:bf:36:50:2b:5c:72:55:7f:11:c8:01:ff:3a:
		46:b9:19:60:28
	ASN1 OID: prime256v1
	NIST CURVE: P-256

	openssl x509 -noout -text -in credentials/test/attestation/test-DAC-${VID}-${PID}-cert.pem

	Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2875998130766646679 (0x27e9990fef088d97)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: CN = Matter Test PAI, 1.3.6.1.4.1.37244.2.1 = hexVendorId
        Validity
            Not Before: Jun 28 14:23:43 2021 GMT
            Not After : Dec 31 23:59:59 9999 GMT
        Subject: CN = Matter Test DAC 0, 1.3.6.1.4.1.37244.2.1 = hexVendorId, 1.3.6.1.4.1.37244.2.2 = hexProductId
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:4f:93:ba:3b:bf:63:90:73:98:76:1e:af:87:79:
                    11:e6:77:e8:e2:df:a7:49:f1:7c:ac:a8:a6:91:76:
                    08:5b:39:ce:6c:72:db:6d:9a:92:b3:ba:05:b0:e8:
                    31:a0:bf:36:50:2b:5c:72:55:7f:11:c8:01:ff:3a:
                    46:b9:19:60:28
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Subject Key Identifier:
                21:0A:CA:B1:B6:5F:17:65:D8:61:19:73:84:1A:9D:52:81:19:C5:39
            X509v3 Authority Key Identifier:
                37:7F:24:9A:73:41:4B:16:6E:6A:42:6E:F5:E8:89:FB:75:F8:77:BB
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:45:02:20:38:8f:c5:0d:3e:90:95:dd:7d:7c:e9:5a:05:19:
        1f:2d:14:08:a3:d7:0e:b5:15:6d:d3:b0:0b:f7:b8:28:4d:bf:
        02:21:00:d4:05:30:43:a6:05:00:0e:b9:99:0d:34:3d:75:fe:
        d3:c1:4e:73:ff:e7:05:64:7a:62:8d:2d:38:8f:fd:4d:ad
