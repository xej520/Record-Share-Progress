0. 安装k8s前需要提前添加yum源
![k8s yum 源](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/9AD0DCC06B354389979F4D4F55B5ADF5/22844)

1. 列出已经安装过的rpm包   
yum list installed | grep kube    
![列出安装过的rpm](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/291F595369D449CCB4288BF31AE89151/22837)  

2. 卸载安装的rpm包  
yum remove kubeadm.x86_64 kubectl.x86_64  kubelet.x86_64 -y   
![卸载rpm](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/E1B594CCD5CD4AD89CDE704557E2B6C4/22840)  

3. 安装指定的kubeadm  
yum install -y kubelet-1.12.1 kubeadm-1.12.1 kubectl-1.12.1   
![安装指定版本的kubeadm](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/55D5FED2AFA440B484570FAB572D022C/22842)  







