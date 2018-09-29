# <center>service account </center> 

**account分为两种**  
- user account， 如kubectl访问APIServer时
    - 具有全局性，权限比较大
- service account, 如Pod中的进程访问Api 
    - 具有限制性，更适合一些轻量级的任务task，更聚焦于授权给某些特定的Pod中的进程Process所使用。