# 自动备份脚本到 Gitee 和 GitHub 使用说明

本说明文档指导如何在 Linux 环境下，自动备份 `/home/xi/printer_data/config` 目录下所有文件和子目录内容至 Gitee 和 GitHub 仓库。脚本支持断网等待，网络恢复后自动推送。

---

## 1. 本地 Git 仓库初始化

### 1.1 进入备份目录
```bash
cd /home/xi/printer_data/config
```

### 1.2 初始化 Git 仓库（如未初始化过）
```bash
git init
```

### 1.3 添加 Gitee 仓库为远程
```bash
git remote add gitee git@gitee.com:你的gitee账号/你的仓库名.git
```

### 1.4 添加 GitHub 仓库为远程
```bash
git remote add github git@github.com:你的github账号/你的仓库名.git
```

### 1.5 配置 SSH 公钥到 Gitee 和 GitHub
- 生成密钥（如没有密钥）：
  ```bash
  ssh-keygen -t rsa -b 4096 -C "你的邮箱"
  ```
- 把 `~/.ssh/id_rsa.pub` 内容分别添加到：
  - Gitee [SSH 公钥管理页面](https://gitee.com/profile/sshkeys)
  - GitHub [SSH 公钥管理页面](https://github.com/settings/keys)

### 1.6 首次全量提交（如本地还没有提交）
```bash
git add .
git commit -m "首次全量备份"
git push gitee master
git push github master
```
> 如遇远程仓库冲突，先执行  
> `git pull gitee master --allow-unrelated-histories --no-rebase`  
> `git pull github master --allow-unrelated-histories --no-rebase`  
> 解决冲突后再推送。

---

## 2. 自动备份脚本设置

1. 编辑脚本 `/home/xi/auto_backup_cfg.sh`：
   ```bash
   nano /home/xi/auto_backup_cfg.sh
   ```
   内容如下：
   ```bash
   #!/bin/bash
   cd /home/xi/printer_data/config
   DELAY=60

   while inotifywait -e close_write,moved_to,create,delete -r .; do
     sleep $DELAY
     git add .
     git diff --cached --quiet || git commit -m "自动备份所有文件 $(date +'%Y-%m-%d %H:%M:%S')"
     # 检查网络是否通畅（ping gitee.com），推送到 Gitee
     while ! ping -c 1 gitee.com &>/dev/null; do
       echo "无网络，等待网络恢复再推送到 Gitee..."
       sleep 10
     done
     git push gitee master

     # 检查网络是否通畅（ping github.com），推送到 GitHub
     while ! ping -c 1 github.com &>/dev/null; do
       echo "无网络，等待网络恢复再推送到 GitHub..."
       sleep 10
     done
     git push github master
   done
   ```

2. 赋予脚本执行权限
   ```bash
   chmod +x /home/xi/auto_backup_cfg.sh
   ```

---

## 3. 配置 systemd 后台服务

1. 创建 systemd 服务文件
   ```bash
   sudo nano /etc/systemd/system/klipper-cfg-backup.service
   ```
   内容如下：
   ```ini
   [Unit]
   Description=Klipper .cfg 自动备份到 Gitee 和 GitHub
   After=network.target

   [Service]
   Type=simple
   User=xi
   WorkingDirectory=/home/xi/printer_data/config
   ExecStart=/home/xi/auto_backup_cfg.sh
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

2. 重新加载 systemd 配置
   ```bash
   sudo systemctl daemon-reload
   ```

3. 设置服务开机自启
   ```bash
   sudo systemctl enable klipper-cfg-backup
   ```

4. 启动服务
   ```bash
   sudo systemctl start klipper-cfg-backup
   ```

5. 检查服务状态
   ```bash
   sudo systemctl status klipper-cfg-backup
   ```

---

## 4. 验证与日常使用

- 每次 `/home/xi/printer_data/config` 目录下有文件或文件夹变动，脚本自动 commit 并在有网络时分别推送到 Gitee 和 GitHub。
- 断网时，脚本会等待网络恢复后自动推送。
- 可随时在两个远程仓库查看备份历史。

---

## 5. 常见问题排查

- 查看 systemd 服务日志
  ```bash
  journalctl -xe -u klipper-cfg-backup
  ```
- 手动运行脚本调试
  ```bash
  /home/xi/auto_backup_cfg.sh
  ```
- 停止服务
  ```bash
  sudo systemctl stop klipper-cfg-backup
  ```

---

## 6. 补充说明

- 每次推送前都检测网络，分别推送到 Gitee 和 GitHub。
- 如需修改脚本，修改后需重启 systemd 服务：
  ```bash
  sudo systemctl restart klipper-cfg-backup
  ```
- 支持全目录递归备份，包括所有子文件夹。
- 可以在 `.gitignore` 文件中设置忽略不需备份的文件。

---
