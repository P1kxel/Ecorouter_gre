# 🔐 Настройка защищённого туннеля EcoRouter

Полный гайд по настройке IPsec IKEv2 между двумя маршрутизаторами Ecorouter.

---

## 1. Создаём gre-тунель на устройстве

```bash
enable 
configure terminal 

interface tunnel.0
description "GRE"
ip address 10.10.10.1/30
ip tunnel <source_ip> <dest_ip> mode gre


exit
write memory
```

---
## И на втором устройстве


```bash
enable 
configure terminal 

interface tunnel.0
description "GRE"
ip address 10.0.0.2/30
ip tunnel <source_ip> <best_ip> mode gre


exit
write memory
```

---

## 2. Включаем протокол IKE

```bash
crypto-ipsec ike enable
```

---

## 3. Настроить профили IPsec

> Для создания туннеля IPsec используется протокол **IKE** (Internet Key Exchange).  
> Есть две фазы построения IPsec туннеля: **IKE фаза 1** и **IKE фаза 2**.

Создаём профиль:

```bash
crypto-ipsec profile CIPROFILE ike-v2
mode tunnel
```

### 🔑 IKE фаза 1

- На IKE этапе 1 два узла договариваются о протоколах шифрования, аутентификации, хеширования и других протоколах, которые они хотят использовать, а также о некоторых других необходимых параметрах.
- На этом этапе устанавливается сеанс **ISAKMP** (Internet Security Association and Key Management Protocol).
- Это также называется туннелем ISAKMP или туннелем первой фазы IKE.
- Туннель IKE фазы 1 используется **только для управляющего трафика**.

```bash
ike-phase1
proposal aes256-sha256-modp2048
auth pre-shared-key P@ssw0rd
exit
```

### 📡 IKE фаза 2

- Этот туннель используется как безопасный метод для организации второго туннеля, называемого туннелем IKE фазы 2 или туннелем IPsec.
- Второй туннель предназначен уже для **непосредственной передачи пользовательских данных**, а также для управляющих данных.
- После завершения фазы 2 IKE появится туннель фазы 2 IKE (или туннель IPsec), который можно использовать для защиты пользовательских данных.

```bash
ike-phase2 
protocol esp 
proposal aes256-sha256 
local-ts <source_ip>
remote-ts <dest_ip>
exit
exit
```

---

## 4. Задать крипто-карту

Необходимо указать, к какому пиру следует применять соответствующий профиль IPsec.

```bash
crypto-map CMAP 10
match peer <test_ip>
set crypto-ipsec profile CIPROFILE 
exit
```

---

## 5. Задать карты фильтрации

- Для каждого маршрутизатора необходимо вычленить исходящий трафик, который нужно зашифровать, и входящий, который нужно дешифровать.
- Исходящий (из локальной сети в туннель) трафик фильтруется по адресам локальной и удалённой подсети.
- Фильтруется любой тип трафика.
- К фильтр-карте привязана криптографическая карта `crypto-map`, которая, в свою очередь, ссылается на профиль `crypto-ipsec`.
- Трафик шифруется в соответствии с параметрами, указанными в профиле IPsec, и отправляется в туннель.

> **Для туннеля используется одна карта фильтрации, которая последовательно обрабатывает исходящий и входящий трафик, и оканчивается разрешающим правилом для прочего трафика:**

```bash
filter-map ipv4 FMAP 5
match gre host <source_ip> host <dest_ip>
set crypto-map CMAP peer <dest_ip>
exit
filter-map ipv4 FMAP 10
match udp host <dest_ip>eq 4500 host <source_ip> eq 4500
set crypto-map CMAP peer <dest_ip>
exit
filter-map ipv4 FMAP 15
match any any any
set accept
exit
```

Применяем на интерфейсах:

```bash
interface inet_interface
set filter-map in FMAP 10
exit
interface tunnel.0
set filter-map in FMAP 10
exit
write memory
```

---

## На втором устройстве мы зеркально выполняем те же самые действия
