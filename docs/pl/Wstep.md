


TWRP recovery (lub inny współczesny recovery) jest idealnym środowiskiem do benchmarku i ogólnie mówiąc uruchamiania dystrybucji Linux. Zawiera w sobie wszystkie binarki od producenta i na ogół działa stabilnie. W tym notatniku wyjaśnię jak zainstalować ubuntu na twrp. 

setup_ubuntu_rootfs.md jest notatnikiem, w którym będę bardziej wyjaśniał jak zrobić własny rootfs. Moja instalacja ubuntu jest do pobrania [tutaj](https://drive.google.com/file/d/1heMCehAZoDS_dTN4GH2TcRQuno63to9P/view?usp=sharing).

Katalogiem, w którym rozpakuję ubuntu rootfs będzie /data/ubuntu. Nie trzeba formatować partycji userdata. Nie wszystkie foldery mają File Based Encryption (FDE). 

Najpierw wrzuć ubuntu rootfs na ramdysk, aby następnie rozpakować na internal storage ubuntu. TWRP pobiera tak mało ramu, że nawet jeśli odpalę kompilację linux kernel na 8 rdzeniach to nadal będzie 3gb wolnej pamięci ram, bez swapu (ani zramu). Końcowa faza budowania, etap linkowania zabiera całą wolną pamięć RAM. Nie będziemy rozpakowywać ubuntu na ramdysku. Zainstalujemy na stałe. Myślę, że twrp ma limit czasowy.

```
adb push  ubuntu_rootfs_shared.tar  /tmp
mkdir -p /data/ubuntu; cd /data/ubuntu
tar xvf /tmp/ubuntu_rootfs_shared.tar
```

Następnie robimy bind-mount, aby był dostęp do różnych linuksowych pseudosystemów plików.

```
for i in dev proc sys tmp ; do   mount -o rbind /$i $i/; done
```
Teraz robimy chroot na ubuntu. Zwróć uwagę, że bez aktualizowania PATH powłoka nie może znaleźć żadnych programów. Ustawiamy governor na performance, aby wszystkie rdzenie działały z maksymalną częstotliwością.

```
chroot .
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$PATH
alias sudo=''
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
bash
```

**Networking**

Teoretycznie możliwe jest ustawienie wifi, ale trzeba załadować moduły do wifi i muszą być one skompilowane razem z kernelem. Mam zamiar w przyszłości wrzucić gotowy twrp z wifi. Teraz ustawimy internet po usb gadget. Host musi mieć uruchomionego Linuksa.

Host commands:

```
#dostosuj do siebie
SRC=wlp3s0
DEST=enx020000000002

sudo iptables -t nat -A POSTROUTING -o $SRC -j MASQUERADE
sudo iptables -A FORWARD -i $SRC -o $DEST -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i $DEST -o $SRC -j ACCEPT

#Ustaw IP
sudo busybox ifconfig $DEST 10.0.0.1 netmask 255.0.0.0 up
sudo sysctl -w net.ipv4.ip_forward=1
```
TWRP Device commands:
(inside ubuntu)
```
#enable rndis
cd /config/usb_gadget/g1
mkdir functions/rndis.usb0
echo "02:00:00:00:00:01" > functions/rndis.usb0/dev_addr
echo "02:00:00:00:00:02" > functions/rndis.usb0/host_addr
ln -s functions/rndis.usb0 configs/b.1/  
#disconnect aby wdrożyć w życie!

busybox ifconfig rndis0 10.0.0.2 netmask 255.0.0.0 up
#Set default gateway!!   
busybox ip route add default via 10.0.0.1 dev rndis0
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/resolv.conf
ping tvn24.pl
```

**Benchmark**

Współczesne mobilne SoC mają architekturę big.LITTLE, co oznacza, że mają 4 rdzenie (klaster) energoszczędne i 4 wydajne rdzenie. Aby poprawnie zrobić benchmark trzeba uwzględnić to, chociażby przypinając dane rdzenie do benchmarku. Oddzielnie "big", oddzielnie "little" testować. Cortex A53 to rdzenie in-order: instrukcje wykonują się zgodnie z kolejnością, rozmiar rdzenia na kawałku krzemu jest malutki, prostsza architektura wewnątrz rdzenia. Wszystkie współczesne rdzenie x86 są out-of-order, poza Intel Atom n270 - który z tego powodu jest STRASZNIE wolny i jest wręcz memem. Teraz wiele zmieniło się i wierzę, że najnowsze cpu Intela stosują też architekturę big.LITTLE, a więc również in-order.

Ten tani cpu nie stosuje out-of-order, ale można zidentyfikować wydajniejsze rdzenie po ich maksymalnej częstotliwości lub "pojemności" rdzenia.

```
for cpu in /sys/devices/system/cpu/cpu*/cpu_capacity; do
    echo "$cpu: $(cat $cpu)"
done



for cpu in /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq; do
    echo "$cpu: $(cat $cpu)"
done

```
