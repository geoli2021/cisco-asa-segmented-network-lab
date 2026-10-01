# 🛡️ Laboratório de Redes e Segurança de Perímetro com Cisco ASA 5506-X e Core L3

Projetamento e implementação de uma infraestrutura corporativa segmentada por VLANs e Zonas de Segurança, utilizando **Switch L3 (Cisco 3560)** para roteamento inter-VLAN e **Firewall Cisco ASA 5506-X** para inspeção de estado (*Stateful Inspection*), ACLs de perímetro e regras de Identity NAT.

---

## 📐 Topologia da Rede

```text
[ PC-Admin ] (192.168.10.50) -- VLAN 10 (ADMIN)
      │
      ▼
[ SW-Acesso ] (2960 - L2 Trunk)
      │ (Gi0/1 - Trunk Dot1q)
      ▼
[ SW-Core-L3 ] (3560 - SVI Vlan10: 192.168.10.1)
      │ (Gi0/2 - Routed Port `no switchport` - 10.0.0.1/30)
      ▼
[ ASA0 (Firewall ASA 5506-X) ]
  ├── Interface Inside  (Gi1/1): 10.0.0.2/30   (Security Level: 100)
  ├── Interface Outside (Gi1/2): 172.16.0.1/30 (Security Level: 0)
  └── Interface DMZ     (Gi1/3): 192.168.30.1/24 (Security Level: 50)
      │
      ▼
[ Web Server ] (192.168.30.10 - HTTP/ICMP)
```

---

## 🎯 Objetivos do Laboratório

1. **Segmentação Layer 2/Layer 3**: Criação e transporte de VLANs via links Trunking no switch de acesso até o Core L3.
2. **Roteamento Inter-VLAN e Roteamento Estático**: Conversão de porta física do Core L3 em porta roteada (`no switchport`) e estabelecimento de tabela de rotas com a interface `inside` do Firewall.
3. **Segurança de Perímetro e Zonas DMZ**: Configuração de níveis de segurança (*Security Levels*) no Cisco ASA para isolamento da DMZ em relação à rede corporativa interna.
4. **Inspeção de Estado e Filtragem**:
   - Inspeção de pacotes ICMP e TCP via `global_policy`.
   - Access Control Lists (ACLs) bidirecionais permitindo tráfego legítimo de gerenciamento e requisições HTTP.
   - Implementação de **Identity NAT (No-NAT)** para preservação de endereçamento IP entre redes confiáveis e a DMZ.

---

## 🛠️ Principais Desafios Resolvidos Durante o Projecto

* **Portas L2 vs L3 no Switch Core**: Resolução de falhas de roteamento ao converter interfaces de camada 2 em portas roteadas L3 puras com o comando `no switchport`.
* **Tráfego de Retorno Bloqueado na DMZ**: Identificação de descartes de pacotes de retorno (DMZ → Inside) e correção via ACLs de entrada na interface `dmz` (`DMZ_IN`) e isenção de tradução NAT (`object network NET_INSIDE` / `nat (inside,dmz) static`).
* **Stateful Inspection no ASA**: Ajuste na `global_policy` para garantir inspeção ativa do protocolo ICMP e sessões HTTP (porta 80).

---

## 💻 Configurações Aplicadas (Destaques)

### 1. SW-Core-L3 (Porta Roteada para o ASA)
```cisco
ip routing
!
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
!
interface GigabitEthernet0/2
 description LINK-PARA-ASA-INSIDE
 no switchport
 ip address 10.0.0.1 255.255.255.252
!
ip route 0.0.0.0 0.0.0.0 10.0.0.2
```

### 2. Cisco ASA 5506-X (Rotas, Identity NAT e ACLs)
```cisco
interface GigabitEthernet1/1
 nameif inside
 security-level 100
 ip address 10.0.0.2 255.255.255.252
!
interface GigabitEthernet1/3
 nameif dmz
 security-level 50
 ip address 192.168.30.1 255.255.255.0
!
! Rotas Estáticas de Retorno
route inside 192.168.10.0 255.255.255.0 10.0.0.1
route inside 192.168.20.0 255.255.255.0 10.0.0.1
!
! Identity NAT (No-NAT entre Inside e DMZ)
object network NET_INSIDE
 subnet 192.168.10.0 255.255.255.0
 nat (inside,dmz) static 192.168.10.0
!
! Rules / Access Lists
access-list INSIDE-IN extended permit icmp any any
access-list INSIDE-IN extended permit tcp any host 192.168.30.10 eq 80
access-group INSIDE-IN in interface inside

access-list DMZ_IN extended permit ip host 192.168.30.10 any
access-group DMZ_IN in interface dmz
!
! Policy Map / Inspection
policy-map global_policy
 class inspection_default
  inspect icmp
```

---

## ✅ Validação e Testes

- [x] **Ping bidirecional ICMP** entre PC-Admin (`192.168.10.50`) e Servidor Web (`192.168.30.10`).
- [x] **Acesso HTTP** via Web Browser do PC-Admin carregando a aplicação no IP `http://192.168.30.10`.
- [x] **Inspeção de pacotes e contadores de ACL (`hitcnt`)** registrados com sucesso na CLI do Cisco ASA.

---

## 🧰 Ferramentas Utilizadas

* **Cisco Packet Tracer** (v8.x)
* **Cisco IOS Software** (3560 / 2960 Switches)
* **Cisco ASA Software** (v9.6+)
