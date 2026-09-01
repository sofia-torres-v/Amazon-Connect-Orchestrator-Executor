# 🏟️ Sports Reservations Serverless Chatbot - Backend

Backend serverless en AWS para gestionar créditos y reservas deportivas mediante conversación natural. Implementa AWS Lambda en Python, DynamoDB como base de datos y se integra con Amazon Lex y Amazon Connect para el flujo conversacional. Desplegado con AWS SAM y totalmente sin servidores.

[![AWS](https://img.shields.io/badge/AWS-Serverless-orange?logo=amazon-aws)](https://aws.amazon.com/)
[![Python](https://img.shields.io/badge/Python-3.9-blue?logo=python)](https://www.python.org/)
[![SAM](https://img.shields.io/badge/SAM-CLI-yellow)](https://aws.amazon.com/serverless/sam/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## Tabla de Contenidos

- [Características](#-características)
- [Arquitectura](#-arquitectura)
- [Tecnologías](#-tecnologías)
- [Prerequisitos](#-prerequisitos)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Instalación Local](#-instalación-local)
- [Deployment](#-deployment)
- [Testing](#-testing)
- [Configuración de Servicios AWS](#-configuración-de-servicios-aws)
- [Variables de Entorno](#-variables-de-entorno)
- [API Reference](#-api-reference)
- [Licencia](#-licencia)

---

## Características

### Funcionalidades Principales

- ✅ **Consulta de Saldo**: Verificar créditos disponibles del cliente
- ✅ **Carga de Créditos**: Sistema de recarga con múltiples métodos de pago
- ✅ **Reserva de Canchas**: Gestión de reservas para fútbol y voley
- ✅ **Validaciones Inteligentes**: 
  - Verificación de fechas futuras con zona horaria correcta
  - Validación de créditos suficientes
  - Pre-llenado automático de información
- ✅ **Base de Conocimientos**: Respuestas automáticas con Amazon Q
- ✅ **Escalamiento a Agentes**: Transferencia fluida a soporte humano

### Características Técnicas

- **100% Serverless**: Sin servidores que mantener
- **Auto-escalable**: De 0 a millones de requests
- **Pago por uso**: Solo pagas lo que usas
- **Seguro**: IAM roles con mínimo privilegio
- **Logs Completos**: CloudWatch para debugging
- **Zona Horaria**: Manejo correcto de Buenos Aires (UTC-3)

---

## Arquitectura

### Diagrama de Alto Nivel

```
┌─────────────────┐
│  Amazon Connect │ ← Frontend (Chat/Voz)
│  Contact Flow   │
└────────┬────────┘
         │
┌────────▼────────┐
│   Lex Bots      │ ← Conversational AI
│  • Clasificador │
│  • Router       │
└────────┬────────┘
         │
┌────────▼────────────────────────┐
│  AWS Lambda (Python 3.9)        │ ← Backend 
│  ├─ check-balance               │
│  ├─ router                      │
│  │  ├─ reserve_court.py         │
│  │  └─ load_credits.py          │
│  └─ text-parser                 │
└────────┬────────────────────────┘
         │
┌────────▼────────┐
│   DynamoDB      │ ← Database
│  • Customers    │
│  • Reservations │
└─────────────────┘
```

### Flujo de una Reserva

```
Usuario: "Quiero reservar cancha de futbol"
    ↓
Lex Bot Clasificador → Identifica intención
    ↓
Connect → Enruta a Bot Router
    ↓
Bot Router → Pide datos (DNI, fecha, hora)
    ↓
Lambda Router → reserve_court.py
    ├─ Valida fecha futura ✓
    ├─ Verifica cliente existe ✓
    ├─ Valida créditos suficientes ✓
    ├─ Crea reserva en DynamoDB
    └─ Descuenta créditos
    ↓
Usuario: "✅ Reserva confirmada! Código: RES-ABC123"
```
---

## Tecnologías

### AWS Services

| Servicio | Uso | Capa |
|----------|-----|------|
| **AWS Lambda** | Funciones serverless (Python 3.9) | Compute |
| **Amazon DynamoDB** | Base de datos NoSQL | Data |
| **AWS SAM** | Infraestructura como código | IaC |
| **CloudWatch** | Logs y monitoreo | Observability |
| **IAM** | Roles y permisos | Security |

### Integraciones

| Servicio | Tipo | Propósito |
|----------|------|-----------|
| **Amazon Lex** | Middleware | NLU y gestión conversacional |
| **Amazon Connect** | Middleware | Enrutamiento y contact center |
| **Amazon Q** | Middleware | Knowledge base con RAG |

### Librerías Python

```txt
boto3>=1.26.0      # SDK de AWS
pytz>=2023.3       # Manejo de zonas horarias
```

---

## Prerequisitos

### Software Requerido

- [AWS CLI](https://aws.amazon.com/cli/) v2.x
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) v1.x
- [Python](https://www.python.org/downloads/) 3.9+
- [Git](https://git-scm.com/)

### Cuenta AWS

Necesitas una cuenta de AWS con permisos para:
- ✅ AWS Lambda
- ✅ Amazon DynamoDB
- ✅ IAM (crear roles)
- ✅ CloudFormation
- ✅ CloudWatch Logs

### Configurar AWS CLI

```bash
aws configure
# AWS Access Key ID: TU_ACCESS_KEY
# AWS Secret Access Key: TU_SECRET_KEY
# Default region: us-east-1
# Default output format: json
```

---

## Estructura del Proyecto

```
sports-reservations-serverless-chatbot-aws/
│
├── functions/                      
│   │
│   ├── check-balance/              
│   │   ├── index.py
│   │   └── requirements.txt
│   │
│   ├── router/                    
│   │   ├── handlers/
│   │   │   ├── __init__.py
│   │   │   ├── load_credits.py    
│   │   │   └── reserve_court.py  
│   │   ├── index.py                
│   │   ├── utils.py                
│   │   └── requirements.txt
│   │
│   └── text-parser/            
│       ├── index.py
│       └── requirements.txt
│
├── events/                         
│   ├── check-balance-event.json
│   ├── reserve-event.json
│   └── load-credits-event.json
│
├── docs/                         
│   ├── architecture.md
│   ├── api-reference.md
│   └── deployment-guide.md
│
├── .gitignore                   
├── template.yaml                
├── samconfig.toml                  
├── README.md                     
└── LICENSE                         
```

---

## Instalación Local

### 1. Clonar el Repositorio

```bash
git clone https://github.com/tu-usuario/sports-reservations-serverless-chatbot-aws.git
cd sports-reservations-serverless-chatbot-aws
```

### 2. Instalar Dependencias (Opcional - para desarrollo local)

```bash
# Para cada Lambda
cd functions/router
pip install -r requirements.txt -t .
cd ../..

cd functions/check-balance
pip install -r requirements.txt -t .
cd ../..

cd functions/text-parser
pip install -r requirements.txt -t .
cd ../..
```

> **Nota**: SAM hace esto automáticamente durante `sam build`, pero es útil para desarrollo local.

### 3. Verificar SAM

```bash
sam --version
# Debe mostrar: SAM CLI, version 1.x.x
```

---

## Deployment

### Deployment Completo (Primera Vez)

```bash
# 1. Construir el proyecto
sam build

# 2. Desplegar con configuración guiada
sam deploy --guided

# Responde las preguntas:
# Stack Name: sports-reservations-backend
# AWS Region: us-east-1
# Confirm changes before deploy: Y
# Allow SAM CLI IAM role creation: Y
# Disable rollback: N
# Save arguments to configuration file: Y
# SAM configuration file: samconfig.toml
```

### Despliegues Subsecuentes

```bash
# Más rápido - usa configuración guardada
sam build && sam deploy
```

### Deployment a Diferentes Entornos

```bash
# Desarrollo
sam deploy --config-env dev

# Producción
sam deploy --config-env prod
```

### Verificar Deployment

```bash
# Listar stacks
aws cloudformation list-stacks

# Ver recursos creados
aws cloudformation describe-stack-resources \
  --stack-name sports-reservations-backend
```

---

## Testing

### Testing Local con SAM

```bash
# Probar Lambda específica
sam local invoke CheckBalanceFunction \
  -e events/check-balance-event.json

# Probar Router con reserva
sam local invoke RouterFunction \
  -e events/reserve-event.json

# Probar Router con carga de créditos
sam local invoke RouterFunction \
  -e events/load-credits-event.json
```

### Ver Logs en Tiempo Real

```bash
# Logs de una Lambda específica
sam logs -n CheckBalanceFunction --tail

# Logs del Router
sam logs -n RouterFunction --tail

# Buscar errores
aws logs filter-log-events \
  --log-group-name /aws/lambda/sports-bot-router \
  --filter-pattern "ERROR"
```

### Testing en AWS (después de deploy)

```bash
# Invocar Lambda directamente
aws lambda invoke \
  --function-name sports-credits-check-balance \
  --payload '{"customer_dni": "12345678"}' \
  response.json

cat response.json
```

---

## Configuración de Servicios AWS

### Amazon Lex - Bot Router

**Este bot maneja las transacciones (reservas y cargas)**

```yaml
Bot: bot-reservas
Language: Spanish (ES)
Alias: PROD

Intents:
  ReserveCourtIntent:
    Utterances:
      - "Quiero reservar una cancha"
      - "Reservar cancha de {court_type}"
    Slots:
      - sl_customer_dni (AMAZON.Number)
      - slt_court_types (Custom: futbol, voley)
      - sl_date (AMAZON.Date)
      - sl_time (AMAZON.Time)
      - sl_confirmation (AMAZON.Confirmation)
    Fulfillment: sports-bot-router

  LoadCreditsIntent:
    Utterances:
      - "Quiero cargar créditos"
      - "Cargar {amount} créditos"
    Slots:
      - sl_customer_dni (AMAZON.Number)
      - sl_amount (AMAZON.Number)
      - slt_payment_methods (Custom: efectivo, tarjeta)
      - sl_confirmation (AMAZON.Confirmation)
    Fulfillment: sports-bot-router
```

**Dar permisos a Lex:**

```bash
aws lambda add-permission \
  --function-name sports-bot-router \
  --statement-id AllowLexInvoke \
  --action lambda:InvokeFunction \
  --principal lexv2.amazonaws.com \
  --source-arn "arn:aws:lex:REGION:ACCOUNT:bot-alias/BOT_ID/ALIAS_ID"
```

### Amazon Lex - Bot Clasificador

**Este bot decide qué hacer con cada mensaje del usuario**

```yaml
Bot: bot-clasificador-q
Language: Spanish (ES)
Alias: demo

Settings:
  Amazon Q Integration: Enabled
  Assistant: sports-assistant
  Knowledge Base: sports-knowledge

System Prompt: (Ver docs/system-prompt.txt)
```

### Amazon Connect

**Importar Contact Flow:**

```bash
# 1. En Connect Console → Contact Flows → Create
# 2. Import flow (JSON)
# 3. Configurar bloques:
#    - Bot Clasificador: Asociar bot-clasificador-q
#    - Bot Router: Asociar bot-reservas
#    - Lambdas: Usar ARNs del output de SAM
# 4. Publish
```

**Dar permisos a Connect:**

```bash
# Para cada Lambda
aws lambda add-permission \
  --function-name sports-credits-check-balance \
  --statement-id AllowConnectInvoke \
  --action lambda:InvokeFunction \
  --principal connect.amazonaws.com \
  --source-arn "arn:aws:connect:REGION:ACCOUNT:instance/INSTANCE_ID"
```

---

## Variables de Entorno

Las Lambdas reciben estas variables automáticamente desde `template.yaml`:

| Variable | Lambda | Valor | Descripción |
|----------|--------|-------|-------------|
| `CUSTOMERS_TABLE` | check-balance, router | `sports-customers` | Tabla de clientes |
| `RESERVATIONS_TABLE` | router | `sports-reservations` | Tabla de reservas |
| `LOG_LEVEL` | Todas | `INFO` | Nivel de logs |

**Configuradas en `template.yaml`:**

```yaml
Environment:
  Variables:
    CUSTOMERS_TABLE: !Ref CustomersTable
    RESERVATIONS_TABLE: !Ref ReservationsTable
    LOG_LEVEL: INFO
```

---

## API Reference

### Lambda: check-balance

**Entrada:**
```json
{
  "customer_dni": "12345678"
}
```

**Salida:**
```json
{
  "statusCode": 200,
  "message": "Tienes 150 créditos disponibles."
}
```

---

### Lambda: router (ReserveCourtIntent)

**Entrada (de Lex):**
```json
{
  "sessionState": {
    "intent": {
      "name": "ReserveCourtIntent",
      "slots": {
        "sl_customer_dni": {"value": {"interpretedValue": "12345678"}},
        "slt_court_types": {"value": {"interpretedValue": "futbol"}},
        "sl_date": {"value": {"interpretedValue": "2025-11-30"}},
        "sl_time": {"value": {"interpretedValue": "18:00"}},
        "sl_confirmation": {"value": {"interpretedValue": "si"}}
      }
    }
  },
  "invocationSource": "FulfillmentCodeHook"
}
```

**Salida (para Lex):**
```json
{
  "sessionState": {
    "dialogAction": {"type": "Close"},
    "intent": {"state": "Fulfilled"}
  },
  "messages": [{
    "contentType": "PlainText",
    "content": "✅ ¡Reserva confirmada!\n\n📋 Código: RES-ABC12345\n🏟️ Cancha: Futbol\n📅 Fecha: 30/11/2025\n🕐 Hora: 18:00\n💰 Costo: 50 créditos\n\nNuevo saldo: 100 créditos"
  }]
}
```

---

### Lambda: router (LoadCreditsIntent)

**Entrada (de Lex):**
```json
{
  "sessionState": {
    "intent": {
      "name": "LoadCreditsIntent",
      "slots": {
        "sl_customer_dni": {"value": {"interpretedValue": "12345678"}},
        "sl_amount": {"value": {"interpretedValue": "100"}},
        "slt_payment_methods": {"value": {"interpretedValue": "efectivo"}},
        "sl_confirmation": {"value": {"interpretedValue": "si"}}
      }
    }
  },
  "invocationSource": "FulfillmentCodeHook"
}
```

**Salida (para Lex):**
```json
{
  "sessionState": {
    "dialogAction": {"type": "Close"},
    "intent": {"state": "Fulfilled"}
  },
  "messages": [{
    "contentType": "PlainText",
    "content": "✅ ¡Carga exitosa!\n\n💰 Créditos agregados: 100\n📊 Saldo anterior: 50\n📈 Nuevo saldo: 150 créditos\n💳 Método de pago: efectivo"
  }]
}
```

---

### Lambda: text-parser

**Entrada:**
```json
{
  "qicSummaryIn": "<SummaryItems><Item>Cliente pidió reserva</Item><Item>Reserva cancelada</Item></SummaryItems>"
}
```

**Salida:**
```json
{
  "qicSummaryOut": "Resumen de conversación:\n- Cliente pidió reserva\n- Reserva cancelada"
}
```

---

## DynamoDB Schema

### Tabla: sports-customers

```json
{
  "customer_dni": "12345678",        
  "credits": 150,                  
  "created_at": "2025-11-22T10:00:00-03:00",  
  "last_load": "2025-11-22T15:30:00-03:00"    
}
```

### Tabla: sports-reservations

```json
{
  "reservation_id": "RES-ABC12345",  
  "customer_dni": "12345678",        
  "court_type": "futbol",            
  "reservation_date": "2025-11-30", 
  "reservation_time": "18:00",      
  "reservation_datetime": "2025-11-30 18:00",  
  "cost": 50,                        
  "status": "confirmed",            
  "created_at": "2025-11-22T16:00:00-03:00" 
}
```

---

## Licencia

Este proyecto está bajo la Licencia MIT - ver el archivo [LICENSE](LICENSE) para más detalles.

---

## Autor

**Sofia U. Torres**

- [GitHub](https://github.com/sofia-torres-v)

---
