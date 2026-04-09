**Prompt [No Arch Linux, como configurar o systemd-boot... | Experimente no Copilot Chat](https://m365.cloud.microsoft/chat/entity1-d870f6cd-4aa5-4d42-9626-ab690c041429/eyJpZCI6IlZYTmxjbFl4ZkdoMGRIQnpPaTh2YzNWaWMzUnlZWFJsTFdsdWRDNXZabVpwWTJVdVkyOXRMM3hQU1VRNk5URXpObVkyWWpJdE1HWTVOUzAwWkRVM0xXRmtZVEV0T0dVeE1UZzNaVE01TkRnNWZEZ3hObVkzT0RCa0xUTmhNak10TkRCaU1pMDRaamcyTFdNM1lUaG1PV0ZqTnpjek1ud3lNREkyTFRBMExUQTVWREUxT2pJeE9qRTFMamcwTkRBNU5UaGEiLCJzY2VuYXJpbyI6InNoYXJlTGlua1ZpYVJpY2hDb3B5IiwicHJvcGVydGllcyI6eyJwcm9tcHRTb3VyY2UiOiJ1c2VyIiwiY2xpY2tUaW1lc3RhbXAiOiIyMDI2LTA0LTA5VDE1OjIxOjE1Ljk0M1oifSwiY2hhdFR5cGUiOiJ3ZWIiLCJ2ZXJzaW9uIjoxLjF9)**
No Arch Linux, como configurar o systemd-boot para gerar arquivos de kernel UKI em /boot/EFI/Linux usando o dracut?
**Resposta**
A ideia é usar **UKI (Unified Kernel Image)** + **systemd‑boot** seguindo exatamente o layout que o boot loader espera, e deixar o **dracut** gerar o `.efi` final já com kernel + initramfs + cmdline + microcode.

No Arch isso funciona muito bem hoje. Abaixo está o **procedimento completo, do zero, focado em `/boot/EFI/Linux/*.efi`**, que é o caminho padrão do systemd‑boot para UKIs.

***

## 1. Pré‑requisitos

Pacotes necessários:

```bash
pacman -S systemd-bootctl dracut linux linux-firmware
```

(opcional, mas **fortemente recomendado**)

```bash
pacman -S intel-ucode   # Intel
# ou
pacman -S amd-ucode     # AMD
```

***

## 2. Layout correto do ESP

O ESP precisa estar montado assim:

```text
/boot
└── EFI
    ├── Linux
    │   └── arch-linux.efi
    └── systemd
```

Confirme o mount:

```bash
findmnt /boot
```

Deve ser algo como:

```text
/dev/nvme0n1p1 on /boot type vfat (rw,relatime)
```

***

## 3. Instalar e ativar o systemd‑boot

```bash
bootctl install
```

Verifique:

```bash
bootctl status
```

Você **não criará entradas `.conf` manuais**, porque o systemd‑boot detecta automaticamente UKIs em:

```text
/boot/EFI/Linux/*.efi
```

***

## 4. Criar o arquivo de cmdline do kernel (obrigatório para UKI)

Crie:

```bash
mkdir -p /etc/kernel
```

```bash
nano /etc/kernel/cmdline
```

Exemplo típico (ajuste UUIDs):

```text
root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rw quiet loglevel=3
```

Se usar LUKS ou LVM, coloque os parâmetros normais aqui (`rd.luks.name=…`, `rd.lvm.lv=…`, etc).

***

## 5. Configurar o dracut para gerar UKI

### 5.1 Diretório de configuração

```bash
mkdir -p /etc/dracut.conf.d
```

### 5.2 Configuração mínima para UKI

Crie o arquivo:

```bash
nano /etc/dracut.conf.d/uki.conf
```

Conteúdo recomendado:

```ini
# Habilita UKI
uefi=yes
hostonly=yes

# Caminho do kernel
kernel_image="/usr/lib/modules/%k/vmlinuz"

# Onde o UKI final será gerado
uefi_stub="/usr/lib/systemd/boot/efi/linuxx64.efi.stub"
uefi_splash_image=""
uefi_secureboot_key=""
uefi_secureboot_cert=""

# Diretório esperado pelo systemd-boot
uefi_dir="/boot/EFI/Linux"

# Nome do arquivo EFI final
uefi_filename="arch-linux-%k.efi"
```

> `%k` será substituído pela versão do kernel (ex: `6.7.9-arch1-1`).

***

## 6. Incluir microcode no UKI

Crie:

```bash
nano /etc/dracut.conf.d/microcode.conf
```

### Intel

```ini
early_microcode=yes
```

O dracut irá automaticamente puxar `/boot/intel-ucode.img`.

### AMD

Idem, desde que o pacote `amd-ucode` esteja instalado.

***

## 7. Teste manual da geração do UKI

Execute:

```bash
dracut --force --kver $(uname -r)
```

Verifique:

```bash
ls -lh /boot/EFI/Linux/
```

Saída esperada:

```text
arch-linux-6.7.9-arch1-1.efi
```

***

## 8. Testar no boot

Reinicie o sistema.

Na tela do **systemd‑boot**, você verá algo como:

```text
Arch Linux (Unified Kernel Image)
```

Sem arquivos `.conf` em `/boot/loader/entries`.

***

## 9. Automatizar com pacman hook (recomendado)

Para que o UKI seja recriado automaticamente a cada upgrade de kernel:

```bash
mkdir -p /etc/pacman.d/hooks
```

```bash
nano /etc/pacman.d/hooks/90-dracut-uki.hook
```

Conteúdo:

```ini
[Trigger]
Type = Package
Operation = Install
Operation = Upgrade
Operation = Remove
Target = linux

[Action]
Description = Generating UKI with dracut
When = PostTransaction
Exec = /usr/bin/dracut --force
```

***

## 10. Checklist rápido

✅ ESP montado em `/boot`  
✅ systemd‑boot instalado  
✅ `/etc/kernel/cmdline` criado  
✅ dracut com `uefi=yes`  
✅ UKI gerado em `/boot/EFI/Linux/*.efi`  
✅ Nenhum `.conf` manual necessário

***

## Observações importantes

*   O **systemd‑boot não lê GRUB configs**
*   UKI elimina initramfs solto e erros de sync
*   Secure Boot pode ser adicionado depois (assinando o `.efi`)
*   Rollback fica muito mais limpo (um `.efi` = um boot)

***

Se quiser, posso:

*   adaptar o exemplo para **LUKS + LVM**
*   mostrar **Secure Boot com sbctl**
*   comparar **mkinitcpio vs dracut para UKI**
*   gerar um **runbook em Markdown padrão SELIC**

É só dizer.

**Prompt [Como não tenho muito espaço no /boot atual, que é... | Experimente no Copilot Chat](https://m365.cloud.microsoft/chat/entity1-d870f6cd-4aa5-4d42-9626-ab690c041429/eyJpZCI6IlZYTmxjbFl4ZkdoMGRIQnpPaTh2YzNWaWMzUnlZWFJsTFdsdWRDNXZabVpwWTJVdVkyOXRMM3hQU1VRNk5URXpObVkyWWpJdE1HWTVOUzAwWkRVM0xXRmtZVEV0T0dVeE1UZzNaVE01TkRnNWZHWXpOalF5T1dOaUxUSXhNREF0TkRRNU5pMWhPR1ZsTFRjMU9EZzJaV1pqTVRnNFkzd3lNREkyTFRBMExUQTVWREU0T2pBek9qQTRMamszTnpFeE56RmEiLCJzY2VuYXJpbyI6InNoYXJlTGlua1ZpYVJpY2hDb3B5IiwicHJvcGVydGllcyI6eyJwcm9tcHRTb3VyY2UiOiJ1c2VyIiwiY2xpY2tUaW1lc3RhbXAiOiIyMDI2LTA0LTA5VDE4OjAzOjA5LjA3MVoifSwiY2hhdFR5cGUiOiJ3ZWIiLCJ2ZXJzaW9uIjoxLjF9)**
Como não tenho muito espaço no /boot atual, que é uma partição ESP,  pretendo criar uma partição XBOOTLDR para o /boot/EFI/Linux no mesmo disco. O que muda na configuração proposta? Posso montar /boot, criar um mount point /boot/EFI/Linux dentro dela, e montar a partição XBOOTLDR nesse mount point? Ou seria melhor montar a partição ESP atual em um mount point  /efi e montar a partição XBOOTLDR em /boot/EFI/Linux? Minha questão é que configuração atual usa mkinitcpio gerando arquivos em /boot. Até migrar para UKI gostaria de manter a configuração corrente funcionando.
**Resposta**
Ótima pergunta — e dá para fazer uma migração **sem quebrar o setup atual com mkinitcpio**, desde que você respeite uma regra bem importante do **systemd‑boot + Boot Loader Specification**:

> **Arquivos `*.conf` em `ESP:/loader/entries/` só podem referenciar arquivos (kernel/initrd/etc.) dentro do ESP; e `XBOOTLDR:/loader/entries/` só pode referenciar arquivos dentro do XBOOTLDR.** [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

Além disso, o **systemd‑boot** varre automaticamente:

*   entradas Type #1 (`/loader/entries/*.conf`) **no ESP e no XBOOTLDR** [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)
*   UKIs Type #2 (`/EFI/Linux/*.efi`) **no ESP e no XBOOTLDR** [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

E o **BLS/UAPI.1** define o XBOOTLDR como partição opcional (no mesmo disco do ESP), com GUID próprio, e recomenda que ambos usem filesystem legível pelo firmware (na prática, **VFAT**) — inclusive sugerindo que o mesmo tipo de FS seja usado em ambos. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

***

# O que muda na configuração ao introduzir XBOOTLDR?

### 1) Onde ficam as coisas (modelo “correto”)

O arranjo mais “limpo” e que simplifica a migração é:

*   **ESP** montado em **`/efi`** (pequeno, só boot manager + configs)
*   **XBOOTLDR** montado em **`/boot`** (grande, kernels/initramfs/entries/UKIs)

Isso é exatamente o cenário citado como recomendável quando o ESP é pequeno (ex.: dual boot com Windows). [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

### 2) Instalação do systemd‑boot com dois mountpoints

Quando você separa ESP e XBOOTLDR, você passa a usar:

```bash
bootctl --esp-path=/efi --boot-path=/boot install
```

Isso está documentado no fluxo de instalação usando XBOOTLDR. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

### 3) Paths do dracut para UKI

Se **XBOOTLDR estiver montado em `/boot`**, então o diretório “padrão” de UKI fica natural:

*   UKI em: **`/boot/EFI/Linux/*.efi`**  (na partição XBOOTLDR)

E o systemd‑boot vai auto-descobrir esses UKIs. [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

***

# Sobre suas duas ideias de montagem

## A) “Manter /boot no ESP e montar o XBOOTLDR em /boot/EFI/Linux”

**Tecnicamente dá**, mas **não é o melhor** para o seu objetivo.

**Prós:**

*   Você consegue começar a colocar UKIs em `/boot/EFI/Linux` (que estaria no XBOOTLDR) **sem mexer no mkinitcpio**, porque ele continuaria escrevendo initramfs em `/boot` (ESP).

**Contras (importantes):**

*   **Não resolve o problema de espaço agora**, porque o `mkinitcpio` e o pacote do kernel (no Arch) continuam usando `/boot` — ou seja, o ESP continua enchendo.
*   Você cria um layout “meio híbrido”, e fica fácil se confundir sobre “o que está em qual partição”.
*   Você não aproveita o benefício principal do XBOOTLDR: tirar kernel/initramfs do ESP. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

**Então**: útil como fase temporária *somente* se você quiser começar a testar UKI sem tocar no boot atual — mas não como estado final.

## B) “Montar ESP em /efi e montar XBOOTLDR em /boot (logo /boot/EFI/Linux fica no XBOOTLDR)”

✅ **Esse é o recomendado**.

Você ganha:

*   espaço em `/boot` (agora é XBOOTLDR maior)
*   mantém systemd‑boot no ESP (onde ele normalmente fica), e kernels/initramfs/UKIs no XBOOTLDR [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)
*   reduz risco de quebrar por confusão de paths/partições

***

# Como manter o mkinitcpio funcionando durante a migração (sem “big bang”)

A melhor estratégia é fazer em **duas fases**, com **rollback fácil**.

## Fase 1 — Criar XBOOTLDR e “espelhar” o boot atual lá (sem mudar mounts ainda)

Objetivo: você continua bootando pelo que já funciona no ESP, mas já prepara uma opção equivalente no XBOOTLDR.

1.  Crie a partição XBOOTLDR no mesmo disco do ESP e marque o tipo GPT correto:  
    GUID: `bc13c2ff-59e6-4262-a352-b275fd6f7172` [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

2.  Formate **VFAT** (é o mais compatível com UEFI; o BLS diz que para a maioria dos sistemas isso significa VFAT, e que ESP/XBOOTLDR devem ser legíveis pelo firmware). [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

> O ArchWiki também menciona que pode ser qualquer FS que o seu UEFI consiga ler (mas VFAT é o “safe bet”). [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

3.  Monte temporariamente em algum lugar, por exemplo:

```bash
mkdir -p /mnt/xbootldr
mount /dev/<sua_particao_xbootldr> /mnt/xbootldr
```

4.  Copie kernel e initramfs atuais para o XBOOTLDR:

```bash
cp -av /boot/vmlinuz-linux /mnt/xbootldr/
cp -av /boot/initramfs-linux*.img /mnt/xbootldr/
# se você usa microcode externo:
cp -av /boot/intel-ucode.img /mnt/xbootldr/ 2>/dev/null || true
cp -av /boot/amd-ucode.img   /mnt/xbootldr/ 2>/dev/null || true
```

5.  Copie **as entradas do loader** para o XBOOTLDR:

```bash
mkdir -p /mnt/xbootldr/loader/entries
cp -av /boot/loader/entries/*.conf /mnt/xbootldr/loader/entries/
```

6.  **Importante:** garanta que essas entries referenciem paths que existam no XBOOTLDR (normalmente já é `/vmlinuz-linux` e `/initramfs-linux.img`, o que fica OK após a cópia).  
    A regra é: entry em `boot:/loader/entries/*.conf` só usa arquivos em `boot:/`. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

7.  Reinicie e, no menu do systemd‑boot, teste a entry que veio do XBOOTLDR.

*   O systemd‑boot lista entradas de **ESP e XBOOTLDR**. [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

✅ Se bootou, você validou que o XBOOTLDR está OK **antes** de mexer em `/etc/fstab`.

***

## Fase 2 — Virar o “/boot de verdade” para o XBOOTLDR (e mover ESP para /efi)

Agora você resolve o problema de espaço e mantém mkinitcpio funcionando.

1.  Crie mountpoint do ESP:

```bash
mkdir -p /efi
```

2.  Ajuste `/etc/fstab` para:

*   ESP → `/efi`
*   XBOOTLDR → `/boot`

Exemplo (ajuste UUIDs):

```fstab
UUID=<UUID_ESP>      /efi   vfat  umask=0077,noatime  0  2
UUID=<UUID_XBOOTLDR> /boot  vfat  umask=0077,noatime  0  2
```

3.  Reinstale/atualize o systemd‑boot apontando os caminhos corretos:

```bash
bootctl --esp-path=/efi --boot-path=/boot install
```

(Esse é o procedimento “com XBOOTLDR”.) [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

4.  Garanta que o `loader.conf` esteja no ESP (agora em `/efi/loader/loader.conf`), pois o loader conf fica no ESP. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

✅ A partir daqui, o **mkinitcpio continua funcionando** como sempre — só que agora `/boot` é maior (XBOOTLDR), então:

*   `initramfs-linux.img` / `fallback` vão para o XBOOTLDR
*   o kernel (`vmlinuz-linux`) também fica no XBOOTLDR
*   e suas entries no XBOOTLDR continuam coerentes (entry e arquivos na mesma partição), respeitando a regra do systemd‑boot. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

***

# E a migração para UKI com dracut, depois?

Quando você estiver pronto para UKI:

*   gere UKIs em: **`/boot/EFI/Linux/*.efi`** (isso é XBOOTLDR montado em `/boot`)
*   o systemd‑boot vai autodetectar UKIs em `/EFI/Linux/` no ESP e no XBOOTLDR — **não precisa de `.conf`** para UKI. [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

Ou seja: **na configuração proposta inicialmente**, praticamente **não muda nada** do ponto de vista do dracut **se** `/boot` já for XBOOTLDR — o `uefi_dir="/boot/EFI/Linux"` continua válido, só que agora ele aponta para a partição grande.

***

# Resumo (decisão prática)

✅ **Melhor opção para seu caso (pouco espaço e quer manter mkinitcpio até migrar):**  
**ESP em `/efi` + XBOOTLDR em `/boot`**, e faça a migração em duas fases como acima. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

⚠️ **Montar XBOOTLDR “dentro” de `/boot/EFI/Linux` com `/boot` ainda no ESP:**  
funciona para **testar UKI**, mas **não resolve** o problema de espaço e aumenta chance de confusão.

***

# Perguntas rápidas (pra eu te dar um passo-a-passo 100% aderente ao seu cenário)

1.  Você está usando **systemd‑boot já hoje** ou ainda GRUB?
2.  Seu `/boot` atual (ESP) tem também o diretório `loader/entries` com `.conf`?
3.  Você usa **LUKS/dm-crypt**, LVM, btrfs subvol, ou root “simples”? (isso impacta cmdline e initramfs/UKI)

Se você me disser isso e o output de:

```bash
lsblk -f
bootctl status
```

eu monto um runbook bem objetivo com **comandos exatos** (incluindo fallback/rollback), no estilo checklist.

**Prompt [Esqueci de comentar que a configuração corrente... | Experimente no Copilot Chat](https://m365.cloud.microsoft/chat/entity1-d870f6cd-4aa5-4d42-9626-ab690c041429/eyJpZCI6IlZYTmxjbFl4ZkdoMGRIQnpPaTh2YzNWaWMzUnlZWFJsTFdsdWRDNXZabVpwWTJVdVkyOXRMM3hQU1VRNk5URXpObVkyWWpJdE1HWTVOUzAwWkRVM0xXRmtZVEV0T0dVeE1UZzNaVE01TkRnNWZEZGxPVEZqTjJJMExXUmlaVGt0TkRnd015MWhZemRpTFRWaVl6VmhPR000WlRFek5Id3lNREkyTFRBMExUQTVWREU0T2pBM09qUTRMak16TlRReE9UVmEiLCJzY2VuYXJpbyI6InNoYXJlTGlua1ZpYVJpY2hDb3B5IiwicHJvcGVydGllcyI6eyJwcm9tcHRTb3VyY2UiOiJ1c2VyIiwiY2xpY2tUaW1lc3RhbXAiOiIyMDI2LTA0LTA5VDE4OjA3OjQ4LjQ1MFoifSwiY2hhdFR5cGUiOiJ3ZWIiLCJ2ZXJzaW9uIjoxLjF9)**
```
Esqueci de comentar que a configuração corrente usa GRUB.
Não uso e por enquanto não vou gerar nenhuma configuração em /loader/entries/.
Não uso LUKS/dm-crypt.
O /boot (ESP) é uma partição simples.
Os demais filesystem estão em volumes LVM usando xfs.
Não uso btrfs.

Minha dúvida agora é, como fazer o grub usar os arquivos copiados do /boot atual para o novo /boot em partição XBOOTLDR? 

Para evitar essa questão sem precisar que o GRUB acesse a partição XBOOTLDR imaginei os passos abaixo para migrar:
1) Deixar o grub intocado e já criar, ainda que de forma manual, arquivos em "partição XBOOLTLDR"/EFI/Linux usando dracut. (Como fazer isso?)
2) Instalar o systemd-boot (bootctl install).
3) Bootar para teste. A UEFI vai tentar bootar usando os arquivos UKI da partição XBOOTLDR, já que a instalação do systemd-boot o torna o boot loader prioncipal sem remover o grub que continua instalado e disponível na UEFI.
4) Em caso de problema posso entrar no menu de boot usando F12 para corrigir.
5) Depois de testar tudo e validar o boot usando os arquivos UKI (/EFI/Linux/*.efi), desintalo mkinitcpio, grub e removo os arquivos desnecessários da partição ESP.
6) Configuro a geração automárica de imagens UKI em /boot/EFI/Linux/ quando houver atualização de kernel. (Como fazer isso?)

Essa estratégia funciona? Deixei escapar algum detalhe?
```

**Resposta**
Sua estratégia **é válida** e, do jeito que você descreveu, ela é justamente o tipo de migração “segura” (testa sem quebrar o boot atual). Só tem alguns **detalhes críticos** para não ser pego de surpresa — principalmente sobre **onde o systemd‑boot vai procurar o UKI**, sobre **como o dracut embute a cmdline**, e sobre um **efeito colateral do `bootctl install`** no ESP.

Vou dividir em:

1.  **GRUB usando o novo XBOOTLDR (se você quisesse)**
2.  **Estratégia que você propôs (GRUB intocado + UKI no XBOOTLDR + systemd‑boot)** — validação e “pegadinhas”
3.  **Como gerar UKI manualmente com dracut no XBOOTLDR (sem mexer no GRUB)**
4.  **Como automatizar a geração do UKI quando o kernel atualizar** (com pacman hook)

***

## 1) “Como fazer o GRUB usar os arquivos copiados do /boot atual para o novo /boot no XBOOTLDR?”

Se o objetivo é **manter GRUB** carregando kernel+initramfs, a forma “normal” de fazê-lo usar o novo armazenamento é: **fazer o `/boot` real virar o XBOOTLDR** (maior) e montar o ESP em outro lugar (ex.: `/efi`). Isso é exatamente o cenário do XBOOTLDR: separar artefatos grandes (kernel/initramfs) do ESP pequeno. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

Mas como você **explicitamente quer evitar** que o GRUB tenha que acessar o XBOOTLDR (e não quer usar `/loader/entries`), então a melhor resposta prática é:

> **Você não precisa (nem vale a pena) fazer o GRUB “usar” o XBOOTLDR nessa fase.**  
> Use o GRUB como fallback e valide o UKI com systemd‑boot.

Isso deixa o GRUB “intocado”, do jeito que você quer.

***

## 2) A estratégia que você propôs funciona? (Sim) — mas observe estes detalhes

A lógica geral está correta:

*   `systemd-boot` **pode coexistir** com GRUB: `bootctl install` cria uma entrada UEFI (“Linux Boot Manager”) e normalmente a coloca primeiro na ordem de boot, **sem remover** outras entradas. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)
*   O `systemd-boot` **auto-detecta UKIs** em `/EFI/Linux/*.efi` **no ESP e também no XBOOTLDR**, sem precisar de arquivos em `/loader/entries`. [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Unified_kernel_image)
*   E ele **só consegue iniciar binários EFI** que estejam no **ESP** ou no **XBOOTLDR no mesmo disco**. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

### Pegadinhas / detalhes que você não pode deixar escapar

#### (A) Firmware “Fast Boot” pode ignorar o XBOOTLDR

Alguns firmwares, com “fast boot” ativado, **podem pular a leitura de partições além do ESP**, e aí o systemd‑boot pode não encontrar as entradas/UKIs no XBOOTLDR. Solução: desabilitar o fast boot no setup. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

#### (B) `bootctl install` pode sobrescrever o “fallback loader” do ESP

O `bootctl install` copia o systemd‑boot para o ESP e **sobrescreve** `ESP:/EFI/BOOT/BOOTX64.EFI` (ou IA32 equivalente). Isso pode impactar cenários onde algo dependia desse caminho. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

> Não costuma “quebrar o Windows” (porque o Windows usa `EFI/Microsoft/Boot/Bootmgfw.efi`), mas é um efeito colateral real — só esteja ciente.

#### (C) UKI no XBOOTLDR: precisa ser `.efi` e estar em `/EFI/Linux/`

O systemd‑boot só trata como UKI auto-detectável o que estiver em `/EFI/Linux/` com extensão `.efi`. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html)

#### (D) Cmdline embutida: dracut usa `kernel_cmdline=...` (não confie que `/etc/kernel/cmdline` será usado)

O ArchWiki do dracut explica que a cmdline pode ser definida em `/etc/dracut.conf.d/*.conf` via `kernel_cmdline=...` e que o dracut a “sourcia” para dentro do initramfs.   
E, para UKI, você quer **ter certeza** que a cmdline embutida contém pelo menos o `root=...` e (no seu caso) parâmetros úteis para LVM se necessário. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

> (Detalhe relacionado) Quando Secure Boot está ativo, UKIs com `.cmdline` embutida **ignoram** opções passadas pelo bootloader; sem Secure Boot, as opções passadas podem sobrescrever. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Unified_kernel_image)

***

## 3) Como fazer o passo (1): gerar UKI “manual” no XBOOTLDR com dracut (sem mexer no GRUB)

A ideia aqui é: **montar o XBOOTLDR em um ponto qualquer (ex.: `/xbootldr`)**, gerar UKIs em `/xbootldr/EFI/Linux/`, instalar systemd‑boot no ESP (onde já está o GRUB), e testar.

### 3.1 Criar/formatar/montar o XBOOTLDR (uma vez)

*   O XBOOTLDR, em GPT, tem GUID **`bc13c2ff-59e6-4262-a352-b275fd6f7172`** e deve ficar **no mesmo disco do ESP**. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)
*   Em geral, use **VFAT** por compatibilidade com firmware. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

Exemplo de montagem em runtime (sem tocar no `/boot` do GRUB):

```bash
mkdir -p /xbootldr
# adicione no /etc/fstab (exemplo):
# UUID=xxxx-xxxx  /xbootldr  vfat  noatime,umask=0077  0  2
mount /xbootldr
mkdir -p /xbootldr/EFI/Linux
```

> Esse “/xbootldr” é só para o Linux em runtime escrever arquivos; no boot, o systemd‑boot vai localizar a partição pelo tipo GUID e pelo disco. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html)

### 3.2 Instalar dracut e configurar para UKI

Instale dracut:

```bash
pacman -S dracut
```

O dracut pode produzir UKI com `--uefi` ou com `uefi="yes"` na configuração. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

Crie uma config para **cmdline** (importante!):

```bash
mkdir -p /etc/dracut.conf.d
nano /etc/dracut.conf.d/cmdline.conf
```

Exemplo (ajuste para seu VG/LV e root real em XFS/LVM):

```ini
kernel_cmdline="root=/dev/mapper/<VG>-<LV> rw rootfstype=xfs"
```

O ArchWiki do dracut documenta exatamente esse mecanismo (`kernel_cmdline=` em `.conf`). [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

Agora uma config para gerar UKI (mínima) — o essencial é “ativar uefi” e apontar o stub:

```bash
nano /etc/dracut.conf.d/uki.conf
```

Conteúdo sugestivo:

```ini
uefi="yes"
hostonly="yes"
uefi_stub="/usr/lib/systemd/boot/efi/linuxx64.efi.stub"
```

> O ArchWiki confirma que o dracut gera UKI com `--uefi` ou `uefi="yes"`. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

### 3.3 Gerar UKI manualmente no XBOOTLDR

O `dracut` permite escolher o arquivo de saída e a versão do kernel. Ele gera a imagem e sobrescreve com `--force`. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut), [\[man7.org\]](https://www.man7.org/linux/man-pages/man8/dracut.8.html)

Para o kernel em execução:

```bash
kver="$(uname -r)"
dracut --force --uefi "/xbootldr/EFI/Linux/arch-linux-${kver}.efi" "${kver}"
```

Depois confira:

```bash
ls -lh /xbootldr/EFI/Linux/
```

**Dica de inspeção:** `lsinitrd` funciona para ver o que foi embutido. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

***

## 4) Passos (2)–(5): instalar systemd‑boot, testar, manter rollback com GRUB

### 4.1 Instalar systemd‑boot no ESP (mantendo GRUB)

Rode:

```bash
bootctl install
```

Isso instala o systemd‑boot no ESP e cria a entrada UEFI; ele tenta localizar o ESP em `/efi`, `/boot`, `/boot/efi` e coloca a nova entrada no boot order.   
**Cuidado**: sobrescreve `ESP:/EFI/BOOT/BOOTX64.EFI`. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

### 4.2 Testar boot

No reboot, selecione “Linux Boot Manager” (systemd‑boot).  
Ele deve listar automaticamente o UKI do XBOOTLDR (`/EFI/Linux/*.efi`) sem nenhum `.conf`. [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Unified_kernel_image)

Se falhar, seu rollback é exatamente o que você disse: F12/Boot Menu e escolher GRUB (entrada UEFI antiga continua lá). [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

### 4.3 Só depois de validar…

Aí sim faz sentido remover GRUB/mkinitcpio e limpar o ESP — e ainda assim eu manteria **pelo menos um caminho de fallback** (ex.: manter o systemd‑boot atualizado no ESP). A remoção do mkinitcpio é recomendada **após** validar o dracut (o ArchWiki sugere isso explicitamente). [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

***

## 5) Passo (6): geração automática de UKI em `/boot/EFI/Linux/` após update de kernel

Você tem duas abordagens:

### Abordagem A (mais simples e controlável): *pacman hook* customizado para UKI

Crie um script que:

1.  garante que o XBOOTLDR esteja montado (por enquanto em `/xbootldr`; depois você pode mudar para `/boot`)
2.  gera UKIs para kernels instalados (ou pelo menos para o kernel recém-instalado)

**Exemplo de script** `/usr/local/sbin/uki-dracut-build-all`:

```bash
#!/usr/bin/env bash
set -euo pipefail

MNT="/xbootldr"                 # depois você troca para /boot quando fizer o cutover
OUTDIR="${MNT}/EFI/Linux"

mkdir -p "${OUTDIR}"

# Gera para todos os kernels presentes em /usr/lib/modules
for d in /usr/lib/modules/*; do
  kver="$(basename "$d")"
  dracut --force --uefi "${OUTDIR}/arch-linux-${kver}.efi" "${kver}"
done
```

> O dracut suporta `--force` e a seleção de versão via argumento de kernel version; e pode gerar UKI com `--uefi`. [\[man7.org\]](https://www.man7.org/linux/man-pages/man8/dracut.8.html), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

Permissões:

```bash
chmod +x /usr/local/sbin/uki-dracut-build-all
```

**Hook** `/etc/pacman.d/hooks/95-uki-dracut.hook`:

```ini
[Trigger]
Type = Package
Operation = Install
Operation = Upgrade
Target = linux
Target = linux-lts
Target = systemd
Target = intel-ucode
Target = amd-ucode

[Action]
Description = Rebuilding UKIs (dracut) into XBOOTLDR
When = PostTransaction
Exec = /usr/local/sbin/uki-dracut-build-all
```

Por que incluir `systemd`/`ucode`? Porque UKI costuma embutir stub/microcode; mudanças nesses pacotes devem regenerar os artefatos. (O systemd‑boot/UKI são sensíveis a atualizações do stub e microcode). [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Unified_kernel_image), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

### Abordagem B: deixar os hooks padrão gerarem initramfs e depois migrar (menos ideal)

O pacote do dracut no Arch inclui hooks para gerar initramfs em upgrade de kernel.   
Mas isso normalmente gera **initramfs solto** (não UKI), e você quer UKI. Então você acabaria com artefatos redundantes. [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

Quando você já estiver 100% em UKI, vale lembrar:

*   o ArchWiki do dracut menciona formas de evitar imagens não usadas e até como “parar” hooks que geram initramfs redundantes, e também como desativar hooks do mkinitcpio (via symlink para `/dev/null`). [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)

***

## 6) Conclusão: sua estratégia “GRUB como fallback + UKI no XBOOTLDR + systemd‑boot para testes” é boa

✅ **Funciona**, desde que você cuide de:

*   XBOOTLDR no mesmo disco do ESP (requisito) [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)
*   firmware não ignorar XBOOTLDR (desativar fast boot se necessário) [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)
*   UKIs em `/EFI/Linux/*.efi` com extensão `.efi` [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot), [\[man7.org\]](https://www.man7.org/linux//man-pages/man7/sd-boot.7.html)
*   cmdline embutida corretamente via `kernel_cmdline=` no dracut [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Dracut)
*   entender que `bootctl install` sobrescreve o fallback loader `EFI/BOOT/BOOTX64.EFI` [\[wiki.archlinux.org\]](https://wiki.archlinux.org/title/Systemd-boot)

***

## Próximo passo (pra eu te dar um “procedimento fechado” sem chute)

Me manda o output de:

```bash
lsblk -f
findmnt -no SOURCE /
cat /etc/fstab
```

Com isso eu te devolvo:

*   a `kernel_cmdline=` **exata** para o seu root em LVM+XFS
*   os arquivos `/etc/dracut.conf.d/*.conf` prontos
*   e o hook de pacman ajustado (linux vs linux-lts, ucode que você usa, etc.)

Se você usa Intel ou AMD (microcode), me diga também qual.

**Prompt [Para o kernel_cmdline posso usar o conteúdo de... | Experimente no Copilot Chat](https://m365.cloud.microsoft/chat/entity1-d870f6cd-4aa5-4d42-9626-ab690c041429/eyJpZCI6IlZYTmxjbFl4ZkdoMGRIQnpPaTh2YzNWaWMzUnlZWFJsTFdsdWRDNXZabVpwWTJVdVkyOXRMM3hQU1VRNk5URXpObVkyWWpJdE1HWTVOUzAwWkRVM0xXRmtZVEV0T0dVeE1UZzNaVE01TkRnNWZHRTFOamhrTXpkbUxUa3daak10TkRBd1pDMDVNVFU1TFRka05tSmlZVE15WlRFM1pYd3lNREkyTFRBMExUQTVWREU0T2pRd09qVTNMakk1TURjNU5ERmEiLCJzY2VuYXJpbyI6InNoYXJlTGlua1ZpYVJpY2hDb3B5IiwicHJvcGVydGllcyI6eyJwcm9tcHRTb3VyY2UiOiJ1c2VyIiwiY2xpY2tUaW1lc3RhbXAiOiIyMDI2LTA0LTA5VDE4OjQwOjU3LjM2M1oifSwiY2hhdFR5cGUiOiJ3ZWIiLCJ2ZXJzaW9uIjoxLjF9)**

```
Para o kernel_cmdline posso usar o conteúdo de /proc/cmdline, certo?
Ex:
Se /proc/cmdline for...

	BOOT_IMAGE=(hd0,msdos1)/vmlinuz-4.18.0-553.85.1.el8_10.x86_64 root=/dev/mapper/vg_root-root ro nofb splash=quiet crashkernel=auto rd.lvm.lv=vg_root/root rd.lvm.lv=vg_root/swap rhgb quiet audit=1 audit_backlog_limit=8192

...uso o trecho...

	root=/dev/mapper/vg_root-root ro nofb splash=quiet crashkernel=auto rd.lvm.lv=vg_root/root rd.lvm.lv=vg_root/swap rhgb quiet audit=1 audit_backlog_limit=8192

...no kernel_cmdline, certo?

Isso é só um exemplo que eu peguei de uma máquina Red Hat arbitrária que eu tenho acesso.
O cmdline da minha máquina Arch eu só vou saber quando chegar em casa.

Uso os kernels linux e linux-zen.
Depois de criar a partição XBOOTLDR com mais espaço devo instalar o linux-lts e talvez eu habilite o fallback.
Minha máquina é Intel, um Dell XPS 8960.
O que é nano? (Contém ironia :)! Uso vim ou neovim.)
```

**Resposta**
Sim — **usar o conteúdo de `/proc/cmdline` como base para o `kernel_cmdline=` é exatamente o caminho mais seguro** (porque reflete o que está funcionando hoje). A única cautela é **remover/ajustar alguns campos** que fazem sentido apenas no contexto do GRUB (ou de outra distro) e **garantir que o parâmetro `root=` continue correto**. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[systemd.io\]](https://systemd.io/BOOT/)

Abaixo deixo um guia bem prático (e “vim-friendly” 😄).

***

## 1) Posso usar o `/proc/cmdline`? Sim, mas com “higienização”

O que você sugeriu está correto: **você não deve colocar `BOOT_IMAGE=...`** no `kernel_cmdline`, porque isso é “metadado” do bootloader (GRUB) e não precisa (nem ajuda) quando você vai bootar via UKI/systemd-stub. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

### Exemplo que você deu (RHEL)

`/proc/cmdline`:

    BOOT_IMAGE=(hd0,msdos1)/vmlinuz-4.18.0-553.85.1.el8_10.x86_64 root=/dev/mapper/vg_root-root ro nofb splash=quiet crashkernel=auto rd.lvm.lv=vg_root/root rd.lvm.lv=vg_root/swap rhgb quiet audit=1 audit_backlog_limit=8192

Para `kernel_cmdline`, você pegaria **basicamente tudo menos o `BOOT_IMAGE=...`**, então seu “trecho” está no caminho certo:

    root=/dev/mapper/vg_root-root ro nofb splash=quiet crashkernel=auto rd.lvm.lv=vg_root/root rd.lvm.lv=vg_root/swap rhgb quiet audit=1 audit_backlog_limit=8192

✅ **Isso funciona como base.** [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

***

## 2) O que eu recomendo ajustar (especialmente ao sair do GRUB/RHEL → UKI/Arch)

Pense em três categorias:

### (A) **Remover quase sempre**

*   `BOOT_IMAGE=...` (sempre remover) [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)
*   Qualquer `initrd=...` (não apareceu no seu exemplo, mas às vezes aparece) — em UKI o initramfs está embutido; `initrd=` passado na cmdline não é desejável. [\[systemd.io\]](https://systemd.io/BOOT/)

### (B) **Manter (importante para boot)**

*   `root=...` ✅ (essencial)
*   Para **LVM**: `rd.lvm.lv=VG/LV` costuma ser útil para garantir descoberta precoce do LV no initramfs (especialmente em setups que dependem do initramfs “entender” LVM). Você usa LVM no root, então **tende a ser uma boa manter**. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)
*   `rootfstype=xfs` (opcional, mas OK; às vezes acelera/evita autodetecção errada)

### (C) **Manter se você realmente quer o comportamento**

*   `ro` vs `rw`: muitas distros bootam `ro` e depois o systemd remonta `rw`. Tanto faz na maioria dos casos; se você quer “igual ao atual”, mantenha `ro`.
*   `quiet`, `loglevel=`, `audit=`, `audit_backlog_limit=`: comportamento/telemetria (mantém se você usa)
*   `crashkernel=auto`: só se você usa kdump (no Arch normalmente não, mas pode)
*   `rhgb` / `splash=quiet`: isso é bem “Red Hat-ish”. Em Arch geralmente é irrelevante, mas não costuma quebrar. Eu **não colocaria** por padrão num Arch “limpo”, mas se estava no `/proc/cmdline` do *seu* Arch e você gosta do comportamento, ok.

> **Dica prática:** para o primeiro teste de UKI, use uma cmdline **mínima e previsível**, e só depois “recoloque perfumarias”.

***

## 3) Uma cmdline “mínima boa” para seu caso (LVM + XFS, sem LUKS)

Algo nessa linha costuma ser suficiente:

```text
root=/dev/mapper/vg_root-root ro rootfstype=xfs rd.lvm.lv=vg_root/root
```

Se você tem swap em LV e usa resume/hibernação, aí sim entram coisas como `resume=` etc.

E você pode acrescentar seus parâmetros de logging/audit:

```text
root=/dev/mapper/vg_root-root ro rootfstype=xfs rd.lvm.lv=vg_root/root rd.lvm.lv=vg_root/swap audit=1 audit_backlog_limit=8192 quiet
```

O ponto é: **sempre garanta `root=` correto** e **mantenha `rd.lvm.lv=` ao menos para o root** quando o root é LV. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

***

## 4) Onde configurar isso no dracut (do jeito “certo”)

No Arch, uma forma direta é criar um arquivo, por exemplo:

`/etc/dracut.conf.d/cmdline.conf`

com:

```ini
kernel_cmdline="root=/dev/mapper/vg_root-root ro rootfstype=xfs rd.lvm.lv=vg_root/root rd.lvm.lv=vg_root/swap quiet audit=1 audit_backlog_limit=8192"
```

O ArchWiki do dracut documenta exatamente esse mecanismo (`kernel_cmdline=` em `.conf`). [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

**Vim edition™**:

```bash
sudoedit /etc/dracut.conf.d/cmdline.conf
```

***

## 5) Observação importantíssima: UKI + systemd-boot e “override” de cmdline

Quando você embute a `.cmdline` dentro do UKI:

*   **Sem Secure Boot**: o systemd-boot pode passar opções e **elas podem sobrescrever** a `.cmdline` embutida. [\[wiki.gentoo.org\]](https://wiki.gentoo.org/wiki/Unified_Kernel_Image), [\[systemd.io\]](https://systemd.io/BOOT/)
*   **Com Secure Boot ativo**: UKIs com `.cmdline` embutida **ignoram** opções passadas pelo bootloader (por segurança). [\[wiki.gentoo.org\]](https://wiki.gentoo.org/wiki/Unified_Kernel_Image), [\[systemd.io\]](https://systemd.io/BOOT/)

Como você quer **não depender de `/loader/entries`** e pretende bootar “auto-discovery”, embutir uma cmdline correta no UKI é o que torna o boot “self-contained”. [\[systemd.io\]](https://systemd.io/BOOT/), [\[kernel.org\]](https://www.kernel.org/pub/linux/utils/boot/dracut/dracut.html)

***

## 6) Seu cenário de kernels (linux, linux-zen, talvez linux-lts) — o que muda na prática

Nada “conceitual”: você vai acabar gerando **um UKI por versão de kernel instalada**.

O systemd‑boot **auto-detecta UKIs** em `/EFI/Linux/*.efi` tanto no **ESP** quanto no **XBOOTLDR** (mesmo disco). [\[kernel.org\]](https://www.kernel.org/pub/linux/utils/boot/dracut/dracut.html), [\[wiki.gentoo.org\]](https://wiki.gentoo.org/wiki/Unified_Kernel_Image), [\[systemd.io\]](https://systemd.io/BOOT/)

Então uma estratégia simples é nomear assim:

*   `/EFI/Linux/arch-linux-<kver>.efi`
*   `/EFI/Linux/arch-zen-<kver>.efi`
*   `/EFI/Linux/arch-lts-<kver>.efi`

Não existe obrigação do nome, mas **claridade ajuda**.

> **Sobre “fallback”:** com dracut “normal”, o fallback (no sentido mkinitcpio) não é tão necessário em muitos setups, mas você pode manter um UKI “rescue”/“fallback” se quiser. (Podemos montar isso depois que você tiver o `/proc/cmdline` real da sua máquina.)

***

## 7) (Extra) Como extrair seu `/proc/cmdline` e “limpar” automaticamente

Quando você estiver em casa, você pode pegar e remover o BOOT\_IMAGE com um one-liner:

```bash
cmdline="$(</proc/cmdline)"
clean="$(printf '%s\n' "$cmdline" | sed -E 's/(^| )BOOT_IMAGE=[^ ]+//g; s/^ +| +$//g')"
printf '%s\n' "$clean"
```

Aí você revisa e cola em `kernel_cmdline="..."`.

***

## 8) Humor obrigatório: “O que é nano?” 😄

Nano é aquele editor que aparece quando você erra algum alias e o universo te pune. Brincadeira.  
Vamos de `vim`/`nvim` sem culpa. 😉

***

## Próximo passo (pra eu te fechar um procedimento 100% exato)

Quando você estiver com sua máquina Arch, me mande:

```bash
cat /proc/cmdline
lsblk -f
```

Com isso eu te devolvo:

*   `kernel_cmdline=` **exato** (incluindo `root=` correto do seu LV)
*   exemplo de comando para gerar **UKI para linux + linux-zen + linux-lts**
*   e o **hook do pacman** para rebuild automático de UKI (incluindo `intel-ucode` e `systemd` como triggers), alinhado com sua estratégia de coexistência GRUB → systemd‑boot. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[wiki.gentoo.org\]](https://wiki.gentoo.org/wiki/Unified_Kernel_Image), [\[systemd.io\]](https://systemd.io/BOOT/)

**Prompt [Entendo que, com a configuração proposta, o... | Experimente no Copilot Chat](https://m365.cloud.microsoft/chat/entity1-d870f6cd-4aa5-4d42-9626-ab690c041429/eyJpZCI6IlZYTmxjbFl4ZkdoMGRIQnpPaTh2YzNWaWMzUnlZWFJsTFdsdWRDNXZabVpwWTJVdVkyOXRMM3hQU1VRNk5URXpObVkyWWpJdE1HWTVOUzAwWkRVM0xXRmtZVEV0T0dVeE1UZzNaVE01TkRnNWZHUXdZVE0xWVdJMkxUSmlNV1V0TkdSak15MWlNMkUzTFdabVlqbGlPREEwWlRjeVpYd3lNREkyTFRBMExUQTVWREU0T2pVeE9qSXdMalEwTURnNU16WmEiLCJzY2VuYXJpbyI6InNoYXJlTGlua1ZpYVJpY2hDb3B5IiwicHJvcGVydGllcyI6eyJwcm9tcHRTb3VyY2UiOiJ1c2VyIiwiY2xpY2tUaW1lc3RhbXAiOiIyMDI2LTA0LTA5VDE4OjUxOjIwLjUzOVoifSwiY2hhdFR5cGUiOiJ3ZWIiLCJ2ZXJzaW9uIjoxLjF9)**

```
Entendo que, com a configuração proposta, o comando "dracut --force --kver $(uname -r)" vai gerar a imagem .efi  para o kernel corrente, no local correto, e incluindo o kernel, o initrd e a cmdline. Ele inclui automaticamente o arquivo microcode do intel-ucode? Ou é necessário informar isso na configuração/linha de comando?
```

**Resposta**
Você está **quase** certo — só um ajuste importante:

*   `dracut --force --kver $(uname -r)` **por si só** tende a gerar **apenas um initramfs “normal”** (no local padrão da distro/pacote), **não necessariamente um UKI `.efi`**, a menos que você esteja usando `--uefi` **ou** tenha `uefi="yes"` na configuração e esteja gerando para um **arquivo `.efi`**. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[edu4rdshl.dev\]](https://edu4rdshl.dev/posts/uki-secure-boot-on-archlinux-systemd-boot-walkthrough/)
*   Para produzir um **UKI de verdade**, o ArchWiki do dracut é bem direto: *“dracut can produce unified kernel images with the `--uefi` command line option or with the `uefi="yes"` configuration option.”* [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

Dito isso, vamos à sua pergunta central:

## Microcode (intel-ucode): dracut embute automaticamente?

✅ **Sim — se o microcode estiver instalado, o dracut embute automaticamente nas imagens que ele gera.**  
O próprio ArchWiki do dracut diz: *“The files created through these commands embed any installed Microcode images.”* [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

Ou seja:

*   Se você tiver `intel-ucode` instalado, o dracut **não precisa** que você aponte explicitamente `intel-ucode.img` na linha de comando para ele ser incluído. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

***

## Então qual é o “comando correto” para gerar o `.efi` no local certo?

Para UKI, eu recomendo ser explícito (menos ambiguidade):

```bash
sudo mkdir -p /boot/EFI/Linux
sudo dracut --force --uefi "/boot/EFI/Linux/arch-linux-$(uname -r).efi" "$(uname -r)"
```

Isso garante:

*   saída `.efi` em `/boot/EFI/Linux/…` (layout que o systemd-boot autodetecta) [\[wiki.gentoo.org\]](https://wiki.gentoo.org/wiki/Unified_Kernel_Image), [\[systemd.io\]](https://systemd.io/BOOT/)
*   que você está realmente pedindo **modo UKI/UEFI** (`--uefi`) [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

> Observação: o systemd‑boot autodetecta UKIs em `/EFI/Linux/*.efi` (no ESP e também no XBOOTLDR) sem precisar de `/loader/entries`. [\[wiki.gentoo.org\]](https://wiki.gentoo.org/wiki/Unified_Kernel_Image), [\[kernel.org\]](https://www.kernel.org/pub/linux/utils/boot/dracut/dracut.html), [\[systemd.io\]](https://systemd.io/BOOT/)

***

## E a cmdline? Ela entra “automaticamente”?

A cmdline **só entra automaticamente** se você **configurar** isso para o dracut.

No dracut, a forma suportada/documentada é definir `kernel_cmdline="..."` em `/etc/dracut.conf.d/*.conf`. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

Exemplo:

`/etc/dracut.conf.d/cmdline.conf`

```ini
kernel_cmdline="root=/dev/mapper/vg_root-root ro rootfstype=xfs rd.lvm.lv=vg_root/root quiet"
```

Sem isso, o UKI pode ficar sem cmdline embutida (e aí você teria que passar opções via bootloader/entrada). O ArchWiki do dracut descreve explicitamente esse mecanismo de `kernel_cmdline`. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

***

## Como eu verifico se o microcode foi realmente embutido?

Use `lsinitrd` (o ArchWiki recomenda para inspecionar o conteúdo gerado):

```bash
sudo lsinitrd /boot/EFI/Linux/arch-linux-$(uname -r).efi | less
```

Você deve ver referências ao microcode (Intel) dentro da imagem se o pacote estiver instalado. O `lsinitrd` é a ferramenta sugerida para inspecionar o que foi incluído. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

***

## Resposta direta (sem rodeio)

*   **Microcode Intel (`intel-ucode`)**: **dracut inclui automaticamente** nas imagens que gera, desde que esteja instalado. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)
*   Para garantir que você está gerando um **UKI `.efi`** no caminho que o systemd‑boot autodetecta, use `--uefi` e especifique o arquivo de saída em `/boot/EFI/Linux/*.efi`. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/), [\[wiki.gentoo.org\]](https://wiki.gentoo.org/wiki/Unified_Kernel_Image), [\[systemd.io\]](https://systemd.io/BOOT/)
*   Para garantir que a **cmdline** vá dentro do UKI, configure `kernel_cmdline="..."` em `/etc/dracut.conf.d/`. [\[uapi-group.org\]](https://uapi-group.org/specifications/specs/boot_loader_specification/)

***

Se você quiser, eu monto um “modelo final” já considerando seus kernels **linux**, **linux-zen** e (futuro) **linux-lts**, com nomes consistentes e um hook do pacman para rebuild automático — e sem nano, prometo 😄.


