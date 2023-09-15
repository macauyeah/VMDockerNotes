# Oracle Database in Docker

雖然筆者之前有提過，Docker並不是萬能，Docker在管理有狀態應用(Stateful Application)的情況下，只能走單機路線。但因為Docker實在很方便，所以連Oracle Database這類強狀態應用也有出[Docker版本](https://github.com/oracle/docker-images/blob/main/OracleDatabase/SingleInstance/README.md)。當然，它在預設的情況下，只能在單機下操作。

不過即使在單機操作下，還是有一些跟其他Docker Image有差異的地方，需要特別拿出來聊聊。

假設根據官方的教學，跑起了一個oracle19c的Docker Container。再查看當中的Process，你會發現有一個內部PID為1的runOracle.sh

```
sudo docker container exec -it oracle19c top
## top output
##    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
##      1 oracle    20   0   11712   2756   2480 S   0.0  0.0   0:00.04 runOracle.sh
sudo docker container top oracle19c
## UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
## 54321               1069154             1069135             0                   Sep14               ?                   00:00:00            /bin/bash /opt/oracle/runOracle.sh
```

在Docker中這個PID為1的Process是很重要的，它是判斷整個Container有沒有運行的依據。它就是當初在Docker Image中Entrypoint或CMD指定的那個指令生起的Process。Docker daemon要進行停止指令，要停止container時，也是對著PID為1的那個process來處理。

一般的情況下，如果PID為1的那個process可以無腦地停了、重開，那一切都好辦。但在Oracle Database的情況下，就不適合。因為Database始乎都是有交易概念的(Transaction)，它的停止並不是殺了process就了事，它還要考慮HDD操作中，有那些可以被考慮為完成，有那些下次要還原(undo)、重做(redo)。如果殺了process就等於Oracle 的Shutdown Abort，有機會下次開機會，就會有交易異常而且無法決定該如何操作。

大家需要先進入Docker container，經sqlplus進行必要的關閉Database指令。但此時，PID為1的那個process，其實還在進行中，在Docker 層面，它就像是Docker Container還在正常運行中，只是Database離線了。又因為sqlplus關閉Database並不是馬上有結果的，所以在整體關閉時可能需要串連command。就像

```
# in container
echo "shutdown immediate;" | sqlplus -s / as sysdba && kill 1

# in docker host, by script
sudo docker container exec -it oracle19c /bin/sh -c 'echo "shutdown immediate;" | sqlplus -s / as sysdba' \
	&& sudo docker container stop oracle19c
```