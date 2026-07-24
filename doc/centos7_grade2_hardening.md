CentOS7 等保2.0二级主机安全加固极简操作手册
 
适配GitHub归档、可直接导出PDF
对标国标：GB/T 22239-2019 网络安全等级保护基本要求（二级）
统一规范：安全风险→等保合规要求→加固操作→验证命令→截图归档路径
 
目录
 
1. 身份鉴别加固
2. 访问控制加固
3. 安全审计加固
4. 入侵防范加固
5. MariaDB数据库加固（适配DVWA靶场）
6. 加固前后总对比表
 
 
 
1 身份鉴别加固
 
1.1 禁止root远程SSH登录
 
- 安全风险：root超级管理员账号可公网直接SSH登录，账号口令一旦泄露，攻击者可完全接管整台服务器。
- 等保合规要求：禁止超级权限账号远程直接登录，身份权限隔离管控。
 
bash
  
# 备份原始配置文件
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
# 编辑SSH主配置
vim /etc/ssh/sshd_config
# 写入配置
PermitRootLogin no
# 配置语法校验，防止SSH服务瘫痪
sshd -t
# 重载服务生效
systemctl restart sshd
 
 
- 验证命令： sshd -T | grep permitrootlogin 
- 截图路径： screen/ssh/root_login_disable.png 
 
1.2 SSH登录失败最大重试次数限制
 
- 安全风险：系统默认允许6次密码尝试次数，攻击者可利用字典工具持续暴力破解账号口令。
- 等保合规要求：具备登录失败处置能力，多次认证失败自动断开会话，抵御暴力破解。
 
bash
  
vim /etc/ssh/sshd_config
MaxAuthTries 3
systemctl restart sshd
 
 
- 验证命令： sshd -T | grep maxauthtries 
- 截图路径： screen/ssh/max_retry.png 
 
1.3 修改SSH默认22监听端口
 
- 安全风险：22端口是全网自动化扫描、僵尸机攻击最高频端口，极易被批量探测爆破。
- 等保合规要求：缩减服务器暴露攻击面，不使用默认管理端口。
 
bash
  
vim /etc/ssh/sshd_config
Port 22022
systemctl restart sshd
 
 
- 验证命令： netstat -lntp | grep sshd 
- 截图路径： screen/ssh/change_port.png 
 
1.4 闲置SSH会话自动超时踢出
 
- 安全风险运维人员长时间挂机登录，账号被旁人盗用劫持。
- 等保合规要求：对长时间闲置会话自动强制终止。
 
bash
  
vim /etc/ssh/sshd_config
# 300秒无数据收发开始计时
ClientAliveInterval 300
# 连续2次无响应直接断开
ClientAliveCountMax 2
 
 
- 生效说明：5分钟无任何操作，系统自动下线SSH连接
- 验证方式：查看sshd完整运行配置参数
 
1.5 全局系统密码复杂度策略
 
- 安全风险：无强制密码规则，弱口令、短口令、简单重复口令极易被破解。
- 等保合规要求：强制设置口令长度、字符组合复杂度策略。
 
bash
  
vim /etc/security/pwquality.conf
minlen = 8
minclass = 3
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
maxrepeat = 3
 
 
- 规则说明：密码最少8位，必须包含大写、小写、数字、特殊符号中3种类型；连续相同字符不超过3位。
- 验证方式：新建测试用户，尝试设置简单弱口令，系统直接拒绝。
 
 
 
2 访问控制加固
 
2.1 普通用户精细化SUDO最小权限分权
 
- 安全风险：运维人员直接使用root账号操作，或普通用户拥有完整root全部权限，一旦账号泄露，服务器完全失控。
- 等保合规要求：严格遵循最小权限原则，权限拆分，仅分配岗位必需权限。
 
bash
  
# 创建日常运维普通账号
useradd secuser
passwd secuser
# 加入wheel权限组
usermod -aG wheel secuser
# 精细化授权配置
visudo
# 仅开放指定运维命令
secuser ALL=(ALL) /usr/bin/systemctl,/usr/bin/firewall-cmd,/usr/bin/journalctl
 
 
- 权限范围：仅允许重启系统服务、配置防火墙、查看系统日志，禁止修改系统关键配置文件。
- 验证方式：切换secuser账号，分别测试授权命令、未授权命令执行效果。
- 截图路径： screen/permission/sudo_limit.png 
 
2.2 Firewalld防火墙最小端口放行
 
- 安全风险：防火墙默认开放22高危端口，多余端口暴露在外网环境。
- 等保合规要求：边界访问控制，仅放行业务运行必需端口。
 
bash
  
# 永久删除默认22端口放行规则
firewall-cmd --permanent --remove-port=22/tcp
# 放行自定义SSH端口
firewall-cmd --permanent --add-port=22022/tcp
# 重载防火墙策略生效
firewall-cmd --reload
 
 
- 验证命令： firewall-cmd --list-ports 
- 截图路径： screen/firewall/port_policy.png 
 
2.3 系统777高危权限文件清理
 
- 安全风险：777权限代表所有用户均可读写、执行，恶意程序可随意篡改系统文件。
- 等保合规要求：系统关键文件权限严格管控，禁止全局开放权限。
 
bash
  
# 全盘扫描777权限高危文件
find / -type f -perm 777 2>/dev/null
# 安全权限整改示例
chmod 644 目标文件名
 
 
 
 
3 安全审计加固
 
3.1 开启系统审计服务auditd
 
bash
  
# 开机自启并立即运行审计服务
systemctl enable --now auditd
 
 
- 等保合规要求：服务器关键操作全程审计记录，行为可完整溯源。
 
3.2 SSH暴力破解登录日志审计排查
 
bash
  
# 筛选所有SSH密码登录失败记录
grep Failed /var/log/secure
 
 
3.3 日志留存配置
 
修改 logrotate 轮转配置，系统审计日志最低留存6个月，满足等保长期溯源审计硬性要求。
 
 
 
4 入侵防范加固
 
4.1 关闭程序Core内核转储
 
- 安全风险：程序崩溃时会生成内存转储文件，容易泄露漏洞敏感内存数据。
 
bash
  
echo "* soft core 0" >> /etc/security/limits.conf
 
 
4.2 关闭ICMP重定向，防范路由欺骗攻击
 
bash
  
sysctl -w net.ipv4.accept_redirects=0
 
 
 
 
5 MariaDB数据库加固（适配DVWA环境）
 
- 安全风险：数据库默认存在匿名空账号、root账号支持外网远程访问、自带无用test测试库。
- 等保合规要求：清理数据库默认冗余账号，限制数据库远程访问权限，收紧账号安全策略。
 
bash
  
# MariaDB一键安全初始化
mysql_secure_installation
 
 
- 主要整改项：设置数据库root强密码、删除匿名用户、禁止root外网访问、删除默认test空数据库。
 
 
 
6 加固前后总对比表
 
加固项目 加固前默认状态 加固后合规状态 
Root远程SSH登录 允许直接登录 完全禁止 
SSH密码最大重试次数 6次 3次 
SSH监听端口 默认22端口 自定义22022端口 
闲置会话策略 永久保持在线 5分钟无操作自动下线 
登录密码规则 无强制约束 8位长度+3类字符组合限制 
管理员运维权限 直接使用root操作 普通账号+sudo最小按需授权 
防火墙端口策略 开放默认高危端口 仅放行业务必需端口 
系统安全审计 无规范留存策略 全操作日志长期可溯源 
 
 
 
GitHub 仓库整体目录结构
 
plaintext
  
README.md
doc/
└── centos7_grade2_hardening.md
screen/
├── ssh/
├── permission/
└── firewall/
 
 
使用说明
 
1. 使用Typora打开本md文档，可一键批量导出完整PDF文件；
2. 所有实操截图按照文档内标注路径，分别放入screen各个子文件夹；
3. 整套文档可直接作为网络安全运维、等保测评方向求职作品集。
