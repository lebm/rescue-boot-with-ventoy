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

