# vcluster-lab

Création automatisée de **clusters Kubernetes virtuels (vCluster)** avec quotas, sur un cluster Kubernetes existant.

Chaque vCluster est un control plane Kubernetes complet (API server, controller-manager, base de données) qui tourne dans un pod, dans un namespace dédié du cluster hôte. L'utilisateur en est administrateur : il crée ses namespaces, ses CRDs et ses rôles, sans rien voir de l'hôte ni des autres vClusters.

## Contenu du dépôt

| Fichier | Rôle |
| --- | --- |
| `create-vcluster.yml` | Playbook Ansible : pose les questions, calcule les quotas, crée le vCluster |
| `templates/vcluster.yaml.j2` | Modèle de configuration d'un vCluster (version, ressources, placement, quotas, sync Ingress) |
| `storageclass-vcluster.yaml` | StorageClass Longhorn à 1 réplica pour l'état des vClusters de dev |
| `ingress-nginx-vcluster-values.yaml` | Valeurs Helm de l'Ingress controller dédié aux vClusters |

## Prérequis

- Un cluster Kubernetes avec `kubectl` configuré (testé avec Kubernetes v1.30.4)
- Longhorn installé (StorageClass `longhorn`)
- Helm 3 ou 4
- Ansible (testé avec 2.10)
- Le CLI vCluster :

```bash
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64"
sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster
vcluster --version          # testé avec 0.37.4
```

## Mise en place (une seule fois)

### 1. StorageClass légère pour les vClusters

Chaque vCluster stocke son état sur un volume. Par défaut, Longhorn copie chaque volume 3 fois, ce qui charge la RAM des nœuds. Cette classe n'en garde qu'une copie. Elle n'est **pas** la classe par défaut du cluster.

```bash
kubectl apply -f storageclass-vcluster.yaml
kubectl get storageclass    # "(default)" doit rester sur longhorn
```

### 2. Choisir les nœuds qui hébergent les control planes

Le control plane d'un vCluster est sensible aux lenteurs : sur un nœud saturé en mémoire, il redémarre en boucle. On désigne des nœuds qui ont de la RAM réellement disponible (`free -h` sur le nœud) :

```bash
kubectl label node <noeud-1> <noeud-2> vcluster-controlplane=true
kubectl get nodes -L vcluster-controlplane
```

Le template place les control planes **uniquement** sur ces nœuds.

### 3. Ingress controller dédié aux vClusters

Un second ingress-nginx, séparé de celui de production, sert les Ingress des vClusters (classe `vcluster`, NodePort 30180/30543). Un vCluster ne peut ainsi pas intercepter le trafic des applications de production.

Vérifier que les NodePorts 30180 et 30543 sont libres, puis :

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm template ingress-nginx-vcluster ingress-nginx/ingress-nginx --version 4.14.3 \
  --namespace ingress-nginx-vcluster -f ingress-nginx-vcluster-values.yaml | grep -E "^kind:|^  name:"
helm install ingress-nginx-vcluster ingress-nginx/ingress-nginx --version 4.14.3 \
  --namespace ingress-nginx-vcluster --create-namespace -f ingress-nginx-vcluster-values.yaml
kubectl get ingressclass    # nginx et vcluster, avec deux controllers différents
```

La commande `helm template` permet de vérifier, avant l'installation, qu'aucun objet ne porte le nom d'un objet existant.

## Créer un vCluster

```bash
mkdir -p clusters
PYTHONNOUSERSITE=1 ansible-playbook create-vcluster.yml
```

Le playbook demande :

| Question | Défaut |
| --- | --- |
| Nom du vCluster | — |
| Environnement (`dev` = 1 réplica, `prod` = 3 réplicas) | `dev` |
| Version Kubernetes du vCluster | `v1.30.4` |
| Nombre de control planes (1 ou 3) | `1` |
| CPU garanti / maximum (cœurs) | `1` / `2` |
| RAM garantie / maximum (Gi) | `1` / `2` |
| Stockage total (Gi) | `10` |
| Nombre maximum de pods | `10` |

Il valide les réponses, ajoute au quota la part consommée par le control plane et CoreDNS, génère `clusters/<nom>.yaml`, affiche un résumé, demande confirmation, crée le vCluster et affiche le quota appliqué.

`PYTHONNOUSERSITE=1` évite qu'une version récente de Jinja2 installée avec `pip --user` casse Ansible 2.10 (erreur `environmentfilter`).

## Utiliser un vCluster

```bash
vcluster list
vcluster connect <nom> --namespace vcluster-<nom> -- kubectl get ns
kubectl get pods -n vcluster-<nom> -o wide              # pods réels sur l'hôte
kubectl describe resourcequota -n vcluster-<nom>        # quota appliqué
```

Les namespaces et Deployments créés dans le vCluster n'existent pas sur l'hôte. Seuls les pods, Services, PVC et Ingress y sont recopiés, sous un nom traduit (`nginx-xxxx-x-demo-x-<nom>`).

## Supprimer un vCluster

```bash
vcluster delete <nom> --namespace vcluster-<nom>
```

Cela supprime le namespace hôte, les pods et le volume du vCluster.

## Points d'attention

- Dans un vCluster, un Ingress doit utiliser `ingressClassName: vcluster`. Le forçage de cette classe n'est pas encore en place.
- Si la création échoue à la validation, le CLI a déjà créé le namespace `vcluster-<nom>` : le supprimer avant de relancer avec le même nom.
- La version du chart vCluster est fixée à `0.37.4` dans le playbook, pour des créations reproductibles.
