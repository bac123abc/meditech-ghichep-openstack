# Hướng dẫn đóng image CentOS 7 với Cloud-init (không dùng LVM)

## Bước 1: Tạo máy ảo bằng kvm

Bạn có thể dử dụng virt-manager hoặc virt-install để tạo máy ảo

Ở đây mình sử dụng virt-install

``` sh
# qemu-img create -f qcow2 /tmp/centos.qcow2 10G
# virt-install --virt-type kvm --name centos --ram 1024 \
  --disk /tmp/centos.qcow2,format=qcow2 \
  --network bridge=br0 \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=rhel7 \
  --location=/tmp/CentOS-7-x86_64-1611.iso
```

**Một số lưu ý trong quá trình cài đặt**

- Thay đổi Ethernet status (mặc định là off). Bên cạnh đó, hãy chắc chắn máy ảo nhận được dhcp

<img src="http://i.imgur.com/2so4Nlo.png">

- Đối với phân vùng dữ liệu, có một vài tùy chọn, ở đây mình không dùng LVM

Sau khi đã hoàn tất cài đặt, tiến hành gỡ bỏ ổ đĩa, libvirt yêu cầu bạn gán một ổ đĩa trống tại nơi trước đó cd-rom được gán, có thể là `hda`. Bạn có thể xem nó bằng câu lệnh ` virsh dumpxml vm-image`

``` sh
# virsh dumpxml centos
<domain type='kvm' id='19'>
  <name>centos</name>
...
    <disk type='block' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='hda' bus='ide'/>
      <readonly/>
      <address type='drive' controller='0' bus='1' target='0' unit='0'/>
    </disk>
...
</domain>
```

Chạy câu lệnh sau để gỡ bỏ nó và reboot lại máy.

``` sh
# virsh attach-disk --type cdrom --mode readonly centos "" hda
# virsh reboot centos
```

## Bước 2 : Cài các dịch vụ cần thiết

**ACPI service**

Để cho phép hypervisor có thể reboot hoặc shutdown instance, bạn sẽ phải cài acpi service.

``` sh
# yum install acpid
# systemctl enable acpid
```

**Cloud-init**

Để có thể nhận metadata, có một vài phương thức, ở đây mình dùng clou-init.

Tiến hành cài đặt dịch vụ:

`# yum install cloud-init`

Thay đổi cấu hình mặc định bằng cách sửa đổi file `/etc/cloud/cloud.cfg`. Ở đây mình sẽ thay đổi tên account được dùng bởi cloud-init thành admin

``` sh
users:
  - name: admin
    (...)
```

**cloud-utils-growpart**

Để pân vùng root có thể tự động resize thì ta cần cài đặt `cloud-utils-growpart`

`# yum install cloud-utils-growpart`

## Bước 3: Hủy bỏ zeroconf route

Để instance có thể access metadata service, ta cần disable default zeroconf route.

`# echo "NOZEROCONF=yes" >> /etc/sysconfig/network`

## Bước 4: Cấu hình consolse

Để sử dụng nova console-log, bạn cần phải làm theo một số bước sau :

- Thay đổi `GRUB_CMDLINE_LINUX`  option trong file `/etc/default/grub` từ `rhgb quiet` sang `console=tty0 console=ttyS0,115200n8`

- Chạy câu lệnh để lưu lại

`grub2-mkconfig -o /boot/grub2/grub.cfg`

## Bước 5: Tắt máy ảo

`# poweroff`

## Bước 6: Xóa bỏ MAC address details

`# virt-sysprep -d centos`

## Bước 7: Undefine the libvirt domain

`# virsh undefine centos`

## Bước 8: Giảm kích thước image

`# virt-sparsify --compress centos.img centos_shrink.img --convert qcow2`

**Link tham khảo**

https://docs.openstack.org/image-guide/centos-image.html
