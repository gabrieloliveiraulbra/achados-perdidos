# Artefato A2 - Diagrama de Arquitetura

Este documento contém o **Diagrama de Arquitetura (Artefato A2)** da solução conteinerizada "Achados & Perdidos", representando as fronteiras de rede, os contêineres (`nginx`, `web`, `db` e `adminer`), portas publicadas, volumes nomeados e pontos de montagem.

---

## Diagrama de Arquitetura do Ambiente

```mermaid
graph TB
    %% =========================================================================
    %% DELIMITAÇÃO 1: FORA DA REDE INTERNA (MÁQUINA HOST)
    %% =========================================================================
    subgraph Host["FORA DA REDE INTERNA (Máquina Host)"]
        direction TB

        Client["Cliente Externo<br/>(Navegador / curl)<br/><code>localhost:8080</code>"]
        DevAdmin["Desenvolvedor / DBA<br/>(Painel Adminer - Debug)<br/><code>localhost:8081</code>"]
        InitSQL["Arquivo Local no Host:<br/><code>./db/init.sql</code>"]
        NginxConf["Arquivo de Configuração:<br/><code>./nginx/default.conf</code>"]

        subgraph HostStorage["Armazenamento dos Volumes no Host"]
            VolFotos[("Volume Nomeado: <b>fotos</b><br/>(Armazenado no Host)")]
            VolBanco[("Volume Nomeado: <b>dados_banco</b><br/>(Armazenado no Host)")]
        end
    end

    %% =========================================================================
    %% DELIMITAÇÃO 2: DENTRO DA REDE INTERNA (REDE BRIDGE)
    %% =========================================================================
    subgraph RedeInterna["DENTRO DA REDE INTERNA (rede_interna - Bridge)"]
        direction TB

        subgraph NginxContainer["Contêiner: nginx (Proxy Reverso)"]
            NginxProcess["Servidor Nginx (nginx:alpine)<br/>Fronteira de Entrada HTTP"]
            NginxPort["Porta Interna: <code>80</code>"]
            NginxConfMount["Ponto de montagem:<br/><code>/etc/nginx/conf.d/default.conf</code>"]
        end

        subgraph WebContainer["Contêiner: web (API)"]
            WebProcess["API Python/Flask + Gunicorn<br/>Usuário: <code>appuser</code> (não-root)<br/><b>SEM porta publicada no Host</b>"]
            WebPort["Porta Interna: <code>5000</code>"]
            WebUploads["Diretório montado:<br/><code>/app/uploads</code>"]
        end

        subgraph DbContainer["Contêiner: db (Banco de Dados)"]
            DbProcess["PostgreSQL (<code>postgres:16-alpine</code>)<br/><b>SEM porta publicada no Host</b>"]
            DbPort["Porta Interna: <code>5432</code><br/>(DNS interno: <code>db</code>)"]
            DbData["Diretório montado:<br/><code>/var/lib/postgresql/data</code>"]
            DbInitMount["Ponto de montagem:<br/><code>/docker-entrypoint-initdb.d/init.sql</code>"]
        end

        subgraph AdminerContainer["Contêiner: adminer (Debug)"]
            AdminerProcess["Adminer (DB Web Client)<br/>Perfil: <code>debug</code>"]
            AdminerPort["Porta Interna: <code>8080</code>"]
        end
    end

    %% Portas Publicadas no Host
    Client -- "Porta publicada no Host: 8080:80 (HTTP)" --> NginxPort
    DevAdmin -. "Porta publicada (profile debug): 8081:8080" .-> AdminerPort

    %% Tráfego e Comunicação na Rede Interna
    NginxProcess -- "Proxy Reverso: proxy_pass http://web:5000" --> WebPort
    WebProcess -- "Comunicação via DNS Interno: db:5432 (TCP)" --> DbPort
    AdminerProcess -. "Conexão de depuração via DNS: db:5432" .-> DbPort

    %% Montagens de Volumes Nomeados
    VolFotos -. "Volume fotos montado em /app/uploads" .-> WebUploads
    VolBanco -. "Volume dados_banco montado em /var/lib/postgresql/data" .-> DbData

    %% Bind Mounts
    InitSQL -. "Bind mount (:ro) em /docker-entrypoint-initdb.d/init.sql" .-> DbInitMount
    NginxConf -. "Bind mount (:ro) em /etc/nginx/conf.d/default.conf" .-> NginxConfMount

    %% Estilização
    classDef hostArea fill:#f8f9fa,stroke:#495057,stroke-width:2px;
    classDef storageArea fill:#fff9db,stroke:#f59f00,stroke-width:2px;
    classDef networkArea fill:#e7f5ff,stroke:#1971c2,stroke-width:2px,stroke-dasharray: 4 4;
    classDef proxyBox fill:#e3fafc,stroke:#1098ad,stroke-width:2px;
    classDef containerBox fill:#ffffff,stroke:#2b8a3e,stroke-width:2px;
    classDef debugBox fill:#fff0f6,stroke:#d6336c,stroke-width:2px,stroke-dasharray: 3 3;
    classDef clientBox fill:#f1f3f5,stroke:#212529,stroke-width:2px;

    class Host hostArea;
    class HostStorage,VolFotos,VolBanco,InitSQL,NginxConf storageArea;
    class RedeInterna networkArea;
    class NginxContainer,NginxProcess,NginxPort,NginxConfMount proxyBox;
    class WebContainer,DbContainer,WebProcess,WebPort,WebUploads,DbProcess,DbPort,DbData,DbInitMount containerBox;
    class AdminerContainer,AdminerProcess,AdminerPort debugBox;
    class Client,DevAdmin clientBox;
```

---

## Mapeamento dos Elementos Arquiteturais

| Categoria | Elemento | Descrição / Configuração |
| :--- | :--- | :--- |
| **Fronteira Externa** | Máquina Host | Ambiente hospedeiro onde rodam o cliente (`curl`/navegador) e o Docker Engine. |
| **Fronteira Externa** | Cliente HTTP | Acesso à API via `http://localhost:8080`. |
| **Fronteira Externa** | Acesso Debug (Adminer) | Interface web do banco de dados via `http://localhost:8081` (ativo apenas com `--profile debug`). |
| **Fronteira Externa** | Script SQL Local | Arquivo `./db/init.sql` no Host contendo o schema DDL inicial. |
| **Fronteira Externa** | Configuração Nginx | Arquivo `./nginx/default.conf` no Host com as diretivas de proxy reverso. |
| **Fronteira Externa** | Armazenamento de Volumes | Áreas físicas no Host gerenciadas pelo Docker para persistência dos dados. |
| **Fronteira Interna** | `rede_interna` | Rede bridge definida pelo usuário (`driver: bridge`), provendo isolamento e resolução DNS automática. |
| **Proxy Reverso** | `nginx` | Contêiner com imagem `nginx:alpine`, escutando na porta 80 e publicado no Host como `8080:80`. Encaminha requisições para `web:5000`. |
| **Contêiner Web** | `web` | API Python/Flask com Gunicorn escutando na porta interna 5000 sob o usuário não-privilegiado `appuser`. **Sem portas expostas no Host**. |
| **Contêiner DB** | `db` | Banco PostgreSQL (`postgres:16-alpine`) escutando na porta interna 5432 via DNS (`db`), **sem porta publicada no Host** (RNF03). |
| **Painel de Depuração** | `adminer` | Serviço opcional (`profiles: ["debug"]`) para gerenciamento web do banco de dados na porta 8081 do host. |
| **Volume de Fotos** | `fotos` | Volume nomeado montado no contêiner `web` em `/app/uploads`. |
| **Volume de Banco** | `dados_banco` | Volume nomeado montado no contêiner `db` em `/var/lib/postgresql/data`. |
| **Bind Mount (SQL)** | `./db/init.sql` | Arquivo do Host montado como somente leitura (`:ro`) em `/docker-entrypoint-initdb.d/init.sql` no contêiner `db`. |
| **Bind Mount (Nginx)** | `./nginx/default.conf` | Arquivo do Host montado como somente leitura (`:ro`) em `/etc/nginx/conf.d/default.conf` no contêiner `nginx`. |
