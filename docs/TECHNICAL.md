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
git clone https://github.com/sofia-torres-v/appbarrio-aws-serverless.git
cd appbarrio-aws-serverless
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

## 3. Pruebas locales con SAM CLI

> ⚠️ **Requisito:** [Docker](https://www.docker.com/) debe estar instalado y en ejecución para simular el entorno Lambda localmente.

Puedes probar las funciones Lambda localmente utilizando los eventos simulados ubicados en la carpeta `events/`:

**Consulta de saldo/disponibilidad:**

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

El proyecto implementa el **Principio de Mínimo Privilegio (PoLP)** mediante políticas gestionadas nativas de AWS SAM en el archivo `template.yaml`:

| Función                | Política             | Alcance                                                              |
| :--------------------- | :------------------- | :------------------------------------------------------------------- |
| `RouterFunction`       | `DynamoDBCrudPolicy` | Lectura y escritura sobre `sports-customers` y `sports-reservations` |
| `CheckBalanceFunction` | `DynamoDBReadPolicy` | Solo lectura sobre `sports-customers`                                |

**Permisos de conectividad (Resource-based Policies):**

Se configuran explícitamente permisos para que `connect.amazonaws.com` y `lexv2.amazonaws.com` puedan invocar las funciones Lambda de manera autorizada.

## 5. Esquema de base de datos (DynamoDB)

### `sports-customers`

| Atributo       | Tipo   | Rol           |
| :------------- | :----- | :------------ |
| `customer_dni` | String | Partition Key |

### `sports-reservations`

| Atributo         | Tipo   | Rol           |
| :--------------- | :----- | :------------ |
| `reservation_id` | String | Partition Key |

**GSI — `CustomerIndex`:**

| Atributo               | Tipo   | Rol   |
| :--------------------- | :----- | :---- |
| `customer_dni`         | String | HASH  |
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

## 7. Configuración de Amazon Lex V2 (`bot-reservas`)

| Configuración        | Valor                                                      |
| :------------------- | :--------------------------------------------------------- |
| **Intención**        | `ReserveCourtIntent`                                       |
| **Slots requeridos** | `sl_customer_dni`, `slt_court_types`, `sl_date`, `sl_time` |
| **Fulfillment**      | Lambda `sports-credits-router`                             |

## 8. Flujo de la conversación

```text
Usuario → Amazon Connect → bot-clasificador-q (Amazon Q in Connect)
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Respuesta RAG   bot-reservas      Handoff
              (FAQ, precios)  (Lex V2)          (Agente humano)
                                    │
                              ┌─────┼─────┐
                              ▼           ▼
                        check-balance   router
                        (Lambda)        (Lambda)
                              │           │
                              ▼           ▼
                          DynamoDB    DynamoDB
```
