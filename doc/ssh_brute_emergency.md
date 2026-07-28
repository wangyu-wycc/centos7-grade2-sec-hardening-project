
SSH暴力破解应急处置闭环方案
 
仓库目录对应：
 
- 文档存放路径： doc/ssh_brute_emergency.md 
- 截图存放路径： screen/ssh_brute/ 
 
一、整体流程
 
模拟爆破攻击 → 日志排查统计攻击IP → 防火墙永久封禁IP → 验证拦截效果 → 恢复规则 → 整理归档截图与报告
 
二、分步实操命令+截图要求
 
步骤1：模拟SSH暴力破解，生成失败登录日志
 
1. 本地电脑使用FinalShell执行SSH连接服务器，连续多次输入错误密码
 
bash
  
ssh wy@192.168.122.137
 
 
截图1：模拟输错密码的报错界面，命名： 1_simulate_attack.png 
 
步骤2：筛选SSH登录失败日志
 
登录CentOS服务器执行命令，抓取爆破记录
 
bash
  
# 筛选所有SSH密码登录失败日志
grep "Failed password" /var/log/secure
 
 
截图2：原始失败日志输出，命名： 2_failed_log.png 
 
步骤3：统计攻击IP爆破频次
 
bash
  
# 统计每个IP的失败登录次数，从高到低排序
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
 
 
截图3：IP攻击频次统计结果，命名： 3_ip_count.png 
 
步骤4：防火墙永久拉黑恶意攻击IP
 
bash
  
# 配置永久黑名单规则
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="攻击IP地址" reject'
# 重载防火墙使规则生效
firewall-cmd --reload
 
 
截图4：拉黑命令执行结果，命名： 4_black_ip_exec.png 
 
步骤5：校验防火墙黑名单规则
 
bash
  
firewall-cmd --list-all
 
 
截图5：查看已生效的封禁规则，命名： 5_check_black_rule.png 
 
步骤6：验证IP拦截生效
 
本地再次发起SSH连接测试，连接会直接拒绝/超时
 
bash
  
ssh wy@服务器IP
 
 
截图6：SSH连接失败验证拦截效果，命名： 6_block_verify.png 
 
步骤7：闭环收尾，删除测试拉黑规则
 
bash
  
firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="攻击IP地址" reject'
firewall-cmd --reload
 
 
截图7：删除封禁规则，恢复连通性，命名： 7_restore_rule.png 
 
三、完整处置报告（可直接粘贴到 医生/ssh_brute_emergency.md ）
 
markdown
  
# SSH暴力破解应急处置闭环报告
## 1. 事件概述
模拟外部对公网CentOS 7服务器SSH端口发起密码暴力破解攻击，产生大量登录失败日志，按照安全运营标准完成排查、封禁、验证、优化全流程闭环处置，符合等保入侵防范要求。

## 2. 环境信息
操作系统：CentOS 7
SSH日志路径：/var/log/secure
防火墙组件：firewalld
攻击源IP：【192.168.122.1】

## 3. 事件排查过程
### 3.1 模拟攻击行为
本地多次错误输入SSH账号密码，生成爆破日志


### 3.2 筛选失败登录日志
执行日志筛选命令，抓取所有登录失败记录
bash
grep "Failed password" /var/log/secure
 
 
3.3 统计攻击IP频次
 
对攻击IP进行次数统计，定位高频恶意IP
 
bash
  
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
 
 
4. 应急处置操作
 
对恶意IP配置防火墙永久拦截策略，阻断攻击源
 
bash
  
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="攻击IP" reject'
firewall-cmd --reload
 
 
校验防火墙规则确认配置生效：
 
bash
  
firewall-cmd --list-all
 
 
5. 处置效果验证
 
使用被封禁IP再次发起SSH连接，访问被强制拦截，处置生效
 
bash
  
ssh wy@192.168.122.137
 
 
6. 闭环收尾操作
 
测试完成后删除临时封禁规则，恢复正常网络访问
 
bash
  
firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="192.168.122.1" reject'
firewall-cmd --reload
 
 
7. 长效加固优化方案（等保合规优化）
 
1. 禁止root账号远程SSH登录，规避超级账号爆破风险
2. 配置系统密码复杂度策略，提升口令安全强度
3. 修改SSH默认22端口，降低端口扫描命中率
4. 限制SSH最大认证重试次数，减少暴力破解尝试空间
 
