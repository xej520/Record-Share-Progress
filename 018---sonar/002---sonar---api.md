1. 认证用户    
    POST api/authentication/login     

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |login|string|required|用户登录|
    |password|string|required|用户密码|  

2.  查询所有相关规则的集合  
    GET api/rules/search

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |p|Integer|optional|第几页|
    |ps|Integer|optional|每页显示的条数|
    |s|String|optional|排序|
    |asc|Boolean|optional|Ascending sort|

3. 查询相关规则的集合   
   GET api/rules/search   
   |名字|类型|是否必须|说明|
   |:---|:---|:---|:---|
   |qprofile|String|optional|过滤规则集的键；<br>尽在参数activation设置时使用 
   |activation|Boolean|optional|对规则集按照激活或者停用状态进行过滤。<br> 如果未设置参数'qprofile'，则忽略| 
   |p|Integer|optional|第几页|
   |ps|Integer|optional|每页显示的条数|
   |s|String|optional|排序|
   |asc|bool|optional|Ascending sort| 

4. 列出所有的规则集   
    GET api/qualityprofiles/search
    无参数   
5. 根据语言来查询规则集   
    GET api/qualityprofiles/search  
    
    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |language|String|optional|Language key. <br>If provided, only profiles for the given language are returned. <br>It should not be used with 'defaults', 'projectKey or 'profileName' at the same time|
6. 根据规则集名称来查询   
     GET api/qualityprofiles/search  
    
    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |profileName|String|optional|Profile name. <br>It should be always used with the 'projectKey' or 'defaults' parameter|
7.  
    GET api/qualityprofiles/inheritance

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |profileKey|String|optional|规则集的键|   

8. 激活规则集中的一条规则   
    POST api/qualityprofiles/activate_rule

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |profile_key|String|required|规则集的键|
    |rule_key |String|required|规则的键|

9. 禁用规则集中的一条规则   
    POST api/qualityprofiles/deactivate_rule

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |profile_key|String|required|规则集的键|
    |rule_key |String|required|规则的键|
10. 复制规则集   
    POST api/qualityprofiles/copy   
    
    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |fromKey|String|required|规则集的键|
    |toName |String|required|新规则集的名称|

11. 创建规则集    
    POST api/qualityprofiles/create
    
    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |language|String|required|language|
    |name |String|required|规则集的名称|
12. 删除规则集   
    POST api/qualityprofiles/delete  

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |profileKey|String|required|规则集的键|   

13. 给工程添加规则集   
    POST api/qualityprofiles/add_project

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |projectKey|String|required|工程的键|
    |profileName|String|required|规则集名称| 

14. 将工程中的规则集移除   
    POST api/qualityprofiles/remove_project

    |名字|类型|是否必须|说明|  
    |:---|:---|:---|:---|
    |projectKey|String|required|工程的键|
    |profileName|String|required|规则集名称| 




