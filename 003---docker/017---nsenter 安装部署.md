`使用nsenter工具管理docker容器`  
参考文献：  
https://blog.csdn.net/stf1065716904/article/details/74198836  
https://blog.csdn.net/qq_39629343/article/details/80170164  

# 1. nsenter介绍
&ensp;&ensp;&ensp;&ensp;对于运行在后台的Docker容器，我们经常需要做的事情是进入到容器中，docker为我们提供了docker exec 、docker attach 命令，并且还提供了nsenter工具，外部工具供我们使用。  
`docker attach存在的问题是：`   
当多个窗口同时attach到同一个容器时，所有的窗口都会同步的显示，假如其中的一个窗口发生阻塞时，其它的窗口也会阻塞，  
docker attach命令可以说
是最不方便的进入后台docker容器的方法。  
docker exec命令是在docker 1.3之后增加的一个比docker attach命令更加方便的命令。和docker exec差不多方便的命令是`nsenter工具`。

# 2. nsenter安装  
- 查看支持的版本  
https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/    

![nsenter当前支持的版本](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/0ECBE91DF8D34F6991D143403A3747D2/22859)

- 下载nsenter源码  
wget https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.32/util-linux-2.32.tar.gz  
![下载nsenter源码](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/0AC659835ABF4908AF198CDE6D225958/22857)  
- 解压  
tar -xzvf util-linux-2.32.tar.gz  
cd  util-linux-2.32  

- 生成Makefile，为下一步编译做准备  
./configure --without-ncurses --prefix=/usr/local/bin
![生成makefile](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/F1E1268CBFC042D982FFCA3D4CDA8D5B/22869)  
![显示makefile文件](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/2FA33DD631FF4BFB9BEFC460F674E7C1/22867)
- 开始编译  
make nsenter 
![编译过程](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/2A6C2B4D031A47039778430744376B71/22865)  

- 查看nsenter帮助文档 
方式一：
![nsenter --help](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/FBB486AE5B2042BCB2F047EB22C8A8A4/22872)  

方式二：
![man nsenter](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/1E654C0F95CE47F6BE4F592DF2248CFD/22874)  


# 3. nsenter使用  
- 获取容器ID号  
docker ps 
![获取容器ID](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/F826D02EB7394DFB8E54ABB5ED598CB6/22876)  


- 获取容器进程的PID的方法  
```
格式:  
PID=$(docker inspect --format "{{ .State.Pid}}"  <container id>)
```  

- 通过得到的这个PID，就可以连接到这个容器：
```
格式:  
nsenter --target    $PID   --mount --uts   --ipc   --net   --pid
```
![进入容器](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/37B0DAB36E1B41D691250E7DC6DBB262/22883)  



# 4. 安装过程中，报的问题  
- 问题一:  configure: error: no acceptable C compiler found in $PATH

![make 时出错](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/7A4DE706A1904B99A9B6C070CE2DD324/22861)  

解决措施：  
![解决措施](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/DF8384EEDB5F41DBB8253A90D9DEE3E4/22863)  



































