# Troubleshooting

## I CronJob non fanno nulla

Controlli base:

```bash
oc get cj -n ocp-namespace-shutdown
oc get job -n ocp-namespace-shutdown
oc get events -n ocp-namespace-shutdown --sort-by=.lastTimestamp
```

Verifica timezone e schedule:

```bash
oc get cj shutdown-target-namespaces -n ocp-namespace-shutdown -o yaml | egrep 'schedule|timeZone|suspend'
oc get cj verify-target-namespaces-zero -n ocp-namespace-shutdown -o yaml | egrep 'schedule|timeZone|suspend'
```

## Test manuale

```bash
oc create job test-shutdown-001 --from=cronjob/shutdown-target-namespaces -n ocp-namespace-shutdown
oc logs -f job/test-shutdown-001 -n ocp-namespace-shutdown

oc create job test-verify-001 --from=cronjob/verify-target-namespaces-zero -n ocp-namespace-shutdown
oc logs -f job/test-verify-001 -n ocp-namespace-shutdown
```

## Verifica RBAC

```bash
oc auth can-i list namespaces \
  --as system:serviceaccount:ocp-namespace-shutdown:ns-shutdown-sa

oc auth can-i list deployments.apps -n mb-mutui-be \
  --as system:serviceaccount:ocp-namespace-shutdown:ns-shutdown-sa

oc auth can-i patch deployments/scale -n mb-mutui-be \
  --as system:serviceaccount:ocp-namespace-shutdown:ns-shutdown-sa

oc auth can-i patch statefulsets/scale -n mb-mutui-be \
  --as system:serviceaccount:ocp-namespace-shutdown:ns-shutdown-sa
```

## Verifica ConfigMap

```bash
oc describe cm ns-shutdown-config -n ocp-namespace-shutdown
```

## Vedere gli alert

```bash
oc get alertingrule -n openshift-monitoring
oc describe alertingrule ns-shutdown-verify-alerts -n openshift-monitoring
```

## Sospendere velocemente

```bash
oc patch cronjob shutdown-target-namespaces -n ocp-namespace-shutdown -p '{"spec":{"suspend":true}}'
oc patch cronjob verify-target-namespaces-zero -n ocp-namespace-shutdown -p '{"spec":{"suspend":true}}'
```
