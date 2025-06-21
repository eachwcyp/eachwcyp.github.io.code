

####    如何重启服务器：

   cd /home/admintalu/application
​           sh  Restart-eaechbuy51-module-system-prod.sh

#### 本地代码路径：

D:\sourcecode\opensource\BCSCLG\eachwcyp\gitee\Business-API_jxwc

#### 管理后台界面：

本地源码路径
D:\sourcecode\opensource\BCSCLG\eachwcyp\gitee\business-portal_jxwc

部署路径：
/usr/share/nginx/html/eachbuybackstage

#### 前端配置文件nginx:

      location ^~ /admin {
            proxy_pass              https://112.126.81.113:9090/admin;
            proxy_set_header        Host 112.126.81.113;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        }


                location /eachbuybackstage{
                    alias   html/eachbuybackstage;
                    index  index.html;
                    try_files $uri $uri/ /eachbuybackstage/index.html;
                }














