# docker syslog

平常大家在做單機app時，寫log有很多選擇，最簡單就是寫在檔案中。但在docker container裏面，寫檔案時要注意怎樣保留log檔，避免因為重建container時不見了。

docker 大部份官方預設image，都把log導向至stdout和stderr。這是方便docker做管理，也方便大家使用統一的docker logs指令來查看，即使到了Swarm mode底下，docker service logs也是同樣原理，使用差異不大，頂多就是不保證log的實時性。

如果網路延遲不計較的話，最大問題也是logs怎樣保存的做法。預設就是container刪走的時候，logs也會一借走。單機模式下，沿用最普遍的方法寫log的做法不是不可行，只是考慮到在極端情況下，同一個node(節點)中，有可能同時運作同一個service(服務)的多個分身(replica)，這裏它們寫檔案時就有機會互相搶佔。

筆者認為，比較合理的是外部提供的服務，例如syslog，把寫檔的操作交給節點的Host OS處理。然後就保證好每筆log都會是一條完整的記錄。

以下就以linux Host裏面的syslog，為大家簡介一下設定的步驟。

## 設定docker 導向 syslog
把該主機的docker daemon (```/etc/docker/daemon.json```)，設定使用syslog driver，並以特定的方式編寫syslog tag。
```json
//file: /etc/docker/daemon.json
{
  "log-driver": "syslog",
  "log-opts": {
    "tag": "dockercontainer/{{.ImageName}}/{{.Name}}/{{.ID}}"
  }
}
```

無腦設定已完成，重啟docker就可以了。
```bash
sudo systemctl restart docker
```

但為了日後管理方便，能把docker log放進獨立的一個檔案中，會更易找問題。所以我們可以進一步設定syslog。我們以Ubuntu 22.04為例，可以在/etc/rsyslog.d/下增加一個設定檔(```/etc/rsyslog.d/*.conf```)，指定看到syslog tag以dockercontainer為首的記錄，都要獨立抽出來。
```conf
# file: /etc/rsyslog.d/51-docker.conf
:syslogtag,startswith,"dockercontainer" -/var/log/dockercontainer.log
```

為免有檔案權限問題，手動指定檔案的所有權後，才正式重啟syslog。然後所有相關記錄都會寫在/var/log/dockercontainer.log
```bash
sudo touch /var/log/dockercontainer.log
sudo chown syslog:adm /var/log/dockercontainer.log
sudo systemctl restart rsyslog
```

## 滾滾滾滾滾動的log檔
檔案一天一天地長大，如果可以，還是自動清掉太舊的記錄為妙。Linux Syslog，通常也會配著logrotate使用。

筆者亦以Ubuntu 22.04為例子，做了個最簡單的自動滾Log功能。目標就是當log檔案大於1M後，就要重開log檔。舊的log檔最多保留7份，多了就刪掉最舊的。
```conf
# file: /etc/logrotate.d/rsyslog-dockercontainer
/var/log/dockercontainer.log
{
        rotate 7
        size 1M
        missingok
        notifempty
        compress
        delaycompress
        sharedscripts
        postrotate
                /usr/lib/rsyslog/rsyslog-rotate
        endscript
}
```
加了設定後，什麼都不用重啟，因為它是Ubuntu 的排程動作，到執行時就會以最新的設定檔執行，詳見```/etc/cron.daily/logrotate```.

有需要手動測試的話，需要手動呼叫```/usr/sbin/logrotate```。加入-d參數後，會被視為debug mode，這是官方的說法，但因為debug mode沒有執行效果，更加像是linux中常見的dry run mode。
```bash
# dry run only, no effect
sudo /usr/sbin/logrotate -d /etc/logrotate.conf
# do the real task and show verbose output
sudo /usr/sbin/logrotate -v /etc/logrotate.conf
```