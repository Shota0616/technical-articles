自宅LinuxサーバにKVM + QEMUをインストールしてみた

# 概要

最近VMware ESXiの無償版の提供が終了してしまったので、別の選択肢としてKVMを利用してみました。初めて触るので不慣れですが、調べつつVM立てるところまでは行けました。

https://kb.vmware.com/s/article/96168?lang=en_US

# KVMインストール

https://www.redhat.com/ja/topics/virtualization/what-is-KVM
> KVM (Kernel-based Virtual Machine：カーネルベースの仮想マシン) は、Linux® に組み込まれたオープンソースの仮想化テクノロジーです。KVM を使用すると、Linux をハイパーバイザーとして機能させることができます。

KVMとは、Linux組み込みのハイパーバイザのことです。KVM自身はエミュレーションを行わずに後述のQEMUが仮想マシンのエミュレーションを行ってKVMは仮想ハードウェアの処理と物理ハードウェアの仲介役を行っています。

ロード済のカーネルモジュールの一覧で確認してみます。KVMはLinuxに組み込まれているので、インストールは不要です。
```
lsmod | grep kvm
```
```
kvm_intel             368640  0
kvm                  1032192  1 kvm_intel
```

# QEMUインストール

仮想化マシンのエミュレータのことで、エミュレーションの対象はCPU,メモリ,I/Oデバイスに大別されます。


```
apt install qemu-kvm
```
```
vi /etc/libvirt/qemu.conf
```
```
user = "root"
group = "root"
dynamic_ownership = 1
```
:::note warn
rootで実行してしまっていますが、あまり良くないかもです。一般ユーザで実行するべきです。
:::

# libvirtインストール

libvirtは仮想化アプリケーションやハイパーバイザを操作するための抽象化レイヤーを提供しているソフトウェア郡です。

```
apt install virt-top libguestfs-tools virt-manager libvirt-daemon-system virtinst libvirt-clients bridge-utils
```

```
vi /etc/libvirt/libvirtd.conf 

auth_unix_ro = "none"
auth_unix_rw = "none"
unix_sock_group = "libvirt"
unix_sock_ro_perms = "0777"
unix_sock_rw_perms = "0770"
```

```
systemctl status libvirtd
systemctl enable libvirtd
systemctl start libvirtd
systemctl status libvirtd
usermod -aG kvm $USER
usermod -aG libvirt $USER
```
ディレクトリ作成
```
mkdir -p /home/shared/images
```

# cockpit

GUIの管理ツールとして`Virtual Machine Manager（virt-manager）`も存在しますが、redhatとしてはcockpitを推奨しているようなのでcockpitを使用します。

```
apt install cockpit
. /etc/os-release
sudo apt install -t ${VERSION_CODENAME}-backports cockpit
```
```
systemctl status cockpit
systemctl start cockpit
systemctl status cockpit
```

以下アクセス
https://{IP}:9090/

仮想マシン作成のプラグインをインストールする。
```
apt install cockpit-machines
```
cockpitのGUI

![スクリーンショット 2024-03-23 22.04.05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/8057cd96-20d3-eca9-5a11-e2ad7fa58d4e.png)


# ユーザーグループ修正

```
groups $(whoami)

usermod -aG kvm $(whoami)
usermod -aG libvirt $(whoami)
usermod -aG libvirtd $(whoami)

groups $(whoami)
```


# VM用のディレクトリ作成

```
# ディレクトリ作成
mkdir -p /home/shared/libvirt/images/
mkdir -p /home/shared/libvirt/images/isos/

# 権限とかグループ追加
groupadd shared
usermod --append --groups shared $(whoami)
chown -R root:shared /home/shared/
chmod -R o-rwx /home/shared/
```


# VM操作コマンド

ここはメモ感覚です。

VM一覧
```
virsh list
```

VMディスク一覧
```
virsh domblklist [VM名前]
```

VM停止
```
virsh shutdown [VM名前]
```

ネットワーク設定
```
virsh net-list
```

VMのIP確認
```
virsh net-dhcp-leases default
```

pool確認
```
virsh pool-list
```

pool削除
```
virsh pool-destroy [pool名前]
```

