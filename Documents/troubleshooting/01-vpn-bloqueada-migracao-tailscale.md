# [VPN] Bloqueio de Portas em Redes Públicas (Migração para Tailscale)

## 🚨 O Problema (Sintomas)
Durante o acesso aos serviços do Homelab a partir de uma rede Wi-Fi pública (biblioteca), não foi possível estabelecer ligação. O "handshake" do cliente VPN para o host local não se concluía e o túnel não transmitia tráfego. Todos os serviços confinados (Nginx Proxy Manager, Portainer e serviços-core) ficaram inacessíveis remotamente.

## 🔍 Análise (Root Cause Analysis - RCA)
1. **Comportamento Partilhado:** O túnel estabelecia comunicação perfeita em ligações móveis (4G/5G). A falha era exclusiva ao estar conectado naquela rede pública Wi-Fi em específico.
2. **Diagnóstico Técnico:** O **WireGuard** standalone comunica via troca de datagramas via protocolo UDP através de portas predefinidas (no caso, 51820 UDP). Por motivos de controlo (bloqueio de descargas peer-to-peer / botnets), a firewall restritiva da biblioteca negava e desfazia ligações intermédias UDP nas saídas (`EGRESS`). 
3. Como os clientes clássicos de WireGuard não executam facilmente *fallback* para outras portas padrão abertas ou para TCP nativo, o túnel simplesmente tombava perante o bloqueio de rede ("UDP Blocking").

## 🛠️ A Solução (Arquitetura ZTNA & DERP Relays)
A solução passou por descartar o modelo clássico do WireGuard e implementar **Tailscale** (Zero-Trust Network Access). 
* O Tailscale gere lógicas avançadas de roteamento (*Hole Punching / NAT Traversal*). Em redes restritivas que bloqueiam tráfego UDP (como a biblioteca), o cliente executa *fallback* automático, encapsulando o tráfego via **DERP** (*Designated Encrypted Relay for Packets*) sobre TCP na porta 443 (HTTPS), perfurando a barreira de rede de modo transparente.
* O servidor físico foi configurado como **Subnet Router** (`--advertise-routes=${LAN_SUBNET}`), garantindo acesso direto aos IPs locais da infraestrutura através do túnel cifrado.

## ⚙️ Integração e Resolução de Atritos (O Verdadeiro Desafio)
A simples troca de VPN introduziu problemas de camada aplicacional que exigiram engenharia de tráfego:

1. **Split DNS para Manter SSL Válido:** 
   O acesso remoto via IP quebrava o Nginx Proxy Manager (NPM) e os certificados SSL. Para resolver, configurei **Split DNS** no *control plane* do Tailscale. Todo o tráfego para `*.${DDNS_DOMAIN}` foi forçado a resolver no meu servidor AdGuard local. Como o AdGuard já estava configurado para apontar estes domínios para o IP interno do servidor (`${SERVER_IP}`), o Tailscale interceta o pedido, traduz para o IP local e injeta-o no túnel, permitindo aceder aos serviços via domínio original de qualquer parte do mundo.

2. **Source NAT (SNAT) do Docker vs. Access Lists (NPM):**
   Mesmo com o tráfego a chegar, o NPM bloqueava a ligação com erro `HTTP 403 Forbidden`. O diagnóstico profundo revelou o problema: o motor de rede do Docker (bridge) executa **SNAT** no tráfego que entra via `tailscale0`. O IP de origem do cliente (`${TAILSCALE_IP}`) era mascarado pelo IP do *gateway* interno do Docker.
   * **Resolução Cirúrgica:** Em vez de comprometer o isolamento dos contentores ou expor portas locais, executei uma auditoria aos logs nativos do proxy (`docker exec npm sh -c 'grep 403 /data/logs/*'`) para identificar exatamente qual o IP do gateway que estava a bater na firewall aplicacional. 
   * Identificado o IP temporário do motor Docker (`${DOCKER_GATEWAY_IP}`), adicionei uma exceção estrita (`${DOCKER_GATEWAY_IP}/32`) na Access List do NPM. A segurança lateral foi mantida e o tráfego foi libertado.

## 🛡️ Prevenção e Benefícios Resultantes
1. **Resiliência Máxima e Fim de Interrupção:** Conectividade de rede integral, livre da infraestrutura alheia estar fechada ou ser muito restrita.
2. **Postura Extra de Segurança e Isolamento:** Redução direta da Superfície de Ataque. A implementação do tailscale ditou que eu já **não careço ou exijo port forwarding configurado no meu Router Principal**. Pude aceder ao Gateway de Casa e eliminar a rota "wan -> 51820 UDP" do NAT de receção do edifício, garantindo na prática agora um sistema estanque com **0 Incoming Active Ports**.
