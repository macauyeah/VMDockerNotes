# Schedule Job with Docker

在Linux底下，crontab是一個最簡單建立Schedule Job的方法。大家用crontab -e 就可以進入設定。
```
# crontab -e
*/1 * * * * /opt/run.sh
```

其中每個星號，順序代表的是分、時、日、月、星期。上面的例子就是不論何月何日何時，只要每一分鐘就執行一次```/opt/run.sh```

## Singleton Job
問題是，實際情況下，你想執行程式的時間都不一定會少於1分鐘。所以你總是有機會上一個job未跑完，下一個job就開始了。為了保障自已，需要一些參考機制，去決定是否讓job開始跑。

有些情況，可能你會想用job server去做監管，但若只為單線執行的工作，起一個job server還是會增加管理上的複雜性。

最簡單的做法，就是根據不同的程式語言，使用file lock（鎖上）的機制，先上鎖，再做事。但要注意考慮有沒有出現異常情況，令你自己反鎖自己。即是你的process死了，但不懂自己解鎖，這樣以後你也不能再執行了。
 
在Linux Bash Shell下，就有一個很簡單的做法，就是使用flock指令。用它的最大好處，就是從OS層面下，去鎖上。只要process結束了，不論正常還是不正常結束，都會自動解鎖。

以下例子就是在執行```/opt/run.sh```前，先要取得```/tmp/run.lockfile```的鎖。如果沒法取鎖，就自動放棄執行後面的指令。
```
flock -n /tmp/run.lockfile /opt/run.sh
```

```
# crontab -e
*/1 * * * * flock -n /tmp/run.lockfile /opt/run.sh
```

## Timeout
引入singleton的概念後，其實會引發另一個問題。因為異常的情況，還有機會是不生不死，process hang。所以我們還需要設定一個最大的執行時間，讓你的process在異常的情況下，被強行清走。

例如，ping指令在linux預設是永遠不會自動停止的，可以模擬process hang的情況。如果我們想定時從外部收走ping process，就可以使用timeout指令。以下指令就是2分鐘後殺指ping process。

```
# in file /opt/run.sh
timeout 2m ping localhost
# to check process id, you could use
# > ps aux | grep ping
# you will see two different id for ping and timeout
```

配合errorcode使用，你可能還會在想在timeout時送出一個email通知自已。

```
# in file /opt/run.sh
timeout 2m ping localhost
exitCode=$?
if [[ $exitCode -eq 124 ]]; then
    echo "timeout"
    # send email alert with timeout
elif [[ $exitCode -gt 0 ]]; then
    echo "exit with error"
    # send email alert with error exit
else
    echo "exit normal"
fi
```

配合docker使用，你可能需要考慮signal怎樣傳遞。

在筆者測試的環境中，似乎SIGTERM會被擋，也有可能是SIGTERM太強，它只把前景的docker container run收走，但其內的ping process還在docker daemon中行走。所以最後改用SIGINT，讓docker container run可以好好地把SIGINT傳入其內。
```
# It seems that docker captured the SIGTERM. Send SIGINT instead

# in file /opt/run.sh
timeout --signal=SIGINT 10s docker container run --rm pingtest -c 20
exitCode=$?
if [[ $exitCode -eq 124 ]]; then
    echo "timeout"
    # send email alert with timeout
elif [[ $exitCode -gt 0 ]]; then
    echo "exit with error"
    # send email alert with error exit
else
    echo "exit normal"
fi
```

還有一個常見的問題，就是中間夾雜了sudo，這樣前面的signal也是傳不過去，都被sudo擋了。所以只能把整段script都變成sudo或直接使用root執行；又或者把你的user加入docker group，執行docker command時就不用sudo。
```
# not work if wrapped sudo inside
timeout -signal=SIGINT 10s sudo docker container run --rm pingtest -c 20
timeout 10s sudo docker container run --rm pingtest -c 20

# workaround, add sudo outside of the whole script;
sudo /opt/run.sh
```

Full demo, github repo [cronjobWithDocker](https://github.com/macauyeah/cronjobWithDocker.git)
