---
title: hexo部署 git相关
date: 2019-03-29 13:53:11
tags:
- hexo
- git
---

今天好囧啊因为好久没更新博客了，更新博客的时候发现部署出现问题。

正常用命令


        hexo g
        hexo d


然后报错

            
        Error: git@github.com: Permission denied (publickey).
        fatal: Could not read from remote repository.

        Please make sure you have the correct access rights
        and the repository exists.



记录一下解决方法：

##### 1. 先自查 public key #####


        ssh -vT git@github.com


结果报错


        ...
        debug1: Trying private key: /Users/ella/.ssh/id_xmss
        debug1: No more authentication methods to try.
        git@github.com: Permission denied (publickey).



##### 2. 去 github settings 里， SSH and GPG keys，新建 SSH key #####


        vim ~/.ssh/id_rsa.pub


把内容粘贴进github new ssh key 里，保存。



##### 3. 检查 "_config.yml" 文件 #####


        deploy:
          type: git
          repository: git@github.com:jinlygenius/jinlygenius.github.io.git
          branch: master


##### 4. 重新测试 #####


        ssh -vT git@github.com



这回返回成功了


        ...
        Hi jinlygenius! You've successfully authenticated, but GitHub does not provide shell access.
        debug1: channel 0: free: client-session, nchannels 1
        Transferred: sent 2684, received 2228 bytes, in 0.8 seconds
        Bytes per second: sent 3305.6, received 2744.0
        debug1: Exit status 1



##### 5. 还是部署出错，需要把 .deploy_git 彻底删了 #####



        rm -rf .deploy_git



##### 6. 终于部署成功了~~ T_T #####



