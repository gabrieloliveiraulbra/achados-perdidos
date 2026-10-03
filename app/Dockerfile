# Partir de uma imagem oficial e enxuta do Python 3.12 (variante slim)
FROM python:3.12-slim

# Criar diretório de trabalho
WORKDIR /app

# cria o usuario comum (pra nao rodar como root)
RUN useradd -m appuser

# cria a pasta de uploads
RUN mkdir -p /app/uploads

# ja da a permissao dessa pasta base pro usuario novo (requisito RNF04)
RUN chown -R appuser:appuser /app

# Copiar e instalar o requirements.txt ANTES de copiar o código (aproveita o cache de camadas)
COPY requirements.txt .

# Instalar dependências sem guardar cache do pip
RUN pip install --no-cache-dir -r requirements.txt

# Copiar o restante do código da aplicação
COPY . .

# Garantir que o usuário appuser seja dono de todos os arquivos copiados
RUN chown -R appuser:appuser /app

# Trocar para o usuário sem privilégios
USER appuser

# Documentar a porta
EXPOSE 5000

# Iniciar com o Gunicorn escutando em 0.0.0.0:5000 (forma exec com colchetes)
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:5000", "app:app"]