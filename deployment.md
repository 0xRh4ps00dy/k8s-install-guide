# Guia de Kubernetes: Alta Disponibilitat i Resiliència (3 Nodes)

Aquesta guia detalla com desplegar, gestionar i provar aplicacions robustes en un clúster de Kubernetes, utilitzant un servidor Nginx com a exemple pràctic.

## 1. Preparació de l'Aplicació (Docker)

Abans de Kubernetes, hem de tenir la imatge disponible. Encara que utilitzem Nginx oficial, aquí tens el flux de treball per a imatges pròpies:

1. Construcció: `docker build -t el-teu-usuari/nginx-custom:v1 .`
2. Limitació en Docker: `docker run -d -m 400m --cpu="1" el-teu-usuari/nginx-custom:v1`
3. Pujar a DockerHub: `docker push el-teu-usuari/nginx-custom:v1`

## 2. El Deployment: Configuració d'Alta Disponibilitat

El Deployment és el que garanteix la persistència. Si un pod mor, el Deployment n'aixeca un altre.

Fitxer `deployment-nginx.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: web-server
spec:
  replicas: 3  # Una còpia per a cada node del teu clúster
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        # Sondes per evitar Downtime
        livenessProbe: # Comprova si el contenidor està viu
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
        readinessProbe: # Comprova si el contenidor està llest per rebre trànsit
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
```

Aplicar: `kubectl apply -f deployment-nginx.yaml`

## 3. Exposar l'Aplicació (Service)

Un servei permet que el trànsit arribi als pods mitjançant etiquetes (`labels`).

- Exposar via comanda: `kubectl expose deployment nginx-deployment --port=80 --type=NodePort --name=nginx-service`
- Veure el port assignat: `kubectl get svc nginx-service` (Busca el port 3XXXX).
- Gestió de Labels: `kubectl get pods --show-labels`

## 4. Diccionari de Comandes de Gestió

Aquí tens les comandes essencials per controlar el teu clúster:

| Comanda | Descripció |
| --- | --- |
| `kubectl get nodes` | Llista els 3 nodes i el seu estat. |
| `kubectl get pods -o wide` | Mostra els pods i en quina màquina física estan corrent. |
| `kubectl describe pod [NOM]` | Detalls complets i historial d'esdeveniments d'un pod. |
| `kubectl logs -f [NOM]` | Veure els logs de Nginx en temps real.
| `stern web-server` | Veure logs de tots els pods del deployment simultàniament. |
| `kubectl exec -ti [NOM] -- /bin/bash` | Entrar dins d'un contenidor Nginx per fer proves. |
| `kubectl scale deployment nginx-deployment --replicas=5` | Augmentar o reduir el nombre de còpies al moment. |
| `kubectl edit deployment nginx-deployment` | Modificar la configuració en calent (com un fitxer de text). |
| `kubectl get events --sort-by=.metadata.creationTimestamp` | Veure què està passant al clúster cronològicament. |

## 5. Prova de Foc: Alta Disponibilitat en acció

L'objectiu és demostrar que el clúster és capaç de sobreviure a fallades.

### Pas 1: Observació

Obre una terminal i deixa aquesta comanda corrent:
`watch -n 1 kubectl get pods -o wide`

### Pas 2: El "Desastre"

En una altra terminal, elimina un dels pods que estigui en estat `Running`:
`kubectl delete pod [NOM_D_UN_POD]`

### Pas 3: La Reacció de Kubernetes

Observaràs el següent:

1. El pod seleccionat passa a estat `Terminating`.
2. A l'instant, apareix un nou pod amb un nom diferent.
3. Kubernetes manté sempre el número de 3 rèpliques que vas definir al Deployment.

Prova avançada: Si apagues físicament una de les 3 màquines, Kubernetes esperarà uns minuts i automàticament mourà els pods d'aquella màquina cap a les dues que queden vives.

## 6. Neteja de Recursos

Si vols esborrar-ho tot per començar de zero:

- Esborrar el deployment i el servei: `kubectl delete deployment nginx-deployment` i `kubectl delete svc nginx-service`
- Neteja total (Compte!): `kubectl delete all --all`
