# Lab 08 — Restart Policies

## Always

kubectl apply -f manifests/restart-always.yaml

Siempre reinicia el contenedor.

---

## Never

kubectl apply -f manifests/restart-never.yaml

No reinicia aunque falle.

---

## OnFailure

kubectl apply -f manifests/restart-onfailure.yaml

Reinicia solo si exit code != 0.

---

## Comparación

Always → Deployments  
OnFailure → Jobs  
Never → testing/debug