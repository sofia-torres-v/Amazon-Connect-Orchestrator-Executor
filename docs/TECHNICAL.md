# Documentación Técnica - AppBarrio

← [Volver al README principal](../README.md)

Especificación técnica de los flujos, configuración, payloads y esquemas de base de datos del proyecto AppBarrio.

## 1. Prerrequisitos y configuración del entorno

Asegúrate de tener instaladas las siguientes herramientas en tu entorno local:

- **AWS CLI v2** configurado (`aws configure`)
- **AWS SAM CLI**
- **Python 3.13** (versión especificada en la infraestructura)
- **Docker** (necesario para pruebas locales con `sam local invoke`)

## 2. Guía de instalación y despliegue

### Paso 1: Clonar el repositorio

```bash
git clone https://github.com/sofia-torres-v/Amazon-Connect-Orchestrator-Executor.git
cd Amazon-Connect-Orchestrator-Executor
```

### Paso 2: Compilar el proyecto con AWS SAM

SAM lee el archivo `template.yaml`, descarga las dependencias de Python de cada función Lambda y prepara los paquetes para despliegue:

```bash
sam build
```

### Paso 3: Despliegue guiado a AWS

Para realizar el despliegue inicial interactivo (crea el stack de CloudFormation y configura el archivo `samconfig.toml`):

```bash
sam deploy --guided
```

> Durante el asistente, se solicitará el ARN de tu instancia de Amazon Connect para configurar los permisos de invocación.

### Qué crea SAM y qué configuras a mano

| Lo despliega `sam deploy` | Lo configuras por separado |
| :-- | :-- |
| Funciones Lambda (`check-balance`, `router`, `text-parser`) | Instancia de Amazon Connect |
| Tablas DynamoDB | Asistente y agente de Amazon Q in Connect |
| Roles y políticas IAM | Documentos en S3 para el RAG |
| Permisos de invocación para Connect y Lex | Bots de Lex V2 (`bot-reservas`, `bot-clasificador-q`) |
| | Widget de chat e interfaz web |

## 3. Pruebas locales con SAM CLI

> ⚠️ **Requisito:** [Docker](https://www.docker.com/) debe estar instalado y en ejecución para simular el entorno Lambda localmente.

Puedes probar las funciones Lambda localmente utilizando los eventos simulados ubicados en la carpeta `events/`:

**Consulta de saldo:**

```bash
sam local invoke CheckBalanceFunction -e events/check-balance-event.json
```

**Router de reservas:**

```bash
sam local invoke RouterFunction -e events/reserve-court-event.json
```

**Carga de créditos:**

```bash
sam local invoke RouterFunction -e events/load-credits-event.json
```

## 4. Seguridad y gestión de permisos (IAM)

El proyecto aplica el **Principio de Mínimo Privilegio (PoLP)** mediante políticas gestionadas de AWS SAM en `template.yaml`:

| Función | Política | Alcance |
| :-- | :-- | :-- |
| `RouterFunction` | `DynamoDBCrudPolicy` | Lectura y escritura sobre `sports-customers` y `sports-reservations` |
| `CheckBalanceFunction` | `DynamoDBReadPolicy` | Solo lectura sobre `sports-customers` |

**Permisos de conectividad (Resource-based Policies):**

Se configuran explícitamente permisos para que `connect.amazonaws.com` y `lexv2.amazonaws.com` puedan invocar las funciones Lambda de manera autorizada.

## 5. Esquema de base de datos (DynamoDB)

### `sports-customers`

| Atributo | Tipo | Rol |
| :-- | :-- | :-- |
| `customer_dni` | String | Partition Key |

### `sports-reservations`

| Atributo | Tipo | Rol |
| :-- | :-- | :-- |
| `reservation_id` | String | Partition Key |

**GSI — `CustomerIndex`:**

| Atributo | Tipo | Rol |
| :-- | :-- | :-- |
| `customer_dni` | String | HASH |
| `reservation_datetime` | String | RANGE |

## 6. API Reference (Payloads de integración)

### Entrada a Lambda `router` desde Lex V2

```json
{
  "sessionState": {
    "intent": {
      "name": "ReserveCourtIntent",
      "slots": {
        "sl_customer_dni": { "value": { "interpretedValue": "76543210" } },
        "slt_court_types": { "value": { "interpretedValue": "futbol" } },
        "sl_date": { "value": { "interpretedValue": "2026-11-01" } },
        "sl_time": { "value": { "interpretedValue": "19:00" } }
      }
    }
  },
  "invocationSource": "FulfillmentCodeHook"
}
```

### Respuesta para Lex V2 (Salida)

```json
{
  "sessionState": {
    "dialogAction": { "type": "Close" },
    "intent": { "state": "Fulfilled" }
  },
  "messages": [
    {
      "contentType": "PlainText",
      "content": "✅ ¡Reserva confirmada!\n\n📋 Código: RES-8F92A1\n🏟️ Cancha: Fútbol\n📅 Fecha: 01/11/2026\n🕐 Hora: 19:00"
    }
  ]
}
```

## 7. Configuración de los bots de Amazon Lex V2

### `bot-reservas` (ejecutor transaccional)

Es el bot tradicional del proyecto: tiene intents y slots, y delega el fulfillment en Lambda.

| Configuración | Valor |
| :-- | :-- |
| **Intención** | `ReserveCourtIntent` |
| **Slots requeridos** | `sl_customer_dni`, `slt_court_types`, `sl_date`, `sl_time` |
| **Fulfillment** | Lambda `sports-credits-router` |

### `bot-clasificador-q` (conector)

Bot que Amazon Connect requiere para integrar Amazon Q in Connect. Solo existe el intent que se genera al activar la integración; no se construyó lógica de intents en este bot. Toda la interpretación del mensaje ocurre en Amazon Q in Connect.

## 8. Flujo de la conversación

```text
Usuario → Amazon Connect → bot-clasificador-q (conector) → Amazon Q in Connect (orquestador)
                                                                   │
                                   ┌───────────────────────────────┼───────────────────────┐
                                   ▼                               ▼                       ▼
                             Respuesta RAG                    bot-reservas              Handoff
                             (FAQ, precios, S3)               (Lex V2, ejecutor)        (agente humano
                                                                   │                     + text-parser)
                                                             ┌─────┴─────┐
                                                             ▼           ▼
                                                       check-balance   router
                                                         (Lambda)     (Lambda)
                                                             │           │
                                                             ▼           ▼
                                                         DynamoDB    DynamoDB
```