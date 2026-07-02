# Vault — init & unseal (mode persistant)

En mode non-dev avec storage `file` persistant, Vault démarre **scellé** (sealed)
et non initialisé au premier déploiement. Contrairement au mode dev, ça nécessite
une init manuelle une fois, mais aussi un **unseal après chaque redémarrage du pod**
(y compris quand tu stoppes/redémarres le lab le soir), sauf si tu configures
l'auto-unseal (overkill pour un lab).

## 1. Init (une seule fois, à la création du volume)

```bash
kubectl exec -n vault vault-0 -- vault operator init -key-shares=1 -key-threshold=1
```

Ça retourne 1 "Unseal Key" et 1 "Initial Root Token" — **à conserver précieusement**
(hors du repo Git, ex: gestionnaire de mots de passe).

## 2. Unseal

```bash
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY>
```

À refaire à chaque redémarrage du pod `vault-0` tant que le volume persiste
(les données restent, mais Vault redémarre scellé).

## 3. Login + activer le moteur KV v2

```bash
kubectl exec -n vault vault-0 -- vault login <ROOT_TOKEN>
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2
```

## 4. Mettre à jour le token ESO

Remplacer `REPLACE_WITH_ROOT_TOKEN_FROM_INIT` dans
`manifests/vault-bootstrap/cluster-secret-store.yaml` par le root token obtenu
à l'étape 1 (ou mieux : créer un token/policy dédié avec accès restreint à
`secret/alerta/*` plutôt que le root token).

## 5. Écrire les secrets applicatifs

```bash
kubectl exec -n vault vault-0 -- vault kv put secret/alerta/postgres \
  username=alerta password=<ton-password>

kubectl exec -n vault vault-0 -- vault kv put secret/alerta/server \
  database_url=postgres://alerta:<ton-password>@alerta-postgres-rw:5432/alerta \
  secret_key=<random> admin_users=admin@poc.local admin_key=<random>
```

## Idée d'automatisation future

Un petit script `unseal.sh` lancé au démarrage du lab (avec la clé stockée
localement hors Git) pour éviter de retaper la commande manuellement chaque matin.
