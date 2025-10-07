## Azure end-to-end setup with az CLI (WebApp Python, App Insights 404 alert, Azure SQL, CSV hosting, Azure Boards)

This guide provisions and configures all required Azure resources via az CLI for the DisplayCSV-based WebApp.

### Prerequisites
- az CLI installed and logged in: `az login`
- Sufficient permissions on the target subscription
- Optional: mssql tools (`sqlcmd`, `bcp`) for SQL seed and CSV export
- Optional: Azure DevOps CLI extension (for Boards)

### Variables (edit before running)
```bash
SUBSCRIPTION_ID="<your-subscription-id>"
LOCATION="eastus"
RG="rg-displaycsv"
PLAN="plan-displaycsv-b1"
WEBAPP="app-displaycsv-$RANDOM"
RUNTIME="PYTHON:3.12"

# Application Insights
AI_NAME="ai-displaycsv"

# Storage for hosting CSV (static website)
STG="stgdisplaycsv$RANDOM"   # must be globally unique

# Azure SQL
SQL_SVR="sqldisplaycsv$RANDOM"
SQL_ADMIN="sqladminuser"
SQL_PASS="ChangeMe_12345!"   # follow Azure SQL password rules
SQL_DB="dbdisplaycsv"

# Azure Boards (Azure DevOps)
ADO_ORG="https://dev.azure.com/<your-org>"
ADO_PROJECT="DisplayCSV"
```

### Setup defaults
```bash
az account set --subscription "$SUBSCRIPTION_ID"
az configure --defaults location=$LOCATION group=$RG
```

### Resource Group
```bash
az group create -n $RG -l $LOCATION
```

### App Service Plan (Linux B1) and WebApp (Python 3.12)
```bash
az appservice plan create -n $PLAN --sku B1 --is-linux
az webapp create -n $WEBAPP -g $RG -p $PLAN --runtime $RUNTIME

# Optional: enable build during deploy for Oryx
az webapp config appsettings set -g $RG -n $WEBAPP --settings SCM_DO_BUILD_DURING_DEPLOYMENT=true
```

### Application Insights and linkage to WebApp
```bash
# Ensure extension (safe if already present)
az extension add --name application-insights --only-show-errors || true

# Create App Insights (Workspace-based by default)
az monitor app-insights component create \
  --app $AI_NAME \
  --location $LOCATION \
  -g $RG

# Connect WebApp to App Insights (sets APPLICATIONINSIGHTS_CONNECTION_STRING)
az monitor app-insights component connect-webapp \
  --app $AI_NAME \
  -g $RG \
  --web-app $WEBAPP
```

### Configure CSV_URL for the WebApp (placeholder for now)
```bash
CSV_URL="https://raw.githubusercontent.com/<user>/<repo>/main/data/export.csv"
az webapp config appsettings set -g $RG -n $WEBAPP --settings CSV_URL="$CSV_URL"
```

### Create Action Group for email notifications
```bash
AG_NAME="ag-displaycsv-email"
EMAIL="you@example.com"   # change to your email

az monitor action-group create \
  -g $RG \
  -n $AG_NAME \
  --short-name displaycsv \
  --action email emailAction $EMAIL

AG_ID=$(az monitor action-group show -g $RG -n $AG_NAME --query id -o tsv)
```

### 404 Alert (Log alert on Application Insights requests where resultCode == 404)
```bash
AI_ID=$(az monitor app-insights component show -g $RG -a $AI_NAME --query id -o tsv)

ALERT_NAME="alert-404"
QUERY="requests | where toint(resultCode) == 404"

az monitor scheduled-query create \
  --name $ALERT_NAME \
  --resource-group $RG \
  --scopes $AI_ID \
  --description "HTTP 404 >= 1 in 5m" \
  --severity 2 \
  --evaluation-frequency 5m \
  --window-size 5m \
  --condition "count '$QUERY' > 0" \
  --action-groups $AG_ID
```

### Simulate a 404 to trigger the alert
```bash
echo "Triggering 404..."
curl -sSf https://$WEBAPP.azurewebsites.net/does-not-exist || true
```

### Azure SQL Server and Database
```bash
# Create logical server
az sql server create \
  -n $SQL_SVR \
  -g $RG \
  -l $LOCATION \
  -u $SQL_ADMIN \
  -p "$SQL_PASS"

# Allow your client IP (or a safe IP range)
MY_IP=$(curl -s https://api.ipify.org)
az sql server firewall-rule create -g $RG -s $SQL_SVR -n AllowMyIP --start-ip-address $MY_IP --end-ip-address $MY_IP

# Create database (General Purpose S0 default)
az sql db create -g $RG -s $SQL_SVR -n $SQL_DB --service-objective S0

SQL_FQDN=$(az sql server show -g $RG -n $SQL_SVR --query fullyQualifiedDomainName -o tsv)
echo "SQL FQDN: $SQL_FQDN"
```

### Seed schema and data (Gestão de Cursos example) using sqlcmd
Save the script below as `sql/schema_seed.sql` locally, then run:
```bash
sqlcmd -S $SQL_FQDN -d $SQL_DB -U $SQL_ADMIN -P "$SQL_PASS" -i sql/schema_seed.sql
```

Sample schema/data:
```sql
CREATE TABLE instrutores (
  instrutor_id INT IDENTITY(1,1) PRIMARY KEY,
  nome NVARCHAR(100) NOT NULL,
  email NVARCHAR(200) NOT NULL UNIQUE
);

CREATE TABLE cursos (
  curso_id INT IDENTITY(1,1) PRIMARY KEY,
  titulo NVARCHAR(200) NOT NULL,
  categoria NVARCHAR(100) NOT NULL,
  carga_horaria INT NOT NULL,
  instrutor_id INT NOT NULL REFERENCES instrutores(instrutor_id)
);

CREATE TABLE inscricoes (
  inscricao_id INT IDENTITY(1,1) PRIMARY KEY,
  curso_id INT NOT NULL REFERENCES cursos(curso_id),
  aluno_nome NVARCHAR(150) NOT NULL,
  aluno_email NVARCHAR(200) NOT NULL,
  data_inscricao DATE NOT NULL DEFAULT GETDATE()
);

INSERT INTO instrutores (nome, email) VALUES
('Ana Souza','ana.souza@example.com'),
('Bruno Lima','bruno.lima@example.com');

INSERT INTO cursos (titulo, categoria, carga_horaria, instrutor_id) VALUES
('Python para Dados','Programação',40,1),
('SQL Essencial','Banco de Dados',24,2),
('Visualização com Power BI','BI',32,2);

INSERT INTO inscricoes (curso_id, aluno_nome, aluno_email, data_inscricao) VALUES
(1,'Carla Mendes','carla@example.com','2025-09-01'),
(1,'Diego Alves','diego@example.com','2025-09-03'),
(2,'Eduardo Reis','edu@example.com','2025-09-05'),
(2,'Fernanda Dias','fer@example.com','2025-09-07'),
(3,'Gustavo Nogueira','gus@example.com','2025-09-09'),
(3,'Helena Prado','helena@example.com','2025-09-11');

-- View for export with 5+ columns
CREATE OR ALTER VIEW vw_inscricoes_export AS
SELECT i.inscricao_id, c.titulo AS curso, c.categoria, c.carga_horaria,
       ins.nome AS instrutor, i.aluno_nome, i.aluno_email, i.data_inscricao
FROM inscricoes i
JOIN cursos c ON c.curso_id = i.curso_id
JOIN instrutores ins ON ins.instrutor_id = c.instrutor_id;
```

### Export CSV from Azure SQL (bcp) and host in Storage Static Website
```bash
# Create Storage Account (with static website)
az storage account create -n $STG -g $RG -l $LOCATION --sku Standard_LRS
az storage blob service-properties update --account-name $STG --static-website --index-document index.html

STG_CONN=$(az storage account show-connection-string -g $RG -n $STG --query connectionString -o tsv)
WEB_CONTAINER="\$web"  # literal $web container for static site

# Export CSV locally via bcp (ensure bcp installed)
CSV_LOCAL="export.csv"
bcp "$SQL_DB.dbo.vw_inscricoes_export" out "$CSV_LOCAL" -S $SQL_FQDN -U $SQL_ADMIN -P "$SQL_PASS" -c -t"," -r"\n" -q

# Upload CSV to static website container
az storage blob upload \
  --connection-string "$STG_CONN" \
  --container-name $WEB_CONTAINER \
  --name export.csv \
  --file "$CSV_LOCAL" \
  --overwrite true

CSV_URL=$(az storage account show -n $STG -g $RG --query "primaryEndpoints.web" -o tsv)export.csv
echo "CSV URL: $CSV_URL"

# Update WebApp setting to consume this CSV
az webapp config appsettings set -g $RG -n $WEBAPP --settings CSV_URL="$CSV_URL"
```

### Deploy the WebApp code
Choose one option:

Option A) Quick deploy from current folder
```bash
az webapp up -n $WEBAPP -g $RG -p $PLAN --runtime $RUNTIME --sku B1
```

Option B) Zip deploy (from a built artifact)
```bash
zip -r deploy.zip .
az webapp deployment source config-zip -g $RG -n $WEBAPP --src deploy.zip
```

### Verify WebApp and CSV rendering
```bash
az webapp browse -g $RG -n $WEBAPP
```

### Azure Boards via Azure DevOps CLI
```bash
# Install extension (safe if already present)
az extension add --name azure-devops --only-show-errors || true

# Set defaults
az devops configure --defaults organization=$ADO_ORG project=$ADO_PROJECT

# Create project (if it doesn't exist). Process can be Agile or Scrum
az devops project create --name $ADO_PROJECT --process Agile --visibility private || true

# Create Epic and Feature
EPIC_ID=$(az boards work-item create --type Epic --title "Observabilidade e Dados" --query id -o tsv)
FEAT_ID=$(az boards work-item create --type Feature --title "WebApp DisplayCSV e Monitoramento" --relation-urls \
  $(az boards work-item show --id $EPIC_ID --query _links.html.href -o tsv) --relation-types Parent --query id -o tsv)

# Create 3 PBIs/User Stories and link to Feature
US1_ID=$(az boards work-item create --type "User Story" --title "Banco & CSV" --query id -o tsv)
US2_ID=$(az boards work-item create --type "User Story" --title "WebApp & CSV por URL" --query id -o tsv)
US3_ID=$(az boards work-item create --type "User Story" --title "Observabilidade (404)" --query id -o tsv)

for ID in $US1_ID $US2_ID $US3_ID; do
  az boards work-item relation add --id $ID --relation-type Parent --target-id $FEAT_ID
done

# Add sample acceptance criteria and tasks
az boards work-item update --id $US1_ID --fields "Acceptance Criteria=Schema com PK/FK; SELECT; CSV 5+ colunas/10+ linhas; URL pública válida."
az boards work-item update --id $US2_ID --fields "Acceptance Criteria=CSV_URL configurada; app exibe dados; erro tratado URL inválida."
az boards work-item update --id $US3_ID --fields "Acceptance Criteria=AI ativo; regra 404 ≥ 1/5min; e-mail recebido ao simular 404."

az boards work-item create --type Task --title "Modelar tabelas e chaves" --relation-ids $US1_ID --relation-types Parent
az boards work-item create --type Task --title "Inserir dados de exemplo" --relation-ids $US1_ID --relation-types Parent
az boards work-item create --type Task --title "Exportar e publicar CSV" --relation-ids $US1_ID --relation-types Parent

az boards work-item create --type Task --title "Configurar CSV_URL e deploy" --relation-ids $US2_ID --relation-types Parent
az boards work-item create --type Task --title "Print da tela do app" --relation-ids $US2_ID --relation-types Parent

az boards work-item create --type Task --title "Habilitar AI e criar alerta 404" --relation-ids $US3_ID --relation-types Parent
az boards work-item create --type Task --title "Simular 404 e evidências" --relation-ids $US3_ID --relation-types Parent
```

### Clean up (optional)
```bash
az group delete -n $RG --yes --no-wait
```

### Notes
- If GitHub Actions deployment is preferred, create the workflow in your repo and set publish profile or OIDC credentials. The WebApp resource is already ready to receive deployments.
- The alert may take a few minutes to activate after creation; ensure at least one 404 occurs during the evaluation window.
- For CSV hosting, GitHub raw URLs also work; update `CSV_URL` accordingly if you choose that route.

