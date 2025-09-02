# Clasificación de Documentos con AI Services en AWS

Este documento guía paso a paso un flujo end-to-end que demuestra el rol del arquitecto como integrador de servicios ML, usando S3, Lambda, Textract, Comprehend, DynamoDB y API Gateway.

⸻

## Escenario

Clasificación automática de documentos (ejemplo: facturas y contratos) en una aplicación empresarial.

Flujo:
	
    1.	Usuario sube un PDF a S3.
	2.	Lambda invoca Textract para extraer texto.
	3.	Lambda invoca Comprehend para análisis de entidades y sentimiento.
	4.	Resultados almacenados en DynamoDB.
	5.	Exponer resultados mediante API Gateway.

⸻

## Prerrequisitos
	•	AWS CLI v2 instalado y autenticado.
	•	Python 3.10+.
	•	Permisos para crear IAM, S3, DynamoDB, Lambda y API Gateway.

⸻

## Crear Recursos Base

### 1. S3 Bucket

Creación de Bucket en Region de Norte Virginia:

```
REGION=us-east-1
BUCKET=nombre_bucket
aws s3 mb s3://$BUCKET --region $REGION
```

### 2. DynamoDB

Creación de Tabla de Dynamodb para guardar los resultados
```
aws dynamodb create-table \
  --table-name DocAnalysis \
  --attribute-definitions AttributeName=document_id,AttributeType=S \
  --key-schema AttributeName=document_id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```
### 3. Lambda

Crear una Funcion Lambda process_document con permisos basicos.

Modificar el Rol de ejecución agregando los permisos:
```
{
			"Effect": "Allow",
			"Action": [
				"s3:GetObject"
			],
			"Resource": "arn:aws:s3:::*/*"
		},
		{
			"Effect": "Allow",
			"Action": [
				"textract:StartDocumentTextDetection",
				"textract:GetDocumentTextDetection"
			],
			"Resource": "*"
		},
		{
			"Effect": "Allow",
			"Action": [
				"comprehend:DetectSentiment",
				"comprehend:DetectEntities",
				"comprehend:DetectDominantLanguage"
			],
			"Resource": "*"
		},
		{
			"Effect": "Allow",
			"Action": [
				"dynamodb:PutItem",
				"dynamodb:GetItem"
			],
			"Resource": "arn:aws:dynamodb:*:*:table/DocAnalysis"
		}
```

Código:
```
import os, boto3, time, urllib.parse

textract = boto3.client('textract')
comprehend = boto3.client('comprehend')
dynamodb = boto3.resource('dynamodb')
TABLE = dynamodb.Table(os.environ['TABLE_NAME'])

def lambda_handler(event, context):
    for rec in event.get('Records', []):
        bucket = rec['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(rec['s3']['object']['key'])
        doc_id = f"s3://{bucket}/{key}"

        # Textract
        resp = textract.start_document_text_detection(
            DocumentLocation={'S3Object': {'Bucket': bucket, 'Name': key}}
        )
        job_id = resp['JobId']
        status = 'IN_PROGRESS'
        while status == 'IN_PROGRESS':
            jr = textract.get_document_text_detection(JobId=job_id)
            status = jr['JobStatus']
            time.sleep(2)

        lines = [b['Text'] for b in jr.get('Blocks', []) if b['BlockType']=='LINE']
        text = "\n".join(lines)

        # Comprehend
        lang = comprehend.detect_dominant_language(Text=text)['Languages'][0]['LanguageCode']
        sentiment = comprehend.detect_sentiment(Text=text[:4500], LanguageCode=lang)
        entities = comprehend.detect_entities(Text=text[:4500], LanguageCode=lang)

        TABLE.put_item(Item={
            'document_id': doc_id,
            'sentiment': sentiment['Sentiment'],
            'entities': [e['Text'] for e in entities['Entities']],
            'ingested_at': int(time.time())
        })
```
### 4. Vincular S3 → Lambda

Crear disparador con el evento s3:ObjectCreated:* y sufijo .pdf

### 5. Probar Flujo

Sube un PDF al bucket y verifica la tabla de DynamoDB

### 6. API Gateway

Crear segunda Lambda (get_doc_result) que consulte DynamoDB.
Exponerla mediante API Gateway HTTP API en la ruta GET /doc/{doc_id}.

Agregar el siguiente permiso:
```
{
			"Effect": "Allow",
			"Action": [
				"dynamodb:PutItem",
				"dynamodb:GetItem"
			],
			"Resource": "arn:aws:dynamodb:*:*:table/DocAnalysis"
		}
```

Codigo:
```
import os, json, boto3, urllib.parse
from boto3.dynamodb.types import TypeDeserializer
from decimal import Decimal

dynamodb = boto3.client('dynamodb')
TABLE = os.environ['TABLE_NAME']

def _extract_doc_id(event):
    # Extrae la ruta de la key para DynamoDB
    raw_path = event.get("path") or event.get("rawPath") or ""
    prefix = "/doc/"
    if raw_path.startswith(prefix):
        return urllib.parse.unquote(raw_path[len(prefix):])
    return None

def _to_plain(o):
    """Convierte tipos Decimals a tipos JSON válidos."""
    if isinstance(o, Decimal):
        # si es entero, devuélvelo como int; si no, como float
        return int(o) if o % 1 == 0 else float(o)
    if isinstance(o, list):
        return [_to_plain(x) for x in o]
    if isinstance(o, dict):
        return {k: _to_plain(v) for k, v in o.items()}
    return o

def lambda_handler(event, context):
    """Función Lambda para obtener un documento de DynamoDB."""
    doc_id = _extract_doc_id(event)
    if not doc_id:
        return {
            "statusCode": 400,
            "headers": {"Content-Type": "text/plain"},
            "body": "Missing 'doc_id' (use /doc/{doc_id} o ?doc_id=...)"
        }

    try:
        res = dynamodb.get_item(
            TableName=TABLE,
            Key={'document_id': {'S': doc_id}}
        )
    except Exception as e:
        return {"statusCode": 500, "headers":{"Content-Type":"text/plain"}, "body": f"DynamoDB error: {str(e)}"}

    if 'Item' not in res:
        return {"statusCode": 404, "headers":{"Content-Type":"text/plain"}, "body": "Not found"}

    d = TypeDeserializer()
    item = {k: d.deserialize(v) for k, v in res['Item'].items()}

    # Conversión segura a tipos JSON
    item = _to_plain(item)

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(item)
    }
```

### 7. Probar API

En un navegador probar la URL:

https://{api-id}.execute-api.{region}.amazonaws.com/prod/doc/s3%3A%2F%2Fml-docs-demo-jdzn%2FAWS-Certified-SysOps-Administrator-Associate_Exam-Guide.pdf