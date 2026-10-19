# Artefato A2 - Diagrama de Arquitetura

Este documento contém o **Diagrama de Arquitetura (Artefato A2)** da solução conteinerizada "Achados & Perdidos", representando as fronteiras de rede, contêineres, portas publicadas, volumes nomeados e pontos de montagem.

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
        InitSQL["Arquivo Local no Host:<br/><code>./db/init.sql</code>"]

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

        subgraph WebContainer["Contêiner: web"]
            WebProcess["API Python/Flask + Gunicorn<br/>Usuário: <code>appuser</code> (não-root)"]
            WebPort["Porta Interna: <code>5000</code>"]
            WebUploads["Diretório montado:<br/><code>/app/uploads</code>"]
        end

        subgraph DbContainer["Contêiner: db"]
            DbProcess["PostgreSQL (<code>postgres:16-alpine</code>)<br/><b>NÃO possui porta publicada no Host</b>"]
            DbPort["Porta Interna: <code>5432</code><br/>(DNS interno: <code>db</code>)"]
            DbData["Diretório montado:<br/><code>/var/lib/postgresql/data</code>"]
            DbInitMount["Ponto de montagem:<br/><code>/docker-entrypoint-initdb.d/init.sql</code>"]
        end
    end

    %% Conexão de Porta Publicada no Host
    Client -- "Porta publicada no Host: 8080:5000" --> WebPort

    %% Conexão de Rede Interna
    WebProcess -- "Comunicação interna via DNS (host: db, porta: 5432)" --> DbPort

    %% Montagens de Volumes Nomeados
    VolFotos -. "Volume fotos montado em /app/uploads" .-> WebUploads
    VolBanco -. "Volume dados_banco montado em /var/lib/postgresql/data" .-> DbData

    %% Bind Mount do init.sql
    InitSQL -. "Bind mount somente leitura (:ro) em /docker-entrypoint-initdb.d/init.sql" .-> DbInitMount

    %% Estilização
    classDef hostArea fill:#f8f9fa,stroke:#495057,stroke-width:2px;
    classDef storageArea fill:#fff9db,stroke:#f59f00,stroke-width:2px;
    classDef networkArea fill:#e7f5ff,stroke:#1971c2,stroke-width:2px,stroke-dasharray: 4 4;
    classDef containerBox fill:#ffffff,stroke:#2b8a3e,stroke-width:2px;
    classDef clientBox fill:#f1f3f5,stroke:#212529,stroke-width:2px;

    class Host hostArea;
    class HostStorage,VolFotos,VolBanco,InitSQL storageArea;
    class RedeInterna networkArea;
    class WebContainer,DbContainer,WebProcess,WebPort,WebUploads,DbProcess,DbPort,DbData,DbInitMount containerBox;
    class Client clientBox;
```

---

## Mapeamento dos Elementos Arquiteturais

| Categoria | Elemento | Descrição / Configuração |
| :--- | :--- | :--- |
| **Fronteira Externa** | Máquina Host | Ambiente hospedeiro onde rodam o cliente (`curl`/navegador) e o Docker Engine. |
| **Fronteira Externa** | Cliente HTTP | Acesso à API via `http://localhost:8080`. |
| **Fronteira Externa** | Script SQL Local | Arquivo `./db/init.sql` no Host contendo o schema DDL inicial. |
| **Fronteira Externa** | Armazenamento de Volumes | Áreas físicas no Host gerenciadas pelo Docker para persistência dos dados. |
| **Fronteira Interna** | `rede_interna` | Rede bridge definida pelo usuário (`driver: bridge`), provendo isolamento e resolução DNS automática. |
| **Porta Publicada** | `8080:5000` | Mapeamento que direciona o tráfego do Host (`:8080`) para a porta da aplicação (`:5000`). |
| **Contêiner Web** | `web` | API Python/Flask com Gunicorn escutando na porta 5000 sob o usuário não-privilegiado `appuser`. |
| **Contêiner DB** | `db` | Banco PostgreSQL (`postgres:16-alpine`) escutando na porta interna 5432 via DNS (`db`), **sem porta publicada no Host** (RNF03). |
| **Volume de Fotos** | `fotos` | Volume nomeado montado no contêiner `web` em `/app/uploads`. |
| **Volume de Banco** | `dados_banco` | Volume nomeado montado no contêiner `db` em `/var/lib/postgresql/data`. |
| **Bind Mount** | `./db/init.sql` | Arquivo do Host montado como somente leitura (`:ro`) em `/docker-entrypoint-initdb.d/init.sql` no contêiner `db`. |
