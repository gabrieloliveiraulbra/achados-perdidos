# Achados & Perdidos - API (Docker)

## Objetivo do Projeto
O "Achados & Perdidos" é uma aplicação web (API Python/Flask) com banco de dados PostgreSQL desenvolvida para o Setor de Atendimento ao Estudante. O objetivo deste projeto de infraestrutura é conteinerizar a aplicação garantindo que ela possa ser executada em qualquer máquina com um único comando, mantendo a persistência dos dados (itens e fotos) mesmo quando o ambiente for reiniciado, além de preparar a arquitetura para uma futura migração para a nuvem.

## Arquitetura do Sistema (Artefato A2)

A arquitetura completa do sistema conteinerizado, incluindo delimitação de fronteiras de rede (`rede_interna`), roteamento via proxy reverso Nginx, contêineres (`web`, `db`, `nginx` e `adminer`), volumes e bind mounts, está documentada de forma isolada com diagrama Mermaid em:

👉 **[docs/arquitetura.md](docs/arquitetura.md)**


## Pré-requisitos
Para executar este projeto, você precisará ter instalado em sua máquina:
* **Docker** e **Docker Compose** (Docker Desktop recomendado para Windows).
* Um terminal (PowerShell, Bash, etc.) para rodar os comandos e ferramentas como `curl` para testar as requisições HTTP.

## Configuração do Ambiente (.env)
Antes de subir a aplicação pela primeira vez, é necessário configurar as variáveis de ambiente com as credenciais do banco de dados:

1. Na raiz do projeto, localize o arquivo modelo chamado `.env.example`.
2. Faça uma cópia deste arquivo e renomeie a cópia para `.env`.
3. Abra o arquivo `.env` e preencha os valores reais para as senhas e usuários, se desejar (ou apenas mantenha os valores de teste já presentes). **Atenção:** O arquivo `.env` contém dados sensíveis e é ignorado pelo Git (`.gitignore`).

## Operação do Ambiente (Passo a Passo)

### 1. Subir o ambiente
Para construir a imagem da aplicação e iniciar todos os serviços em segundo plano (detached mode), execute na raiz do projeto:
```bash
docker compose up -d --build
```
*Aguarde alguns segundos até que o terminal indique que o banco de dados está `Healthy`, a aplicação web está `Started` e o proxy reverso Nginx está `Started`.*

### 2. Testar o ambiente
Você pode testar se a API está no ar rodando o healthcheck através do Nginx:
```bash
curl http://localhost:8080/health
```

### 3. Modo de Depuração / Painel Adminer (Desafio Bônus)
Para iniciar o ambiente incluindo o painel de gerenciamento visual do PostgreSQL (Adminer):
```bash
docker compose --profile debug up -d
```
*Acesse no navegador: `http://localhost:8081` (Sistema: PostgreSQL, Servidor: `db`, Usuário: `aluno`, Base: `achados_db`).*

### 4. Parar o ambiente
Para desligar os contêineres sem remover as redes e sem apagar os volumes de dados:
```bash
docker compose stop
```
*(Para religar, basta rodar `docker compose start`)*.

### 5. Destruir o ambiente (Reset total)
Para parar os contêineres, remover a rede interna e **apagar definitivamente** os volumes (destruindo todos os dados do banco e as fotos salvas), execute:
```bash
docker compose down -v
```

---

## Tabela de Rotas da API

A aplicação web roda internamente na porta 5000 (isolada na rede interna), sendo roteada e acessível no Host pela porta **8080** através do proxy reverso **Nginx**.

| Método | Rota | Descrição |
| :--- | :--- | :--- |
| **GET** | `/health` | Verifica a saúde da API e o status de conexão com o banco de dados. |
| **GET** | `/itens` | Retorna a lista de todos os objetos cadastrados no banco de dados. |
| **GET** | `/itens?local=texto` | Filtra e retorna a lista de objetos pelo local informado. |
| **POST** | `/itens` | Cadastra um novo objeto. Espera um form-data com `descricao`, `local` e um arquivo opcional `foto`. |
| **GET** | `/fotos/<nome>` | Acessa e exibe uma foto salva fisicamente no volume (ex: `eadc9baeea.jpg`). |
| **GET** | `/estatisticas` | Retorna o total numérico de itens no banco e de arquivos físicos no volume. |