# docker syslog

## Configure docker to syslog
Configure docker daemon to write log to localhost syslog
```json
//file: /etc/docker/daemon.json
{
  "log-driver": "syslog",
  "log-opts": {
    "tag": "dockercontainer/{{.ImageName}}/{{.Name}}/{{.ID}}"
  }
}
```

Apply change by restarting docker
```bash
sudo systemctl restart docker
```

Add configuration in rsyslog to filter all tag starts with "dockercontainer", put them into a seperate file. Because when we running in swarm mode, it will generate much much logs and not easily to rotate the log message.
```conf
# file: /etc/rsyslog.d/51-docker.conf
:syslogtag,startswith,"dockercontainer" -/var/log/dockercontainer.log
```

Avoid file permission problem, create the log file by command, and restart rsyslog
```bash
sudo touch /var/log/dockercontainer.log
sudo chown syslog:adm /var/log/dockercontainer.log
sudo systemctl restart rsyslog
```

## Rotate the log file
After adding a new log file, /var/log/dockercontainer.log, it will getting bigger and bigger. Apply logrotate to rotate and dispose the old logs.

Add configuration in logrotate.
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
No need to restart anything, it will take effect when next crontab daily job run (/etc/cron.daily/logrotate).

You might want to verify the configuration manually. You can dry run (debug mode) the configuration or run it directly with verbose output.
```bash
# dry run only, no effect
sudo /usr/sbin/logrotate -d /etc/logrotate.conf
# take effect now
sudo /usr/sbin/logrotate -v /etc/logrotate.conf
```