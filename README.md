所為何來
=
做為一公開的網頁伺服器，連入的安全性是要謹慎的。  
本文將針對登入帳號、SSH 及 IP 入侵規則進行補強。

適用環境
=
Ubuntu Server 12.04  
fail2ban 0.8.10

安裝方式
=
###登入
請從本機以 root 身份登入，或從遠端以 SSH 協定登入：  
(本文以伺服器的 IP 為 <code>10.10.10.10</code> 做示範，請自行修改)
```bash
ssh root@10.10.10.10
```
如出現類似訊息，仍請回答 <code>yes</code>：
```text
The authenticity of host '10.10.10.20 (10.10.10.20)' can't be established.
ECDSA key fingerprint is 11:22:33:44:55:66:77:88:99:00:aa:bb:cc:dd:ee:ff.
Are you sure you want to continue connecting (yes/no)?
```

###更改 root 的密碼
如 root 的密碼為他方給予，或自己想換組密碼，可於此時輸入：
```bash
passwd
```

###新增使用者
如沒有 root 之外的使用者，或想新增一網站管理帳號：  
(本文以 <code>WebAdmin</code> 做為示範帳號)
```bash
adduser WebAdmin
```
設定過程中，請雙次輸入密碼；其餘之姓名、電話資訊，可自行填入或留白。  
需要注意的是，所新增的帳號並未擁有 sudo 權限，所以輸入以下指令來編輯權限設定：
```bash
visudo
```
找到文中以下字句處：
```text
# User privilege specification
root    ALL=(ALL:ALL) ALL
```
於其下一行添加：  
(<code>WebAdmin</code> 為本文示範帳號)
```text
WebAdmin   ALL=(ALL:ALL) ALL
```
存檔後退出編輯，如此，此帳號即有使用 sudo 的權限。

###修改 SSH 的設定
開啟 SSH 的設定檔：
```bash
nano /etc/ssh/sshd_config
```
請分別找到以下二行：
```text
Port 22
PermitRootLogin yes
```
Port 預設值為 22，為防止惡意連線，請更換為 1025 ～ 65536 間的數字，並請不要忘記你的設定 (本文以 <code>1010</code> 做為示範)。  
PermitRootLogin 為設定 root 可否登入的選項，預設為 <code>yes</code>，本文建議更改為 <code>no</code>。  
於文件最後，可添加以下二句，：
```text
UseDNS no
AllowUsers WebAdmin
```
UseDNS 設定為 <code>no</code> 後，可使伺服器省略過 DNS 反向查詢，加快 SSH 登入速度；  
AllowUsers 則用於限定可 SSH 登入之帳號 (本文限定 <code>WebAdmin</code> 為唯一登入帳號)。
  
存檔後退出編輯，再重新啟動 SSH：
```bash
reload ssh
```

###測試
請先不要登出，保持連線，並另開一終端機介面，輸入以下指令：
(<code>WebAdmin</code> 及 <code>10.10.10.10</code> 則皆為本文示範之帳號及伺服器 IP)
```bash
ssh -p 1010 WebAdmin@10.10.10.10
```
其中，-p 參數代表要指定連線的 Port，<code>1010</code> 即本文先前所指定的 Port 值。  
若可成功登入，代表設定無誤，即可將原來 root 的連線登出。  
  
日後也請使用所指定的帳號與 Port 進行登入及操作。

###安裝 Fail2ban 防入侵套件
安裝指令：
```bash
sudo apt-get install fail2ban
```
製作本機設定檔，並編輯之：
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local &&
sudo nano /etc/fail2ban/jail.local
```
其中，DEFAULT 段落會如下所示：
```text
[DEFAULT]

# "ignoreip" can be an IP address, a CIDR mask or a DNS host
ignoreip = 127.0.0.1/8
bantime  = 600
maxretry = 3

# "backend" specifies the backend used to get files modification. Available
# options are "gamin", "polling" and "auto".
# yoh: For some reason Debian shipped python-gamin didn't work as expected
#      This issue left ToDo, so polling is default backend for now
backend = auto

#
# Destination email address used solely for the interpolations in
# jail.{conf,local} configuration files.
destemail = root@localhost

```
段落末行的 <code>destemail = root@localhost</code> 可自行更換為自己的 Email，當 Fail2ban 有阻擋 IP 時，即會發信通知 (但先決條件是，你的伺服務要設定好 Mail Server)。  
餘選項維持預設即可，有興趣者可自行研究更改。  
  
SSH 的段落中：
```text
[ssh]

enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 6
```
請將 port 值指定為你設定 SSH 登入 Port，以本文為例，將為<code>port = 1010</code>。  
存檔後退出編輯，再重新啟動 Fail2ban：
```bash
sudo service fail2ban restart
```
  
DONE.
<br>
<br>

補充說明
=
欲查看 Fail2ban 的運作情況，可使用：
```bash
sudo iptables -L
```
查看 Fail2ban 已阻擋的 IP：
```bash
sudo fail2ban-client status
```
編輯 Fail2ban 的阻擋清單：
```bash
sudo nano /etc/fail2ban/filter.d
```

參考資源
=
* [Initial Server Setup with Ubuntu 12.04 | DigitalOcean](https://www.digitalocean.com/community/articles/initial-server-setup-with-ubuntu-12-04)
* [How to Protect SSH with fail2ban on Ubuntu 12.04 | DigitalOcean](https://www.digitalocean.com/community/articles/how-to-protect-ssh-with-fail2ban-on-ubuntu-12-04)
* [Fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page)
* [Fail2ban - Community Ubuntu Documentation](https://help.ubuntu.com/community/Fail2ban)
