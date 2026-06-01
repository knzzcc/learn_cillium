手動建/etc/vbox/networks.conf?

下午5:53





2 / 2

## Claude responded: 對，手動建，內容就一行：

對，手動建，內容就一行：

**Mac / Linux：**

bash

```bash
sudo mkdir -p /etc/vbox
echo "* 0.0.0.0/0 ::/0" | sudo tee /etc/vbox/networks.conf
```

完成後確認：

bash

```bash
cat /etc/vbox/networks.conf
# 應該顯示：* 0.0.0.0/0 ::/0
```

**Windows 不需要做這個**，這個問題只有 Mac/Linux 上的 VirtualBox 才會遇到。

原因是新版 VirtualBox 預設只允許 `192.168.56.0/21` 這個範圍建立 host-only network，但 Vagrant 的 VM 可能用其他 IP 段，這個檔案是告訴 VirtualBox「任何 IP 範圍都允許」。

你是 Mac 還是 Windows？