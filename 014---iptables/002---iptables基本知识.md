1. MASQUERADE 与 SNAT ?  
```
都是对源地址，进行伪装，转换。  
MASQUERADE比SNAT好处是：  它能动态的从网卡里获取地址。   
```  

2. 规则
- 匹配规则   
![基本匹配规则](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/44DFEA0BA94A4A588CE6B3CE41450779/21902)  
- 处理动作   
![处理动作](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/8DADDF3BB4F04F2EBE513F8C1D0CDD37/21904)  
RUTURN: 返回主链继续匹配   



# iptables实际操作之规则查询   
以表作为入口  
![](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/7D25D073754E4B1AA0873CE224810B57/21906)    

## 以filter表为例  
1. 场景一： 假如我们要进入某些IP来访问我们的主机，过滤规则应该添加在哪些链中呢？  
![input链](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/D264CC5319CD485A98175DA5F960A407/21908)   

![](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/2C06455F0CA94B3FBBECB1B05264414A/21910)   




