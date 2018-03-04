# Jenkins Git插件离线安装

Jenkins部署的服务器没有连接外网时，可以通过下载 `.hpi` 文件并上传至jenkins以离线安装插件。

插件下载地址：[http://updates.jenkins-ci.org/download/plugins/](http://updates.jenkins-ci.org/download/plugins/)

插件之间存在依赖关系，缺失依赖时会导致安装失败，不过我们可以通过报错信息查看该插件安装失败是因为缺少了哪些插件，再尝试安装缺失的插件。

以下为安装Jenkins Git插件时可行的安装顺序之一（由上至下）：

- structs

- credentials

- display-url-api

- mailer

- scm-api

- workflow-step-api

- workflow-scm-step

- workflow-api

- script-security

- junit

- matrix-project

- ssh-credentials

- apache-httpcomponents-client-4-api

- jsch

- git-client

- git

