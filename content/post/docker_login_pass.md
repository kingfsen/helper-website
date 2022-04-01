---
title: "Docker Login登录凭证安全存储"
date: 2019-05-11T14:10:54+08:00
description: "详细介绍Docker登录用户名密码管理方式"
Tags:
- docker
Categories:
- docker
---

Docker利用docker login命令来校验用户镜像仓库的登录凭证，实际并不是真正意义上的登录(Web Login)，仅仅是一种登录凭证的试探校验，如果用户名密码正确，Docker则会把用户名、密码
以及仓库域名等信息进行base64编码保存在Docker的配置文件中，在Linux中文件路径是$HOME/.docker/config.json。


登录Docker官方镜像仓库

```docker
[root@vm ~]# docker login -u lovemm -p mylovemm520
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

指定域名登录其它仓库

```docker
[root@vm ~]# docker login hub.test.company.com
Username: lovemm
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

查看/root/.docker/config.json文件(文件中数据非真实数据)

```docker
[root@vm ~]# cat .docker/config.json
{
        "auths": {
                "https://index.docker.io/v1/": {
                        "auth": "zZW46a2luluZ2ZzZW4xMDI2Z2Za2"
                },
                "hub.test.company.com": {
                        "auth": "0xVWYWRtaW46zzOKhkaOeVpaGJ3bz0YUmhjbmt0VFds="
                }
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/18.09.2 (linux)"
        }
}
```

用户名密码可直接通过如下命令解码为明文

```bash
echo 'zZW46a2luluZ2ZzZW4xMDI2Z2Za2' | base64 --decode
```

从config.json数据结构可知，Docker针对每一个镜像仓库，只会保存最近一次有效的用户名密码，之后执行`docker login $domain`会直接使用config.json中对应域名的用户名密码进行登录。
当处理完毕之后，可以执行`docker logout hub.test.company.com`将指定仓库的用户登录凭证从config.json中删除。

```docker
[root@vm ~]# docker logout 
Removing login credentials for https://index.docker.io/v1/
```

Docker直接将仓库的用户名密码明文保存在配置文件中非常不安全，除非用户每次在与镜像仓库交互完成之后手动执行`docker logout`删除，这种明文密码很容易被他人窃取，
Docker也考虑到这一点，针对不同的平台，其提供了不同的辅助工具将仓库的登录凭证保存到其他安全系数高的存储中。

- D-Bus Secret Service
- Apple macOS keychain
- Microsoft Windows Credential Manager
- [pass](https://www.passwordstore.org/)

以上辅助工具均可在[docker github](https://github.com/docker/docker-credential-helpers/releases)下载。

![docker_helper](/blog/docker_login_process/001.png)

docker在Linux平台上支持pass、secret service，其中pass依赖了gpg，下面在以CentOS系统为例，将Docker的Credential store切换到pass存储，不再写入config.json文件中。

- **检查gpg**

    ```bash
      [root@vm ~]# gpg --version
      gpg (GnuPG) 2.0.22
      libgcrypt 1.5.3
      Copyright (C) 2013 Free Software Foundation, Inc.
      License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
      This is free software: you are free to change and redistribute it.
      There is NO WARRANTY, to the extent permitted by law.

      Home: ~/.gnupg
      Supported algorithms:
      Pubkey: RSA, ?, ?, ELG, DSA
      Cipher: IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256, TWOFISH,
              CAMELLIA128, CAMELLIA192, CAMELLIA256
      Hash: MD5, SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
      Compression: Uncompressed, ZIP, ZLIB, BZIP2
    ```
    
- **安装pass**
   
     ```bash
     yum install -y pass
     ```
     
     安装完成之后通过如下命令验证
     
     ```bash
     [root@vm ~]# pass version
      ============================================
      = pass: the standard unix password manager =
      =                                          =
      =                  v1.7.3                  =
      =                                          =
      =             Jason A. Donenfeld           =
      =               Jason@zx2c4.com            =
      =                                          =
      =      http://www.passwordstore.org/       =
      ============================================
     ```
     
     执行pass的基本命令
     
     ```bash
     [root@vm ~]# pass
      Error: password store is empty. Try "pass init".
      [root@vm ~] pass init
      Usage: pass init [--path=subfolder,-p subfolder] gpg-id...
     ```
     
     从上面的输出信息可知，pass init需要一个gpg-id，通过gpg生成一个key即可。
     
- **gpg生成key**

      先查看是否已有gpg key
      
      ```bash
      [root@vm ~]# gpg --list-keys
      ```   
      
      当前没有生成任何key，下面通过命令生成一个key，生成key是一个交互过程，需要输入key类型、长度、过期时间等相关信息，请记住设置的操作密码，类似Java中的keystore也有个密码。
      
      ```bash
        [root@vm ~]# gpg --gen-key
        gpg (GnuPG) 2.0.22; Copyright (C) 2013 Free Software Foundation, Inc.
        This is free software: you are free to change and redistribute it.
        There is NO WARRANTY, to the extent permitted by law.

        Please select what kind of key you want:
           (1) RSA and RSA (default)
           (2) DSA and Elgamal
           (3) DSA (sign only)
           (4) RSA (sign only)
        Your selection? 1
        RSA keys may be between 1024 and 4096 bits long.
        What keysize do you want? (2048) 4096
        Requested keysize is 4096 bits
        Please specify how long the key should be valid.
                 0 = key does not expire
              <n>  = key expires in n days
              <n>w = key expires in n weeks
              <n>m = key expires in n months
              <n>y = key expires in n years
        Key is valid for? (0) 0
        Key does not expire at all
        Is this correct? (y/N) y

        GnuPG needs to construct a user ID to identify your key.

        Real name: kingfsen
        Email address: kingfsen@gmail.com
        Comment: 
        You selected this USER-ID:
            "kingfsen <kingfsen@gmail.com>"
        Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
        You need a Passphrase to protect your secret key.
        ┌─────────────────────────────────────────────────────┐
        │ Enter passphrase                                    │
        │                                                     │
        │                                                     │
        │ Passphrase **************__________________________ │
        │                                                     │
        │       <OK>                             <Cancel>     │
        └─────────────────────────────────────────────────────┘
         ─────────────────────────────────────────────────────┐
        │ Please re-enter this passphrase                     │
        │                                                     │
        │ Passphrase **************__________________________ │
        │                                                     │
        │       <OK>                             <Cancel>     │
        └─────────────────────────────────────────────────────┘
      ```
      
      密码输入之后，很快就会生成key了。
      
      ```bash
      We need to generate a lot of random bytes. It is a good idea to perform
      some other action (type on the keyboard, move the mouse, utilize the
      disks) during the prime generation; this gives the random number
      generator a better chance to gain enough entropy.
      We need to generate a lot of random bytes. It is a good idea to perform
      some other action (type on the keyboard, move the mouse, utilize the
      disks) during the prime generation; this gives the random number
      generator a better chance to gain enough entropy.
      gpg: key 5BAC1C87 marked as ultimately trusted
      public and secret key created and signed.

      gpg: checking the trustdb
      gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
      gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
      pub   4096R/5BAC1C87 2019-05-11
            Key fingerprint = E3E2 B354 73DD D511 3059  E213 1A08 85F9 5BAC 1C87
      uid                  kingfsen <kingfsen@gmail.com>
      sub   4096R/69AA61E2 2019-05-11
      ```
      
      > * 注意：如果系统上没有安装rng-tools，gpg在生成key的最后这一步会卡住，无法进行操作了。
      
      安装rng-tools包
      
      ubuntu
      
      ```bash
      # apt-get install rng-tools
      # rng -r /dev/urandom
      ```
      
      centos
      
      ```bash
      # yum install -y rng-tools
      # rngd -r /dev/urandom
      ```
      
- **pass初始化**

    查看gpg已生成的key
    
    ```bash
      [root@vm ~]# gpg --list-keys
      /root/.gnupg/pubring.gpg
      ------------------------
      pub   4096R/5BAC1C87 2019-05-11
      uid                  kingfsen <kingfsen@gmail.com>
      sub   4096R/69AA61E2 2019-05-11
    ```
    
    执行命令初始化pass
    
    ```bash
    [root@vm ~]# pass init "5BAC1C87"
    Password store initialized for 5BAC1C87
    ```
    
    执行pass insert命令验证是否成功
    
    ```bash
    [root@vm ~]# pass init 5BAC1C87
    Password store initialized for 5BAC1C87
    You have new mail in /var/spool/mail/root
    [root@vm ~]# pass insert Gmail/kingfsen@gmail.com
    Enter password for Gmail/kingfsen@gmail.com: 
    Retype password for Gmail/kingfsen@gmail.com: 
    [root@vm ~]# pass show Gmail/kingfsen@gmail.com
     ────────────────────────────────────────────────────────────────────────────────────┐
    │ Please enter the passphrase to unlock the secret key for the OpenPGP certificate:  │
    │ "kingfsen <kingfsen@gmail.com>"                                                    │
    │ 4096-bit RSA key, ID 69AA61E2,                                                     │
    │ created 2019-05-11 (main key ID 5BAC1C87).                                         │
    │                                                                                    │
    │                                                                                    │
    │ Passphrase **************_________________________________________________________ │
    │                                                                                    │
    │            <OK>                                                  <Cancel>          │
    └────────────────────────────────────────────────────────────────────────────────────┘
    sun1026
    ```
    
    pass已经可以用于管理敏感信息了。
    
- **安装Docker Credential辅助工具**

    ```bash
    [root@vm ~]# wget https://github.com/docker/docker-credential-helpers/releases/download/v0.6.0/docker-credential-pass-v0.6.0-amd64.tar.gz 
    [root@vm ~]# tar -xf docker-credential-pass-v0.6.0-amd64.tar.gz 
    [root@vm ~]# chmod +x docker-credential-pass
    [root@vm ~]# mv docker-credential-pass /usr/local/bin/
    [root@vm ~]# docker-credential-pass 
    Usage: docker-credential-pass <store|get|erase|list|version>
    [root@vm ~]# docker-credential-pass version
    0.6.0
    ```
    
- **修改Docker配置**

    清空.docker/config.json文件内容，然后将下面配置写入config.json文件中，注意credsStore是各辅助安装包名字的尾缀
    
    ```bash
    {
        "credsStore": "pass"
    }
    ```
    
    config.json保存之后，执行docker login操作试试。
    
    ```bash
    [root@vm ~]# docker login hub.test.company.com -u 2000014559
    Password: 
    Error saving credentials: error storing credentials - err: exit status 1, out: `pass store is uninitialized`
    ```
    
    报错了，这是因为还没有初始化docker password store，辅助工具并不会自动初始化，需要手动操作。
    
- **初始化docker password store**

    执行pass insert插入docker password store条目，密码：pass is initialized
    
    ```bash
    [root@vm ~]# pass insert docker-credential-helpers/docker-pass-initialized-check
    An entry already exists for docker-credential-helpers/docker-pass-initialized-check. Overwrite it? [y/N] y
    Enter password for docker-credential-helpers/docker-pass-initialized-check: 
    Retype password for docker-credential-helpers/docker-pass-initialized-check: 
    ```
    
    通过如下命令验证是否初始化成功，请注意输出结果，pass show执行的过程中无需输入密码。
    
    ```bash
    [root@vm ~]# pass show docker-credential-helpers/docker-pass-initialized-check
    pass is initialized
    [root@vm ~]# docker-credential-pass list
    {}
    ```
    
    再次执行docker login登录镜像仓库，同时查看$HOME/.docker/config.json文件内容。
    
    ```bash
    [root@vm ~]# docker login hub.test.company.com -u 2000014559
    Password: 
    Login Succeeded
    You have new mail in /var/spool/mail/root
    [root@vm ~]# cat .docker/config.json
    {
            "auths": {
                    "hub.test.company.com": {}
            },
            "HttpHeaders": {
                    "User-Agent": "Docker-Client/18.09.2 (linux)"
            },
            "credsStore": "pass"
    }
    ```
    
    用户名、密码此时并未保存在config.json中，而是保存在加密文件中了。
    
    ```bash
    [root@vm ~]# docker-credential-pass list
    {"hub.test.company.com":"2000014559"}
    ```
    
    ```bash
    [root@vm ~]# pass
    Password Store
    ├── docker-credential-helpers
    │   └── aHViLmtjrzXSe3l1bi5jb20=
    │       └── 2000014559
    └── Gmail
        └── kingfsen@gmail.com
    ```
    aHViLmtjrzXSe3l1bi5jb20=就是仓库域名的base64编码值，查看原始密码信息。
    
    ```bash
    [root@vm ~]# pass show docker-credential-helpers/aHViLmtjrzXSe3l1bi5jb20=/2000014559
    ```
    
    保存密码文件路径
    
    ```bash
    [root@vm aHViLmtjrzXSe3l1bi5jb20=]# pwd
    /root/.password-store/docker-credential-helpers/aHViLmtjrzXSe3l1bi5jb20=
    [root@vm aHViLmtjrzXSe3l1bi5jb20=]# ls
    2000014559.gpg
    ```
    