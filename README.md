## BASADA PARA GITHUB ACTIONS

La pipeline tendra los siguientes chequeos; 
- SAST - SonarQube 
- DAST - OWASP ZAP 
- Secret Scanning 
- SBOM  
- SCA 
- Inegracion de alertas en Slack/Teams 

---

## Aplicacion web vulnerable y SAST 

Buscaremos la aplicacion web vulnerable basada en Javascript, en este caso usaremos 'juice-shop' de OWASP.

Buscamos integrar una herramienta que analice el codigo estatico, usaremos SonarQube. 

1. Creamos la cuenta de SonarQube
2. Vinculamos nuestro repositorio a SonaQube
3. Vamos a la informacion del repostiorio que queremos analizar
4. Copiamos las project y organization key
5. En la direccion ROOT del directorio creamos el archivo llamado 'sonar-project.properties'
6. Dentro de la carpeta colocamos lo siguiente
``` YML
sonar.projectKey=AQUI_VA_LA_PROJECT_KEY
sonar.organization=AQUI_VA_ORGNIZATION_KEY
```
7. Luego de crear esto, iremos a SonarQube y generarmos nuestro Token
8. Perfil - Configuraciones - Se curity - Creamos el Token y lo copiamos
9. En el repositorio vamos al Secret Mangament, creamos el secreto que se llame 'SONAR_TOKEN' y colocamos el Token creado anteriormente
10. Luego colocaremos en en Actions el job para que corra el workflow

``` YML

sonarqube:

name: SonarQube

runs-on: ubuntu-latest

steps:

- uses: actions/checkout@v4

with:

fetch-depth: 0

- name: SonarQube Scan

uses: SonarSource/sonarqube-scan-action@v4

env:

SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

```


----

## Agregaremos a la pipeline un Secret Scanning

Usamos gitleaks

``` YML
gitleaks:

name: Run Gitleaks

runs-on: ubuntu-latest

steps:

- name: Checkout code

uses: actions/checkout@v4

with:

fetch-depth: 0

- name: Run Gitleaks

uses: gitleaks/gitleaks-action@v2

with:

config-path: .gitleaks.toml

fail: true

report-format: json

report-path: gitleaks-report.json

  

- name: Upload Gitleaks report

uses: actions/upload-artifact@v4

with:

name: gitleaks-report

path: gitleaks-report.json
```

Buscamos en alguna parte del repositorio, parte de codigo y colocamos la siguiente variable, para ver que funciona la pipeline

export type AWS_SECRET_ACCESS_KEY = "AKIAEXAMPLE1234567890FAKE"

---

## Agregamos una pipeline que ejecute un escaneo SBOM


El crear esto nos ahorrara y facilitara auditar todo lo que tiene el codigo, librerias, dependencias. En otras palabras es hacer un inventario de lo que vendria a ser nuestro codigo, listando todo y tener mas facilidad al momento de ver lo que tiene. 

¿Para qué sirve un SBOM?
Seguridad → Identifica vulnerabilidades en dependencias. 
Cumplimiento → Ayuda a cumplir normativas (ej. NIST, ISO 27001).
Gestión de dependencias → Permite entender qué componentes usa tu software.
Transparencia → Facilita auditorías y revisiones de terceros.

``` YML

enerate-sbom:

name: Generate SBOM

runs-on: ubuntu-latest

steps:

- name: Checkout code

uses: actions/checkout@v4

  

- name: Install Syft

run: |

curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

syft --version # Verificar la instalación

  

- name: Generate SBOM (CycloneDX Format)

run: syft . -o cyclonedx-json > sbom.json

  

- name: Upload SBOM artifact

uses: actions/upload-artifact@v4

with:

name: sbom

path: sbom.json

retention-days: 7

```
----

## Agregaremos un Analisis DAST a la pipeline

En este caso usaremos OWASP ZAP

Tipos de análisi:s que realiza ZAP:
Escaneo Pasivo
Inspecciona el tráfico HTTP entre el cliente y el servidor sin modificar las solicitudes.
Detecta encabezados de seguridad faltantes, cookies inseguras, exposición de información, etc.
No genera carga en el servidor.

Escaneo Activo:
Envía solicitudes maliciosas para identificar vulnerabilidades explotables.
Detecta inyección SQL, XSS, SSRF, IDOR, CSRF, etc.
Puede generar carga en el servidor y afectar el rendimiento.

Ataques en Fuzzing:
Prueba entradas con datos aleatorios para detectar fallos de validación y seguridad.
Útil para encontrar fallos en formularios, API y autenticaciones.

Escaneo de API (REST y SOAP):
Analiz endpoints de API mediante OpenAPI, GraphQL, Postman, etc.
Detecta fallos en la autorización, validación de entradas y exposición de datos sensibles.

Análisis en entornos CI/CD:
Puede integrarse en GitHub Actions, Jenkins, GitLab CI, etc.
Automatiza pruebas de seguridad en cada despliegue.


En este ejemplo, corremos el dockerfile y luego ejecutamos el escaneo de vulnerabilidades en la aplicacion ya corriendo. 

``` YML
zap_scan:

runs-on: ubuntu-latest

name: Escanear pagina web

steps:

- name: Checkout code

uses: actions/checkout@v4

  

- name: Buildear y correr juice-shop

run: |

git clone https://github.com/juice-shop/juice-shop.git

cd juice-shop

docker build -t juice-shop .

docker run -d -p 3000:3000 --name juice-shop juice-shop

sleep 10 # Espera a que el servicio se inicie

curl -I http://localhost:3000 || (echo "Juice Shop no está corriendo" && exit 1)

  

- name: ZAP Scan

uses: zaproxy/action-baseline@v0.14.0

with:

docker_name: 'ghcr.io/zaproxy/zaproxy:stable'

target: 'http://localhost:3000'

rules_file_name: '.zap/rules.tsv'

cmd_options: '-a'

  

- name: Stop and Remove Container

if: always()

run: docker stop juice-shop && docker rm juice-shop

```

----
