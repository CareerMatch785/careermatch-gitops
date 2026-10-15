# Configuracion de secretos en AWS

Este documento describe como External Secrets Operator (ESO) obtiene credenciales de AWS Secrets Manager y las publica como Kubernetes Secrets en el namespace `carrermatch`. No contiene contrasenas ni API keys.

## Flujo

1. Argo CD instala ESO en el namespace `external-secrets`.
2. El ServiceAccount `external-secrets` asume un rol IAM mediante IRSA y el proveedor OIDC de EKS.
3. Un `SecretStore` del namespace `carrermatch` accede a AWS Secrets Manager en `us-east-1`.
4. Los `ExternalSecret` copian los valores a Kubernetes Secrets; los Deployments los consumen mediante `secretKeyRef`.
5. ESO vuelve a consultar AWS cada hora (`refreshInterval: 1h`).

## Rol IAM e IRSA

El rol utilizado es `arn:aws:iam::954976288182:role/careermatch-external-secrets-local-role`. Su politica de permisos `careermatch-external-secrets-openai-read` permite solamente `GetSecretValue` y `DescribeSecret` para los secretos RDS y OpenAI configurados.

La relacion de confianza IAM debe conservar el acceso existente del usuario `israel` y permitir al ServiceAccount del operador asumir el rol usando el proveedor OIDC del cluster:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::954976288182:user/israel"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::954976288182:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/5546F8435093C7DA15A806B394265719"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/5546F8435093C7DA15A806B394265719:aud": "sts.amazonaws.com",
          "oidc.eks.us-east-1.amazonaws.com/id/5546F8435093C7DA15A806B394265719:sub": "system:serviceaccount:external-secrets:external-secrets"
        }
      }
    }
  ]
}
```

La anotacion IRSA se configura en [argocd/external-secrets-aws.yaml](argocd/external-secrets-aws.yaml). Si se modifica esa anotacion, Argo CD debe sincronizar el manifiesto y el pod controlador debe reiniciarse para recibir `AWS_ROLE_ARN` y el token OIDC:

```powershell
kubectl -n external-secrets rollout restart deployment/external-secrets
kubectl -n external-secrets rollout status deployment/external-secrets
```

## Secretos sincronizados

| AWS Secrets Manager | Claves/propiedades de origen | Kubernetes Secret | Consumidores |
| --- | --- | --- | --- |
| `rds!db-49717e06-e255-40d7-b1c6-4056221ecc11` | `username`, `password` | `careermatch-db-secret`: `DB_USERNAME`, `DB_PASSWORD` | `job-service`, `cv-service` |
| `careermatch/local/openai-4VMatx` | valor completo del secreto | `openai-secret`: `OPENAI_API_KEY` | `cv-service`, `job-source-service` |

La sincronizacion esta declarada en [apps/careermatch/templates/external-secret.yaml](apps/careermatch/templates/external-secret.yaml). Los valores generados son Secrets de Kubernetes namespaced en `carrermatch`; los Deployments no leen directamente AWS Secrets Manager.

El secreto de OpenAI en AWS es un objeto JSON. El `ExternalSecret` usa `remoteRef.property: OPENAI_API_KEY` para copiar solo la API key como valor plano a `openai-secret`; no elimines esa propiedad mientras el secreto remoto siga siendo JSON.

## URL de base de datos

`DB_URL` no se guarda en AWS Secrets Manager: se configura como dato de conexion no secreto en [apps/careermatch/values-aws.yaml](apps/careermatch/values-aws.yaml). Actualmente usa el endpoint RDS de `us-east-1`, puerto `5432` y base `careermatch`.

Los Deployments `job-service` y `cv-service` leen `DB_USERNAME` y `DB_PASSWORD` desde `careermatch-db-secret`, mientras que `DB_URL` viene de los valores Helm. En el modo local (`externalSecrets.enabled: false`) el chart conserva las variables de entorno de `values-local.yaml`.

## Permisos IAM

La politica adjunta al rol necesita estas acciones sobre los dos ARN indicados:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadRDSSecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:954976288182:secret:rds!db-49717e06-e255-40d7-b1c6-4056221ecc11-snMl6B"
    },
    {
      "Sid": "ReadOpenAISecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:954976288182:secret:careermatch/local/openai-4VMatx"
    }
  ]
}
```

No se requiere una resource policy adicional en los secretos para este acceso de la misma cuenta. RDS administra el valor del secreto; la lectura se autoriza con la politica de identidad del rol.

## Argo CD y Helm

La aplicacion `careermatch-aws` usa `apps/careermatch/values-aws.yaml` y despliega en `carrermatch`. Los manifiestos ESO se habilitan con `externalSecrets.enabled: true`. Los cambios del chart o de `argocd/external-secrets-aws.yaml` deben publicarse en la rama Git configurada en Argo CD para que se sincronicen.

El chart requiere `jobService.env.DB_URL` y `cvService.env.DB_URL` cuando ESO esta habilitado. Si falta una, Helm detiene el render para evitar lanzar un contenedor con URL vacia.

## Verificacion

Estos comandos muestran estados y nombres, no los valores de los secretos:

```powershell
kubectl -n external-secrets get serviceaccount external-secrets -o yaml
kubectl -n carrermatch get secretstore aws-secretsmanager
kubectl -n carrermatch get externalsecret careermatch-db careermatch-openai
kubectl -n carrermatch get secret careermatch-db-secret openai-secret
kubectl -n carrermatch get deployments
```

El estado esperado es `SecretStore READY=True`, ambos `ExternalSecret READY=True` y los Secrets presentes. Para validar el chart antes de publicarlo:

```powershell
helm template careermatch-aws apps/careermatch --namespace carrermatch --kube-version 1.35.8 -f apps/careermatch/values-aws.yaml
```

## Problemas comunes

- `secret "careermatch-db-secret" not found`: el Secret AWS no se proyecta automaticamente a Kubernetes. Revisa que exista el `ExternalSecret`, que su `SecretStore` este `Ready` y que el target se llame `careermatch-db-secret`.
- `SecretStore ValidationFailed` con `no EC2 IMDS role found`: comprueba la anotacion IRSA del ServiceAccount, la relacion OIDC de confianza y que el controlador haya reiniciado despues de aplicar la anotacion.
- `ExternalSecret SecretSyncedError`: revisa el mensaje de estado, el ARN y las propiedades de origen (`username`/`password`); no imprimas el contenido del Secret para diagnosticarlo.
- El pod ya estaba ejecutandose cuando se actualizo un Secret: Kubernetes no actualiza automaticamente variables de entorno dentro de un proceso activo. Reinicia el Deployment consumidor despues de la rotacion si necesita cargar inmediatamente el nuevo valor.
- Argo CD muestra `DB_URL must be set`: completa `jobService.env.DB_URL` y `cvService.env.DB_URL` en `values-aws.yaml`.
