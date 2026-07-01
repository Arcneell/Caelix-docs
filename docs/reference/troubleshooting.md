# Troubleshooting

Guide de résolution des problèmes courants.

## Diagnostic rapide

```bash
# Vérifier l'état général
bin/caelix status

# Valider la configuration
bin/caelix validate

# Diagnostic complet
bin/caelix doctor

# Voir les logs daemon
tail -50 .caelix/logs/caelix-daemon.log | python3 -m json.tool

# Derniers incidents
tail -20 .caelix/incidents/incidents.log
```

## Problèmes courants

### Le daemon ne démarre pas

**Symptôme** : `bin/caelix run` échoue immédiatement.

**Vérifications** :

1. Docker est-il accessible ?
   ```bash
   docker info
   ```

2. Le manifest est-il valide ?
   ```bash
   bin/caelix validate
   ```

3. Le répertoire CAELIX_DATA est-il accessible en écriture ?
   ```bash
   ls -la .caelix/
   ```

### Un service ne démarre pas

**Symptôme** : le conteneur n'est jamais créé.

**Vérifications** :

1. L'image existe-t-elle ?
   ```bash
   docker pull <image>
   ```

2. Le service est-il suspendu ?
   ```bash
   ls .caelix/state/<app>.suspend_reconcile
   # Si le fichier existe :
   bin/caelix resume <app>
   ```

3. Le service est-il en pause manuelle ?
   ```bash
   ls .caelix/state/<app>.manual_pause
   # Si le fichier existe :
   bin/caelix resume <app>
   ```

### Health checks échouent en permanence

**Symptôme** : le compteur `.fail` ne cesse d'augmenter.

**Vérifications** :

1. L'URL de health est-elle accessible ?
   ```bash
   curl -v <health_url>
   ```

2. Le port est-il correctement exposé ?
   ```bash
   docker port caelix-<app>
   ```

3. Le service met-il du temps à démarrer ? Augmentez le grace period :
   ```ini
   post_repair_grace = 10
   ```

4. En mode strict, l'URL cible-t-elle localhost ?
   ```bash
   CAELIX_STRICT_LOCAL=1 bin/caelix doctor
   ```

### Le service redémarre en boucle

**Symptôme** : restart constant, notifications en rafale.

**Causes possibles** :

- **OOM** : le conteneur est tué par manque de mémoire
  ```bash
  docker inspect caelix-<app> | grep OOMKilled
  ```
  Solution : augmenter `memory_limit_mb`

- **Crash au démarrage** : l'application plante immédiatement
  ```bash
  docker logs caelix-<app>
  ```

- **Port occupé** : le port est déjà utilisé par un autre processus
  ```bash
  ss -tlnp | grep <port>
  ```

### Les notifications Discord ne fonctionnent pas

**Vérifications** :

1. Le webhook est-il configuré ?
   ```bash
   cat etc/notify.ini
   ```

2. Le webhook est-il valide ?
   ```bash
   curl -X POST <webhook_url> \
     -H "Content-Type: application/json" \
     -d '{"content": "Test Caelix"}'
   ```

3. `enabled` est-il à `1` ?

### L'autoscale ne fonctionne pas

**Vérifications** :

1. `autoscale = 1` est-il défini ?
2. `socat` est-il installé ?
   ```bash
   which socat
   ```

3. La plage de ports est-elle libre ?
   ```bash
   ss -tlnp | grep -E '185[0-9]{2}'
   ```

4. Vérifier les backends :
   ```bash
   cat .caelix/autoscale/<app>.backends
   ```

### La console web ne se connecte pas

**Vérifications** :

1. Le conteneur UI tourne-t-il ?
   ```bash
   docker ps | grep caelix-ui
   ```

2. Le socket Docker est-il monté ?
   ```bash
   docker inspect caelix-ui | grep docker.sock
   ```

3. Le token d'auth est-il correct (si activé) ?

### Cluster : une action, un log ou une sauvegarde semble sans effet

En cluster, un service tourne sur **un nœud précis** — pas forcément celui qui porte
la VIP. La console résout automatiquement le nœud hôte pour chaque action/vue ; en
ligne de commande, un `docker ps`/`docker logs caelix-<app>` sur le nœud VIP ne verra
**pas** un conteneur hébergé ailleurs.

- **Trouver le nœud d'un service** : `caelix vip-status` (leader/VIP), puis
  `docker ps --filter name=caelix-<app>` sur chaque nœud, ou l'état observé publié dans
  le store (`caelix/nodes/<id>/observed`).
- **Agir sur le bon nœud** : passez par la console (elle cible le nœud de la ligne), ou
  ciblez le démon distant via l'en-tête `X-Caelix-Node` / `DOCKER_HOST=tcp://<ip-nœud>:2375`.
- **Sauvegardes** : elles s'exécutent et sont stockées **sur le nœud qui détient les
  données** ; `status`/`list`/`download` agrègent les nœuds. Une archive absente du nœud
  VIP est normale — elle est sur le nœud hôte.
- **Après une bascule VIP** : la console, l'ingress HTTPS (certs + routes répliqués) et
  toutes les actions cluster-aware continuent sur le nouveau leader. Si l'ingress ne
  répond pas, vérifiez que le nœud VIP porte bien `:80`/`:443`
  (`ss -tlnp | grep -E ':80 |:443 '`) et que `routes.conf` est présent
  (`.caelix/autoscale/routes.conf`).

## Logs

### Logs daemon (JSON lines)

```bash
# Derniers logs
tail -20 .caelix/logs/caelix-daemon.log

# Formatter en JSON lisible
tail -5 .caelix/logs/caelix-daemon.log | python3 -m json.tool

# Filtrer par niveau
grep '"level":"error"' .caelix/logs/caelix-daemon.log
```

### Logs conteneur

```bash
docker logs caelix-<app> --tail 50
docker logs caelix-<app> -f  # suivi temps réel
```

### Incidents

```bash
# Texte
cat .caelix/incidents/incidents.log

# JSONL (avec jq)
cat .caelix/incidents/$(date +%Y-%m-%d).jsonl | jq .
```

## Mode debug

Pour plus de détails dans les logs :

```bash
CAELIX_LOG_LEVEL=debug bin/caelix once
```

Ou dans le manifest :

```ini
[orchestrator]
log_level = debug
```

## Reset complet

!!! danger "Attention"
    Cette opération supprime tout l'état de Caelix. Les conteneurs ne sont pas affectés.

```bash
# Supprimer tout l'état
rm -rf .caelix/

# Relancer
bin/caelix once
```

Caelix recréera le répertoire `.caelix/` et reconvergera vers l'état désiré.
