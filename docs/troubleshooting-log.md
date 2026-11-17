# Bitácora de Troubleshooting

Registro de las 3 fallas provocadas y resueltas durante el despliegue en Kubernetes (k3s sobre AWS EC2), namespace `practica-s12`.

---

## Escenario A: CrashLoopBackOff por Secret mal inyectado

- **Síntoma:** el pod `backend` quedó en estado `CrashLoopBackOff` tras reiniciar el deployment.
- **Comandos usados:**
  ```bash
  kubectl get pods -n practica-s12
  kubectl logs deploy/backend -n practica-s12
  kubectl describe pod -l app=backend -n practica-s12
  ```
- **Causa raíz:** la clave `DB_PASSWORD` no existía en el Secret (se había renombrado a `DB_PASS`), por lo que la variable de entorno llegaba vacía y el contenedor terminaba con el error `DB_PASSWORD no definida`.
- **Solución:** restaurar la clave `DB_PASSWORD` en `02-db-secret.yaml`, aplicar con `kubectl apply` y reiniciar con `kubectl rollout restart deploy/backend`.
- **Resultado:** el pod volvió a `1/1 Running`.

---

## Escenario B: Falla de red hacia la base de datos

- **Síntoma:** el pod nuevo del backend se quedó bloqueado en `Init:0/1`.
- **Comandos usados:**
  ```bash
  kubectl get pods -n practica-s12
  kubectl logs -l app=backend -n practica-s12 -c wait-for-db
  kubectl get svc,endpoints -n practica-s12
  ```
- **Causa raíz:** el ConfigMap `app-config` tenía `DB_PORT: "5433"`, pero el Service `db-service` y el endpoint de PostgreSQL escuchan en el puerto `5432`. El init container `wait-for-db` esperaba indefinidamente una conexión que nunca llegaba.
- **Solución:** corregir `DB_PORT` a `"5432"` en `01-security-configmap.yaml`, aplicar y reiniciar el deployment.
- **Resultado:** el init container terminó y el backend pasó a `1/1 Running`.

---

## Escenario C: Sondas de salud (probes) incorrectas

- **Síntoma:** el contenedor del backend se reiniciaba constantemente (columna `RESTARTS` en aumento).
- **Comandos usados:**
  ```bash
  kubectl get pods -n practica-s12
  kubectl describe pod -l app=backend -n practica-s12
  ```
- **Causa raíz:** el `livenessProbe` apuntaba a la ruta `/healthz`, que no existe en la aplicación y devuelve HTTP 404. Kubernetes interpretaba el contenedor como no saludable y lo reiniciaba.
- **Solución:** cambiar `path: /healthz` por `path: /` en el `livenessProbe` de `03-app-deployment.yaml` y volver a aplicar.
- **Resultado:** el contenedor se estabilizó, con `RESTARTS` sin aumentar.
