devsecops-pipeline-demo
Repositorio de demostración para una pipeline CI/CD de Jenkins que realiza SAST, SCA, escaneo de imágenes, DAST y despliega en staging (docker-compose). Este repositorio incluye una aplicación Node.js deliberadamente vulnerable para ser usada en laboratorios.

Contenido
Jenkinsfile - pipeline declarativa

src/ - aplicación Node.js vulnerable

Dockerfile - construye la imagen de la app

docker-compose.yml - despliegue en staging

scripts/ - scripts auxiliares para ejecutar Semgrep, Dependency-Check, Trivy, ZAP

Requisitos previos
Servidor Jenkins (con agentes habilitados para Docker) o máquina con Docker.

Docker instalado y usuario con permiso para ejecutar comandos docker.

Ejecución rápida local
Construir y ejecutar la app localmente:

bash 
docker-compose up --build -d
Visitar http://localhost:3000

Ejecutar escaneo semgrep:
-------------------------------------------------------------------
bash
./scripts/run_semgrep.sh
-------------------------------------------------------------------
Ejecutar dependency-check:
bash ./scripts/run_dependency_check.sh
-------------------------------------------------------------------
Construir imagen:
bash
docker build -t devsecops-labs-app:local .
Ejecutar trivy:
./scripts/run_trivy.sh devsecops-labs-app:local
-------------------------------------------------------------------
Ejecutar escaneo ZAP:
bash
./scripts/run_zap.sh http://localhost:3000
-------------------------------------------------------------------
Notas
Este repositorio es intencionalmente inseguro — no usar en producción.

Para aplicar políticas de bloqueo, extienda el Jenkinsfile para analizar resultados y fallar las builds cuando se superen los umbrales.