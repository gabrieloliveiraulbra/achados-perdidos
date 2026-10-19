# Reflexão Teórica e Equivalência em Nuvem (Artefato A5)

### Q1. Por que o serviço web consegue se conectar ao banco usando o nome `db` em vez de um endereço IP? Quem resolve esse nome e o que mudaria se os contêineres fossem criados com `docker run` na rede bridge padrão?
O serviço web consegue se conectar porque ambos os contêineres estão inseridos na mesma rede definida pelo usuário (`rede_interna`). O próprio serviço de DNS interno embutido do Docker (embedded DNS) resolve o nome do serviço `db` para o seu IP dinâmico correspondente. Se usássemos o `docker run` na rede `bridge` padrão, essa resolução automática de nomes não funcionaria; teríamos que apelar para a flag defasada `--link` ou descobrir e configurar os IPs manualmente, o que não é escalável e quebra se o IP mudar após um reinício.

### Q2. Qual a diferença entre um volume nomeado e um bind mount? Justifique por que o `init.sql` foi montado como bind mount e os dados do banco como volume nomeado.
Volumes nomeados são gerenciados integralmente pelo Docker em áreas reservadas do host e são focados em persistência de dados de longo prazo (como provado na Tarefa 5, onde a foto não sumiu após o `down`). Bind mounts mapeiam diretamente um arquivo/pasta específica do host para o contêiner. Usamos o bind mount no `init.sql` pois ele é um arquivo do repositório de código que só precisa ser injetado para leitura inicial. Já o volume nomeado foi usado nos dados do banco (`dados_banco`) para proteger as informações independentemente da pasta onde o projeto for clonado, isolando-as no gerenciamento seguro do Docker.

### Q3. Se você alterar o `init.sql` (por exemplo, adicionando uma coluna) depois que o ambiente já subiu uma vez e executar `docker compose up -d` de novo, a mudança é aplicada? Por quê? Como aplicar a alteração sem perder dados?
Não, a mudança não será aplicada. O script `init.sql` mapeado no entrypoint da imagem oficial do PostgreSQL só é lido e executado na primeira inicialização, ou seja, quando o volume do banco de dados está completamente vazio. Reiniciar com `up -d` apenas sobe o banco existente sem reexecutar os scripts. Para alterar a tabela sem perder dados, teríamos que conectar manualmente no banco (via `docker compose exec db psql`) e executar um `ALTER TABLE`, ou, em um cenário de produção, utilizar ferramentas de migração de banco como o Flyway ou Alembic.

### Q4. Por que `depends_on` sozinho, sem `condition: service_healthy`, não garante que a aplicação consiga se conectar ao banco na primeira tentativa?
O comando `depends_on` padrão garante apenas a ordem de subida (o contêiner do banco inicia antes do web), mas não garante prontidão (readiness). O PostgreSQL demora alguns segundos após ligar para montar os processos e começar a aceitar conexões. Sem o `condition: service_healthy`, a aplicação Python tentaria conectar antes da porta 5432 estar efetivamente ouvindo, causando erro de conexão (`Connection Refused`). O healthcheck garante que a aplicação só inicie quando o `pg_isready` atestar que o banco está pronto para serviço.

### Q5. A ordem das instruções `COPY requirements.txt` -> `RUN pip install` -> `COPY app.py` influencia o tempo de build. Explique usando o conceito de cache de camadas e cite o que acontece quando só o `app.py` muda.
A ordem influencia drasticamente devido ao cache de camadas do Docker (layer caching). As dependências (`requirements.txt`) sofrem alterações com frequência muito menor do que o código-fonte (`app.py`). Colocando a instalação primeiro, o Docker armazena o pesado resultado do `pip install` em cache. Se apenas o `app.py` for modificado nas próximas vezes, o Docker reaproveitará a camada pronta do `pip` e reconstruirá apenas a camada final (rápida) do código, economizando tempo de build.

### Q6. Por que a porta 5432 do banco não deve ser publicada no host? Considerando que as fotos podem mostrar documentos pessoais esquecidos, relacione a sua configuração com a LGPD e proponha duas medidas adicionais de proteção.
Publicar a porta 5432 (ex: `ports: - "5432:5432"`) deixaria o banco de dados diretamente exposto e acessível de fora, aumentando riscos de invasões e vazamentos. Pela LGPD, dados sensíveis em fotos (como RGs) devem seguir o princípio da segurança. Ao isolar o banco na `rede_interna`, blindamos a base de acessos externos diretos. Como medidas adicionais para a API, poderíamos: 1) Proteger a rota do `/itens` e `/fotos` com autenticação (Token JWT ou Login), e 2) Implementar HTTPS com Nginx na frente para criptografar o tráfego da rede e evitar interceptações.

### Q7. Tente executar `docker compose up -d --scale web=3`. O que acontece e por quê? O que seria necessário para rodar três réplicas da aplicação? As réplicas poderiam compartilhar o volume `fotos` com segurança?
Ao executar esse comando, o Docker retorna um erro de conflito de portas (port binding collision). Isso acontece porque tentamos vincular a mesma porta `8080` do computador host em três contêineres web diferentes. Para rodar três réplicas, seria necessário remover a definição explícita de `ports` no serviço web e colocar um contêiner de proxy reverso (como Nginx) atuando como balanceador de carga. As três réplicas poderiam sim compartilhar o volume nomeado de `fotos`, pois estariam apenas lendo e escrevendo arquivos.

---

### 5.1 Do ambiente local para a nuvem

O ambiente local é um protótipo do que será publicado na nuvem. A tabela abaixo lista os serviços gerenciados equivalentes em cada provedor de computação em nuvem:

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
| :--- | :--- | :--- | :--- |
| **Contêiner web** | Azure Container Instances (ACI) ou App Service | Amazon ECS (Fargate) ou AppRunner | Google Cloud Run |
| **Contêiner db + volume dados_banco** | Azure Database for PostgreSQL | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| **Volume fotos** | Azure Blob Storage | Amazon S3 | Google Cloud Storage |
| **rede_interna** | Azure Virtual Network (VNet) | Amazon VPC | Google Virtual Private Cloud (VPC) |
| **Arquivo .env (segredos)** | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| **Imagem construída localmente** | Azure Container Registry (ACR) | Amazon Elastic Container Registry (ECR) | Google Artifact Registry |

**Adaptação na Migração:**
A peça que mais exigiria mudanças na arquitetura do sistema seria o **Volume fotos**[cite: 9]. Quando migrarmos o armazenamento em disco do contêiner para o serviço nativo de nuvem (como o S3 na AWS ou Blob Storage na Azure), o código em Python precisará ser reescrito[cite: 9]. Ao invés de salvar arquivos localmente na pasta `/app/uploads` usando `os.path` e retorná-los com a rota Flask `send_from_directory`, a API precisará integrar o SDK (ex: `boto3`) da provedora de nuvem, passando a fazer o upload da imagem diretamente no bucket de armazenamento em nuvem e a retornar o link HTTPS gerado pelo provedor para a visualização dos arquivos.