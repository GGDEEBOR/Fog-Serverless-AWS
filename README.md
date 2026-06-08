# Sistema IoT–Fog–Cloud para Detección de Eventos Críticos  
Arquitectura híbrida con Fog Computing y Serverless en AWS

Este proyecto implementa un sistema IoT completo que integra dispositivos físicos, procesamiento en Fog y una arquitectura Cloud Serverless, con el objetivo de detectar eventos críticos de manera eficiente, reducir tráfico innecesario hacia la nube y demostrar buenas prácticas de diseño en arquitecturas distribuidas.

---

## 👥 Integrantes del equipo

- Edson Bryan Béjar Román  
- Katherine Nikole Béjar Román  

---

## 🧠 Introducción

En los últimos años, el crecimiento acelerado de los dispositivos IoT ha generado un escenario donde enormes volúmenes de datos son producidos continuamente por sensores distribuidos. Tradicionalmente, estos datos se envían directamente a la nube para su procesamiento; sin embargo, este enfoque introduce latencias innecesarias, incrementa costos operativos y satura tanto la red como los sistemas cloud.

Gran parte de los datos generados por sensores físicos corresponden a información repetitiva o ruido, lo cual provoca procesamiento innecesario que no aporta valor real. Esto evidencia la necesidad de replantear cómo y dónde se procesan los datos en sistemas IoT.

Frente a esta problemática, surge el enfoque de Fog Computing, donde el procesamiento se realiza cerca de la fuente de datos. Esto permite analizar información en tiempo real, filtrar lecturas irrelevantes y detectar únicamente eventos significativos antes de enviarlos a la nube.

En este contexto, el presente trabajo propone el diseño e implementación de un sistema IoT que combina una capa Fog con una arquitectura serverless en AWS, demostrando cómo una correcta separación de responsabilidades permite construir una solución resiliente, desacoplada y escalable.

---

## 🏗️ Desarrollo – Arquitectura del Sistema

La solución sigue un enfoque IoT–Fog–Cloud, compuesto por tres capas claramente diferenciadas.

### 1️⃣ Dispositivo IoT – Arduino + Sensor Ultrasónico

- Captura mediciones físicas reales (distancia)
- Envío periódico por comunicación serial
- Formato JSON
- Sin lógica compleja (productor simple de datos)

El Arduino actúa como un nodo IoT realista: bajo costo, recursos limitados y generación continua de datos.

---

### 2️⃣ Capa Fog – Next.js (Node.js)

La PC funciona como nodo Fog, encargándose de:

- Lectura del puerto serial
- Filtrado de ruido
- Modelo de estados (NORMAL, WARNING, CRÍTICO)
- Confirmación por lecturas consecutivas
- Generación de eventos semánticos
- Envío de eventos a la nube vía HTTP
- Streaming en tiempo real usando Server-Sent Events (SSE)

El Fog envía solo eventos confirmados, no lecturas crudas.

---

### 3️⃣ Cloud – Arquitectura Serverless en AWS

Componentes principales:

- API Gateway (HTTP API) como punto de entrada
- Lambda Ingest para validación y normalización
- Amazon SQS para desacoplo y amortiguación
- Dead Letter Queue (DLQ) para manejo de fallos
- Lambda Worker para lógica de negocio
- DynamoDB para persistencia de eventos
- SNS para notificaciones críticas
- CloudWatch para observabilidad

---

## ⚙️ Implementación

### Arduino (IoT)

El Arduino realiza la lectura del sensor HC-SR04, brinda feedback local con LEDs y buzzer, y envía datos en formato JSON cada segundo.  
Ejemplo de estructura del mensaje enviado por Serial:

    {
      "device_id": "SENSOR_01",
      "distance": 18
    }

El baudrate utilizado es 115200 y cada mensaje se envía en una línea independiente.

---

## 🚀 Comandos de Despliegue y Operación en AWS

### Desplegar la infraestructura con CloudFormation

    aws cloudformation deploy --template-file stack.yaml --stack-name sensor-detect --capabilities CAPABILITY_NAMED_IAM

---

### Obtener el endpoint del API Gateway

    aws cloudformation describe-stacks --stack-name sensor-detect --query "Stacks[0].Outputs"

---

### Agregar correo para alertas (SNS)

    aws sns subscribe --topic-arn arn:aws:sns:us-east-1:396608808300:sensor-detect-alerts --protocol email --notification-endpoint kbejarr@gmail.com

---

### Ver logs en tiempo real (CloudWatch)

    aws logs tail /aws/lambda/sensor-detect-ingest --follow
    aws logs tail /aws/lambda/sensor-detect-worker --follow

---

### Probar el endpoint del API Gateway (PowerShell)

    Invoke-RestMethod `
      -Uri "https://kv43buk7s0.execute-api.us-east-1.amazonaws.com/prod/event" `
      -Method POST `
      -Headers @{ "Content-Type" = "application/json" } `
      -Body (@{
        device_id = "SENSOR_01"
        timestamp = "2025-12-29T01:01:00Z"
        distance  = 17
        evento    = "EVENTO_RARO"
        nivel     = "CRITICO"
      } | ConvertTo-Json)

---

### Eliminar toda la infraestructura

    aws cloudformation delete-stack --stack-name sensor-detect

---

### Verificar el estado del delete

    aws cloudformation describe-stacks --stack-name sensor-detect

Si el stack fue eliminado correctamente, debe aparecer un mensaje similar a:

    An error occurred (ValidationError):
    Stack with id sensor-detect does not exist


## Documentación

📄 [Informe Final](Fog_Serverless.pdf)
