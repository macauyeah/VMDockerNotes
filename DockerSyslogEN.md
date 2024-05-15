# docker syslog

When we build a single instance container, we have mulitple opitons for logging. The easiest one is log to file. But we need to map it to presistent storage, so that it won't be disappear after re-creating container.

Most of Docker hub official images, they redirect logs to stdout and stderr. Then the command ```docker logs``` will handle that all. Amazing!

Even in Swarm mode, they provide ```docker service logs``` to help you get aggregate log across nodes. Amazing!!! 

But the biggest problem of the default behavior is that, the logs will be gone when the container is deleted.

You might think that, you can simply write log to host os presistent storage through bind mount. It might not be okay if multiple replica of a service running at the same node. The logs might be ruined by mixing multiple messages.

A better option could be sending logs to ***syslog***, a robust service in Host OS. 

Following, is an example of handling docker logs with Ubuntu 22.04 ***syslog*** and ***logrotate***.

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

Add configuration in logrotate. Aim to archive the log file when it grows over 1M and create a new one instead. Keep at most 7 old log archive, delete the oldest if more than 7.
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
# do the real task and show verbose output
sudo /usr/sbin/logrotate -v /etc/logrotate.conf
```