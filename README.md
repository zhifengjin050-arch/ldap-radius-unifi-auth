# **LDAP \+ FreeRADIUS \+ Unifi 企业级认证方案**

**⚠️ 安全提示 / Security Notice**

生产环境请务必修改默认密码（如 123456），并妥善保管 clients.conf 中的共享密钥。  
本文档基于 Docker Compose 部署，适用于 1Panel、群晖或纯 Linux 环境。

## **📖 项目简介**

本项目提供一套完整的企业级无线网络认证方案。通过 **Go-Ldap-Admin** 管理用户，**FreeRADIUS** 处理 802.1X 认证，配合 **Unifi** 控制器实现“一人一号”登录及动态 VLAN 下发。

**核心优势：**

* **可视化管理**：使用 Web 界面管理组织架构和员工账号。  
* **安全接入**：WPA2-Enterprise 企业级认证，杜绝万能钥匙共享 WiFi。  
* **动态 VLAN**：根据部门（如研发、市场）自动划分网络 VLAN，无需多 SSID。

## **🛠️ 部署步骤**

### **第一步：启动基础服务 (LDAP)**

1. 确保本地有 docker-compose.yml 文件。  
2. 启动 LDAP 和管理后台：  
   docker-compose up \-d go-ldap-admin mysql openldap

3. **配置加密方式（关键）**：  
   * 访问管理后台 http://\<你的IP\>:8888 (默认账号 admin / 密码 123456)。  
   * 进入 **配置中心**，将“用户密码加密方式”修改为 **clear (明文)**。  
   * *原因：FreeRADIUS 需要读取明文或特定哈希来验证 PAP/CHAP 协议。*  
4. **创建用户**：  
   * 在“用户管理”中新建测试用户，记住 Uid (用户名) 和 Password。

### **第二步：配置 FreeRADIUS**

FreeRADIUS 配置较复杂，建议先临时启动容器提取默认配置，再进行修改。

#### **1\. 提取配置文件**

\# 临时启动  
docker-compose up \-d freeradius

\# 将容器内配置拷贝到本地 ./freeradius/config 目录  
docker cp freeradius:/etc/raddb ./freeradius/config

\# 【非常重要】修正权限 (容器内 radius 用户 ID 为 100\)  
\# 如果是 Linux/Mac 请执行：  
sudo chown \-R 100:101 ./freeradius/config

#### **2\. 修改配置对接 LDAP**

以下文件均在 ./freeradius/config/ 目录下修改：

* **编辑 mods-available/ldap**：  
  ldap {  
      server \= "192.168.x.x"  \# 宿主机真实IP (不要填 127.0.0.1)  
      identity \= "cn=admin,dc=eryajf,dc=net"  
      password \= "admin\_password"  
      base\_dn \= "dc=eryajf,dc=net"

      update {  
          \# 开启密码头支持  
          control:Password-With-Header \+= 'userPassword'  
      }  
  }

* **启用模块**：  
  \# 在 mods-enabled 目录下建立软链接  
  cd mods-enabled  
  ln \-s ../mods-available/ldap ldap

#### **3\. 开启 Unifi 接入 (clients.conf)**

编辑 clients.conf，允许 Unifi 设备连接：

client unifi\_ac {  
    ipaddr \= 0.0.0.0/0      \# 生产环境建议限制为 AP 的 IP 段  
    secret \= testing123     \# 【重要】Unifi 控制器里要填这个  
}

#### **4\. 延长证书有效期**

默认证书只有 60 天，建议修改为 10 年。

* 修改 certs/ca.cnf, certs/server.cnf 中的 default\_days \= 3650。  
* 进入容器重新生成证书：  
  docker exec \-it freeradius /etc/raddb/certs/bootstrap

  *(或者删除 certs 下的旧文件，重启容器让其自动生成)*

### **第三步：配置 Unifi 控制器**

1. 登录 Unifi Network 后台。  
2. 进入 **Settings \-\> Profiles \-\> RADIUS**。  
3. **Create New RADIUS Profile**：  
   * **IP Address**: 填宿主机 IP。  
   * **Port**: 1812。  
   * **Shared Secret**: 填 clients.conf 里设置的密码 (如 testing123)。  
4. 进入 **WiFi 设置**：  
   * **Security**: WPA2 Enterprise。  
   * **RADIUS Profile**: 选择刚才创建的 Profile。

## **🚀 进阶：动态 VLAN 配置**

实现根据 LDAP 用户属性（如部门字段 businessCategory）自动分配 VLAN。

### **1\. 开启 EAP 透传**

编辑 mods-available/eap，找到 peap 块：

peap {  
    use\_tunneled\_reply \= yes  \# 确保此处为 yes  
}

### **2\. 映射 LDAP 属性**

编辑 mods-available/ldap，将 LDAP 字段映射到 FreeRADIUS 临时变量：

update {  
    control:Password-With-Header \+= 'userPassword'  
    \# 将 businessCategory 映射到 Tmp-String-0  
    request:Tmp-String-0 := 'businessCategory'  
}

### **3\. 编写判断逻辑**

编辑 sites-available/inner-tunnel，在 post-auth 块中添加：

post-auth {  
    \# 逻辑：如果 LDAP 属性为 "it"，则下发 VLAN 10  
    if (request:Tmp-String-0 \== "it") {  
        update reply {  
            Tunnel-Type \= VLAN  
            Tunnel-Medium-Type \= IEEE-802  
            Tunnel-Private-Group-Id \= "10"  
        }  
    }  
    \# 默认 VLAN  
    else {  
        update reply {  
            Tunnel-Type \= VLAN  
            Tunnel-Medium-Type \= IEEE-802  
            Tunnel-Private-Group-Id \= "1"  
        }  
    }  
    \# ... 其他原有配置  
}

## **🔧 常用调试命令**

如果认证失败，请停止当前容器，使用 **Debug 模式** 启动，查看详细日志：

\# 停止正常运行的容器  
docker stop freeradius

\# 启动调试模式 (注意替换路径)  
docker run \-it \--rm \--network host \\  
  \-v $(pwd)/freeradius/config:/etc/raddb \\  
  freeradius/freeradius-server:latest \-X

* **测试连接工具** (radtest)：  
  \# 格式: radtest \<用户名\> \<密码\> \<RadiusIP\> \<端口\> \<Secret\>  
  radtest testuser 123456 127.0.0.1 0 testing123  
