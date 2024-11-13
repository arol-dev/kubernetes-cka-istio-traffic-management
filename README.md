# Istio Traffic Management Lab - CKA Course

Este laboratorio práctico está diseñado para aprender a implementar estrategias de backup y restauración de etcd, un componente esencial en el control plane de Kubernetes. Asegúrate de seguir cada paso cuidadosamente y practicar varias veces para estar completamente preparado para el examen.

## Prerrequisitos

- **VirtualBox** (se necesita **Python** y **pywin32** como prerrequisitos).
- **Vagrant**.
- **MobaXterm** para sesiones SSH.

## Objetivos

1. Instalar y configurar Istio en un clúster de Kubernetes, habilitando la gestión de tráfico avanzada.
2. Activar la inyección de sidecars para gestionar el tráfico entre servicios.
3. Configurar un API Gateway de Istio para controlar el tráfico de entrada al clúster.
4. Aplicar reglas de **Request Routing** para dirigir el tráfico a versiones específicas de servicios.
5. Simular fallos mediante la **Fault Injection** (inyección de latencia y errores).
6. Implementar políticas de **Traffic Shifting** para dividir el flujo entre versiones.
7. Configurar **Circuit Breaking** para mejorar la estabilidad ante fallos.
8. Aplicar **Traffic Mirroring** para duplicar el tráfico hacia nuevas versiones para pruebas sin impacto en usuarios.
9. Configurar gateways de **Ingress y Egress** para gestionar tráfico externo.
10. Practicar técnicas básicas de solución de problemas en Istio.

## Contenido del Repositorio

Este repositorio incluye:

- Una carpeta `scripts` con dos scripts que proporcionan soporte durante el laboratorio.
- Un fichero `Vagrantfile` que permite automatizar el despliegue de tres VMs en VirtualBox.

Las VMs consisten en:

- 1 nodo master.
- 2 nodos worker.

## Paso 1: Despliegue de las VMs

1. Clona el repositorio en tu entorno local:

   ```bash
   git clone https://github.com/arol-dev/kubernetes-cka-backup-recovery-etcd.git
   cd kubernetes-cka-backup-recovery-etcd
   ```

2. Dentro del repositorio, ejecuta el siguiente comando para desplegar las VMs:

   ```bash
   vagrant up
   ```

   Esto comenzará a desplegar tres VMs en VirtualBox: un nodo master y dos worker nodes. Espera unos minutos para que el proceso termine.

3. Verifica el estado de las VMs con:

   ```bash
   vagrant status
   ```

   Asegúrate de que las tres máquinas estén en estado `running`.

4. Obtén la configuración SSH para conectarte a las máquinas:

   ```bash
   vagrant ssh-config
   ```

   Guarda los detalles proporcionados, ya que los necesitarás en el siguiente paso.

## Paso 2: Conectar a las VMs con MobaXterm

1. Abre **MobaXterm** y utiliza la configuración SSH obtenida anteriormente para conectarte a las tres máquinas.
   - No se requiere un usuario específico, deja el campo vacío.
   - Si se te solicita usuario o contraseña, utiliza la cadena `vagrant`.

## Paso 3: Desplegar la Aplicación BookInfo de Istio

1. Crea el namespace para BookInfo:
   ```bash
   kubectl create ns bookinfo
   ```
2. Despliega la aplicación:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
   ```

3. Verifica que los pods se hayan desplegado correctamente en el namespace bookinfo.
   ```bash
   kubectl get pods -n bookinfo
   ```

4. Expone la aplicación localmente (opcional):
   ```bash
   kubectl port-forward svc/productpage 9080:9080 -n bookinfo
   ```
5. Prueba la aplicación desde el navegador (opcional):
   - [http://127.0.0.1:9080/productpage](http://127.0.0.1:9080/productpage)

6. Si la aplicación se ve lenta o incorrecta, reinicia el `coredns` (opcional):
   ```bash
   kubectl -n kube-system rollout restart deployment coredns
   ```

## Paso 4: Instalación de Istio

1. **Descargar Istio y Actualizar PATH**
    ```bash
    curl -L https://istio.io/downloadIstio | sh -
    export PATH="$PATH:/home/vagrant/istio-1.23.3/bin"
    ```

2. **Pre-check de la Instalación de Istio**
    ```bash
    istioctl x precheck
    ```

3. **Instalar Istio Usando el Perfil Demo**
    ```bash
    istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y
    ```

## Paso 5: Habilitar Istio en el Clúster

1. **Habilitar la Inyección de Istio en el Namespace Default**
    ```bash
    kubectl label namespace default istio-injection=enabled
    ```

## Paso 6: Instalar el API Gateway (Kubernetes)

1. **Instalar los CRDs del API Gateway**
    ```bash
    kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null ||     { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.1.0" | kubectl apply -f -; }
    ```

## Paso 7: Traffic Management

Esta sección explora cómo gestionar el flujo de tráfico entre servicios en un entorno Kubernetes usando Istio.

La gestión de tráfico en Istio (Traffic Management) es una funcionalidad avanzada que permite controlar cómo las solicitudes se enrutan y distribuyen entre los servicios dentro de un clúster de Kubernetes. Con Istio, es posible definir reglas detalladas para dirigir el tráfico, dividirlo entre diferentes versiones de un servicio (por ejemplo, para pruebas A/B o despliegues canary), simular fallos (latencia o errores), y gestionar el flujo de datos de entrada y salida del clúster.

Entre las capacidades clave de la gestión de tráfico en Istio están:

- Request Routing
- Fault Injection
- Traffic Shifting
- Circuit Breaking
- Mirroring
- Ingress Gateway
- Egress Gateway

## 7.1 Request Routing

#### Descripción:
El enrutamiento de solicitudes permite dirigir el tráfico a versiones específicas de un servicio según reglas, como encabezados HTTP, rutas de URL o porcentajes de tráfico. Es útil para implementaciones canary o pruebas A/B.

#### Ejemplo:
```bash
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/networking/destination-rule-all.yaml -n bookinfo
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/networking/virtual-service-all-v1.yaml
```

**Explicación**: En este ejemplo, se aplican reglas de enrutamiento para el servicio Bookinfo. `destination-rule-all.yaml` define todas las versiones del servicio, mientras que `virtual-service-all-v1.yaml` dirige todo el tráfico a `reviews:v1`.

**Resultado Esperado**: Todo el tráfico de los usuarios se dirige a `reviews:v1`, por lo que solo debería aparecer `reviews:v1` en las respuestas.

## 7.2 Fault Injection (Inyección de Fallos)

#### Descripción:
La inyección de fallos permite simular problemas en el servicio, como latencia o errores, para probar la resiliencia de un sistema en condiciones no ideales.

#### Ejemplo:
```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

**Explicación**: Este archivo aplica una regla que introduce un retraso de 7 segundos en las respuestas del servicio `ratings`, simulando latencia.

**Resultado Esperado**: Al acceder a `productpage`, se notará un retraso en el servicio `ratings`, permitiendo observar los efectos de la latencia en los servicios dependientes.

## 7.3 Traffic Shifting o Despliegue Canary (Cambio de Tráfico)

#### Descripción:
El cambio de tráfico permite redirigir gradualmente el tráfico entre versiones de un servicio en proporciones específicas, útil para despliegues canary.

#### Ejemplo:
```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

**Explicación**: Esta regla divide el tráfico entre `reviews:v1` y `reviews:v2` según una proporción definida. Modifica la configuración para probar diferentes divisiones de tráfico, como 50%-50%.

**Resultado Esperado**: El tráfico al servicio se dividirá entre `reviews:v1` y `reviews:v2`, y las respuestas deberían reflejar ambas versiones.

## 7.4 Circuit Breaking (Corte de Circuito)

#### Descripción:
El corte de circuito ayuda a proteger los servicios al detener el tráfico hacia instancias que fallan continuamente, mejorando así la resiliencia del sistema.

#### Ejemplo:
```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-circuit-breaker
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

**Explicación**: Este ejemplo configura un límite de conexiones concurrentes a `reviews` y activa el corte de circuito si una instancia falla repetidamente.

**Resultado Esperado**: Durante un fallo, la instancia problemática será temporalmente excluida, protegiendo la estabilidad general del sistema.

## 7.5 Mirroring

#### Descripción:
El espejado permite duplicar solicitudes a una nueva versión de un servicio para realizar pruebas, enviando tráfico a una nueva versión sin afectar a los usuarios finales.

#### Ejemplo:
```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-mirroring
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    mirror:
      host: reviews
      subset: v2
    mirrorPercentage:
      value: 50.0
EOF
```

**Explicación**: Esta configuración espeja el 50% del tráfico de `reviews:v1` a `reviews:v2`, permitiendo probar `v2` sin afectar el flujo principal de usuarios.

**Resultado Esperado**: `reviews:v2` recibirá tráfico espejado, permitiendo observar su comportamiento bajo condiciones de tráfico real.

## 7.6 Ingress Gateway

1. **Configuración del Ingress Gateway**
    - **Instalar httpbin**:
        ```bash
        kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml
        ```
    - **Crear Gateway**:
        ```yaml
        kubectl apply -f - <<EOF
        apiVersion: networking.istio.io/v1
        kind: Gateway
        metadata:
          name: httpbin-gateway
        spec:
          selector:
            istio: ingressgateway
          servers:
          - port:
              number: 80
              name: http
              protocol: HTTP
            hosts:
            - "httpbin.example.com"
        EOF
        ```

2. **Revisar Detalles del Gateway**
    ```bash
    kubectl get gateway.networking.istio.io
    kubectl get gateway # Puede no funcionar ya que utiliza Kubernetes Gateway
    ```

## 7.6 Egress Gateway

1. **Cambiar Tipo de Servicio y Añadir Anotación**
    ```bash
    kubectl edit gateway httpbin-gateway
    # Añadir la anotación:
    networking.istio.io/service-type: ClusterIP
    ```

2. **Port-Forward y Prueba de Acceso**
    ```bash
    curl -s -I -HHost:httpbin.example.com "http://localhost:10000/status/200"
    ```

## Troubleshooting Envoy Proxy Sidecar
Para solucionar problemas, inspecciona los logs del contenedor sidecar en los pods con Istio inyectado.