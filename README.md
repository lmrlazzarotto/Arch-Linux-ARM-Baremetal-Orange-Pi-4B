# Arch-Linux-ARM-Baremetal-Orange-Pi-4B

### Compilação da Cadeia de Boot (Firmware de Baixo Nível)

O projeto Arch Linux ARM é purista e o mais agnóstico de hardware possível. O sistema operacional exige que o operador construa e posicione o bootloader apropriado na unha. Recuso sumariamente a utilização de binários pré-compilados de fóruns não auditados de terceiros. A única maneira de garantir estabilidade é a compilação cruzada (*cross-compilation*) a partir dos repositórios fonte originais.

O calcanhar de Aquiles da Orange Pi 4B é o treinamento inicial da memória RAM. A configuração padrão `orangepi-rk3399_defconfig` possui a diretiva `CONFIG_TPL=y` cravada em seu código. Isso força o U-Boot a tentar inicializar o silício por conta própria usando o TPL (*Tiny Program Loader*) de código aberto. O grande problema é que esse TPL possui falhas conhecidas de temporização para memórias LPDDR4 no RK3399, detectando a matriz incorretamente como DDR3 a 400MHz (ou 50MHz em algumas revisões) e inevitavelmente colapsando com a falha de treinamento `rk3399_dmc_init DRAM init failed -22` em um bootloop infinito.

Para garantir estabilidade de nível corporativo e extirpar este obstáculo em definitivo, não confiaremos no arquivo `idbloader.img` gerado automaticamente pelo comando `make`. Vamos assumir o controle e forjar este artefato primário manualmente, fundindo o binário proprietário da Rockchip diretamente com o estágio secundário do U-Boot.

#### Preparação da Toolchain e Compilação do ARM Trusted Firmware (ATF)

O ambiente host de compilação (preferencialmente uma máquina física x86_64 robusta rodando Linux) exige a presença da toolchain GNU `aarch64-linux-gnu-gcc`. A forja do firmware de nível de exceção seguro (EL3) é o primeiro passo:

```bash
# Clone e compilação rigorosa do ARM Trusted Firmware direto da fonte upstream
git clone https://github.com/ARM-software/arm-trusted-firmware.git
cd arm-trusted-firmware

# Limpeza de artefatos de compilações anteriores
make realclean

# Compilação direcionada à plataforma RK3399
make CROSS_COMPILE=aarch64-linux-gnu- PLAT=rk3399 bl31

# Exporte o caminho absoluto do artefato gerado como variável de ambiente
export BL31=$(pwd)/build/rk3399/release/bl31/bl31.elf
cd ..

```

#### Compilação Híbrida do U-Boot

**O Cofre e o U-Boot**

```bash
# Clone da árvore mainline do U-Boot
git clone https://source.denx.de/u-boot/u-boot.git

# Clone o cofre da Rockchip para obter os blobs proprietários de RAM
git clone https://github.com/rockchip-linux/rkbin.git

```

**Limpeza, Defconfig e Compilação**

Ajuste a árvore de dependências para o hardware correto e dispare a compilação:

```bash
cd u-boot

# Purga de qualquer artefato de compilação anterior
make mrproper

# Aplica o defconfig base da placa
make orangepi-rk3399_defconfig

# Dispara a compilação paralela do U-Boot e do seu estágio secundário (SPL)
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

```

> **AVISO CRÍTICO:** Se a variável de ambiente `$BL31` não houver sido exportada corretamente no passo anterior, o processo falhará silenciosamente com a mensagem de alerta `WARNING: BL31 file bl31.elf NOT found, resulting binary is non-functional` e o artefato vital não será criado de forma utilizável. Verifique os logs.

#### A Engenharia de Montagem do IDBLoader (Bypass LPDDR4)

O arquivo `idbloader.img` gerado automaticamente pelo passo anterior contém o TPL *open-source* defeituoso. Ignore-o sumariamente. A partir da raiz do diretório do U-Boot que você acabou de compilar, execute a seguinte engenharia de montagem:

```bash
# Utilize a ferramenta nativa mkimage do U-Boot para envelopar o blob proprietário de treinamento de DDR da Rockchip como o bloco inicial.
# Nota: Adapte o caminho do .bin para coincidir com a versão LPDDR4 mais recente do seu diretório rkbin.
./tools/mkimage -n rk3399 -T rksd -d ../rkbin/bin/rk33/rk3399_ddr_800MHz_v1.30.bin idbloader.img

# Concatene o Secondary Program Loader (SPL) — recém-compilado na pasta spl/ — diretamente no final do arquivo mágico recém-criado.
cat spl/u-boot-spl.bin >> idbloader.img

```

#### O Saldo da Operação (Artefatos Invioláveis)

A execução imaculada desta cadeia consolida, no diretório raiz do repositório U-Boot, os dois artefatos exigidos para gravar diretamente no disco e tracionar o sistema:

* **`idbloader.img`**: O binário fundido na unha contendo o inicializador de RAM proprietário da Rockchip e o SPL do U-Boot.
* **`u-boot.itb`**: O contêiner *Flattened Image Tree* abrangendo o núcleo do U-Boot e o payload seguro BL31.

Estes arquivos serão salvaguardados. Eles são a chave mestra para o funcionamento do hardware e a fundação da migração baremetal.

---

### Arquitetura do Rootfs, Geometria de Disco e Instalação AArch64

O próximo estágio exige a estruturação completa do ambiente do sistema operacional. Esta fase deve ser primeiramente construída em um cartão MicroSD confiável de classe industrial. Esta mídia de armazenamento autônoma atuará como nosso "veículo de implantação de estadiamento" (*staging deployment vehicle*). É uma negligência arquitetônica inaceitável tentar aplicar as partições diretamente ao eMMC de 16GB utilizando métodos esotéricos sem antes validar o kernel, a árvore de dispositivos e o comportamento térmico no ambiente local seguro do MicroSD.

#### Geometria de Partição do Veículo de Implantação (Cartão SD)

Insira o cartão MicroSD na estação de trabalho host (assumiremos que ele foi enumerado pelo kernel do host como `/dev/sdX`. A identificação correta via comando `lsblk` é responsabilidade exclusiva do operador para evitar destruição de dados do próprio computador).

O dispositivo deve ser purgado de qualquer cabeçalho de assinatura lógica nos primeiros 32 megabytes, aniquilando resíduos de antigas tabelas GPT ou vestígios de BootROM de outras distribuições.

```bash
# Erradicação preventiva de metadados de armazenamento
wipefs -a /dev/sdX
dd if=/dev/zero of=/dev/sdX bs=1M count=32 status=progress

```

Utilizaremos o utilitário determinístico `parted` para forjar a estrutura lógica. A partição que abrigará o kernel (boot) deve começar impreterivelmente no setor `32768s`.

```bash
# Iniciação da Tabela GPT
parted -s /dev/sdX mklabel gpt

# Definição da partição de boot protegendo os 16MB iniciais do disco
parted -s /dev/sdX mkpart boot fat32 32768s 512MiB
parted -s /dev/sdX set 1 boot on

# Definição da partição raiz do sistema (Rootfs) ocupando o restante do espaço
parted -s /dev/sdX mkpart root ext4 512MiB 100%

```

A formatação rigorosa dos sistemas de arquivos com rótulos semânticos (Labels) é um princípio organizador necessário:

```bash
mkfs.vfat -F32 -n "BOOT" /dev/sdX1
mkfs.ext4 -L "ROOT" /dev/sdX2

```

#### Extração de Baixo Nível do Rootfs Arch Linux

A filosofia de distribuição do Arch Linux é purista; ela não oculta o sistema de arquivos por trás de instaladores baseados em interfaces. O tarball contém a árvore de diretórios exata do sistema vivo. A extração deste pacote em um sistema de arquivos externo (ext4 no cartão SD) não tolera falhas na transposição das permissões de propriedade, links simbólicos relativos a bibliotecas vitais em `/usr/lib` e atributos estendidos (ACLs).

A operação a seguir deve obrigatoriamente ser executada com plenos privilégios de root utilizando a ferramenta `bsdtar`, pois versões antigas de tar GNU podem truncar hardlinks complexos.

```bash
# Preparação dos pontos de montagem da arquitetura
mkdir -p /mnt/arch_root
mount /dev/sdX2 /mnt/arch_root

mkdir -p /mnt/arch_root/boot
mount /dev/sdX1 /mnt/arch_root/boot

# Obtenção do artefato criptograficamente auditável
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz

# Extração rigorosa preservando a anatomia dos metadados do POSIX
bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt/arch_root

```

> **Nota de Engenharia:** Certifique-se de que a operação concluiu sem nenhum erro ou interrupção. O silenciamento de um erro de leitura/escrita nesta etapa gerará falhas indetectáveis no tempo de execução de bibliotecas C nativas.

#### Orquestração da Passagem de Parâmetros de Inicialização (Extlinux)

A arquitetura de sistemas baseados em UEFI x86_64 é primariamente dominada pelo carregador GRUB2 ou systemd-boot. Em contraste profundo, o U-Boot nos sistemas Rockchip prefere e prioriza a semântica de script baseada na especificação do projeto Syslinux/Extlinux. Devemos criar e popular manualmente o arquivo `/boot/extlinux/extlinux.conf` na raiz do sistema recém-extraído.

Este arquivo tem o poder executivo de instruir o U-Boot sobre em quais blocos buscar o kernel binário compilado (`Image`), qual arquivo espelha a imagem temporária de drivers de boot (`initramfs-linux.img`) e, crucialmente, qual a topologia de hardware (Device Tree Blob - `.dtb`) a ser injetada na memória ram para conhecimento do Linux.

O Device Tree exato necessário para operar os controladores elétricos de sua placa é o `rk3399-orangepi.dtb`. Apesar da nomenclatura ser "Orange Pi 4", este DTB encapsula corretamente os endereços de memória, controladores de voltagem I2C e a topologia PHY RGMII do Orange Pi 4B para propósitos estritos de networking e boot, ignorando o NPU conforme nosso desejo. Ele já se encontra nativamente instalado no diretório `/boot/dtbs/rockchip/` após a extração do pacote `linux-aarch64` da imagem genérica.

Crie a infraestrutura do diretório de destino:

```bash
mkdir -p /mnt/arch_root/boot/extlinux

```

Edite o arquivo `/mnt/arch_root/boot/extlinux/extlinux.conf` preenchendo-o com o seguinte paradigma rigoroso:

```text
default arch
menu title Orange Pi 4B Arch Linux ARM Boot Menu
prompt 0
timeout 50

label arch
menu label Arch Linux ARM (Mainline Kernel)
linux /Image
initrd /initramfs-linux.img
fdt /dtbs/rockchip/rk3399-orangepi.dtb
append initrd=/initramfs-linux.img root=LABEL=ROOT rw rootwait console=tty1 console=ttyS2,1500000n8 earlycon=uart8250,mmio32,0xff1a0000

```

**Crítica Analítica da Passagem de Parâmetros:**

* **`root=LABEL=ROOT`**: A especificação do destino raiz baseada em LABEL (ou alternativamente, PARTUUID) desvincula o processo de montagem da ordem de inicialização imprevisível do kernel (`/dev/mmcblk0` vs `/dev/mmcblk1`). A numeração varia drasticamente quando cartões SD e o eMMC estão presentes simultaneamente.
* **`console=ttyS2,1500000n8`**: Este parâmetro é absoluto e não negociável. Diferente do padrão industrial obsoleto de 115200 bauds, o multiplexador serial (UART2) de depuração da Rockchip para o RK3399 opera num baud rate de extrema velocidade: 1.500.000 bps. Tentar depurar falhas cegas de inicialização na placa com cabos conversores USB-TTL legados, baseados em clones PL2303, resultará apenas na exibição de lixo digital na saída do console.
* **`earlycon=uart8250,mmio32,0xff1a0000`**: Parâmetro de diagnóstico de sobrevivência extrema. Ele instrui as funções de nível inferior do kernel Linux a injetar logs diretamente no registrador de memória de endereço de hardware físico do subsistema UART (`0xff1a0000`) de forma imediata, contornando a complexidade dos drivers TTY padrões. Sem ele, em caso de pânico (*kernel panic*) induzido pela controladora de eMMC ou falta de energia inicial, o sistema parará de emitir vídeo e sinais de rede, agindo como uma caixa-preta falha.

Em concomitância, o arquivo `/mnt/arch_root/etc/fstab` deve ser imediatamente adaptado para referenciar os pontos de montagem `LABEL=BOOT` para a raiz `/boot` e `LABEL=ROOT` para a raiz do sistema `/`. O uso arbitrário de arquivos fstab padrão gerará pânicos operacionais durante a inicialização do daemon systemd de montagem local.

#### Instanciação Física do Firmware (Flashing Low-Level)

Com as partições logicamente montadas, preenchidas de forma autônoma e os metadados de configuração salvaguardados, o sistema de arquivos deve ser sincronizado com o hardware de armazenamento. Contudo, ele ainda não é inicializável pelo processador.

Os artefatos compilados heroicamente na forja do U-Boot (`idbloader.img` e `u-boot.itb`) devem ser aplicados cruamente nos setores absolutos da mídia usando o utilitário binário `dd`. Devemos acoplar as diretrizes (*flags*) anti-cache e de não-truncamento (`conv=notrunc,fsync` ou `conv=notrunc,sync`) para garantir a persistência química imediata na fita magnética/memória flash do dispositivo e contornar os buffers de escrita preguiçosa do Linux.

```bash
# Gravação do código do Estágio Primário/Secundário (setor 64)
dd if=caminho/para/seu/compilado/idbloader.img of=/dev/sdX seek=64 conv=notrunc,fsync

# Gravação do código Bootloader/ATF (setor 16384)
dd if=caminho/para/seu/compilado/u-boot.itb of=/dev/sdX seek=16384 conv=notrunc,fsync

# Sincronização forçada dos buffers do sistema de arquivos host
sync
umount /mnt/arch_root/boot
umount /mnt/arch_root

```

Este procedimento consagra o cartão MicroSD como uma imagem plenamente autônoma, logicamente particionada, compatível com a arquitetura ARMv8 e de inicialização nativa no Orange Pi 4B.

#### Procedimentos de Primeiro Contato e Alimentação de Energia

Transfira o cartão devidamente formatado para o slot SD do Orange Pi 4B. Conecte o cabo de rede Ethernet na porta física da placa.

A exigência energética na primeira fase de inicialização do RK3399 (especialmente durante as fases simultâneas de descompressão multicore do LZ4 no initramfs operando nos núcleos Cortex-A72 sob frequência de turbo transiente) é brutal. Fontes de alimentação USB de prateleira baseadas em carregadores de telefone genéricos sofrerão queda de tensão súbita (*Voltage Drop / Brownout*). O pino da placa reconhecerá a falta de tensão e o circuito de proteção do PMU cortará as correntes para o SoC resultando num desligamento imediato ou congelamento perpétuo no log `Starting kernel...`. Utilize imperativamente uma fonte comutada AC-DC que entregue incondicionalmente 5 Volts sólidos e no mínimo 3 Amperes ou 4 Amperes via conexão Type-C.

Ao acessar remotamente o sistema provido (encontrando o IP distribuído por seu roteador de borda ou através de uma interface serial conectada aos pinos UART na placa) e autenticando com as credenciais padrão do Arch Linux ARM (`root` com a senha `root` ou `alarm` com senha `alarm`), o primeiro mandamento operacional é preencher a entropia do sistema de assinaturas de pacote pacman.

Sistemas embarcados sem dispositivos de entrada de usuário sofrem severamente para gerar bits aleatórios imprevisíveis logo no primeiro boot, resultando em pausas gigantescas na geração de chaves criptográficas (*keyrings*).

```bash
# Iniciação da base de chaves criptográficas do gestor de pacotes
pacman-key --init

# Preenchimento das assinaturas chaves mestre autorizadas pela distribuição
pacman-key --populate archlinuxarm

```
