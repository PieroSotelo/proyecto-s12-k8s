# Proyecto Práctico S12: Despliegue Cloud, Seguridad y Troubleshooting

Despliegue de una arquitectura distribuida en Kubernetes (k3s) sobre una instancia EC2 de AWS, aplicando hardening de seguridad y resolución de fallas.

## Arquitectura

- **Backend:** Deployment con `nginxinc/nginx-unprivileged` (puerto 8080).
- **Base de datos:** Deployment con PostgreSQL 15 expuesto internamente mediante el Service `db-service` (puerto 5432).
- **Configuración:** ConfigMap `app-config` (host y puerto de la BD) y Secret `db-secret` (credenciales en Base64).
- **Exposición:** Service `backend-service` tipo NodePort (puerto 30080), con el puerto abierto en el Security Group de AWS.
- **Init container:** `wait-for-db` espera a que la base de datos esté disponible antes de iniciar el backend.
- **Namespace:** `practica-s12`.

## Medidas de seguridad (Hardening)

- `runAsNonRoot: true`
- `runAsUser: 10001`
- `readOnlyRootFilesystem: true` (con volúmenes `emptyDir` en `/tmp` y `/var/cache/nginx`)
- `allowPrivilegeEscalation: false`
- Credenciales almacenadas en un Secret codificado en Base64.
- Sondas `livenessProbe` y `readinessProbe` configuradas.

## Estructura del repositorio

```
k8s/
  00-namespace.yaml
  01-security-configmap.yaml
  02-db-secret.yaml
  03-app-deployment.yaml
  04-app-service.yaml
docs/
  troubleshooting-log.md
README.md
```

## Cómo desplegar

```bash
git clone https://github.com/PierroSotelo/proyecto-s12-k8s.git
cd proyecto-s12-k8s
kubectl apply -f k8s/
kubectl get pods -n practica-s12
```

La aplicación queda disponible en `http://<IP-PUBLICA-EC2>:30080`.

## Troubleshooting

Las 3 fallas resueltas (Secret mal inyectado, puerto de BD incorrecto y sondas de salud erróneas) están documentadas en [docs/troubleshooting-log.md](docs/troubleshooting-log.md).
