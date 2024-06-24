# Instalação Samba 4


Samba é Software Livre licenciado sob a Licença Pública Geral GNU , o projeto Samba é membro da Software Freedom Conservancy.
Desde 1992, fornece serviços de arquivo e impressão seguros, estáveis ​​e rápidos para todos os clientes que usam o protocolo SMB/CIFS, como todas as versões de DOS e Windows, OS/2, Linux e muitos outros.

Samba é um componente importante para integrar perfeitamente servidores e desktops Linux/Unix em ambientes Active Directory. Ele pode funcionar como controlador de domínio ou como membro regular do domínio.



## Ambiente de instalação

- SO: Ubuntu 24.04
- Domínio: teste.local
- IP do servidor: 172.18.3.50
- Hostname: srvdc01



## Configuração de Rede

Verifique os seguintes arquivos antes de iniciar a instalação:

1. /etc/hostname

```
srvdc01
```

2. /etc/hosts

```
127.0.0.1       localhost
127.0.1.1       srvdc01.teste.local       srvdc01
```

3. /etc/network/interfaces

```
# The primary network interface
allow-hotplug ens18
iface ens18 inet static
address 172.18.3.50/24
gateway 172.18.3.24
```

4. /etc/resolv.conf

```
search teste.local
nameserver 8.8.8.8
```


## Instalação das dependências

Faça o download do samba com o seguinte comando:

```
wget https://download.samba.org/pub/samba/stable/samba-4.20.2.tar.gz
```

Descompacte o arquivo baixado:

```
tar zxvf samba-4.20.2.tar.gz
```


Entre no diretório para instalação:

```
cd samba-4.20.2
```

Atualmente existe um script para instalação das dependências do samba dentro do diretório de instalação. Entre no diretório da distribuição em específico e execute o script bootstrap.sh.

```
cd bootstrap/generated-dists/ubuntu2204/
```

```
./bootstrap.sh
```

```
cd ../../..
```


## Instalação


Execute o comando abaixo para executar a configuração. O parâmetro -j 4 executa quatro processos em paralelo.

```
./configure -j 4
```

Executar o make e make install para concluir a instalação.

```
make -j 4
```

```
make install -j 4
```

Por padrão os arquivos da instalação do samba ficam em /usr/local/samba
para acessar os binários é preciso adicionar o diretório do samba ao path do sistema.

```
echo "export PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin" >> ~/.bashrc
echo "export PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin" >> /etc/bash.bashrc
```

## Provisionamento do domínio

Para iniciar o provisionamento do domínio execute o comando:

```
samba-tool domain provision --use-rfc2307 --interactive --option="interfaces=lo enp0s3" --option="bind interfaces only=yes"
```

Digite os parâmetros solicitados durante a instalação


```
Realm [teste.local]:  
Domain [TESTE]:  
Server Role (dc, member, standalone) [dc]:  
DNS backend (SAMBA_INTERNAL, BIND9_FLATFILE, BIND9_DLZ, NONE) [SAMBA_INTERNAL]:  
DNS forwarder IP address (write 'none' to disable forwarding) [8.8.8.8]:
Senha de administrador
```


Configurar o kerberos:

```
cp /usr/local/samba/private/krb5.conf /etc/
```

Adicionar as seguintes informações na seção de realms do arquivo de configuração do kerberos.

Antes:
```
[realms]
teste.local = {
        default_domain = teste.local

}
```


Depois:

```
[realms]
teste.local = {
        default_domain = teste.local
        kdc = 172.18.3.50
        admin_server = 172.18.3.50

}
```

## Configuração do systemd para gerenciar o serviço do Samba

Em um DC, o serviço /usr/local/samba/sbin/samba inicia automaticamente os serviços smbd e winbindd necessários como subprocessos. Se você iniciá-los manualmente, o Samba DC não funcionará conforme o esperado. Se o seu provedor de pacotes criou arquivos de serviço Samba adicionais, desative-os e mascare-os para evitar que outros serviços os reativem.


```
systemctl mask smbd nmbd winbind
```

```
systemctl disable smbd nmbd winbind
```



Crie o arquivo /etc/systemd/system/samba-ad-dc.service com um editor de texto

```
nano /etc/systemd/system/samba-ad-dc.service
```

Cole o conteúdo abaixo dentro no arquivo.

```
[Unit]
Description=Samba Active Directory Domain Controller
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
ExecStart=/usr/local/samba/sbin/samba -D
PIDFile=/usr/local/samba/var/run/samba.pid
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```


Recarregue a configuração do systemd

```
systemctl daemon-reload
```

Configure o serviço do samba para ele inicar automaticamente:

```
systemctl enable samba-ad-dc
```


Para parar iniciar o serviço do samba manualmente:

```
service samba-ad-dc start
```

## Adicionar DC secundário

```
samba-tool domain join teste.local DC -U"TESTE\administrator" --option="interfaces=lo enp0s3" --option="bind interfaces only=yes" --dns-backend=SAMBA_INTERNAL --option="dns forwarder=8.8.8.8" --option='idmap_ldb:use rfc2307 = yes'
```
