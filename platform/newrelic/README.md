# New Relic en AWS

Argo CD instala `nri-bundle` en el namespace `newrelic` y registra el
cluster como `careermatch`. Se habilitan metricas de infraestructura,
kube-state-metrics y eventos. Los logs y la instrumentacion APM se
configuran por separado. `lowDataMode` reduce metricas; no garantiza
que el consumo permanezca dentro del plan gratuito.

## Requisitos

- External Secrets Operator instalado y operativo.
- Secreto `careermatch/newrelic` en AWS Secrets Manager, region `us-east-1`,
  con un objeto JSON que contiene la propiedad `licenseKey`.
- El rol IAM del ServiceAccount del operador debe permitir leer ese secreto.
  Agregar este statement a su politica existente, conservando los permisos
  de RDS y OpenAI:

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
  "Resource": "arn:aws:secretsmanager:us-east-1:954976288182:secret:careermatch/newrelic-??????"
}
```

El sufijo de seis caracteres corresponde al sufijo generado por AWS.
Para restringirlo a un ARN exacto, sustituir el patron por el ARN del secreto.
Si utiliza una clave KMS administrada por el cliente, tambien requiere
`kms:Decrypt` para esa clave y acceso en su politica KMS.

## Instalacion

Publicar los archivos en la rama `main` configurada en la Application y aplicar:

```powershell
kubectl apply -f argocd/newrelic-aws.yaml
```

Argo CD crea el namespace y administra tanto el chart como el SecretStore
y el ExternalSecret. Estos recursos no contienen la clave de ingesta.
Los agentes pueden esperar el Secret mientras ESO realiza la primera sincronizacion.

## Verificacion

```powershell
kubectl -n argocd get application newrelic-aws
kubectl -n newrelic get secretstore aws-secretsmanager
kubectl -n newrelic get externalsecret newrelic-license
kubectl -n newrelic get pods
```

El SecretStore y el ExternalSecret deben mostrar `READY=True`; los pods deben
estar listos. Revisar luego el cluster `careermatch` en New Relic.
Un error `AccessDeniedException` en ESO indica que faltan permisos IAM.

Para validar el chart localmente sin conectarse al cluster:

```powershell
helm template newrelic nri-bundle --repo https://helm-charts.newrelic.com --version 8.0.28 --namespace newrelic -f platform/newrelic/values-aws.yaml
```

Referencia: [chart oficial nri-bundle](https://github.com/newrelic/helm-charts/tree/master/charts/nri-bundle).
