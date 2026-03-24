# OCP Namespace Shutdown

Cronjob per forzare a **0 repliche** tutti i `Deployment` e `StatefulSet` di una lista esplicita di namespace OpenShift, con:

- `CronJob` di **shutdown** ogni 15 minuti
- `CronJob` di **verify** 2 minuti dopo
- `AlertingRule` Prometheus su **failed verify job**
- guardrail per evitare namespace di piattaforma

## Struttura

- `manifests/00-namespace.yaml` — namespace tecnico
- `manifests/10-shutdown-stack.yaml` — SA, RBAC, ConfigMap, CronJob shutdown e verify
- `manifests/20-alertingrule.yaml` — alert Prometheus/Alertmanager sul verify failed
- `docs/TROUBLESHOOTING.md` — controlli rapidi e problemi comuni

## Prerequisiti

- accesso `oc` al cluster con permessi per creare risorse nel namespace tecnico e in `openshift-monitoring`
- monitoring OpenShift attivo
- immagine CLI disponibile: `quay.io/openshift/origin-cli:4.16`

## Personalizzazioni minime

### 1. Lista namespace target

Modifica `NS_LIST` in `manifests/10-shutdown-stack.yaml`.

Esempio:

```yaml
  NS_LIST: >-
    mb-mutui-be
    mb-mutui-fe
    mb-mutui-mgw
```

### 2. Schedules

Impostati di default così:

- shutdown: `*/15 * * * *`
- verify: `2,17,32,47 * * * *`
- timezone: `Europe/Rome`

### 3. Runbook URL alert

Aggiorna `runbook_url` in `manifests/20-alertingrule.yaml` con il tuo link interno.

## Installazione

```bash
oc apply -f manifests/00-namespace.yaml
oc apply -f manifests/10-shutdown-stack.yaml
oc apply -f manifests/20-alertingrule.yaml
```

## Test manuale immediato

### Shutdown

```bash
oc create job test-shutdown-001 \
  --from=cronjob/shutdown-target-namespaces \
  -n ocp-namespace-shutdown

oc logs -f job/test-shutdown-001 -n ocp-namespace-shutdown
```

### Verify

```bash
oc create job test-verify-001 \
  --from=cronjob/verify-target-namespaces-zero \
  -n ocp-namespace-shutdown

oc logs -f job/test-verify-001 -n ocp-namespace-shutdown
```

## Controlli utili

```bash
oc get cj -n ocp-namespace-shutdown
oc get job -n ocp-namespace-shutdown
oc get po -n ocp-namespace-shutdown
oc get events -n ocp-namespace-shutdown --sort-by=.lastTimestamp
```

## Modifica lista namespace nel tempo

Aggiorna la ConfigMap via manifest e riapplica:

```bash
oc apply -f manifests/10-shutdown-stack.yaml
```

Oppure patch veloce:

```bash
oc patch cm ns-shutdown-config -n ocp-namespace-shutdown \
  --type merge \
  -p '{"data":{"NS_LIST":"namespace1 namespace2 namespace3 namespaceN"}}'
```

## Sospendere / riattivare

### Sospendere

```bash
oc patch cronjob shutdown-target-namespaces -n ocp-namespace-shutdown -p '{"spec":{"suspend":true}}'
oc patch cronjob verify-target-namespaces-zero -n ocp-namespace-shutdown -p '{"spec":{"suspend":true}}'
```

### Riattivare

```bash
oc patch cronjob shutdown-target-namespaces -n ocp-namespace-shutdown -p '{"spec":{"suspend":false}}'
oc patch cronjob verify-target-namespaces-zero -n ocp-namespace-shutdown -p '{"spec":{"suspend":false}}'
```

## Rimozione safe

```bash
oc delete -f manifests/20-alertingrule.yaml
oc delete -f manifests/10-shutdown-stack.yaml
oc delete -f manifests/00-namespace.yaml
```

## Note 

- Il targeting vero è fatto da `NS_LIST`, non dalla regex di esclusione.
- La regex di esclusione è solo un guardrail aggiuntivo.
- Il verify fallisce con `exit 1` se trova `deploy/sts` non a zero.
- L'alert Prometheus scatta sui Job falliti generati dal CronJob `verify-target-namespaces-zero`.
- Il repo usa un `ClusterRole` dedicato, non `cluster-admin`.
