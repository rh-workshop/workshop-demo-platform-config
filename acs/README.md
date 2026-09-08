# acs — Red Hat Advanced Cluster Security

`company-acs`, Central en el hub y un `SecuredCluster` (sensor + admission
controller + collector) en cada cluster, hub incluido. Cubre firma de
imágenes, políticas de seguridad en BUILD/DEPLOY/RUNTIME, integración con
Quay y reporte periódico de vulnerabilidades.

## GitOps vs Ansible — qué gestiona cada uno

| | Vive en | Por qué |
|---|---|---|
| `Central` (una vez, solo hub) | `gitops/base/central.yaml`, Argo (`config-acs`) | Es un CR: Argo lo reconcilia igual que cualquier otro |
| `SecuredCluster` (en cada cluster) | `gitops/secured/base/`, Argo (AppSet `acs-secured`, fan-out por `environment`) | Mismo criterio — CR reconciliado por el operador |
| `SecurityPolicy` **sin** dependencia de un ID generado en runtime | `gitops/base/policies/`, Argo | Desde ACS 4.11 tienen CRD propio; `disallow-latest-tag` es el ejemplo |
| `SecurityPolicy` de firma (referencian el ID de la `SignatureIntegration`) | `ansible/policies/`, aplicadas por `integrations.yml` | El ID lo genera Central en cada instalación — un ID literal en Git no es reproducible y deja la política inerte |
| Políticas built-in del producto (activar/endurecer) | Sin manifiesto — API de Central, `integrations.yml --tags endurecimiento` | Son `IMPERATIVE`: existen de fábrica en toda instalación, no hay CR que declarar |
| Integraciones (Quay, firma cosign), colecciones, reportes | `ansible/integrations.yml` | Solo existen tras la API de Central; no tienen CRD |
| Init bundles (certificados del sensor) | `ansible/init-bundles.yml` + Policy de ACM `require-acs-init-bundle` | Los emite Central por API en cada instalación; son certificados, no se versionan |

## El init bundle — por qué es un playbook y no Argo

El sensor de cada cluster se identifica ante Central con un init bundle
(3 `Secret` con certificados de cliente). Sin él, el operador se queda en
`Irreconcilable: some init-bundle secrets missing in namespace "stackrox"`,
aunque el `SecuredCluster` aparezca perfectamente sincronizado en Argo — es
el síntoma que más despista al diagnosticar.

1. `acs/ansible/init-bundles.yml` pide a Central un bundle **por ambiente**
   (hub, dev, test, prod, contingencia — revocar uno no tumba el resto) y lo
   deposita, aplanado, en `open-cluster-management` (el único namespace desde
   el que un hub template de ACM puede leer).
2. La Policy `require-acs-init-bundle` (`acm/gitops/base/policies/`) replica
   cada bundle al namespace `stackrox` del cluster que corresponda, según su
   etiqueta `environment`.

```bash
export ACS_HOST=<host-de-central>
export ACS_TOKEN=<token-de-api>   # consola -> Platform Configuration -> Integrations -> API Token, rol Admin
ansible-playbook acs/ansible/init-bundles.yml
# rotar un ambiente:
ansible-playbook acs/ansible/init-bundles.yml -e forzar_reemision=true
```

## Políticas de seguridad — tres capas

1. **CR declarativo por Argo** (`gitops/base/policies/`): el catálogo de
   políticas propias como `SecurityPolicy` CRD, versionado y sincronizado por
   Argo — nunca a mano en la consola. Añadir una política que no dependa de un
   ID generado en runtime va aquí. Organizado en cuatro grupos:
   - **Higiene de imagen**: `disallow-latest-tag`, `resource-limits`,
     `package-manager-run`, `curl-wget-in-image`, `ssh-port-exposed`,
     `required-label-owner`.
   - **Endurecimiento del workload**: `read-only-root-fs`, `no-privileged`,
     `drop-all-capabilities`, `no-privilege-escalation`, `secret-env-var`,
     `run-as-non-root`.
   - **Vulnerabilidades (CVE)**: `fixable-cvss-7`, `fixable-severity`,
     `image-scanned`.
   - **Runtime** (`eventSource: DEPLOYMENT_EVENT`): `interactive-shell`,
     `process-uid-0`, `crypto-mining`, `package-exec-runtime`.

   **Todas arrancan en modo INFORM** (`enforcementActions` vacío): en producción
   se observan primero en Violations y solo después se agregan las acciones de
   enforce, comunicando a los equipos. Excepción: `disallow-latest-tag` ya viene
   con enforce porque es la regla base validada. En las de RUNTIME el enforce
   sería `KILL_POD_ENFORCEMENT` (agresivo) — se activa con criterio, no a ciegas.

   > **Antes de pasar a enforce**, confirmar en la consola (Policy Management) el
   > `fieldName`/valor exacto de las políticas marcadas `# VERIFICAR` en su YAML:
   > algunos nombres de criterio de 4.11 (p. ej. *Allow Privilege Escalation*,
   > *Run as Privileged User*, *Required Label*, *Image Scan Age*) conviene
   > cotejarlos con un `roxctl policy export` real para no dejar la política inerte.
2. **Firma cosign, por Ansible** (`ansible/policies/` + `integrations.yml
   --tags firma`): dos políticas gemelas — `require-image-signature` (BUILD,
   corta el pipeline si la imagen no está firmada) y
   `require-image-signature-deploy` (DEPLOY, admisión — `SCALE_TO_ZERO` si
   algo se salta el pipeline). Con scope a namespaces de negocio
   (`^.*-demo-(dev|test|prod|contingencia)$`): los namespaces de plataforma
   corren imágenes firmadas por Red Hat, no por el pipeline propio.
3. **Endurecimiento de políticas built-in, por Ansible**
   (`integrations.yml --tags endurecimiento`): ACS trae ~90 políticas de
   fábrica, muchas relevantes para producción **deshabilitadas por defecto**.
   Dos grupos:
   - **Bloqueo** (`politicas_bloqueo` en `integrations.yml`): no dependen de
     nada fuera del propio Deployment — root filesystem de solo lectura,
     capabilities, secretos en variables de entorno, CVSS ≥ 7 fixable. Se
     activan CON `enforcementActions`.
   - **Alerta** (`politicas_alerta`): dependen de una convención que la
     plataforma aún no exige — NetworkPolicy por Deployment, anotación
     `owner`/`team` (ningún namespace de negocio la lleva hoy), o son
     `RUNTIME` (`Process with UID 0`: solo detecta el proceso ya corriendo;
     forzar `KILL_POD` a ciegas es agresivo, muchas imágenes de base arrancan
     como root sin que el proceso principal lo sea). Quedan solo alertando
     hasta que se decida forzarlas.

```bash
export ACS_HOST=<host-de-central>
export ACS_TOKEN=<token-de-api>
ansible-playbook acs/ansible/integrations.yml --tags endurecimiento
```

## Integración con Quay

Una `imageintegration` (categoría `REGISTRY`) por organización del registro,
cada una con su robot `+acs-scanner` de solo lectura — los robots de Quay no
cruzan organizaciones. Requiere que
`quay/ansible/organizations.yml` ya haya creado ese robot (404 si no).

## Reporte periódico de vulnerabilidades

Antes vivía solo en la consola ("nada de eso se versiona en Git"); ahora lo
gestiona `integrations.yml --tags reportes`:

- **Notifier SMTP** (`smtp-reportes-vulnerabilidades`) — la credencial sale
  de un `Secret` propio (`acs-smtp`, namespace `stackrox`), sincronizado por
  `ExternalSecret` desde el almacén (`platform/acs/smtp`) — nunca se versiona
  ni se pasa por variable de entorno en texto plano si ya existe el secreto.
- **Colección `todos-los-deployments`** — regla `Deployment` con regex `.*`:
  matchea toda la flota, sin filtrar por cluster ni namespace. Acotarla
  (p. ej. solo prod) es cambiar esta colección, no el `ReportConfiguration`.
- **`ReportConfiguration`** — CVE `CRITICAL`/`IMPORTANT` con parche
  disponible (`fixability: FIXABLE`; un reporte con CVE sin fix es ruido que
  nadie puede accionar), semanal, lunes 07:00.

**API v2, no v1.** `POST/GET /v1/reports/configurations` da 404 en ACS 4.11
— el `ReportConfiguration` solo existe en `/v2/reports/configurations`, a
diferencia del resto de este playbook (políticas, integraciones, firma), que
sí usa `v1`. Verificado en vivo. El body también exige `allVuln: true`
(declara el rango temporal a cubrir; sin él, Central responde 400).

```bash
# Sembrar la credencial SMTP real (un adoptante sustituye el valor):
bao kv put -mount=secret platform/acs/smtp username=<usuario> password=<password>

# ACS_SMTP_HOST/ACS_REPORT_TO en vars/platform-vars.yml siguen en CHANGE_ME_*
# hasta que un adoptante los sustituya por su dominio y su lista real.
export ACS_HOST=<host-de-central>
export ACS_TOKEN=<token-de-api>
ansible-playbook acs/ansible/integrations.yml --tags reportes
```

## Cómo ejecutar los playbooks

```bash
export ACS_HOST=<host-de-central>
export ACS_TOKEN=<token-de-api>   # consola -> Platform Configuration -> Integrations -> API Token, rol Admin
ansible-playbook acs/ansible/init-bundles.yml      # certificados del sensor por ambiente
ansible-playbook acs/ansible/integrations.yml      # Quay + firma + endurecimiento + reportes, todo junto
ansible-playbook acs/ansible/integrations.yml --tags registro         # solo integración con Quay
ansible-playbook acs/ansible/integrations.yml --tags firma            # solo firma cosign
ansible-playbook acs/ansible/integrations.yml --tags endurecimiento   # solo políticas built-in
ansible-playbook acs/ansible/integrations.yml --tags reportes         # solo reporte de vulnerabilidades
```

Ninguno está en `bootstrap/ansible/bootstrap.yml`: a diferencia de Quay, ACS
no tiene huevo-gallina de día-0 (Central ya está vivo cuando se ejecutan) —
se corren sueltos, después de que Argo haya desplegado `config-acs` y los
`SecuredCluster`.

## Cómo verificar que un cluster está bien vigilado

```bash
oc get pods -n stackrox -l app=sensor
oc get pods -n stackrox -l app=admission-control
oc get securedcluster -A -o custom-columns='NS:.metadata.namespace,AVAILABLE:.status.conditions[?(@.type=="Deployed")].status'
```

Si el sensor no arranca, primero revisar el init bundle (`Irreconcilable` en
el `SecuredCluster`), no la Application de Argo — Argo la reporta `Synced`
aunque el operador esté bloqueado por falta de certificados.

## El flujo completo de extremo a extremo

Qué gobierna cada control y en qué orden se pone en marcha. Regla transversal:
**todo es manifiesto o playbook versionado — nada se toca a mano en la consola.**

| # | Control | Dónde vive | Cómo se aplica |
|---|---|---|---|
| 1 | Central + SecuredCluster | `gitops/base/`, `gitops/secured/` | Argo (CRs) |
| 2 | Init bundles (certs del sensor) | `ansible/init-bundles.yml` + Policy ACM | playbook + ACM |
| 3 | Integración con Quay (lectura) | `ansible/integrations.yml --tags registro` | playbook (API) |
| 4 | Firma cosign (`SignatureIntegration` + 2 políticas) | `ansible/policies/` + `--tags firma` | playbook (ID runtime) |
| 5 | Catálogo de políticas propias (19) | `gitops/base/policies/` | Argo (`SecurityPolicy` CRD) |
| 6 | Endurecimiento de built-in | `ansible/integrations.yml --tags endurecimiento` | playbook (API) |
| 7 | Reporte de vulnerabilidades (SMTP + `ReportConfiguration`) | `ansible/integrations.yml --tags reportes` | playbook (API) + ESO |

**Por qué unas por Argo y otras por Ansible:** lo que tiene CRD limpio y no
depende de un ID generado por Central va por Argo (Central, SecuredCluster,
las 19 `SecurityPolicy`). Lo que **solo existe tras la API de Central** —su ID
lo genera la instalación— va por playbook idempotente: integración de registro,
firma (referencia el ID de la `SignatureIntegration`), activar built-ins,
notifier + `ReportConfiguration`. Un ID literal en Git no es reproducible.

**Rollout recomendado (producción):**
1. Argo despliega Central, SecuredClusters y las 19 políticas → todas en INFORM.
2. `--tags registro` + `--tags firma` → ACS puede leer el registro y verificar firmas.
3. Observar **Violations** unos días: ver qué cargas reales incumplirían.
4. Confirmar los `fieldName` marcados `# VERIFICAR` contra la consola.
5. Comunicar a los equipos, luego pasar política a política de INFORM a enforce
   (agregar `enforcementActions`), empezando por las de menor ruido.
6. `--tags endurecimiento` (built-ins) y `--tags reportes` (correo semanal).

**Integración con el pipeline (repo `workshop-pipelines`):** el CI evalúa cada
imagen contra ACS con `roxctl image scan` (CVEs) y `roxctl image check`
(políticas de ciclo BUILD, incluida la firma) — ver `task-image-scan.yaml`. El
gate de admisión (ciclo DEPLOY) lo aplican los `SecuredCluster` con estas mismas
políticas. Build-time y deploy-time son complementarios: el pipeline protege el
camino esperado, la admisión protege el perímetro.
