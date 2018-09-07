[TOC] 
# vendor 常用命令  
## 安装vendor  
>go get -u -v github.com/kardianos/govendor 

## 进入到go项目的目录下  


## 命令格式  
>govendor COMMAND  


## 初始化vendor目录  
>govendor init  

## 命令列表 
|命令|功能|
|:---|:---|
|init|初始化 vendor 目录|
|list|列出所有的依赖包|
|add|添加包到 vendor 目录，如 govendor add +external 添加所有外部包|
|add PKG_PATH| 添加指定的依赖包到 vendor 目录|
|update|从 $GOPATH 更新依赖包到 vendor 目录|
|remove|从 vendor 管理中删除依赖|
|status|列出所有缺失、过期和修改过的包|
|fetch|添加或更新包到本地 vendor 目录|
|sync|本地存在 vendor.json 时候拉去依赖包，匹配所记录的版本|
|get|类似 go get 目录，拉取依赖包到 vendor 目录|  
## status依赖包状态说明：  
|状态|简写|含义|是否需要解决措施|
|:---|:---|:---|:---|
|+local|l|此包已经在当前的工程中，或已经交给vendor管理了|no|
|+external|e|依赖包在gopath了，并不在vendor里|govendor add +e 或者 govendor add 依赖包|
|+std|s|依赖包存在于标准库里|no|
|+excluded|x|external packages explicitly excluded from vendoring|no|
|+unused|u|packages in the vendor folder, but unused|govendor remove unused|
|+program|p|package is a main package|no|
|+outside|+external +missing|||
|+all|+all packages|||


## 举例  
### 首次添加所有外部文件  
>govendor add +external
### 删除没用的外部文件  
>govendor remove unused  
### 再次添加外部文件  
>govendor add code.byted.org/gopkg/mosaic 
### 更新外部文件  
>govendor update code.byted.org/gopkg/mosaic/…
### 添加使用过的外部文件
>govendor add +e