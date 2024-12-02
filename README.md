# Projeto de Monitoramento com Flask, Prometheus e Grafana

## Visão Geral
Este projeto integra uma aplicação web desenvolvida com Flask, um banco de dados MariaDB, um exportador Prometheus para coleta de métricas e dashboards no Grafana para visualização. Ele permite monitorar métricas de desempenho de uma aplicação Flask e de um banco de dados, bem como configurar e exibir dashboards personalizáveis para análise detalhada.

## Tecnologias Utilizadas
- **Back-end**: Flask, Flask-AppBuilder, SQLAlchemy
- **Banco de Dados**: MariaDB
- **Monitoramento**: Prometheus, Prometheus-Flask-Exporter
- **Visualização**: Grafana
- **Testes**: Pytest

---

## Estrutura do Projeto

### 1. Configuração do MySQL Exporter
O arquivo de configuração para o Prometheus Exporter do MariaDB:

```yaml
DATA_SOURCE_NAME: "user:password@(mariadb:3306)/"  # Configura as credenciais e o endereço do MariaDB para coleta de métricas.
```

### 2. Dockerfile para a Aplicação Flask
Arquivo de configuração para containerizar a aplicação Flask.

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py /app/

CMD ["flask", "run", "--host=0.0.0.0"]
```

### 3. Código Principal do Flask (app.py)
Arquivo principal da aplicação Flask.

```python
import time
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_appbuilder import AppBuilder, SQLA
from flask_appbuilder.models.sqla.interface import SQLAInterface
from flask_appbuilder import ModelView
from sqlalchemy.exc import OperationalError
from prometheus_flask_exporter import PrometheusMetrics
import logging

app = Flask(__name__)

metrics = PrometheusMetrics(app)
app.config['SECRET_KEY'] = 'minha_chave_secreta_super_secreta'  # Substitua por uma chave segura
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:root_password@mariadb/school_db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
```

### 4. Requirements.txt
Dependências do projeto.

```txt
Flask==1.1.4
Flask-SQLAlchemy==2.4.4
PyMySQL==0.9.3
Flask-AppBuilder==3.3.0
Werkzeug==1.0.1
MarkupSafe==2.0.1
WTForms==2.3.3
prometheus-flask-exporter==0.18.3
```

### 5. Testes com Pytest
Testes automatizados para verificar a funcionalidade das rotas da API Flask.

```python
import pytest
from flask import Flask
from flask.testing import FlaskClient
from app import app  # Assumindo que seu arquivo principal é app.py

@pytest.fixture
def client():
    with app.test_client() as client:
        yield client

def test_listar_alunos(client: FlaskClient):
    """Testa a rota GET /alunos"""
    response = client.get('/alunos')
    assert response.status_code == 200
    assert isinstance(response.json, list)

def test_adicionar_aluno(client: FlaskClient):
    """Testa a rota POST /alunos"""
    new_aluno = {
        "nome": "Antonio",
        "turma": "9A",
        "ra": "12345"
    }
    response = client.post('/alunos', json=new_aluno)
    assert response.status_code == 201
    assert response.json['message'] == 'Aluno adicionado com sucesso!'
```

### 6. Dockerfile para Grafana
Arquivo de configuração para containerizar o Grafana e provisionar dashboards.

```dockerfile
FROM grafana/grafana:latest

USER root
RUN mkdir /var/lib/grafana/dashboards

COPY provisioning/datasource.yml /etc/grafana/provisioning/datasources/
COPY provisioning/dashboard.yml /etc/grafana/provisioning/dashboards/
COPY dashboards/mariadb_dashboard.json /var/lib/grafana/dashboards/

RUN chown -R 472:472 /etc/grafana/provisioning
USER grafana
```

### 7. Configuração do Dashboard do Grafana
Arquivo JSON de configuração do dashboard.

```json
{
  "uid": "http_performance_dashboard",
  "title": "Monitoramento HTTP - Métricas Principais",
  "tags": ["HTTP", "Performance", "Prometheus"],
  "timezone": "browser",
  "schemaVersion": 16,
  "version": 1,
  "panels": [
    {
      "type": "graph",
      "title": "Taxa de Requisições por Segundo",
      "datasource": "Prometheus",
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 6 },
      "targets": [
        {
          "expr": "rate(prometheus_http_requests_total[1m])",
          "legendFormat": "Requisições/s",
          "refId": "A"
        }
      ],
      "lines": true,
      "linewidth": 2,
      "fill": 2
    }
  ]
}
```

### 8. Configuração de Datasources do Grafana
Arquivo de configuração de fontes de dados.

```yaml
apiVersion: 1
providers:
  - name: "MariaDB Dashboards"
    orgId: 1
    folder: ""
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
```

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 5s
```

### 9. Configuração do Prometheus
Arquivo de configuração do Prometheus.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "mysqld_exporter"
    static_configs:
      - targets: ["mysqld_exporter:9104"]
```

---

## Como Rodar o Projeto

### 1. Iniciando o Banco de Dados MariaDB
```bash
docker build -t mariadb -f Dockerfile.mariadb .
docker run -d --name mariadb -e MYSQL_ROOT_PASSWORD=root_password -e MYSQL_DATABASE=school_db -e MYSQL_USER=flask_user -e MYSQL_PASSWORD=flask_password mariadb
```

### 2. Iniciando a Aplicação Flask
```bash
docker build -t flask-app .
docker run -d --name flask-app --link mariadb:mariadb -p 5000:5000 flask-app
```

### 3. Iniciando o Prometheus
```bash
docker run -d --name prometheus -p 9090:9090 -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

### 4. Iniciando o Grafana
```bash
docker run -d --name grafana -p 3000:3000 -v /path/to/grafana-config:/etc/grafana grafana/grafana
```

# README - Testes e Validações

Este documento fornece um guia detalhado sobre como realizar testes e validações no projeto, abordando as principais etapas e melhores práticas para garantir a qualidade e a integridade do sistema.

## Estrutura do Projeto
Antes de iniciar os testes, é importante ter uma compreensão geral da estrutura do projeto:
- **Aplicativo Flask**: Responsável por fornecer a API e gerenciar a lógica de backend.
- **Prometheus**: Ferramenta de monitoramento e coleta de métricas.
- **Grafana**: Painel de visualização de dados para as métricas coletadas.
- **Banco de dados**: Utilizado para armazenar dados coletados e processados pelo aplicativo.

## Testes de Integração
### Testes no Flask
Para testar a API desenvolvida com Flask, utilizamos o `pytest`.

1. **Instalação do pytest**
   ```bash
   pip install pytest
   ```

2. **Estrutura dos Testes**
   Os testes estão organizados em um diretório chamado `tests/`, onde cada arquivo de teste corresponde a um módulo da aplicação.

3. **Executando os Testes**
   Para executar os testes, execute o seguinte comando na raiz do projeto:
   ```bash
   pytest
   ```

4. **Exemplo de Teste de Endpoint**
   ```python
   def test_status_code(client):
       response = client.get('/api/endpoint')
       assert response.status_code == 200
   ```

## Validações de Coleta de Métricas
### Verificação do Prometheus
Para garantir que o Prometheus está coletando métricas conforme esperado:
1. **Verifique se o Prometheus está rodando**
   ```bash
   docker-compose up -d prometheus
   ```
2. **Acesse o Prometheus**: Visite `http://localhost:9090` e execute consultas para verificar se as métricas estão sendo registradas.
3. **Consulta Exemplo**:
   ```prometheus
   rate(http_requests_total[5m])
   ```

### Testes de Dashboard no Grafana
1. **Verifique se o Grafana está rodando**
   ```bash
   docker-compose up -d grafana
   ```
2. **Acesse o Grafana**: Visite `http://localhost:3000` e verifique se os dashboards estão sendo atualizados com as métricas do Prometheus.
3. **Valide os Gráficos e Painéis**: Certifique-se de que os dados apresentados nos painéis estão corretos e condizem com as consultas do Prometheus.

## Melhorias e Validações Futuras
- **Testes automatizados de fluxo de trabalho**: Implementar testes de fluxo de trabalho para validar interações entre diferentes partes da aplicação.
- **Monitoramento de desempenho**: Adicionar testes de carga para verificar a escalabilidade da API.
- **Validação de Dados**: Criar scripts para verificar a integridade dos dados armazenados no banco de dados.

## Troubleshooting
### Problemas Comuns
- **Erro de Conexão com o Banco de Dados**: Verifique as credenciais e a URL de conexão no arquivo de configuração.
- **Métricas Não Exibidas no Grafana**: Certifique-se de que o Prometheus está configurado para expor as métricas e que o Grafana está apontando para a fonte correta.

## Conclusão
Seguindo este guia, você será capaz de realizar testes e validações completas, garantindo que a aplicação esteja funcionando como esperado e que as métricas estão sendo coletadas e apresentadas corretamente.

