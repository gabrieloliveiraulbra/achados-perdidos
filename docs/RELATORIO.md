# Relatório Técnico - Arquitetura e Decisões

Este documento detalha as configurações e o funcionamento da infraestrutura Docker criada para a aplicação "Achados & Perdidos", cumprindo integralmente as exigências dos artefatos A2 e A3.

---

## Diagrama de Arquitetura do Ambiente (Artefato A2)

O diagrama oficial de arquitetura (Artefato A2), com a especificação visual em Mermaid da delimitação de rede (`rede_interna`), portas publicadas, contêineres (`web` e `db`), volumes e bind mounts, está documentado em seu arquivo dedicado e isolado:

👉 **Consulte o artefato em: [docs/arquitetura.md](arquitetura.md)**


---

## (a) Explicação do Dockerfile (app/Dockerfile)

O `Dockerfile` foi estruturado considerando segurança (usuário sem privilégios) e otimização de cache (ordem das camadas):

1. **`FROM python:3.12-slim`**: Define a imagem base. O uso da versão `slim` reduz drasticamente o tamanho final da imagem, pois traz apenas os pacotes essenciais do sistema operacional Debian.
2. **`WORKDIR /app`**: Cria (se não existir) e define o diretório de trabalho padrão dentro do contêiner.
3. **`RUN useradd -m appuser`**: Cria um usuário restrito do sistema (não-root) para garantir a segurança da execução (RNF04).
4. **`RUN mkdir -p /app/uploads`**: Garante a criação prévia da pasta onde as fotos serão salvas fisicamente.
5. **`COPY requirements.txt .`**: Copia apenas o arquivo de dependências primeiro. Isso é crucial para o cache: se o código (`app.py`) mudar, mas as dependências não, o Docker reaproveita a camada de instalação do `pip`, economizando muito tempo no build.
6. **`RUN pip install --no-cache-dir -r requirements.txt`**: Instala as bibliotecas do Python. A flag `--no-cache-dir` evita que o pip guarde arquivos temporários de cache, diminuindo o tamanho da imagem.
7. **`COPY . .`**: Copia o restante do código da aplicação (`app.py`, etc) para dentro do contêiner.
8. **`RUN chown -R appuser:appuser /app`**: Transfere a propriedade de todos os arquivos da pasta `/app` (incluindo a pasta `uploads`) para o usuário restrito, permitindo que a aplicação consiga gravar fotos lá.
9. **`USER appuser`**: Troca o contexto de execução de root para o usuário restrito que acabamos de criar.
10. **`EXPOSE 5000`**: Serve como documentação indicando que o contêiner vai escutar na porta 5000.
11. **`CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]`**: Comando de inicialização. Usa o Gunicorn no formato *exec* (lista) para escutar as conexões em todas as interfaces de rede do contêiner (`0.0.0.0`) na porta 5000.

---

## (b) Explicação do docker-compose.yml

O arquivo agrupa toda a arquitetura para subir com um único comando, definindo redes, volumes e os dois serviços principais:

### Serviço `db`
* **Imagem**: Usa `postgres:16-alpine` (oficial e leve).
* **Environment**: Puxa as variáveis de banco de dados (`POSTGRES_DB`, etc) do arquivo `.env` sem expor senhas no código (RNF03).
* **Volumes**: 
  * Faz um *bind mount* de `./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro` (somente leitura) para rodar o script na primeira inicialização.
  * Associa o volume nomeado `dados_banco` à pasta interna `/var/lib/postgresql/data` para persistir o banco.
* **Healthcheck**: Executa `pg_isready` repetidamente para saber quando o banco está apto a receber conexões.
* **Rede e Portas**: Fica apenas na `rede_interna`. Não possui a diretiva `ports`, o que o mantém isolado (sem acesso direto do host).
* **Restart**: `unless-stopped` garante que ele reinicie automaticamente caso caia ou o servidor reinicie.

### Serviço `web`
* **Build**: Aponta para o diretório `./app` para construir o Dockerfile localmente.
* **Depends_on**: Usa a condição `service_healthy` apontando para o `db`. Isso trava a inicialização da API até que o PostgreSQL informe que está 100% pronto (RNF05).
* **Ports**: Mapeia a porta `8080` da máquina Host para a porta `5000` do contêiner.
* **Environment**: Passa as credenciais e configuração de host (o `DB_HOST=db` permite a conexão via DNS interno do Docker).
* **Volumes**: Associa o volume nomeado `fotos` à pasta `/app/uploads` do contêiner, garantindo a persistência dos arquivos físicos.
* **Rede**: Fica na `rede_interna` para conseguir enxergar e se comunicar com o banco de dados.

---

## (c) Caminho da Requisição (POST /itens)

O fluxo completo, desde o comando do usuário até a gravação no disco, ocorre da seguinte forma:

1. O cliente (usuário) executa o comando `curl -F...` no terminal do **Host**.
2. A requisição HTTP bate no IP `localhost` e na porta `8080` do **Host**.
3. O Docker recebe essa requisição na porta `8080` e encaminha o tráfego para a porta `5000` do contêiner `web` através da interface de rede em ponte (`bridge`).
4. Dentro do contêiner, o servidor **Gunicorn** intercepta a requisição e a repassa para a aplicação **Flask** (`app.py`).
5. A função `cadastrar()` do Flask valida os dados (texto) e identifica o arquivo de foto enviado.
6. A aplicação gera um nome único (UUID) para a foto e a salva fisicamente no diretório `/app/uploads`. Como este diretório está mapeado para o volume nomeado `fotos`, o arquivo é efetivamente gravado no disco físico do computador host (em `/var/lib/docker/volumes/...`).
7. A aplicação usa a biblioteca `psycopg` para iniciar uma conexão com o serviço `db` na porta `5432` da `rede_interna`. O Docker resolve automaticamente o nome `db` para o IP interno do contêiner do PostgreSQL.
8. A query SQL de `INSERT` é enviada com os textos e o nome do arquivo. O PostgreSQL grava esses metadados nos seus arquivos físicos (que estão dentro do volume `dados_banco` no disco do host).
9. O PostgreSQL retorna o `id` recém-criado.
10. O Flask monta um JSON de resposta (com `id` e `nome_foto`) e o Gunicorn devolve para o Docker, que devolve ao cliente no terminal com o status `201 Created`.

---

## (d) Ciclo de Vida dos Volumes

Com base nos testes de persistência da Tarefa 5, o ciclo de vida dos volumes nomeados configurados (`dados_banco` e `fotos`) tem o seguinte comportamento:

* **Criação:** São criados automaticamente na primeira execução do comando `docker compose up`. 
* **Persistência ao reinício:** Sobrevivem de forma independente ao ciclo de vida efêmero dos contêineres. Ao executar `docker compose down` e `up`, os contêineres e redes são destruídos e recriados do zero, mas os volumes permanecem intactos, salvando os dados e imagens.
* **Armazenamento Físico:** Os dados não ficam dentro do contêiner, mas sim guardados permanentemente no disco da máquina hospedeira.
* **Destruição Voluntária:** Só são removidos permanentemente se houver uma intervenção manual deliberada usando a flag `--volumes` ou `-v` (ex: `docker compose down -v`). Quando isso ocorre, o banco volta a ficar vazio e a inicialização via `bind mount` (init.sql) ocorre novamente na próxima subida.