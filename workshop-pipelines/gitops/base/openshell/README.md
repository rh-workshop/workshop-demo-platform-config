# openshell — aislamiento out-of-process del agente IA

Guardrail de la **capa sobre el kernel** para el agente de triage de CVE: un
gateway que evalua CADA accion del agente (red, ficheros, procesos, inferencia)
contra politica declarativa, **fuera del alcance del agente**. Complementa a Kata
(capa de kernel) y a la `NetworkPolicy` del agente. Ver el diseno completo y el
benchmark en [`docs/ai-agents-devsecops-design.md`](../../../../docs/ai-agents-devsecops-design.md).

> **EXPERIMENTAL.** El propio proyecto declara que el camino de despliegue en
> Kubernetes esta en desarrollo activo ("expect rough edges and breaking
> changes"). Las versiones se **fijan** (nunca `dev`/`latest`) y se sube por PR.
> Producto: [`NVIDIA/OpenShell`](https://github.com/NVIDIA/OpenShell),
> guia OpenShift: https://docs.nvidia.com/openshell/latest/kubernetes/openshift

## Cadena de dependencias (orden de despliegue)

| # | Pieza | Que es | Como se instala |
|---|---|---|---|
| 1 | **Agent Sandbox** CRDs + controller | Proyecto upstream `kubernetes-sigs/agent-sandbox`; el CR `Sandbox` (`agents.x-k8s.io`) que OpenShell usa | cluster-scoped, prerequisito — `agent-sandbox.reference.yaml` documenta el manifiesto oficial |
| 2 | Namespace `openshell` + SCC | El SA `openshell-sandbox` necesita SCC para los pods de sandbox | `namespace.yaml` + `clusterrolebinding-scc.yaml` (Argo) |
| 3 | **OpenShell gateway** | El punto de control out-of-process (chart Helm OCI) | `application-openshell.yaml` (Argo → chart OCI, valores en `values-openshell.yaml`) |
| 4 | **OpenShell workspace** | Recursos namespace-scoped del sandbox, en el ns del pipeline | mismo chart `openshell-workspace`, por overlay |

## Por que hace falta ademas de Kata + NetworkPolicy

Benchmark de Red Hat: Kata detiene el escape de **kernel** pero NO filtra red;
la `NetworkPolicy` corta egress de forma **basica** (por namespace/puerto), pero
OpenShell lo eleva a **politica por-accion evaluada por el gateway** (que
comando, que fichero, que destino, que herramienta) — el modelo "browser tab"
aplicado a agentes: las sesiones estan aisladas y los permisos se verifican en el
runtime ANTES de ejecutar. Es la diferencia entre "el agente no puede llamar a
este puerto" (NetworkPolicy) y "el agente no puede ejecutar `curl` contra nada
que la politica no permita, aunque una prompt injection lo intente" (OpenShell).

## Modo de despliegue elegido: `operator`

El gateway sirve VARIOS namespaces de workspace pre-aprovisionados (GitOps), que
descubre por **label selector** (`operator_namespace_label`) — no auto-crea
namespaces. Encaja con el modelo declarativo del repo: los namespaces existen en
Git, OpenShell solo los reconoce. Falla cerrado si el namespace no esta en la
allowlist.

## Como se conecta con la task del agente

El step del agente (`task-agent-cve-triage.yaml`) se ejecuta DENTRO de un sandbox
de OpenShell en vez de un pod normal: el gateway intercepta sus llamadas de red y
sistema. La integracion final (envolver el step con el supervisor
`openshell-sandbox`) se fija cuando la version soportada estabilice el camino de
Kubernetes; hasta entonces, Kata + NetworkPolicy + guardrails de la task cubren
las dos superficies del benchmark.
