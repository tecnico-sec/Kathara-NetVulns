Instituto Superior Técnico, Universidade de Lisboa

**Segurança Informática em Redes e Sistemas**

# Guia de Laboratório - *Network Vulnerabilities*

## Objectivo

O objectivo deste guia consiste em aprofundar os conhecimentos sobre vulnerabilidades em redes, nomeadamente as vulnerabilidades associadas a protocolos que não cifram as comunicações (telnet) e ataques como *ARP spoofing, TCP reset* e ICMP *echo reply redirection*.

Neste laboratório vamos usar um laboratório Kathará com três máquinas da mesma LAN: *pc1, pc2* e *badpc*.
Execute esse laboratório, observe a topologia da rede e os endereços IP das máquinas.

Para realizar este guia é preciso o *nemesis,* que está instalado na imagem dos laboratórios.
Para obter informação sobre o *nemesis* use `man nemesis`.

## Exercício 1 -- Telnet

O comando *tcpdump* é um *sniffer* simples, que não tem interface gráfica (ao contrário do Wireshark).
O comando *telnet* permite obter um terminal remoto num computador, mas não deve ser usado porque não protege a comunicação.

1.  Corra o comando *tcpdump* no *badpc:*

```bash
tcpdump -i eth0 -X dst <IP do pc1>
```

2.  No *pc1* crie um utilizador chamado *teste* usando o comando:

```bash
adduser teste
```

3.  Corra o seguinte comando de modo a garantir que é possível fazer
    *telnet* para esse PC:

```bash
/etc/init.d/openbsd-inetd restart
```

4.  No *pc2* execute `telnet <IP do pc1>`. 
Repare como o *tcpdump* permite ver (*eavesdrop*) a *password* a ser enviada, caracter por caracter.

5.  Nota final: como pode observar o *telnet* é extremamente inseguro, logo nunca deve ser usado em canais inseguros. 
Uma alternativa é o *ssh*, que veremos mais tarde.

## Exercício 2 -- ARP *spoofing*

Em cada computador existe uma tabela ARP que contém associações entre endereços IP e endereços MAC. 
Um ataque de ARP *spoofing* consiste em alterar essa tabela no computador de uma vítima de modo a interceptar as suas comunicações. 
O objectivo pode ser observar essa comunicação ou realizar um ataque de *man-in-the-middle*.

1.  No *badpc* corra o comando `ping -c 1 <IP alvo>`.
Veja o conteúdo da tabela ARP no *badpc* correndo `arp -a`.
Veja que endereço MAC está associado ao endereço IP de cada PC.

2.  Obtenha o endereço MAC do *badpc* correndo nesse PC: `ifconfig eth0`

3.  Tome nota da correspondência entre cada endereço MAC e cada endereço
    IP para os três PCs.

4.  No *pc1* e no *pc2* faça *ping* ao outro e confirme que as tabelas
    ARP têm os mapeamentos entre endereço IP e endereço MAC (`arp -a`).

5.  As tabelas ARP desempenham um papel importante: quando um PC pretende enviar um pacote para um endereço IP, obtém na sua tabela ARP o endereço MAC correspondente a esse endereço IP (ou ao endereço IP do *router* que encaminhará o pacote para outra rede). 
O ataque consiste em usar o *nemesis* para fazer ARP *spoofing*.
Para o efeito, veja na documentação do *nemesis* (`man nemesis arp`) o que faz o comando abaixo e execute-o no *badpc*:

```bash
nemesis arp -v -S <IP do pc2> -D <IP do pc1> -h <MAC do badpc> -m <MAC do pc1>
```

6.  Para observar o resultado do ataque na tabela ARP, execute o seguinte comando no *pc1: arp -a .* 
Repare que este ataque não é permanente e que tem de repetir o comando acima periodicamente para a alteração permanecer (a cada 10 minutos aproximadamente).

7.  Para observar o efeito do resultado, execute o comando *tcpdump -i eth0* no *badpc* e no *pc2*, execute no *pc1* o comando *ping \<IP do pc2\>* e observe que os pacotes enviados pelo comando *ping* são redirecionados para o *badpc*.

## Exercício 3 -- TCP *reset*

O cabeçalho TCP tem um bit designado RST (*ReSeT*) que permite fechar a ligação TCP. 
Em concreto, quando dois computadores têm uma ligação TCP estabelecida entre eles e um recebe do outro um segmento TCP válido com esse bit posto a 1, fecha a ligação. 
Um ataque TCP *reset* ou RST *hijacking* consiste em enviar um segmento TCP com esse bit a 1 e o
endereço de origem modificado (*spoofed*) de modo a fechar uma ligação.

1.  No *pc1* execute o seguinte comando para observar apenas os segmentos TCP com o bit ACK (bit 13) a 1:

```bash
tcpdump -S -n -e -l "tcp[13]&16 == 16"
```

2.  No *pc2* faça uma ligação *telnet* para o *pc1*, pois esta estabelece uma ligação TCP (o servidor está à escuta no porto 23).

3.  No *badpc* use o *nemesis* para enviar um pacote de TCP *reset* para uma das máquinas:

```bash
nemesis tcp -v -fR -S <IP do pc1> -x 23 -D <IP do pc2> -y <portodo pc2> -s <sequence number>
```

> A informação sobre o *\<porto do PC2\>* e o *\< sequence number\>*
> pode ser obtida no output do *tcpdump.* O *\< sequence number\>* deve
> ser escolhido de modo a que o segmento TCP seja considerado válido
> pelo *pc2.*

4.  Verifique que a ligação foi fechada. 
Qual foi o valor *\< sequence number\>* usado?

## Exercício 4 -- ICMP *echo reply redirection*

O ataque consiste em falsificar (*spoof*) o endereço IP de origem enviado num pacote ICMP *echo request* de modo a que a resposta seja enviada para outro computador. 
Esta técnica é usada como parte de ataques de negação de serviço; é particularmente eficaz se o endereço IP de destino for um endereço de *broadcast* (mas geralmente existem *firewalls* que impedem esse tipo de ataque).

1.  Execute o seguinte comando no *badpc* para observar os endereços IP de origem e destino dos pacotes:

```bash
tcpdump "ip[9]=1"
```

2.  No *badpc* envie um pacote ICMP com um endereço de origem falsificado:

```bash
nemesis icmp -S <IP do pc2> -D <IP do pc1>
```

3.  Observe como o *badpc* consegue enganar o *pc1* fingindo que foi o *pc2* que enviou o pacote ICMP *echo request* (o primeiro pacote de um ping), já que o *pc1* responde ao *pc2* com um pacote ICMP *echo reply*. 
Assim, o *pc2* recebe uma resposta sem nunca ter enviado um pedido.

## Referências

-   Kathará, [https://github.com/KatharaFramework/Kathara/wiki][1]

-   `man nemesis`

-   Nemesis, [https://github.com/troglobit/nemesis][2]

  [1]: http://wiki.netkit.org/
  [2]: https://github.com/troglobit/nemesis
