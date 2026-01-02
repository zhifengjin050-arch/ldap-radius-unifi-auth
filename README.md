# **ldap-radius-unifi-auth**

**⚠️ 安全提示 / Security Notice**

本文档及示例配置文件中涉及的密码（如 123456）、密钥及 IP 地址仅作为演示使用。  
在生产环境部署时，请务必修改为强密码，严格限制防火墙规则，并妥善保管 clients.conf 中的共享密钥。

## **📖 项目简介 (Introduction)**

本项目提供了一套**开箱即用**的企业级网络接入认证解决方案。通过 Docker 容器化编排，深度整合了 **OpenLDAP**（身份源）、**FreeRADIUS**（认证服务）与 **Unifi 网络控制器**。

**核心功能：**

* 🚀 **统一身份管理**：集成 Go-Ldap-Admin 可视化面板，轻松管理员工账号与部门架构。  
* 🛡️ **802.1X 企业级认证**：取代传统的静态 WiFi 密码，实现“一人一号”的安全接入机制。  
* 🔀 **动态 VLAN 下发**：支持根据 LDAP 用户属性（如部门），自动将用户划分至不同的网络 VLAN，实现网络逻辑隔离。

## **📑 目录 (Table of Contents)**

* [环境拓扑](https://www.google.com/search?q=%23-%E7%8E%AF%E5%A2%83%E6%8B%93%E6%89%91)  
* [快速开始](https://www.google.com/search?q=%23-%E5%BF%AB%E9%80%9F%E5%BC%80%E5%A7%8B)  
  * [1\. 部署身份源 (LDAP)](https://www.google.com/search?q=%231-%E9%83%A8%E7%BD%B2%E8%BA%AB%E4%BB%BD%E6%BA%90-ldap)  
  * [2\. 部署认证服务 (Radius)](https://www.google.com/search?q=%232-%E9%83%A8%E7%BD%B2%E8%AE%A4%E8%AF%81%E6%9C%8D%E5%8A%A1-radius)  
* [核心配置指南](https://www.google.com/search?q=%23-%E6%A0%B8%E5%BF%83%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97)  
  * [对接 LDAP](https://www.google.com/search?q=%23%E5%AF%B9%E6%8E%A5-ldap)  
  * [配置 Unifi 客户端](https://www.google.com/search?q=%23%E9%85%8D%E7%BD%AE-unifi-%E5%AE%A2%E6%88%B7%E7%AB%AF)  
* [进阶：动态 VLAN 配置](https://www.google.com/search?q=%23-%E8%BF%9B%E9%98%B6%E5%8A%A8%E6%80%81-vlan-%E9%85%8D%E7%BD%AE)  
* [截图预览](https://www.google.com/search?q=%23-%E6%88%AA%E5%9B%BE%E9%A2%84%E8%A7%88)  
* [故障排查](https://www.google.com/search?q=%23-%E6%95%85%E9%9A%9C%E6%8E%92%E6%9F%A5)

## **🏗️ 环境拓扑**

| 组件 | 镜像/服务 | 作用 | 端口 |
| :---- | :---- | :---- | :---- |
| **IAM (身份源)** | eryajf/openldap:1.4.1 | 存储用户账户、组织架构数据 | 389 |
| **Admin UI** | eryajf/go-ldap-admin | LDAP 的可视化 Web 管理后台 | 8888 |
| **Database** | eryajf/mysql:8.3 | Admin UI 的后端存储 | 3306 |
| **AAA Server** | freeradius/freeradius-server | 处理认证、授权、计费请求 | 1812/udp |
| **NAS (接入)** | Unifi Controller / AP | 发起认证请求的网络设备 | \- |

## **⚡ 快速开始**

### **1\. 部署身份源 (LDAP)**

1. 启动 LDAP 及管理面板服务：  
   docker-compose up \-d go-ldap-admin mysql openldap

2. **初始化配置**：  
   * 访问管理面板：http://Server-IP:8888 (默认账号: admin / 默认密码: \*\*\*\*\*\*)。  
   * **关键步骤**：进入配置中心，将密码加密方式修改为 **clear (明文)**。这是 FreeRADIUS 读取 LDAP 密码进行验证的前提条件。

### **2\. 部署认证服务 (Radius)**

FreeRADIUS 建议使用 **Host 网络模式** 部署。

1. 启动容器并提取配置（首次运行）：  
   \# 临时启动  
   docker-compose up \-d freeradius  
   \# 拷贝配置  
   docker cp freeradius:/etc/raddb ./freeradius/config  
   \# 修正权限 (重要：容器内用户ID为100)  
   sudo chown \-R 100:101 ./freeradius/config

2. 修改 docker-compose.yml 挂载 volumes，然后重启服务。

## **⚙️ 核心配置指南**

以下操作均在宿主机 ./freeradius/config 目录中进行。

### **对接 LDAP**

编辑 mods-available/ldap，填入 LDAP Server IP 和管理员账号，并建立软链接：

ln \-s ../mods-available/ldap mods-enabled/ldap

### **配置 Unifi 客户端**

编辑 clients.conf，添加 Unifi 控制器或 AP 的网段信息：

client 192.168.0.0/24 {  
    secret \= your\_secure\_key  \# 需与 Unifi 设置一致  
    shortname \= unifi\_ap  
}

### **Unifi 控制器设置**

1. 进入 Settings \-\> Profiles \-\> RADIUS.  
2. 新建 Profile，填入 FreeRADIUS 服务器 IP 和 Secret。  
3. 在 WiFi 设置中，Security 选择 WPA Enterprise。

## **🚀 进阶：动态 VLAN 配置**

实现根据 LDAP 用户属性自动将用户分配到不同 VLAN。

1. **开启 EAP 透传**: 修改 mods-available/eap \-\> use\_tunneled\_reply \= yes。  
2. **属性映射**: 修改 mods-available/ldap，将 businessCategory 映射到 Tmp-String-0。  
3. **编写逻辑**: 在 sites-available/inner-tunnel 的 post-auth 中：

if (control:Tmp-String-0 \== "it") {  
    update outer.session-state {  
        \&reply:Tunnel-Type := VLAN  
        \&reply:Tunnel-Medium-Type := IEEE-802  
        \&reply:Tunnel-Private-Group-Id := "10"  
    }  
}

## **📸 截图预览**

### **LDAP 配置**

### **用户管理**

### **Unifi Radius 配置**

## **🔧 故障排查**

* **容器无法启动**：检查 ./freeradius/config 目录权限是否为 100:101。  
* **认证失败**：使用 docker logs freeradius 查看日志，或进入容器运行 freeradius \-X 开启调试模式。  
* **VLAN 未生效**：检查交换机端口是否开启 Trunk 模式，并确认 AP 支持动态 VLAN。