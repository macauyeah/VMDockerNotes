# Swarm mode 上線
之前一直都討論Image 的打包形式，現在聊聊上線時的一些選擇

# command
swarm mode 主要通過"docker service" 指令去產生一堆可以在不同節點上運行的container。為了更加形象地講，我把container稱為Image的分身。

docker service create跟docker container run的感覺很像，兩者都可以指定image 