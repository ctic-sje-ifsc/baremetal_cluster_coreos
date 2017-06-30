# Cluster CoreOS

Nesse repositório é descrito a configuração do [Cluster Coreos](https://coreos.com/os/docs/latest) que dá suporte ao projeto [Kubernetes](https://github.com/ctic-sje-ifsc/kubernetes).

Atualmente possuimos as seguintes especificações de Hardware:

* 1 x Dell PowerEdge R630: 
  * 1 x Intel(R) Xeon(R) CPU E5-2650 v3 @ 2.30GHz
  * 8 x 16 GB DDR4 2133 ECC
  * 4 x Broadcom BCM5720 em agregação de link LACP (IEEE802.3ad)
  
* 4 x HP Z220: 
  * 1 x Intel(R) Xeon(R) CPU E3-1225 V2 @ 3.20GHz 
  * 4 x 4 GB DDR3-1600
  * 2 x Gigabit Ethernet (Intel 82579LM e Realtek RTL8169) em agregação de link LACP (IEEE802.3ad)

__Totalizando no cluster 192 GB de memória RAM e 36 Núcleos de processamento.__


### O Sistema Operacional instalado nos nossos hosts é o [CoreOS Container Linux](https://coreos.com/os/docs/latest/).

"*O CoreOS é um sistema operacional Linux desenvolvido para ser tolerante à falhas, distribuído e fácil de escalar. Ele tem sido utilizado por times de operações e ambientes alinhados com a cultura DevOps. A principal diferença do CoreOS para outras distribuições Linux minimalistas é o fato de ser desenvolvido para suportar nativamente o funcionamento em cluster, possuir poucos binários e não possuir um sistema de empacotamento (como apt-get ou yum). O sistema operacional consite apenas no Kernel e no systemd. Ele depende de containers para gerenciar a instalação de software e aplicações no sistema operacional, provendo um alto nível de abstração. Desta forma, um serviço e todas as suas dependências são empacotadas em um container e podem ser executadas em uma ou diversas máquinas com o CoreOS.*" Fonte: http://www.ricardomartins.com.br/coreos-o-que-e-e-como-funciona/

Para a gerência das configurações utilizamos, como sugerido na documentação oficial do CoreOS, tudo em um arquivo [cloud-config](https://coreos.com/os/docs/latest/cloud-config.html), que pode ser encontrado em cada pasta coreos0, coreos1 ... para cada nó. Por exemplo: [aqui](https://github.com/ctic-sje-ifsc/coreos/blob/master/coreos0/user_data).

## Podemos destacar as seguintes configurações:

* Manter a versão [stable](https://coreos.com/releases) do CoreOS, utilizando determinada janela para reinicio das máquinas em atualização automática e garantindo que somente um nó do cluster reinicie por vez:

```yaml
  update:
    reboot-strategy: "etcd-lock"

  locksmith:
    window-start: Thu 11:30
    window-length: 1h
    
  - path: /etc/coreos/update.conf
    permissions: 0644
    owner: root
    content: |
      GROUP=stable
```

* Configuração da rede e da agregação de link LACP:

```yaml
    - name: down-interfaces.service
      command: start
      content: |
        [Service]
        Type=oneshot
        ExecStart=/usr/bin/ip link set eno1 down
        ExecStart=/usr/bin/ip addr flush dev eno1
        ExecStart=/usr/bin/ip link set eno2 down
        ExecStart=/usr/bin/ip addr flush dev eno2
        ExecStart=/usr/bin/ip link set eno3 down
        ExecStart=/usr/bin/ip addr flush dev eno3
        ExecStart=/usr/bin/ip link set eno4 down
        ExecStart=/usr/bin/ip addr flush dev eno4
        
    - name: systemd-networkd.service
      command: restart
      
  - path: /etc/modprobe.d/bonding.conf
    content: |
      alias bond0 bonding
        options bonding mode=4 miimon=100 lacp_rate=1

  - path: /etc/modules
    content: |
      bonding
      mii
      
  - path: /etc/systemd/network/10-eno.network
    permissions: 0644
    owner: root
    content: |
      [Match]
      Name=eno*
      [Network]
      Bond=bond0
      
  - path: /etc/systemd/network/20-bond.netdev
    permissions: 0644
    owner: root
    content: |
      [NetDev]
      Name=bond0
      Kind=bond
      
  - path: /etc/systemd/network/30-bond-static.network
    permissions: 0644
    owner: root
    content: |
      [Match]
      Name=bond0
      [Network]
      DNS=191.36.8.2
      DNS=191.36.8.3
      Address=191.36.8.8/27
      Gateway=191.36.8.30
      Domains=sj.ifsc.edu.br
```

* Configuração do etcd, que fornece descoberta de configuração e serviço compartilhada para clusters do Container Linux(https://coreos.com/etcd/docs/2.3.7/clustering.html):

```yaml
    - name: etcd2.service
      command: start
      enable: true
      drop-ins:
      - name: 10-environment.conf
        content: |
          [Service]
          Environment="ETCD_ADVERTISE_CLIENT_URLS=http://coreos0.sj.ifsc.edu.br:2379"
          Environment="ETCD_DISCOVERY_SRV=sj.ifsc.edu.br"
          Environment="ETCD_INITIAL_ADVERTISE_PEER_URLS=http://coreos0.sj.ifsc.edu.br:2380"
          Environment="ETCD_INITIAL_CLUSTER_STATE=existing"
          Environment="ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379"
          Environment="ETCD_LISTEN_PEER_URLS=http://coreos0.sj.ifsc.edu.br:2380"
          Environment="ETCD_NAME=coreos0"
```
* Configuração do flannel, é descrito no projeto [kubernetes](https://github.com/ctic-sje-ifsc/kubernetes):

```yaml
    - name: flanneld.service
      drop-ins:
        - name: 50-network-config.conf
          content: |
            [Service]
            ExecStartPre=/usr/bin/etcdctl --discovery-srv sj.ifsc.edu.br set /coreos.com/network/config '{ "Network": "10.2.0.0/16","Backend":{"Type":"vxlan"}}'
        - name: 40-ExecStartPre-symlink.conf
          content: |
            [Service]
            ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env
      command: start
      enable: true
```

* Configuração de data e timezone(https://coreos.com/os/docs/latest/configuring-date-and-timezone.html):

```yaml
      
    - name: systemd-tinesyncd.service
      command: stop
      mask: true
    - name: ntpd.service
      command: start
      enable: true
      
    - name: settimezone.service
      command: start
      content: |
        [Unit]
        Description=Set the time zone

        [Service]
        ExecStart=/usr/bin/timedatectl set-timezone America/Sao_Paulo
        RemainAfterExit=yes
        Type=oneshot

   - path: /etc/systemd/timesyncd.conf
    permissions: 0644
    owner: root
    content: |
      [Time]
      NTP=pool.ntp.br ntp.ufsc.br ntp.cais.rnp.br
      
  - path: /etc/ntp.conf
    content: |
      server pool.ntp.br
      server ntp.ufsc.br
      server ntp.cais.rnp.br

      # - Allow only time queries, at a limited rate.
      # - Allow all local queries (IPv4, IPv6)
      restrict default nomodify nopeer noquery limited kod
      restrict 127.0.0.1
      restrict [::1]
```
