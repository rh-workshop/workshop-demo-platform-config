# Backlog de mejoras

Mejoras acordadas pero **no implementadas todavía**: primero se valida que el
bootstrap arranca de cero de punta a punta; luego se refactoriza sobre una base
que ya sabemos que funciona.

> Lo relativo a **permisos, identidad, segregación de funciones y evidencia para
> auditoría** vive en [`governance-backlog.md`](governance-backlog.md), organizado
> por dominio de decisión.

> **Norma al aplicarlas: no eliminar operadores ni componentes.** Si uno no aplica
> a un entorno, se **comenta** con el motivo — nunca se borra. Un componente
> borrado se pierde para el resto de entornos; uno comentado se reactiva con una
> línea.

## 1. Catálogo de componentes activos (en vez de listas fijas)

**Problema.** Hoy, para no desplegar un componente hay que tocar varios sitios:
`gitops/apps/kustomization.yaml` (la Application), la `OperatorPolicy` que instala
su operador, y su overlay. Es fácil dejar uno a medias — un operador instalado sin
su configuración, o al revés.

**Propuesta.** Una única lista de datos en `clusters.yml`, sin booleanos por
componente ni condicionales en el playbook:

```yaml
components:
  - acm
  - quay
  - acs
  - keycloak
  - connectivity-link
  - cert-manager
  - pipelines
  - monitoring
  # - metallb         # solo bare-metal: en cloud el proveedor ya da LoadBalancers
  # - developer-hub   # el paquete rhdh-operator no está en el catálogo
```

Se implementa en dos capas, ambas por **iteración sobre datos**, no por
ramificación (`when`/`if`), que es lo que se quiere evitar:

1. El playbook genera las Applications recorriendo `components`, en lugar de que
   `gitops/apps/kustomization.yaml` las enumere fijas.
2. Las `OperatorPolicy` se generan filtrando esa misma lista: si un componente no
   tiene Application, su operador tampoco hace falta.

**Ventaja.** Una sola fuente de verdad: comentar una línea desactiva el componente
entero — operador y configuración — sin tocar el playbook.

## 1-bis. Promoción de imágenes entre organizaciones de Quay — HECHO

**Implementado** en `workshop-pipelines/gitops/base/pipeline-promote-image.yaml`
(con un run por salto de la cadena **dev → test → prod → contingencia**:
`pipelinerun-promote-dev-to-test.yaml`, `-test-to-prod.yaml` y
`-prod-to-contingencia.yaml`; los tres invocan el mismo pipeline y solo cambian
`source-org`, `target-org` y el overlay destino):
`skopeo copy --all` entre organizaciones (verificando que el digest en destino
coincide), `kustomize edit set image` sobre el overlay del entorno destino con el
MISMO digest, push con reintento y sync opcional de Argo. La credencial de
promoción (`quay-promotion-credentials`) es **distinta** de la del CI — ver el
README de `workshop-pipelines/` para su creación y para el modelo RBAC que hace
de barrera hacia `prod`/`contingencia`. **Validado en vivo** (2026-09-03):
`PipelineRun promote-demo-service-test-qmb89` en `Completed`, y el ciclo dev→test
completo (PR de promoción con revisión y merge) en `workshop-demo-app-config`.

## 1-ter. SBOM en el pipeline de CI — HECHO

**Implementado** en `workshop-pipelines/gitops/base/task-sbom-generate.yaml`,
enganchado en el CI tras `resolve-digest` (en paralelo con firma y escaneo; la
promoción del digest lo espera): `syft` genera el SPDX y `cosign attest` lo
publica como atestación con la misma llave de firma. No existe imagen Red Hat de
syft (elección documentada en la propia Task); la atestación sí va con la imagen
RHTAS de cosign. **Validado en vivo** (2026-09-03): TaskRuns `*-generate-sbom` en
`Succeeded` sobre builds reales de `demo-producer`, `dotnet-validacion` y `pipeline-v2`.

## 1-quater. Canary automático con Argo Rollouts — HECHO

**Implementado**: `rollouts/gitops/` (CR `RolloutManager` + Application
`config-rollouts` al hub) y, en `workshop-demo-app-config`, el `Rollout`
`canary-service-auto` + `AnalysisTemplate` sobre Prometheus (tasa de éxito y
p95). Convive con el canary manual por pesos de `HTTPRoute` (material del
workshop, intacto) en el path `/canary-auto`. Sin `trafficRouting`: el peso se
aproxima por réplicas — el control fino con Gateway API exige el plugin
community `rollouts-plugin-trafficrouter-gatewayapi`, fuera de la norma "solo
Red Hat".

**Autenticación al Thanos Querier: RESUELTA.** La consulta iba sin token y con
`insecure: true`; ese endpoint exige Bearer token y comprueba el permiso con
SubjectAccessReview, así que respondía 403 y —con `failureLimit: 1`— habría
abortado *todo* despliegue canary aunque la aplicación estuviera sana. Ahora
`serviceaccount-thanos-reader.yaml` (en el repo de aplicaciones) aporta las tres
piezas: ServiceAccount `thanos-reader`, ClusterRoleBinding al rol de solo lectura
`cluster-monitoring-view` y un Secret de tipo `service-account-token` que **el
cluster rellena** (su contenido nunca viaja en Git). La `AnalysisTemplate` lo
inyecta como arg `thanos-token` en la cabecera `Authorization`.

Eso obligó a ampliar el AppProject `workshop-platform`… y **fue un error, que la
auditoría revirtió**. Permitir `ClusterRoleBinding` a un proyecto que despliega
desde el repo de APLICACIONES convertía cualquier commit ahí en cluster-admin
efectivo (basta enlazar `cluster-admin` a una ServiceAccount propia); y permitir
`Secret` dejaba a ese repo SOBRESCRIBIR, con `selfHeal` incluido, los Secrets del
CI que viven en `workshop-demo-dev` — la llave de firma entre ellos. Las tres
piezas (ServiceAccount, ClusterRoleBinding y token) se movieron al repo de
plataforma, a `namespace-governance/gitops/base/namespaces/canary-demo-dev/`,
que es donde ya vive el RBAC por namespace.

**Pendiente de validar en vivo**: el ámbito cluster del `RolloutManager`, que el
token se rellene y que las métricas `http_requests_total` /
`http_request_duration_seconds_bucket` existan (requieren que la app las exponga
y haya un ServiceMonitor).

## 1-quinquies. Previews efímeros por Pull Request — HECHO

No estaba en el backlog original; se añadió al detectar que `overlays/dev` es
único y dos features en paralelo se pisan. **Implementado** en
`gitops/appsets/applicationset-workshop-previews-pr.yaml` (generador
`pullRequest` de GitHub sobre `workshop-demo-app-config`): una Application por
PR abierto en su namespace efímero `demo-service-pr-<n>` (path `/pr-<n>` del
gateway), borrada EN CASCADA al cerrar el PR. Barandilla propia:
`bootstrap/manifests/appproject-workshop-preview.yaml`. El token de GitHub va en
el Secret `github-pr-token` de `openshift-gitops`, nunca en Git.

**Filtro por etiqueta: obligatorio, no opcional (corregido en la auditoría).**
`workshop-demo-app-config` es un repositorio PÚBLICO, así que el generador
`pullRequest` sin filtro desplegaba en el cluster HUB el contenido de cualquier
PR abierto por cualquier persona de internet —incluido un PR desde un fork—, y
el manifiesto del PR decide la imagen (`newName`): ejecución de código arbitrario
en la plataforma sin revisión. Ahora solo se despliegan los PR con la etiqueta
`preview`, que únicamente puede poner quien tiene escritura en el repo.

Pendiente: cuotas/NetworkPolicy de `namespace-governance` para los namespaces
`*-pr-*` (ver 6.4), y validación en vivo — el clúster ya está encendido
(verificado 2026-09-03) pero el `ApplicationSet workshop-previews-pr` sigue sin
generar ninguna Application `*-pr-*`: no hay evidencia de que un PR con la
etiqueta `preview` se haya abierto y probado el flujo completo todavía.

## 1-sexies. Cadena DevSecOps del CI completada — HECHO

**Implementado** en `workshop-pipelines/gitops/base/` (sin pipelines nuevos: las
piezas entran en los tres existentes, porque lo que cambia entre ambientes son
datos, no responsabilidades):

- **Tests unitarios** (`task-go-test.yaml`): `go test ./... -coverprofile` con
  imagen Red Hat (`ubi9/go-toolset`), cobertura como result. El monorepo hoy NO
  tiene tests: la task lo detecta y lo informa sin romper
  (`fail-on-no-tests: "false"`); ponerlo en `"true"` los hará obligatorios.
- **Detección de secretos** (`task-secret-scan.yaml`): gitleaks sobre árbol e
  HISTORIAL completo (el clone del CI pasó a `DEPTH: "0"`; un shallow haría el
  escaneo decorativo y la task lo detecta y corta). Falla CERRADO por defecto
  (`fail-on-violation: "true"`): un secreto filtrado no admite "informar y
  seguir". Tercera excepción documentada a "solo Red Hat" (no existe detector de
  secretos en registry.redhat.io), espejada en `company-tooling` como semgrep y
  syft (`quay/ansible/organizations.yml`).
- **Chequeo de manifiestos** (`task-deployment-check.yaml`): `roxctl deployment
  check` sobre el overlay RENDERIZADO con kustomize — políticas DEPLOY de ACS
  (privilegios, límites, montajes) antes de promocionar el digest. Cero
  dependencias nuevas: misma imagen roxctl y mismas credenciales que
  `image-scan`.
- **Versionado GitFlow** (`resolve-image-tags` + `apply-version-tag` en el CI):
  tag de versión derivado de la rama como ALIAS inmutable del mismo digest
  (`skopeo copy` intra-repo, sin reconstruir). Ver "Versionado por rama" en
  `workshop-pipelines/README.md`.
- **Procedencia en la promoción** (`verify-provenance` en `promote-image`): con
  `require-release-provenance: "true"` (los runs hacia prod y contingencia), el
  commit estampado en `org.opencontainers.image.revision` debe existir en el
  repo de aplicación, ser ancestro de `main` y llevar tag semver.

**Sin validar en vivo** (solo compilación estática con `oc kustomize`): queda
para la siguiente reconstrucción del laboratorio.

## 2. Dominio propio `labjp.xyz` (Cloudflare)

Hoy los hostnames usan el dominio del sandbox RHPDS, que cambia en cada
reaprovisionamiento. Pendiente:

- `ClusterIssuer` de Let's Encrypt con solver **DNS-01 de Cloudflare** (el
  `cert-manager` del cluster solo trae issuers para el dominio del sandbox).
- API Token de Cloudflare (`Zone:DNS:Edit` sobre `labjp.xyz`) por variable de
  entorno, nunca en Git.
- Hostnames objetivo, divididos por ambiente:
  `auth-workshop-{dev,prod}.labjp.xyz` (Keycloak) y
  `gateway-workshop-{hub,dev,prod}.labjp.xyz` (Gateway).
- CNAME en Cloudflare al ELB de cada cluster, **con el proxy desactivado** (nube
  gris): en naranja, Cloudflare termina el TLS y rompe el reto ACME y el
  passthrough al Gateway.

## 3. external-dns

Los CNAME anteriores hay que reapuntarlos a mano cada vez que se reaprovisiona un
cluster (el hostname del ELB cambia). `external-dns` con el proveedor de Cloudflare
los mantendría solos.

## 4. `developer-hub` deshabilitado

El packagemanifest `rhdh-operator` no está en el catálogo de estos clusters, así
que su Application falla al instalar el operador. Queda **comentado, no borrado**,
a la espera de localizar el nombre real del paquete o el catálogo que lo sirve.

## 5. Gateway: de wildcard a hostname fijo

Al pasar de `*.apps.<cluster>` a un hostname único por ambiente, los `HTTPRoute`
de los 5 servicios ya no pueden discriminar por subdominio y tienen que rutear
**por path**. Afecta al repo `workshop-demo-app-config`, no solo a éste.

## 6. Hallazgos de la auditoría de cadena de suministro (los no corregidos)

Auditoría de supply chain / GitOps sobre ambos repos. Los CRÍTICO y ALTO se
corrigieron en el mismo commit; aquí quedan los MEDIO/BAJO, con su motivo.

### 6.1 No hay procedencia in-toto utilizable (SLSA Build L2/L3) — MEDIO

Tekton Chains está configurado (`pipelines/gitops/base/tektonconfig.yaml`:
`artifacts.pipelinerun.format: in-toto`), pero la procedencia no cierra el ciclo:

- Chains identifica el artefacto construido por los results `IMAGE_URL` /
  `IMAGE_DIGEST` de la TaskRun. El pipeline de CI expone los suyos como
  `image-digest` e `immutable-tag`
  (`workshop-pipelines/gitops/base/pipeline-ci-build-image.yaml`), que Chains no
  reconoce: solo hay procedencia de la TaskRun de `buildah`, no del PipelineRun.
- Cuando la **idempotencia** salta el build (el tag `git-<sha>` ya existe), esa
  TaskRun no corre y **no se genera ninguna procedencia** para ese run.
- Nada la **verifica**: sin verificación en el consumidor, la procedencia no
  cuenta para SLSA.

Pendiente: renombrar los results a `IMAGE_URL`/`IMAGE_DIGEST` a nivel de
PipelineRun y añadir una puerta `cosign verify-attestation --type slsaprovenance`
antes de promocionar. No se hizo ahora porque cambia el contrato de results del
pipeline (los `PipelineRun` de `runs/` y cualquier consumidor externo) — el
clúster ya está encendido y disponible para validar (verificado 2026-09-03), así
que la única razón restante para no hacerlo es el alcance del cambio, no la
disponibilidad de infraestructura.

### 6.2 Sin VEX: no hay forma declarativa de decir "este CVE no aplica" — MEDIO

La puerta de escaneo es binaria (`scan-severity-threshold` +
`scan-fail-on-violation`) y el default `"false"` está justificado precisamente
por los CVE de la base UBI9 sin parche. La salida correcta no es bajar el umbral
sino declarar excepciones: **VEX**. Hoy la única vía es el *deferral* manual en
la consola de ACS, que no se versiona y por tanto no es auditable. Pendiente:
consumir los documentos VEX que Red Hat publica para sus imágenes base y
versionar las excepciones propias junto al servicio que las declara.

### 6.3 Egress sin restringir en los namespaces de negocio — MEDIO

`namespace-governance/gitops/base/profiles/app-namespace/` solo declara reglas
de **ingress** (`allow-same-namespace`, `allow-from-platform-gateway`,
`allow-from-monitoring`, `allow-from-openshift-ingress`). Una NetworkPolicy sin
`policyTypes: [Egress]` no restringe la salida: un pod comprometido puede abrir
conexiones a cualquier destino, dentro y fuera del cluster. Pendiente: una
`deny-all-egress` con excepciones explícitas (DNS, gateway, registro, Keycloak).
No se aplica ahora porque romper el egress sin poder probar en un cluster deja
los servicios del workshop sin arrancar.

### 6.4 Namespaces de preview sin cuota, sin LimitRange y sin NetworkPolicy — MEDIO

Las Policies de ACM `require-resourcequota` / `require-limitrange` seleccionan
por nombre (`user*` y sufijo `-demo-dev`), así que los namespaces efímeros
`demo-service-pr-<n>` quedan fuera; tampoco los cubre `namespace-governance`,
que enumera namespaces fijos. Con el filtro por etiqueta ya activo en el
ApplicationSet el riesgo baja mucho (solo despliega un PR que un mantenedor haya
etiquetado), pero un preview sigue corriendo sin techo de recursos ni
aislamiento de red. Pendiente: ampliar el patrón de las Policies a
`demo-service-pr-*`.

### 6.5 Las organizaciones de Quay no corresponden con lo desplegado — HECHO

`quay/ansible/organizations.yml` crea cuatro organizaciones (`company-dev`,
`company-test`, `company-prod`, `company-contingencia`) con el repositorio
`app-workshop`, pero **todos** los overlays de `workshop-demo-app-config`
apuntan a `…/company/demo-service` y `…/company/api-service`: una sola
organización, que ningún playbook gobierna, con repositorios que no existen en
el modelo. La segregación por entorno está escrita pero no conectada: hoy nada
de lo desplegado pasa por la barrera de promoción. Pendiente: renombrar los
repositorios en el modelo (`demo-service`, `api-service`) y reapuntar cada
overlay a la organización de su entorno. Toca los dos repos y obliga a re-subir
las imágenes, así que va con la siguiente reconstrucción del laboratorio.

**Resuelto**: los overlays de `workshop-demo-app-config` apuntan ya a
`company-<ambiente>/<servicio>` (dev con digest real; test/prod/contingencia
como placeholders que rellena la promoción) y `quay/ansible/organizations.yml`
gobierna esas organizaciones desde `vars/platform-vars.yml`.

### 6.6 El ID de la SignatureIntegration de ACS es un valor de otra instalación — MEDIO

`acs/gitops/base/policies/require-image-signature*.yaml` referencian
`io.stackrox.signatureintegration.46ab1fe7-…`. Ese ID lo genera Central al crear
la integración, así que en un cluster nuevo **no existe** y la política queda
inerte: no bloquea nada y aparenta estar activa, que es el peor de los dos
mundos. Pendiente: crear la integración desde `acs/ansible/` y que el playbook
escriba el ID resultante, o usar un ID determinista. Requiere un Central vivo.

### 6.7 Llave de firma cosign estática, sin rotación ni caducidad — MEDIO

La misma llave de larga duración firma imágenes y atestaciones, vive en un
Secret del cluster y no tiene procedimiento de rotación ni fecha de caducidad
(`workshop-pipelines/README.md`, apartado 2). Sin log de transparencia
(desactivado a propósito, ver 6.8), una llave comprometida permite firmar hacia
atrás sin dejar rastro. Pendiente en producción: llave en Azure Key Vault con
rotación anual —o, mejor, firma **keyless** con identidad de workload— y Rekor
interno como evidencia temporal.

### 6.8 Rekor: desactivado hasta tener un log interno — MEDIO (mitigado)

`transparency.enabled` estaba en `"true"` **sin** `transparency.url`, lo que
publica en el log PÚBLICO `rekor.sigstore.dev` el nombre del registro interno,
el repositorio y el digest de cada artefacto corporativo. Se desactivó con el
motivo comentado en el manifiesto y la línea `transparency.url` preparada.
Pendiente: desplegar Rekor con Red Hat Trusted Artifact Signer y reactivar las
dos líneas juntas.

### 6.9 Datos de un entorno concreto en un fichero de ejemplo — HECHO

`workshop-pipelines/runs/pipelinerun-ci-build-image.yaml` lleva cableado el host
real de un sandbox (`…sandbox3572.opentlc.com`), mientras que el run de
promoción usa `CHANGE_ME_QUAY_HOST`. Debe homologarse a `CHANGE_ME_*`.

**Resuelto**: todos los `runs/` usan `CHANGE_ME_*`.

### 6.10 Separar el namespace del CI del namespace de aplicación — MEDIO

El corte correcto, y el que pediría un auditor, es que los Secrets del CI
(llave de firma, PAT de Git, robot de Quay, token de Argo) vivan en un namespace
`workshop-ci` donde **ningún grupo de desarrollo pueda ejecutar cargas**. La
mitigación aplicada (Tekton en solo lectura para `app-developers`) cierra la
exfiltración, pero sigue dependiendo de un Role bien escrito en vez de una
frontera de namespace. Pendiente: mover `workshop-pipelines/gitops/overlays/dev`
a `workshop-ci`, darle su entrada en `namespace-governance` y en las Policies de
cuota, y actualizar los `runs/`. No se hizo ahora porque toca la narrativa de
varias sesiones del workshop y no se puede validar en vivo.

## 7. Decisiones de alcance DevSecOps: lo que NO se implementa, y por qué

Decir explícitamente qué NO se cubre vale más que fingir que sí. Estas tres
decisiones acompañan a la cadena de 1-sexies.

### 7.1 SonarQube — NO se implementa (decisión)

En la parte de SEGURIDAD se solapa con semgrep, que ya corre en el CI; lo que
aportaría de verdad es la métrica de CALIDAD con histórico (deuda técnica,
duplicación, cobertura acumulada). El coste no compensa aquí: es un servidor
con Postgres y mantenimiento propio, y su edición Community **no analiza ramas
ni Pull Requests** — exactamente lo que un flujo GitFlow necesita; la edición
de pago que sí lo hace no se justifica para este material. Se integra SOLO si
el cliente ya tiene un SonarQube corporativo: entonces es una task más
(`sonar-scanner` contra su servidor) sin operar nada nuevo.

### 7.2 DAST — NO se implementa (decisión)

Un DAST (ZAP y similares) exige un entorno desplegado y estable contra el que
atacar, tuning de reglas por aplicación y triage continuo de falsos positivos:
desproporcionado para los servicios del workshop. El `smoke-test` ya cubre la
ALCANZABILIDAD post-despliegue. **Lo que queda sin cubrir y hay que decirlo**:
vulnerabilidades que solo aparecen en ejecución (fallos de autenticación y
sesión, cabeceras, inyección explotable de extremo a extremo). En la cadena
actual las mitigan parcialmente el SAST (el defecto en el código) y las
políticas de Kuadrant (AuthPolicy/RateLimitPolicy delante del servicio).

### 7.3 SCA de dependencias declaradas (go.mod) — roxctl no lo cubre

`roxctl` no tiene ningún subcomando que analice un `go.mod` (ni manifiestos de
dependencias de otros lenguajes) ANTES del build: sus objetos de análisis son
la imagen (`image scan`/`image check`) y los manifiestos de despliegue
(`deployment check`, que sí se integró). La cobertura SCA real llega tras el
build: el SBOM de syft y el escaneo de imagen de ACS ven los módulos Go
compilados en el binario — con Go es una brecha pequeña (todo dependencia
acaba en el binario), pero DESPLAZADA en el tiempo: un CVE en una dependencia
se descubre tras construir, no antes. Si se quisiera cerrar en pre-build haría
falta otra herramienta (p. ej. `osv-scanner`), que hoy no se añade por la norma
de dependencias mínimas; quedaría como cuarta excepción espejada si algún día
compensa.

## 8. Solo `workshop-demo-dev` corre en el HUB; el resto va a su cluster — RESUELTO EN PARTE

**Actualizado (2026-09-04):** `applicationset-workshop-services-dev.yaml` ya usa
el mismo `matrix` (`clusterDecisionResource` × `git`) que sus hermanos
test/prod/contingencia — verificado en vivo. Los 4 servicios didácticos
(`canary`, `bluegreen`, `circuit-breaker`, `api`) ya despliegan en
`cluster-dev`, no en el hub. Solo `workshop-demo-dev` (el servicio original,
con el namespace del CI) sigue en el hub, como asimetría deliberada del
material del taller — los labs y las capturas la asumen.

`namespace-governance-workload/` (nuevo componente, separado de
`namespace-governance/`) ya gobierna la asimetría real: overlay `hub` para
`workshop-demo-dev`, overlay `dev` para los otros 4. Si en el futuro
`workshop-demo-dev` también se mueve a `cluster-dev`, solo hace falta mover su
directorio de un overlay al otro en ese componente y actualizar el destino en
la Application/AppSet correspondiente — ya no hace falta "revisar
namespace-governance" como pendiente aparte.

## 9. Almacenamiento de Quay: S3 nativo vía CredentialsRequest (NooBaa solo sin object storage)

**Hallazgo (verificado en vivo).** En el hub no hay ODF: solo `mcg-operator`
(NooBaa suelto) con un backingstore `pv-pool` de 50Gi sobre `gp3-csi`. Es decir,
Quay guarda blobs en un PVC de EBS que NooBaa *presenta* como S3… en un cluster
AWS que tiene S3 de verdad (el registry interno ya lo usa:
`config.imageregistry` → bucket S3 con credenciales que emite el Cloud
Credential Operator). Es la causa probable del incidente del backingstore
`Rejected`/`ALL_NODES_OFFLINE` con push fallando en `blob upload invalid`: un
pod intermediario que se cae, sin durabilidad ni escalado de S3, limitado a
50Gi, y una premisa falsa ("Quay necesita ODF") en cloud.

**Recomendación.** Backend de almacenamiento de Quay **parametrizable**
(`quay_storage_backend: s3 | noobaa`, default en `vars/platform-vars.yml`,
sobreescribible por cluster en `clusters.yml`):

- **`s3` (default en cloud):** un `CredentialsRequest` propio
  (`quay-enterprise/quay-s3-credentials`, política mínima S3 sobre el bucket)
  que el CCO resuelve; el bootstrap crea el bucket (nombre derivado del infra
  name) y genera el `quay-config-bundle` con `DISTRIBUTED_STORAGE_CONFIG`
  (S3Storage) desde el secreto acuñado; el `QuayRegistry` pasa a
  `objectstorage: managed: false` y `noobaa.yaml`/`backingstore.yaml` salen del
  kustomize de `quay/gitops`.
- **`noobaa` (alternativa on-prem/bare-metal sin object storage):** el camino
  actual, documentado como tal — caso muy real en banca.

**Coordinación pendiente.** El cambio implica migrar los blobs existentes
(re-push/re-espejado: `mirror-tooling.yml` y el CI lo repueblan, pero las
imágenes firmadas viejas se pierden — asumible en el lab) y retirar el
`ignoreDifferences` de NooBaa añadido en `gitops/apps/hub/application-quay.yaml`
(commit 04e8ce6) cuando NooBaa deje de desplegarse. No se implementó en esta
pasada para no romper el CI en curso ni tocar ficheros con dueño en paralelo.

## 10. Criptografía post-cuántica (PQC) en la cadena de suministro — REVISIÓN 1 (roadmap)

**Qué es.** Los algoritmos de firma y cifrado que usamos hoy (ECDSA, RSA) los
rompe un computador cuántico suficientemente grande. NIST ya estandarizó los
reemplazos resistentes a cuántica (2024): **ML-KEM/Kyber** (FIPS 203, intercambio
de claves), **ML-DSA/Dilithium** (FIPS 204, firma) y **SLH-DSA/SPHINCS+**
(FIPS 205, firma basada en hash). El riesgo para banca es *"harvest now, decrypt
later"*: capturar hoy tráfico o artefactos firmados para romperlos cuando exista
la cuántica — un horizonte de años, pero los datos y las firmas de larga vida
(hipotecas, contratos, imágenes de release archivadas) se ven afectados hoy.

**Qué usamos hoy (todo cripto CLÁSICA, ningún PQC):**
- Firma de imágenes cosign → **ECDSA P-256** (`cosign generate-key-pair`).
- mTLS/TLS del gateway, Keycloak, ACS, Quay → RSA/ECDSA.
- Verificación de firma en ACS → valida esas firmas ECDSA.

**Qué revisar en la REVISIÓN 1 (sin tocar lo montado todavía):**
1. **Estado de PQC en cosign/Sigstore** — ¿ya firma con ML-DSA o híbrido? (upstream
   lo está incorporando; confirmar versión y soporte productivo).
2. **crypto-policies de RHEL 10 / OpenSSL 3.x** — perfiles PQC disponibles y si
   OpenShift 4.x los expone en el ingress/API server.
3. **Firma híbrida** (clásica + PQC) como patrón de transición: no se rompe la
   verificación actual y se añade resistencia cuántica.
4. **Inventario de activos criptográficos** (CBOM — Cryptography Bill of Materials):
   qué algoritmos usa cada componente, para saber qué migrar y en qué orden. Encaja
   junto al SBOM que ya generamos.
5. **Alcance realista**: qué firmas/certificados tienen vida larga y merecen PQC
   primero (release archivadas, CA corporativa) vs. lo efímero (tokens, TLS de
   sesión) que se renueva solo.

**Entregable de la revisión.** Nota de roadmap PQC para el cliente (banca,
horizonte 3-5 años): qué es cuántica-vulnerable hoy, prioridad de migración,
y estado real del soporte en la cadena Red Hat. No es un hallazgo de seguridad
que bloquee — es preparación estratégica. Relacionado con el ángulo "agentes de
IA en el flujo" (otra tendencia emergente de DevSecOps): ambos son *futuro*, no
requisito actual.
