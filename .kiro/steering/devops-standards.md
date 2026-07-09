# Estándares DevOps del Equipo

Este archivo define los estándares y convenciones que todos los pipelines y workflows del equipo deben seguir.

## Plataforma de CI/CD

- Todos los pipelines deben implementarse usando **GitHub Actions**
- No se permite el uso de otras plataformas de CI/CD (Jenkins, CircleCI, GitLab CI, etc.) sin aprobación explícita del equipo

## Nomenclatura de Workflows

- Los workflows deben tener **nombres descriptivos en español**
- El campo `name` del workflow debe ser claro y reflejar su propósito
- Ejemplos válidos:
  - `Integración Continua - Validación y Pruebas`
  - `Despliegue a Producción`
  - `Verificación de Calidad de Código`
- Evitar nombres genéricos como `CI`, `Build`, `Test`

## Steps Obligatorios

Todo workflow de integración continua debe incluir los siguientes pasos en este orden:

1. **Lint** — Verificación de estilo y calidad de código
2. **Test** — Ejecución de pruebas unitarias e integración
3. **Build** — Compilación y empaquetado del artefacto

Ejemplo de estructura mínima:

```yaml
jobs:
  lint:
    name: Verificación de Estilo
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  test:
    name: Pruebas Automatizadas
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  build:
    name: Compilación
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

## Versión de Node.js

- Usar **Node.js 20** como versión por defecto en todos los workflows
- Siempre declarar la versión explícitamente usando la action `actions/setup-node`

```yaml
- name: Configurar Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
```

- No usar `node-version: 'latest'` ni rangos como `20.x` — fijar la versión mayor

## Gestión de Secrets

- Todos los secrets deben referenciarse con el prefijo **`CARVAJAL_`**
- Formato: `${{ secrets.CARVAJAL_<NOMBRE_DEL_SECRET> }}`
- Ejemplos:
  - `${{ secrets.CARVAJAL_NPM_TOKEN }}`
  - `${{ secrets.CARVAJAL_AWS_ACCESS_KEY_ID }}`
  - `${{ secrets.CARVAJAL_DATABASE_URL }}`
- No hardcodear valores sensibles en los archivos de workflow
- No usar secrets sin el prefijo `CARVAJAL_` (salvo los secrets nativos de GitHub como `GITHUB_TOKEN`)

## Notificaciones de Fallo en Slack

Todo workflow debe incluir un step de notificación a Slack que se ejecute **únicamente en caso de fallo**. Usar `if: failure()` para condicionar el paso.

```yaml
- name: Notificar fallo en Slack
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "❌ *Fallo en el pipeline*: `${{ github.workflow }}`\n*Repositorio*: ${{ github.repository }}\n*Rama*: ${{ github.ref_name }}\n*Commit*: ${{ github.sha }}\n*Ejecutado por*: ${{ github.actor }}\n*Ver detalles*: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.CARVAJAL_SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

- El webhook de Slack se almacena como `CARVAJAL_SLACK_WEBHOOK_URL`
- El mensaje debe incluir: nombre del workflow, repositorio, rama, commit y enlace directo a la ejecución

## Ejemplo de Workflow Completo

```yaml
name: Integración Continua - Validación y Pruebas

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    name: Verificación de Estilo
    runs-on: ubuntu-latest
    steps:
      - name: Obtener código fuente
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar lint
        run: npm run lint

      - name: Notificar fallo en Slack
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ *Fallo en el pipeline*: `${{ github.workflow }}`\n*Repositorio*: ${{ github.repository }}\n*Rama*: ${{ github.ref_name }}\n*Ver detalles*: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.CARVAJAL_SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

  test:
    name: Pruebas Automatizadas
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Obtener código fuente
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar pruebas
        run: npm test

      - name: Notificar fallo en Slack
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ *Fallo en el pipeline*: `${{ github.workflow }}`\n*Repositorio*: ${{ github.repository }}\n*Rama*: ${{ github.ref_name }}\n*Ver detalles*: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.CARVAJAL_SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

  build:
    name: Compilación y Empaquetado
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Obtener código fuente
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Compilar proyecto
        run: npm run build

      - name: Notificar fallo en Slack
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ *Fallo en el pipeline*: `${{ github.workflow }}`\n*Repositorio*: ${{ github.repository }}\n*Rama*: ${{ github.ref_name }}\n*Ver detalles*: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.CARVAJAL_SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```
