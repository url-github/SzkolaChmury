# [Zadanie domowe nr 10](https://szkolachmury.pl/google-cloud-platform-droga-architekta/tydzien-10-cloud-hybrid-connectivity/zadanie-domowe-nr-10/)

## 1. Zadanie 1
Utworzenie projektu przed przystąpieniem do zadania:
```bash
gcloud projects create "zadanie10"
```

### 1.1 Utworzenie sieci VPC w celu symulacji sieci lokalnej oraz produkcyjnej
```bash
vpcNetwork1="cloud"
vpcNetwork2="on-prem"

# cloud
gcloud compute networks create $vpcNetwork1 --subnet-mode=custom --bgp-routing-mode=global
# on-prem
gcloud compute networks create $vpcNetwork2 --subnet-mode=custom --bgp-routing-mode=global
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute networks list
NAME     SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
cloud    CUSTOM       GLOBAL
on-prem  CUSTOM       GLOBAL
```
</details>

### 1.2 Utworzenie podsieci
Podsieci utworzone zostały w dwóch różnych regionach ze względu na nałożony limit adresów **external IP** per region.
```bash
vpc1subnet1="vpcnetwork1-sub1"
vpc1subnet2="vpcnetwork1-sub2"
vpcRegion1="europe-west1"
vpcRegion2="us-central1"

# cloud
gcloud compute networks subnets create $vpc1subnet1 --network=$vpcNetwork1 --range=10.1.0.0/16 --region=$vpcRegion1
# on-prem
gcloud compute networks subnets create $vpc1subnet2 --network=$vpcNetwork2 --range=10.2.0.0/16 --region=$vpcRegion2
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute networks subnets list
NAME              REGION        NETWORK  RANGE
vpcnetwork1-sub1  europe-west1  cloud    10.1.0.0/16
vpcnetwork1-sub2  us-central1   on-prem  10.2.0.0/16
```

![screen](./img/20200222201006.jpg)
</details>

### 1.3 Dodanie reguł firewall dla ruchu SSH oraz ICMP
```bash
# cloud
gcloud compute firewall-rules create $vpcNetwork1-allow-icmp --direction=INGRESS --network=$vpcNetwork1 --action=ALLOW --rules=icmp --source-ranges=0.0.0.0/0
gcloud compute firewall-rules create $vpcNetwork1-allow-ssh --direction=INGRESS --network=$vpcNetwork1 --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0
# on-prem
gcloud compute firewall-rules create $vpcNetwork2-allow-icmp --direction=INGRESS --network=$vpcNetwork2 --action=ALLOW --rules=icmp --source-ranges=0.0.0.0/0
gcloud compute firewall-rules create $vpcNetwork2-allow-ssh --direction=INGRESS --network=$vpcNetwork2 --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute firewall-rules list
NAME                NETWORK  DIRECTION  PRIORITY  ALLOW   DENY  DISABLED
cloud-allow-icmp    cloud    INGRESS    1000      icmp          False
cloud-allow-ssh     cloud    INGRESS    1000      tcp:22        False
on-prem-allow-icmp  on-prem  INGRESS    1000      icmp          False
on-prem-allow-ssh   on-prem  INGRESS    1000      tcp:22        False
```
![screen](./img/20200222201529.jpg)
</details>

### 1.4 Utworzenie target VPN gateway - Classic VPN
```bash
vpngwName1="gw-$vpcNetwork1"
vpngwName2="gw-$vpcNetwork2"

# cloud
gcloud compute target-vpn-gateways create $vpngwName1 --network $vpcNetwork1 --region $vpcRegion1
# on-prem
gcloud compute target-vpn-gateways create $vpngwName2 --network $vpcNetwork2 --region $vpcRegion2
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute target-vpn-gateways list
NAME        NETWORK  REGION
gw-cloud    cloud    europe-west1
gw-on-prem  on-prem  us-central1
```
![screen](./img/20200222201858.jpg)
</details>

### 1.5 Rezerwacja publicznych adresów IP
```bash
gwIpName1="$vpngwName1-ip"
gwIpName2="$vpngwName2-ip"

# cloud
gcloud compute addresses create $gwIpName1 --region $vpcRegion1
# on-prem
gcloud compute addresses create $gwIpName2 --region $vpcRegion2 

# Zapisanie adresów do zmiennych
gwipAddress1=$(gcloud compute addresses describe $gwIpName1 --region $vpcRegion1 --format='get(address)')
gwipAddress2=$(gcloud compute addresses describe $gwIpName2 --region $vpcRegion2 --format='get(address)')
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute addresses list
NAME           ADDRESS/RANGE  TYPE      PURPOSE  NETWORK  REGION        SUBNET  STATUS
gw-cloud-ip    35.190.211.80  EXTERNAL                    europe-west1          RESERVED
gw-on-prem-ip  35.238.233.74  EXTERNAL                    us-central1           RESERVED
```
![screen](./img/20200222202129.jpg)
</details>

### 1.6 Utworzenie reguł dla ruchu IPSec
```bash
frgwName1="fr-$vpngwName1"
frgwName2="fr-$vpngwName2"

# cloud
gcloud compute forwarding-rules create $frgwName1-esp --ip-protocol ESP --address $gwIpName1 --target-vpn-gateway $vpngwName1 --region $vpcRegion1 
gcloud compute forwarding-rules create $frgwName1-udp500 --ip-protocol UDP --ports 500 --address $gwIpName1 --target-vpn-gateway $vpngwName1 --region $vpcRegion1
gcloud compute forwarding-rules create $frgwName1-udp4500 --ip-protocol UDP --ports 4500 --address $gwIpName1 --target-vpn-gateway $vpngwName1 --region $vpcRegion1

# on-prem
gcloud compute forwarding-rules create $frgwName2-esp --ip-protocol ESP --address $gwIpName2 --target-vpn-gateway $vpngwName2 --region $vpcRegion2
gcloud compute forwarding-rules create $frgwName1-udp500 --ip-protocol UDP --ports 500 --address $gwIpName2 --target-vpn-gateway $vpngwName2 --region $vpcRegion2
gcloud compute forwarding-rules create $frgwName1-udp4500 --ip-protocol UDP --ports 4500 --address $gwIpName2 --target-vpn-gateway $vpngwName2 --region $vpcRegion2
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute forwarding-rules list
NAME                 REGION        IP_ADDRESS     IP_PROTOCOL  TARGET
fr-gw-cloud-esp      europe-west1  35.190.211.80  ESP          europe-west1/targetVpnGateways/gw-cloud
fr-gw-cloud-udp4500  europe-west1  35.190.211.80  UDP          europe-west1/targetVpnGateways/gw-cloud
fr-gw-cloud-udp500   europe-west1  35.190.211.80  UDP          europe-west1/targetVpnGateways/gw-cloud
fr-gw-cloud-udp4500  us-central1   35.238.233.74  UDP          us-central1/targetVpnGateways/gw-on-prem
fr-gw-cloud-udp500   us-central1   35.238.233.74  UDP          us-central1/targetVpnGateways/gw-on-prem
fr-gw-on-prem-esp    us-central1   35.238.233.74  ESP          us-central1/targetVpnGateways/gw-on-prem
```
</details>

### 1.7 Utworzenie Cloud Router
```bash
routerName1="router-$vpcNetwork1"
routerName2="router-$vpcNetwork2"
asnRouter1=65001
asnRouter2=65002

# cloud
gcloud compute routers create $routerName1 --asn $asnRouter1 --network $vpcNetwork1 --region $vpcRegion1
# on-prem
gcloud compute routers create $routerName2 --asn $asnRouter2 --network $vpcNetwork2 --region $vpcRegion2
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute routers list
NAME            REGION        NETWORK
router-cloud    europe-west1  cloud
router-on-prem  us-central1   on-prem
```
![screen](./img/20200222202647.jpg)
</details>

### 1.8 Utworzenie tunelu VPN
Wskazówki dotyczące utworzenia Shared Secret można znaleść w [dokumentacji](https://cloud.google.com/vpn/docs/how-to/generating-pre-shared-key).
```bash
vpnTunelName1="vpn-tunel-$vpcNetwork1"
vpnTunelName2="vpn-tunel-$vpcNetwork2"
sharedSecret=""

# cloud
gcloud compute vpn-tunnels create $vpnTunelName1 --peer-address $gwipAddress2 --ike-version 2 --shared-secret $sharedSecret --router $routerName1 --target-vpn-gateway $vpngwName1 --region $vpcRegion1
# on-prem
gcloud compute vpn-tunnels create $vpnTunelName2 --peer-address $gwipAddress1 --ike-version 2 --shared-secret $sharedSecret --router $routerName2 --target-vpn-gateway $vpngwName2 --region $vpcRegion2
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute vpn-tunnels list
NAME               REGION        GATEWAY     PEER_ADDRESS
vpn-tunel-cloud    europe-west1  gw-cloud    35.238.233.74
vpn-tunel-on-prem  us-central1   gw-on-prem  35.190.211.80
```
![screen](./img/20200222202955.jpg)
</details>

### 1.9 Konfiguracja sesji BGP w sieci `Cloud`

#### 1.9.1 Konfiguracja interfejsu routera
```bash
routerInterfaceName1="$routerName1-interface-1"

gcloud compute routers add-interface $routerName1 --interface-name $routerInterfaceName1 --vpn-tunnel $vpnTunelName1 --region $vpcRegion1 
``` 

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute routers describe $routerName1 --region $vpcRegion1 --format='flattened(interfaces)'
interfaces[0].linkedVpnTunnel: https://www.googleapis.com/compute/v1/projects/zadanie10/regions/europe-west1/vpnTunnels/vpn-tunel-cloud
interfaces[0].name:            router-cloud-interface-1
# Wyświetlenie w tabeli
gcloud compute routers describe $routerName1 --region $vpcRegion1 --format="multi(interfaces:format='table[box](name,linkedVpnTunnel)')"
```
</details>

#### 1.9.2 Powiązanie peeringu BGP do interfejsu
```bash
peerName1="bgp-peer-$routerInterfaceName1"

gcloud compute routers add-bgp-peer $routerName1 --peer-name $peerName1 --peer-asn $asnRouter2 --interface $routerInterfaceName1 --advertisement-mode=DEFAULT --region $vpcRegion1

# Zapisanie adresów IP
bgpIpCloud=$(gcloud compute routers get-status $routerName1 --region $vpcRegion1 --format='get(result.bgpPeerStatus[0].ipAddress)')
bgpIpOnPrem=$(gcloud compute routers get-status $routerName1 --region $vpcRegion1 --format='get(result.bgpPeerStatus[0].peerIpAddress)')
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute routers get-status $routerName1 --region $vpcRegion1 --format='flattened(result.bgpPeerStatu
s[].ipAddress, result.bgpPeerStatus[].peerIpAddress)'
result.bgpPeerStatus[0].ipAddress:     169.254.243.137
result.bgpPeerStatus[0].peerIpAddress: 169.254.243.138
```
![screen](./img/20200222203503.jpg)
![screen](./img/20200222203610.jpg)
</details>

### 1.10 Konfiguracja sesji BGP w sieci `On-Prem`

#### 1.10.1 Konfiguracja interfejsu routera
```bash
routerInterfaceName2="$routerName2-interface-1"

gcloud compute routers add-interface $routerName2 --interface-name $routerInterfaceName2 --vpn-tunnel $vpnTunelName2 --ip-address $bgpIpOnPrem --mask-length 30 --region $vpcRegion2
```
<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute routers describe $routerName1 --region $vpcRegion1 --format='flattened(interfaces)'
interfaces[0].ipRange:         169.254.243.137/30
interfaces[0].linkedVpnTunnel: https://www.googleapis.com/compute/v1/projects/zadanie10/regions/europe-west1/vpnTunnels/vpn-tunel-cloud
interfaces[0].name:            router-cloud-interface-1
```
</details>

#### 1.10.2 Powiązanie peeringu BGP do interfejsu
```bash
peerName2="bgp-peer-$routerInterfaceName2"

gcloud compute routers add-bgp-peer $routerName2 --peer-name $peerName2 --peer-asn $asnRouter1 --interface $routerInterfaceName2 --advertisement-mode=DEFAULT --peer-ip-address $bgpIpCloud --region $vpcRegion2
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute routers get-status $routerName2 --region $vpcRegion2 --format='flattened(result.bgpPeerStatus[].ipAddress, result.bgpPeerStatus[].peerIpAddress)'
result.bgpPeerStatus[0].ipAddress:     169.254.243.138
result.bgpPeerStatus[0].peerIpAddress: 169.254.243.137
```
![screen](./img/20200222204125.jpg)
![screen](./img/20200222204155.jpg)
</details>

### 1.11 Sprawdzenie tablicy routingu

<details>
  <summary><b><i>Pokaż</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute routes list
NAME                            NETWORK  DEST_RANGE   NEXT_HOP                  PRIORITY
default-route-0d5571cf070d278a  cloud    10.1.0.0/16  cloud                     1000
default-route-23012612f2eca906  cloud    0.0.0.0/0    default-internet-gateway  1000
default-route-2ac5791fdbf6d6ff  on-prem  0.0.0.0/0    default-internet-gateway  1000
default-route-7aec4aa938873722  on-prem  10.2.0.0/16  on-prem                   1000
```
![screen](./img/20200222204519.jpg)
</details>

### 1.12 Utworzenie VM
```bash
vmName1="vm-cloud"
vmZone1="$vpcRegion1-b"
vmName2="vm-onprem"
vmZone2="$vpcRegion2-b"
vmType="f1-micro"

gcloud compute instances create $vmName1 --zone=$vmZone1 --machine-type=$vmType --network-interface=network=$vpcNetwork1,subnet=$vpc1subnet1 --image-project=debian-cloud --image=debian-9-stretch-v20191210
gcloud compute instances create $vmName2 --zone=$vmZone2 --machine-type=$vmType --network-interface=network=$vpcNetwork2,subnet=$vpc1subnet2 --image-project=debian-cloud --image=debian-9-stretch-v20191210
```

<details>
  <summary><b><i>Sprawdzenie</i></b></summary>

```bash
bartosz@cloudshell:~ (zadanie10)$ gcloud compute instances list
NAME       ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
vm-cloud   europe-west1-b  f1-micro                   10.1.0.2     35.195.115.163  RUNNING
vm-onprem  us-central1-b   f1-micro                   10.2.0.2     34.70.176.28    RUNNING
```
</details>

<details>
  <summary><b><i>ping</i></b></summary>

![screen](./img/20200222205734.jpg)
![screen](./img/20200222205746.jpg)
</details>

### 1.13 Test wydajności
Dodanie reguł firewall
```bash
gcloud compute firewall-rules create vpnrule-$vpcNetwork1 --network $vpcNetwork1 --allow tcp,udp,icmp --source-ranges 10.2.0.0/16
gcloud compute firewall-rules create vpnrule-$vpcNetwork2 --network $vpcNetwork2 --allow tcp,udp,icmp --source-ranges 10.1.0.0/16
```

<details>
  <summary><b><i>Pomiar wydajności TCP</i></b></summary>

on-prem -> cloud
![screen](./img/20200222211512.jpg)
![screen](./img/20200222211514.jpg)

cloud -> on-prem
![screen](./img/20200222211611.jpg)
![screen](./img/20200222211613.jpg)
</details>

<details>
  <summary><b><i>Pomiar wydajności UDP</i></b></summary>

on-prem -> cloud
![screen](./img/20200222211844.jpg)
![screen](./img/20200222211848.jpg)

cloud -> on-prem
![screen](./img/20200222212028.jpg)
![screen](./img/20200222212031.jpg)
</details>

### 1.14 Usunięcie projektu
```bash
gcloud projects delete "zadanie10"
```

## 2. Zbudowanie odpowiedniego połączenia do aplikacji znajdującej się on-premises.

Główne wymagania, które należy spełnić:

* Przez aplikacje będzie przepływać około 70 GB danych dziennie z dużym natężeniem przez co jakakolwiek przerwa połączenia nie jest akceptowalna.

* Dopasuj odpowiednie rozwiązanie, aby połączenie pomiędzy środowiskiem w Google Cloud, a środowiskiem lokalnym było dodatkowo szyfrowane oraz zawierało odpowiednie przepustowości, aby spełnić dzienny limit.

* Jaki komponent dodałbyś do schematu ,aby przełączanie pomiędzy źródłami przesyłu danych było w pełni automatyczne podczas awarii?

---

Wymagania:

* 70 GB danych dziennie
* zapewnione SLA
* szyfrowane połączenie z odpowiednią przepustowością
* automatyczne przełączenie pomiędzy źródłami przesyłu danych

Do wyboru mamy:
* Dedicated Interconnect
* Partner Interconnect
* Direct Peering
* Carrier Peering
* Cloud VPN

Odpowiednim wyborem będzie **HA Cloud VPN**:
* Przy odpowienim skonfigurowaniu połączenia możemy osiągnąć odpowiednie SLA (GCP zapewnia SLA na poziomie 99.99%).
* Możemy skonfigurować dodatkowe tunele używając kilku połączeń/dostawców zwiększając SLA.
* Jeden tunel VPN może obsługiwać połączenie do 3 Gbps.
* Brak uzależnienia się od providera/jednego połączenia.
* Szyfrowane połączenie IPSec. 
* Łączymy się przez VPN poza GCP, więc naliczane będą koszty zwykłego ruchu ([Cloud VPN pricing](https://cloud.google.com/vpn/pricing#cloud-vpn-pricing_1)), który jest o wiele tańszy w porównaniu z połączeniami Dedicated/Partner Interconnect ([Dedicated Interconnect Pricing tables](https://cloud.google.com/interconnect/pricing#dedicated-pricing-table), [Partner Interconnect Pricing tables](https://cloud.google.com/interconnect/pricing#partner-pricing-table)).
* Direct/Carrier Peering nie zapewniają SLA.

Automatyczne przełączenie pomiędzy źródłami przesyłu danych zapewni **Cloud Router**.