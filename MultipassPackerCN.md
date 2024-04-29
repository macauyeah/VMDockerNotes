# Notes about customizing cloud image for Multipass

## 前言
原本筆者只是想做docker cluster，但因為在實機中建VM極其麻煩，所以就研究了好一陣子如何快速起VM。

以下是筆者有初步研究，但未有完全實行的方向。
- Hyper-V (2023-08-17 更新)
	- 有預設的Ubuntu template，但只有ubuntu desktop版，沒有server版。Desktop gui顯得浪費資源
	- clone VM很費時，需要通過Export，Import達到Clone效果。若好好地處理各VM的HDD存放位置，Export，Import也不錯。
	- 沒有自帶的cloud-init，需要自己想辦法做一些Clone後置的處理。
	- 若要走cloud image + cloud-init，可能需要[Windows ADK](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install)，並配合qemu做轉換。
- Windows Subsystem Linux
	- 起VM很方便，但同一個Linux version只有一個instance
	- 可能需要需要通過Export，Import Instance來達到Clone效果
	- 可能沒有fix ip。可能對於起docker swarm不利。
- Virtual box (2024/01/31 更新)
	- 沒有Ubuntu template
	- 若要clone的話就變得跟Hyper-V差不多。
	- Virtual Box, Hyper-V應該要跟Vagant結合，這樣可以取代multipass + packer的組合。
	- Vagent原本就提供了很多Image Template，官方稱為Box，但主要以供給Virtual Box使用為主。若你需要於Hyper-V執行特定的Linux，可能需要找一些第三方提供的Image Template。

經過一輪資料搜集，發現了一個Ubuntu multipass engine，聲稱可以跨平台快速起VM。裏面有一些很吸引的功能，可以自己建立images、使用固定IP。

那怕即使是沒有snapshot，在自定義images的配合下預裝docker，要隨時加減cluster node都是一件容易的事。

# Ubuntu multipass
## 參考軟硬件需求
醜話說在前頭。經過一輪測試，multipass最大的問題，就是custom image、fix IP都只能在ubuntu中才能使用。以下是筆者成功實現的軟硬件架構。
- 你可以使用實體主機，BIOS提供Virtualization，在實機上安裝Ubuntu，最後行起multipass。
  - 參考配置: 實機:CPU 10 core, 32G RAM, 1T HDD
- 你可以使用實機主機，BIOS提供Virtualization，在實機上安裝Windows 10 和Hyper-V，並在Hyper-V上安裝ubuntu，並對該Ubuntu提供ExposeVirtualizationExtensions。
  - 參考配置: 實機:CPU 10 core, 32G RAM, 1T HDD；VM Ubuntu: CPU 4 core，10G RAM，128G HDD

## 重點
詳細的Proof of Concept，筆者記錄了在
- [Packer template git repo link](https://github.com/macauyeah/ubuntuPackerImage) 
	- [Packer template.pkr.hcl link](https://github.com/macauyeah/ubuntuPackerImage/blob/main/template.pkr.hcl)
	- [Multipass run packer image script link](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerCluster.sh)
- [Multipass Static IP script snippet](MultipassStaticIpCN.md)

在這裏就補充一些重點。
- packer是使用cloud-init和qemu的技術，行起template中指定的cloud image (在筆者的例子中就是ubuntu-22.04-server-cloudimg-amd64.img)
- 大家可以定義image行起後進行一些操作，而那些操作都是經過qemu vnc、ssh進去操作的。
- 操作完後就會直接儲存當時的image。所以在操作結束之前，盡可能地刪cache或刪去不要的user / group settings。
- 最後生成的image，還是一個cloud image。若要再運行它，必需要使用支援cloud-init的VM來讀取。
- cloud-init是用來指定初次運行時要設定的事，例如:hdd size, user account password, ssh key import等。
- 使用工具cloud-localds可以生成一個seed.img，這樣qemu也可以cloud-init。
- Hyper-V應該也可以經過類似方式，進行cloud-init，但筆者未有去實測。如有更簡便的方法請告知。
- multipass預設就已經有cloud-init，在ubuntu就可以直接執行。
- multipass也可以設定不同的cloud-init參數。

## 成品
最後筆者就選擇了用packer用來預裝docker，經mulitpass無腦起VM，再使用 [shell script](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerCluster.sh) 對多個node設定docker，達到即時起docker node的功能。這樣就減省了VM的安裝時間，也省去了docker的安裝問題。

說到底，如果只想測試docker cluster，其實windows, macOS中的multipass也可以實現相同的功能。因為安裝docker那些都可以經過shell script自動化，只是每次重複操作，都變得相當慢。另外，因為multipass在windows, macOS不支援fix ip，對於指定docker cluster interface又會再多一重功夫。