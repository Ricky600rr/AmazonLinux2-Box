# 1. Virtual Machineの準備 ~ 起動
## 1-1. 作業用ディレクトリを作成する
自身の好きな場所に作成する。
```
mkdir -p working_dir
cd working_dir
```
## 1-2. AmazonLinux2のVDIをダウンロードする
公式のURLからダウンロードして作業用ディレクトリに配置する。  
(以下の例では、Homebrewからwgetをインストールしているのでコマンドで実行している)  
```
wget https://cdn.amazonlinux.com/os-images/2.0.20190823.1/virtualbox/amzn2-virtualbox-2.0.20190823.1-x86_64.xfs.gpt.vdi
```
※ 記述時点では最新のVDIだが、更新されている可能性もあるので[公式サイト](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/amazon-linux-2-virtual-machine.html#amazon-linux-2-virtual-machine-download)を確認することをオススメする。
## 1-3. 設定ファイルを作業用ディレクトリにコピーする
```
cp ../meta-data ./
cp ../user-data ./
```
## 1-4. ISOを作成する
```
hdiutil makehybrid -o seed.iso -hfs -joliet -iso -default-volume-name cidata ./
```
- makehybrid           … ハイブリッドイメージとして起動イメージファイルを作成
- -o                   … 起動イメージファイルのファイル名
- -hfs                 … HFS+ファイルシステムを作成する
- -joliet              … ISO9660ファイルシステムを拡張したJolietファイルシステムで作成する
- -iso                 … ISO9660ファイルシステムを作成する
- -default-volume-name … ファイルシステムのデフォルトのボリューム名を指定  
※ AmazonLinux2のイメージ起動時にuser-dataはnocloudという方法で読み込まれる。nocloudでuser-dataファイルをディスクから読み込む場合は、cidataというボリュームラベルが付与されている必要がある。
## 1-5. Virtual Machineの作成する
```
VBoxManage createvm --name vagrant-amznlinux2 --ostype "RedHat_64" --register
```
## 1-6. SATA Controller(仮想ディスク)を設定する
```
VBoxManage storagectl vagrant-amznlinux2 --name "SATA Controller" --add "sata" --controller "IntelAHCI"
VBoxManage storageattach vagrant-amznlinux2 --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium amzn2-virtualbox-2.0.20190823.1-x86_64.xfs.gpt.vdi
```
## 1-7. IDE Controller(起動イメージ)を設定する
```
VBoxManage storagectl vagrant-amznlinux2 --name "IDE Controller" --add "ide"
VBoxManage storageattach vagrant-amznlinux2 --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium seed.iso
```
## 1-8.  基本項目を設定する
ssh はlocal:2222 → VirtualMachine:22とする。
```
VBoxManage modifyvm vagrant-amznlinux2 --natpf1 "ssh,tcp,127.0.0.1,2222,,22" --memory 1024 --vram 16 --audio none --usb off
```
## 1-9. Virtual Machineを起動する。
ヘッドレスモードで起動する。
```
VBoxManage startvm vagrant-amznlinux2 --type headless
```
## 1-10. SSH秘密鍵をダウンロードする。
```
curl -sL https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant -o vagrant.pem
cp vagrant.pem ~/.ssh/
chmod 600 ~/.ssh/vagrant.pem
```
## 1-11. SSHでサーバーに接続する。
```
ssh -o 'StrictHostKeyChecking no' -p 2222 vagrant@localhost -i ~/.ssh/vagrant.pem
```
# 2. Virtual Machineの設定変更
## 2-1. パッケージ追加と更新をする。
VirtualBox Guest Additions のビルドに必要なパッケージをインストールし、その後全てのモジュールを更新する。
```
sudo su
yum install kernel-devel kernel-headers gcc make perl bzip2 mod_ssl
yum update
reboot
```
## 2-2. VirtualBox Guest Additions をインストールする。
```
sudo su
cd
wget http://download.virtualbox.org/virtualbox/6.0.0_RC1/VBoxGuestAdditions_6.0.0_RC1.iso
mkdir /media/VBoxGuestAdditions
mount -t iso9660 -o loop VBoxGuestAdditions_6.0.0_RC1.iso /media/VBoxGuestAdditions
/media/VBoxGuestAdditions/VBoxLinuxAdditions.run
```
インストール出来たら後片付けをする。
```
umount /media/VBoxGuestAdditions
rmdir /media/VBoxGuestAdditions
rm VBoxGuestAdditions_6.0.0_RC1.iso
```
## 2-3. Virtual Machineをキレイにする。
起動時に保存されたMACアドレスの削除
```
rm -f /etc/udev/rules.d/70-persistent-net.rules
```
Bash のコマンド履歴の削除
```
export HISTSIZE=0
```
yum cacheの削除
```
yum clean all
rm -rf /var/cache/yum
```
/var/log ディレクトリに出力されたのログを削除
```
find /var/log -type f -exec cp -f /dev/null {} \;
```
仮想ハードディスクの領域を最適化
```
dd if=/dev/zero of=/ZERO bs=1M
rm -f /ZERO
```
# 3. Vagrant Boxの作成
## 3-1. Virtual Machineを停止する。
```
shutdown -h now
```
## 3-2. Vagrant Boxを作成する。
```
vagrant package --base vagrant-amznlinux2 --output amazonlinux2.box
```
## 3-3. Vagrant Boxを登録する。
```
vagrant box add --name amazonlinux2 amazonlinux2.box
```
## 3-4. 登録されていることを確認する。
```
vagrant box list
```
