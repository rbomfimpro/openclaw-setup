## Openclaw Self-Hosted (Container)

### Configurando o Openclaw de forma segura

#### Setup
Neste setup o Openclaw é instalado via Docker Compose em um servidor Linux pessoal na rede local. Para resolver o hostname, é criada uma entrada DNS local no arquivo `/etc/hosts` do computador cliente (no meu caso, macOS) apontando o IP do servidor para `openclaw.local`.

<br>

#### LLM
É possível utilizar modelos locais caso o servidor possua memória suficiente. Como estou utilizando um servidor modesto, optei por utilizar a LLM do DeepSeek via API.

<br>

#### Requisitos
- Docker e Docker Compose
- [mkcert](https://github.com/FiloSottimo/mkcert)
- Acesso a uma LLM (local ou via API)

<br>

#### Por que configurar TLS?
O Openclaw faz uma série de validações de segurança para garantir que o Gateway seja acessado por uma fonte confiável. O problema é que a maioria dos tutoriais desabilita essas validações por conveniência, comprometendo a segurança do ambiente.

Para manter essas validações ativas e acessar o Gateway de forma segura, este setup utiliza:

- **mkcert** — para criar uma CA (Certificate Authority) local confiável e gerar certificados TLS válidos
- **Caddy** — como proxy reverso, terminando o TLS e encaminhando o tráfego para o container do Openclaw

Além disso, algumas configurações precisam ser definidas no arquivo `openclaw.json` para que o Openclaw reconheça o acesso via proxy como seguro.

<br>

#### Procedimentos — mkcert

> **Nota:** os comandos abaixo devem ser executados no computador **cliente** (o que vai acessar o Openclaw pelo browser), não no servidor.

**IP do servidor (exemplo):** `192.168.1.230`

**1. Instalar a CA local no sistema**

Esse comando cria uma CA raiz e a registra no trust store do sistema operacional e dos navegadores. Só precisa ser executado uma vez.
```bash
mkcert -install
```

**2. Gerar os certificados TLS**

Isso gera um certificado válido para o hostname `openclaw.local` e para o IP do servidor.
```bash
mkcert openclaw.local 192.168.1.230
```

**3. Renomear os arquivos gerados**
```bash
mv openclaw.local+1.pem openclaw-tls.crt
mv openclaw.local+1-key.pem openclaw-tls.key
```

**4. Enviar os certificados para o servidor**
```bash
scp openclaw-tls.crt openclaw-server:/path/to/certs/
scp openclaw-tls.key openclaw-server:/path/to/certs/
```

**5. Configurar o Caddyfile**

Criar o arquivo `Caddyfile` com a rota para o Openclaw:
```
openclaw.local {
    tls /certs/openclaw-tls.crt /certs/openclaw-tls.key
    reverse_proxy openclaw-gateway:18789
}
```

**6. Configurar a entrada DNS local**

No computador cliente, adicionar a entrada no `/etc/hosts`:
```
192.168.1.230   openclaw.local
```

<br>
<br>

### Openclaw Commands

#### Aprovar Dispositivo Local na Rede para acessar o Gateway
```bash
docker compose run --rm openclaw-cli devices list
```

```bash
docker compose run --rm openclaw-cli devices approve <requestId>
```

#### Validar Segurança
```bash
docker compose run --rm openclaw-cli security audit --deep
```

```bash
docker compose run --rm openclaw secrets audit --check
```
