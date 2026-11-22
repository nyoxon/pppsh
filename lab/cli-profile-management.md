# Criar perfil com a CLI: host QEMU → Alpine `.2`

Este procedimento cria um perfil gerenciado no host `192.168.100.1` e provisiona
a VM Alpine `192.168.100.2` como peer PPP. Use uma sessão de console da VM para
consultar a fingerprint do host SSH por um canal independente.

## 1. Conferir a fingerprint no servidor

Na console da VM `.2`, execute como root:

```sh
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Anote o valor `SHA256:...`. Se o arquivo não existir, instale/ative as chaves
de host Ed25519 do OpenSSH antes de prosseguir.

## 2. Instalar a versão atual da CLI no host

No host `192.168.100.1`:

```sh
sudo install -m 0755 ~/Projects/pppsh/pppsh /usr/local/bin/pppsh
```

O host precisa ter `pppd`, `ssh`, `scp`, `ssh-keyscan`, `ssh-keygen`, `awk` e `sort`.
O servidor precisa aceitar chave pública para root em
`/root/.ssh/authorized_keys` e ter `/usr/sbin/pppd`. A CLI não modifica a
política do `sshd`.

## 3. Criar e provisionar o perfil

Execute no host, usando o nome de usuário administrativo que já consegue
acessar a VM por SSH:

```sh
sudo pppsh profile create qemu-host-to-alpine2 \
  --host 192.168.100.2 \
  --admin-user alpine \
  --local-ip 10.77.0.1 \
  --peer-ip 10.77.0.2 \
  --mtu 1400 \
  --source-ip 192.168.100.1 \
  --privilege su
```

Quando a CLI mostrar a fingerprint, compare-a com o valor anotado na console se
possível e digite `yes` para confiar e fixar a chave do servidor. A CLI informa
quando está autenticando a conta `alpine` por SSH/SCP e quando executa `su` para
obter privilégio root. A senha SSH é da conta `alpine`; a senha de `su` é a do
root. A CLI guarda a chave VPN gerada, mas não guarda senhas.

O servidor deve permitir autenticação SSH por chave para root. A CLI adiciona
uma linha dedicada e um wrapper em `/usr/local/libexec/pppsh/`; não habilita
login root no `sshd`.

## 4. Inspecionar e conectar

```sh
sudo pppsh profile list
sudo pppsh profile show qemu-host-to-alpine2
sudo pppsh profile validate qemu-host-to-alpine2
sudo pppsh connect qemu-host-to-alpine2
```

Deixe `connect` em primeiro plano. Em outro terminal do host, verifique o
endereço da interface PPP e alcance a ponta `.2`:

```sh
ip -brief address
ping -I 10.77.0.1 10.77.0.2
```

Interrompa com `Ctrl-C`. O enlace e a interface PPP devem ser removidos pelo
`pppd`, como no teste manual anterior.

## 5. Remover o perfil

Depois de desconectar:

```sh
sudo pppsh profile remove qemu-host-to-alpine2
```

Esse comando autentica novamente na VM, remove a autorização marcada para esse
perfil e apaga seu estado local. `--local-only` deixa a autorização remota
ativa e só deve ser usado intencionalmente.
