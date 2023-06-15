# Notes about customizing cloud image for Multipass

## 前言
原本筆者只是想做docker cluster，但因為在實機中建VM極其麻煩，所以就研究了好一陣子如何快速起VM。

- Hyper-V有預設的Ubuntu template，但只有ubuntu desktop版，沒有server版。而且desktop gui顯得浪費資源，要clone VM也很廢時，放棄。
- Windows Subsystem Linux起VM很方便，但同一個Linux version只有一個instance，沒法起cluster，放棄。
- Virtual box沒有Ubuntu template，若要clone的話就變得跟Hyper-V差不多，放棄。

經過一輪資料搜集，發現了一個Ubuntu multipass engine，聲稱可以跨平台快速起VM。裏面有一些很吸引的功能，可以自己建立images、使用固定IP。

那怕即使是沒有snapshot，在自定義images的配合下預裝docker，要隨時加減cluster node都是一件容易的事。

## 重大問題
醜話說在前頭。經過一輪測試，multipass最大的問題，就是custom image、fix IP都只能在bare metal ubuntu 中才能使用。如果你沒有一台閒置實體機安裝Ubuntu，還是不要隨便試。

## 重點
詳細的Proof of Concept，筆者記錄了在
- [Packer template git repo link](https://github.com/macauyeah/ubuntuPackerImage) 
	- [Packer template.json link](https://github.com/macauyeah/ubuntuPackerImage/blob/main/template.json)
	- [Multipass run packer image script link](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerCluster.sh)
- [Multipass Static IP script snippet](MultipassStaticIpCN.md)
	- 筆者已經沒有多餘的bare metal ubuntu，沒法不斷重複測試這個安裝步驟。

在這裏就補充一些重點。
- packer是使用cloud-init和qemu的技術，行起template中指定的cloud image (在筆者的例子中就是ubuntu-22.04-server-cloudimg-amd64.img)
- 大家可以定義image行起後進行一些操作，而那些操作都是經過qemu vnc、ssh進去操作的。
- 操作完後就會直接儲存當時的image。所以在操作結束之前，盡可能地刪cache或刪去不要的user / group settings。
- 最後生成的image，還是一個cloud image。若要再運行它，必需要使用支援cloud-init的VM來讀取。
- cloud-init是用來指定初次運行時要設定的事，例如:hdd size, user account password, ssh key import等。
- 使用工具cloud-localds可以生成一個seed.img，這樣qemu也可以cloud-init。
- Hyper-V應該也可以經過類似方式，進行cloud-init，但筆者未有去實測。如有更簡便的方法請告知。
- multipass預設就已經有cloud-init，在bare metal ubuntu就可以直接執行。
- multipass也可以設定不同的cloud-init參數。

## 成品
最後筆者就選擇了用packer用來預裝docker，經mulitpass無腦起VM，再使用 [shell script](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerCluster.sh) 對多個node設定docker，達到即時起docker node的功能。這樣就減省了VM的安裝時間，也省去了docker的安裝問題。

說到底，如果只想測試docker cluster，其實windows, macOS中的multipass也可以實現相同的功能。因為安裝docker那些都可以經過shell script自動化，只是每次重複操作，都變得相當慢。另外，因為multipass在windows, macOS不支援fix ip，對於指定docker cluster interface又會再多一重功夫。