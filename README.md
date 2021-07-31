Vscan
================================
Vscan 是一款为红队开发的简单、快速的跨平台打点扫描器。

### 1.options
```
-host                           Host to find ports for		
-top-ports                      Top Ports to scan (full|http|top100|top-1000)		
-iL                             File containing list of hosts to enumerate ports		
-p                              Ports to scan (80, 80,443, 100-200, (-p - for full port scan)		
-ping                           Use ping probes for verification of host		
-ports-file                     File containing ports to enumerate for on hosts		
-o                              File to write output to (optional)		
-json                           Write output in JSON lines Format		
-silent                         Show found ports only in output		
-retries                        DefaultRetriesSynScan, "Number of retries for the port scan probe		
-rate                           DefaultRateSynScan, "Rate of port scan probe requests		
-v                              Show Verbose output		
-no-color                       Don't Use colors in output		
-timeout                        DefaultPortTimeoutSynScan, "Millisecond to wait before timing out		
-exclude-ports                  Ports to exclude from enumeration		
-verify                         Validate the ports again with TCP verification		
-version                        Show version of naabu		
-exclude-hosts                  Specifies a comma-separated list of targets to be excluded from the scan (ip, cidr)		
-exclude-file                   Specifies a newline-delimited file with targets to be excluded from the scan (ip, cidr)		
-debug                          Enable debugging information		
-source-ip                      Source Ip		
-interface                      Network Interface to use for port scan		
-exclude-cdn                    Skip full port scans for CDNs (only checks for 80,443)		
-warm-up-time                   Time in seconds between scan phases		
-interface-list                 List available interfaces and public ip		
-config                         Config file		
-nmap                           Invoke nmap scan on targets (nmap must be installed)		
-nmap-cli                       Nmap command line (invoked as COMMAND + TARGETS)		
-c                              General internal worker threads		
-stats                          Display stats of the running scan		
-scan-all-ips                   Scan all the ips		
-s                              Scan Type (s - SYN, c - CONNECT)	
-http-proxy                     HTTP Proxy, eg http://127.0.0.1:8080        	
```

### 2.功能
#### 2.1 端口扫描

[https://github.com/projectdiscovery/naabu](https://github.com/projectdiscovery/naabu)

一款扫描器必备的东西是端口扫描，虽然massscan和nmap是非常给力的端口扫描工具，要比其他端口扫描器好用，但是由于他们都是C语言开发的，想要集成到vscan里比较繁琐，所以我选择了大方向go语言的一款端口扫描器，他支持CONNECT、SYN扫描，C段扫描等功能，对于我们来说完全足够

#### 2.2 服务识别
[https://github.com/projectdiscovery/httpx](https://github.com/projectdiscovery/httpx)
这是一款http服务快速识别扫描器
对于外网打点来说，最重要的就是web快速扫描，这款识别扫描器非常好用，可以快速识别网站的标题、网址、状态码、指纹等，还可以保留内容


需要注意的是，由于httpx使用了wappalyzergo库作为指纹识别，我们需要把[wappalyzergo](https://github.com/projectdiscovery/wappalyzergo)库整个下载下来，方便后续的指纹添加,这里我添加了一个shiro指纹，可以快速识别服务器是否使用shiro


#### 2.3 漏洞扫描（nday、0day自动利用）
我在pkg包里新建了一个exp版块，建立了一个入口函数check，以后其他所有nday也可以使用同样的入口，方便检测


shiro exp内容：
包含CBC、GCM两种方式的检测，遍历测试一百多个key

漏洞扫描的过程我放到了指纹识别以后多线程并行进行，例如如果识别到使用了shiro服务，则调用shiro.Check
```
matches := r.wappalyzer.Fingerprint(resp.Headers, resp.Data)
for match := range matches {
    technologies = append(technologies, match)
    if match == "Shiro" {
        key := shiro.Check(URL.String())
        if key != ""{
            technologies = append(technologies, "key:"+key)
        }
    }
}
```

#### 2.4 智能后台弱口令扫描，中间件弱口令扫描

弱口令其实是打点一个较为关键的部分，需要人工去抓包使用工具爆破

这里我完成了最简单的没有使用验证码，没有使用vue等前端框架的后台智能爆破，可以一定量的减少手工时间，加快打点速度

内置了两个账号 admin/test，密码为top100，如果成功识别后台会标记为\[登录页\]，成功构建登录包会自动爆破出密码

如：

`http://xxx.xxx.xxx.xxx:8080 [302,200] [登录 - 后台] [exp-shiro|key:Z3VucwAAAAAAAAAAAAAAAA==,Java,登录页,brute-admin|admin:123456] [http://xxx.xxx.xxx.xxx:8080/login;JSESSIONID=8417fe14-f529-46a7-a67e-bbe96429cbd0]`

包含爆破板块
1. 智能后台爆破
2. basic爆破
3. tomcat登录爆破
4. weblogic登录爆破

#### 2.5 敏感文件扫描

扫描 备份文件、swagger-ui、spring actuator、上传接口、测试文件等敏感链接

### 3.演示
```
root@xxx:~/vscan# ./vscan  -iL ../urlx.txt -top-ports http

http://xxx.xxx.xxx [200] [Test Page for Apache Installation] [OpenSSL,aardvark topsites,blognplus,igaming-cms,Windows Server,Apache,68 classifieds,allnewsmanager_net,atomic-photo-album]
http://xxx.xxx.xxx:8001 [302,302,200] [商户管理平台] [Express,Node.js,登录页,Font Awesome] [ http://xxx.xxx.xxx:8001/login ]
http://xxx.xxx.xxx:8004 [302,200] [系统后台] [Node.js,登录页,Font Awesome,Ionicons,Bootstrap,jQuery,Express] [ http://xxx.xxx.xxx:8004/login ]
https://xxx.xxx.xxx [200] [xxxxxx] [Nginx,登录页]
http://xxx.xxx.xxx:8001 [302,302,200] [商户管理平台] [Express,Node.js,登录页,Font Awesome] [ http://xxx.xxx.xxx:8001/login ]
http://xxx.xxx.xxx:8083 [302,302,200] [运营后台] [登录页,Font Awesome,Google Font API,Bootstrap,jQuery] [ http://xxx.xxx.xxx:8083/main/login.html;JSESSIONID=xxx ] [file_fuzz："http://xxx.xxx.xxx:8083/actuator","http://xxx.xxx.xxx:8083/actuator/env"]
http://xxx.xxx.xxx:8004 [302,200] [系统后台] [Ionicons,Bootstrap,jQuery,登录页,Express,Node.js,Font Awesome] [ http://xxx.xxx.xxx:8004/login ]
https://xxx.xxx.xxx [200] [xxxxxx] [Nginx,登录页]
https://xxx.xxx.xxx [200] [xxxxxx] [Nginx,PHP]
http://xxx.xxx.xxx:8002 [302,302,200] [代理商管理平台] [登录页,Font Awesome,Express,Node.js] [ http://xxx.xxx.xxx:8002/login ]
http://xxx.xxx.xxx:8002 [302,302,200] [代理商管理平台] [Font Awesome,Express,Node.js,登录页] [ http://xxx.xxx.xxx:8002/login ]
http://xxx.xxx.xxx:8088 [302,200] [后台管理系统] [Shiro,exp-shiro|key:kPH+bIxk5D2deZiIxcaaaA==,Java,Apache Tomcat,登录页,brute-admin|test:test] [ http://xxx.xxx.xxx:8088/login;jsessionid=xxx ]
http://xxx.xxx.xxx:8081 [302,200] [xxx后台管理系统] [Font Awesome,登录页,Microsoft ASP.NET,IIS,Windows Server] [ http://xxx.xxx.xxx:8081/Login ]
http://xxx.xxx.xxx:8001 [302,302,200] [Data Search] [Java,Google Font API,Bootstrap,jQuery,登录页,Font Awesome,Shiro] [ http://xxx.xxx.xxx:8001/main/login.html;jsessionid=xxx ] [file_fuzz："http://xxx.xxx.xxx:8001/druid/index.html"]
http://xxx.xxx.xxx:8082 [302,200] [运营系统] [Java,Google Font API,Bootstrap,jQuery,登录页,Font Awesome,Shiro] [ http://xxx.xxx.xxx:8082/main/login.html;jsessionid=xxx ]
http://xxx.xxx.xxx:8084 [302,200] [xxx运营系统] [Java,Google Font API,Bootstrap,jQuery,登录页,Font Awesome,Shiro] [ http://xxx.xxx.xxx:8084/main/login.html;JSESSIONID=xxx ]
http://xxx.xxx.xxx:8085 [302,200] [运营系统] [jQuery,登录页,Shiro,Java,Font Awesome,Google Font API,Bootstrap] [ http://xxx.xxx.xxx:8085/main/login.html;jsessionid=xxx ]
http://xxx.xxx.xxx [200] [Test Page for Apache Installation] [Apache,68 classifieds,allnewsmanager_net,blognplus,atomic-photo-album,igaming-cms,OpenSSL,aardvark topsites,Windows Server]
https://xxx.xxx.xxx [403] [403 Forbidden] [Apache,OpenSSL,Windows Server] [file_fuzz："https://xxx.xxx.xxx:443/.git/config","https://xxx.xxx.xxx:443/.svn/entries"]
http://xxx.xxx.xxx [200] [APP管理后台] [Nginx] [file_fuzz："http://xxx.xxx.xxx:80/api/swagger-ui.html","http://xxx.xxx.xxx:80/api/v2/api-docs","http://xxx.xxx.xxx:80/api/v1/api-docs","http://xxx.xxx.xxx:80/api/upload","http://xxx.xxx.xxx:80/upload/"]
```

### 4.TO DO

1.加入socks代理

2.解析http以外的端口指纹

### 5.目前正在做的

1.加入weblogic，jboss等反序列化检测

2.加入其他cms nday



