# Pràctica: Desplegament de Clúster Kubernetes

Aquesta guia descriu els passos necessaris per aixecar un clúster de Kubernetes funcional format per un node Master i nodes Workers.

**Objectiu final:** Tenir el node Master i els nodes Workers connectats i en estat **`Ready`**.

---

## 1. Preparació de l'Entorn (TOTS els nodes)

**Aquests passos s'han d'aplicar tant a la màquina MASTER com als WORKERS.**

Seguirem la guia de referència de [ComputingForGeeks](https://computingforgeeks.com/deploy-kubernetes-cluster-on-ubuntu-with-kubeadm/), però amb les següents **modificacions importants**:

1.  **Sistema Base:** Realitza els **passos 1, 2 i 3** de la guia (actualització, desactivar swap, hostname, mòduls del kernel).
2.  **Container Runtime (Pas 4):** A la guia, tria l'opció d'instal·lar **DOCKER**.
3.  **Kubelet (Pas 5):**
    * Instal·la els paquets necessaris (`kubelet`, `kubeadm`, `kubectl`).
    * Habilita el servei perquè s'iniciï automàticament:
        ```bash
        sudo systemctl enable kubelet
        ```
    * **Molt Important:** Afegeix al fitxer `/etc/hosts` la IP i el nom DNS que utilitzaràs per al Master.

---

## 2. Inicialització del Control Plane (Només MASTER)

Un cop acabada la preparació anterior, executa **només al node Master** la següent comanda.

*El paràmetre `--pod-network-cidr` és la xarxa interna que utilitzarà k8s, per tant, ha de ser una xarxa que no estiguem utilitzant a l'aula.*

```bash
sudo kubeadm init --pod-network-cidr=172.16.0.0/16 --upload-certs --control-plane-endpoint=NOM_DNS
```

> **Notes:**
> * Substitueix `NOM_DNS` pel nom que has configurat a `/etc/hosts`.
> * El paràmetre `--pod-network-cidr=172.16.0.0/16` defineix la xarxa interna dels Pods. Assegura't que no coincideix amb la xarxa física de l'aula.

### 2.1. Configuració d'usuari
Si el procés acaba correctament (*"Your Kubernetes control-plane has initialized successfully!"*), executa això per poder utilitzar el clúster amb el teu usuari:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2.2. Instal·lació de la Xarxa (CNI)

Perquè els nodes es comuniquin i el clúster estigui operatiu, cal instal·lar un plugin de xarxa. Utilitzarem **Calico**.

* **ATENCIÓ:** No segueixis la guia de ComputingForGeeks per a aquest pas.
* Ves a la **[Documentació Oficial de Calico (On-Premises)](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)**.
* Segueix els passos indicats allà per instal·lar:
    1.  El `Tigera Calico operator`.
    2.  Els `custom resource definitions`.

---

## 3. Unió dels Workers (Només WORKERS)

Per afegir els nodes de treball al clúster:

1.  Assegura't que has completat tot el punt **1. Preparació de l'Entorn** en aquest node (incloent-hi el `/etc/hosts` apuntant al Master i l'habilitació del `kubelet`).
2.  Recupera la comanda `kubeadm join` que ha aparegut al final de la instal·lació del Master (comença per `kubeadm join...` i conté el token).
3.  Executa-la amb permisos de `sudo` al node worker.

*Exemple de format (utilitzeu el vostre propi token generat al master):*

```bash
kubeadm join k8s.asix2.iesmontsia.cat:6443 --token <token> ...
```

---

## 4. Validació de la Pràctica

Torna al node Master i comprova l'estat del clúster per verificar que la pràctica està completa.

Requisits per donar la tasca per finalitzada:

1. Tots els nodes han de sortir a la llista.
2. L'estat (`STATUS`) de tots els nodes ha de ser `Read`y.

```
kubectl get nodes -o wide
```

També pots verificar que tots els pods del sistema estan corrent:

```
kubectl get pods --all-namespaces
```
