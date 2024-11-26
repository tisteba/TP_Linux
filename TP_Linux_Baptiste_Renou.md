

# TP Avancé : "Mission Ultime : Sauvegarde et Sécurisation"
## Contexte
Votre serveur critique est opérationnel, mais de nombreuses failles subsistent. Votre objectif est d'identifier les faiblesses, de sécuriser les données et d’automatiser les surveillances pour garantir un fonctionnement sûr à long terme.
---
## Objectifs
1. Surveiller les répertoires critiques pour détecter des modifications suspectes.
2. Identifier et éliminer des tâches malveillantes laissées par des attaquants.
3. Réorganiser les données pour optimiser l’espace disque avec LVM.
4. Automatiser les sauvegardes et surveillances avec des scripts robustes.
5. Configurer un pare-feu pour protéger les services actifs.
---
## Étape 1 : Analyse et nettoyage du serveur
1. **Lister les tâches cron pour détecter des backdoors** :
   - Analysez les tâches cron de tous les utilisateurs pour identifier celles qui semblent malveillantes:

```ps

   [root@localhost ~]# sudo crontab -u attacker -l

```


2. **Identifier et supprimer les fichiers cachés** :
   - Recherchez les fichiers cachés dans les répertoires `/tmp`, `/var/tmp` et `/home`.
   - Supprimez tout fichier suspect ou inconnu:

```ps
[root@localhost ~]# cd /tmp
[root@localhost tmp]# ls -u
[root@localhost tmp]# rm malicious.sh
rm: remove regular file 'malicious.sh'? y
[root@localhost tmp]# ls -alh
[root@localhost tmp]# rm .hidden_file
rm: remove regular file '.hidden_file'? y
[root@localhost tmp]# rm .hidden_script
rm: remove regular file '.hidden_script'? y

[root@localhost /]# cd /var/tmp
[root@localhost tmp]# ls -alh
[root@localhost tmp]# rm .nop
rm: remove regular file '.nop'? y

[root@localhost home]# cd /home
[root@localhost home]# ls -alh
[root@localhost home]# rm -rf attacker

```




3. **Analyser les connexions réseau actives** :
   - Listez les connexions actives pour repérer d'éventuelles communications malveillantes:

```ps
[root@localhost ~]# ss
```

---
## Étape 2 : Configuration avancée de LVM
1. **Créer un snapshot de sécurité pour `/mnt/secure_data`** :
   
```ps
[root@localhost ~]# sudo lvdisplay
[root@localhost ~]# sudo lvcreate -L 1G -s -n secure_data_snapshot /dev/vg_secure/secure_data
```



2. **Tester la restauration du snapshot** :
   - Supprimez un fichier dans `/mnt/secure_data`:
```ps
[root@localhost ~]# cd /mnt/secure_data
[root@localhost secure_data]# ls
lost+found  sensitive1.txt  sensitive2.txt
[root@localhost secure_data]# rm sensitive1.txt
rm: remove regular file 'sensitive1.txt'? y
```
   - Montez le snapshot et restaurez le fichier supprimé.

```ps
[root@localhost secure_data]# sudo mkdir /mnt/secure_data_snapshot
[root@localhost secure_data]# sudo mount /dev/vg_secure/secure_data_snapshot /mnt/secure_data_snapshot
[root@localhost secure_data]# cd /mnt/secure_data_snapshot/
[root@localhost secure_data_snapshot]# ls
[root@localhost secure_data_snapshot]# cp /mnt/secure_data_snapshot/sensitive1.txt /mnt/secure_data


```


3. **Optimiser l’espace disque** :
   - Si le volume logique `secure_data` est plein:

```
[root@localhost ~]# df -h
```
*On voit que le Volume Logique n'est plein qu'a 1%, mais pour le cadre de l'exercice je vais quand même lui ajouter un peu d'espace*
   
    étendez-le en ajoutant de l’espace à partir du groupe de volumes existant:

```ps
[root@localhost vg_secure]# sudo lvextend -L +10G /dev/vg_secure/secure_data

```
*si on voulait ajouter 10 giga, en l'occurence ici ce ne serait probablement pas utile d'en avoir autant*
---



## Étape 3 : Automatisation avec un script de sauvegarde
1. **Créer un script `secure_backup.sh`** :

```ps
[root@localhost home]# dnf install tar
[root@localhost home]# sudo yum install rsync
[root@localhost home]# nano ~/secure_backup.sh
```

```ps
#!/bin/bash


DATE=$(date +%Y%m%d)


SOURCE="/mnt/secure_data"
BACKUP_DIR="/backup"


mkdir -p "$BACKUP_DIR"


rsync -a --exclude '.*' --exclude '*.tmp' --exclude '*~' --exclude '*.swp' "$SOURCE" "$BACKUP_DIR/secure_data_$DATE"

tar -czf "$BACKUP_DIR/secure_data_$DATE.tar.gz" -C "$BACKUP_DIR" "secure_data_$DATE"


rm -rf "$BACKUP_DIR/secure_data_$DATE"
```

2. **Ajoutez une fonction de rotation des sauvegardes** :

*On ajoute cette ligne au script:*

```ps
ls -tp /backup/*.tar.gz | grep -v '/$' | tail -n +8 | xargs -I {} rm -- {}
```


3. **Testez le script** :
   - Exécutez le script manuellement et vérifiez que les archives sont créées correctement:

```ps

[root@localhost home]# chmod u+x ~/secure_backup.sh
[root@localhost home]# sudo chmod 755 /backup
[root@localhost backup]# ~/secure_backup.sh
```
*le fichier d'archive est bien crée au bon endroit*

4. **Automatisez avec une tâche cron** :
   - Planifiez le script pour qu’il s’exécute tous les jours à 3h du matin:

```ps
[root@localhost ~]# crontab -e
```
*on ajoute cette ligne*
```ps
0 3 * * * /bin/bash /root/secure_backup.sh
```

et on vérifie avec :
```ps
[root@localhost ~]# crontab -l
0 3 * * * /bin/bash /root/secure_backup.sh

[root@localhost ~]# chmod +x /root/secure_backup.sh

```

---
## Étape 4 : Surveillance avancée avec `auditd`
1. **Configurer auditd pour surveiller `/etc`** :
   - Ajoutez une règle avec `auditctl` pour surveiller toutes les modifications dans `/etc`:

```ps
[root@localhost ~]# sudo yum install audit
[root@localhost ~]# yum install audit audispd-plugins
[root@localhost ~]# sudo auditctl -w /etc -p wa -k watch_etc

```


2. **Tester la surveillance** :
   - Créez ou modifiez un fichier dans `/etc` et vérifiez que l’événement est enregistré dans les logs d’audit:

```ps
[root@localhost ~]# cd /etc
[root@localhost etc]# mkdir test
[root@localhost etc]# cat /var/log/audit/audit.log | grep "test"
```
*On a bien la ligne correpsondante:

```ps
type=PATH msg=audit(1732550768.120:678): item=1 name="test" inode=5484 dev=fd:00 mode=040755 ouid=0 ogid=0 rdev=00:00 obj=unconfined_u:object_r:etc_t:s0 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0OUID="root" OGID="root"
```


3. **Analyser les événements** :

   - Recherchez les événements associés à la règle configurée et exportez les logs filtrés dans `/var/log/audit_etc.log`:

```ps
[root@localhost etc]# sudo ausearch -k watch_etc > /var/log/audit_etc.log
```



---
## Étape 5 : Sécurisation avec Firewalld
1. **Configurer un pare-feu pour SSH et HTTP/HTTPS uniquement** :
   - Autorisez uniquement les ports nécessaires pour SSH et HTTP/HTTPS.
   - Bloquez toutes les autres connexions.

```ps
[root@localhost etc]# sudo systemctl status firewalld
[root@localhost etc]# sudo firewall-cmd --add-port=22/tcp --permanent
[root@localhost etc]# sudo firewall-cmd --add-port=80/tcp --permanent
[root@localhost etc]# sudo firewall-cmd --add-port=443/tcp --permanent
[root@localhost etc]# sudo firewall-cmd --set-default-zone=drop
[root@localhost etc]# sudo firewall-cmd --reload
```

2. **Bloquer des IP suspectes** :
   - À l’aide des logs d’audit et des connexions réseau, bloquez les adresses IP malveillantes identifiées.

```ps
[root@localhost ~]# sudo ausearch -m USER_AUTH | grep "attacker"
```
*je n'ai pas trouvé de ip correspondant a une activité malveillante mais j'ai bien repéré de l'activité*

```ps
type=USER_AUTH msg=audit(1732470499.659:400): pid=2067 uid=0 auid=0 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:authentication grantors=pam_rootok acct="attacker" exe="/usr/bin/su" hostname=? addr=? terminal=/dev/pts/0 res=success'
```

3. **Restreindre SSH à un sous-réseau spécifique** :
   - Limitez l’accès SSH à votre réseau local uniquement (par exemple, 192.168.x.x):

```ps
[root@localhost ~]# sudo firewall-cmd --permanent --new-zone=local-only
[root@localhost ~]# sudo firewall-cmd --permanent --zone=local-only --add-source=192.168.1.0/24
[root@localhost ~]# sudo firewall-cmd --permanent --zone=local-only --add-service=ssh
[root@localhost ~]# sudo firewall-cmd --permanent --zone=local-only --change-interface=enp0s8
[root@localhost ~]# sudo firewall-cmd --reload
```
---


# DLC : "L'ultime défi du SysAdmin"

---

### Étape 1 : Analyse avancée et suppression des traces suspectes

1. **Rechercher des utilisateurs récemment ajoutés :**
   - Identifiez les utilisateurs ajoutés récemment en inspectant les logs de sécurité:

```ps
[root@localhost ~]# sudo grep "new user" /var/log/secure
```


2. **Trouver les fichiers récemment modifiés dans des répertoires critiques :**
   - Analysez les fichiers modifiés dans `/etc`, `/usr/local/bin` ou `/var` au cours des 7 derniers jours :
     ```ps
[root@localhost ~]# sudo find /etc /usr/local/bin /var -type f -mtime -7
     ```

3. **Lister les services suspects activés :**
   - Vérifiez tous les services activés au démarrage pour identifier ceux qui ne devraient pas être présents :
     ```ps
     [root@localhost ~]# sudo systemctl list-unit-files --state=enabled
     ```

4. **Supprimer une tâche cron suspecte :**
   - Identifiez et supprimez une tâche cron malveillante ajoutée à un utilisateur spécifique, par exemple `attacker` :

```ps
[root@localhost ~]# sudo crontab -u attacker -r
```

*aucune tache n'est n'est associé a "attacker" avec cette commande*
---

## Étape 2 : Configuration avancée de LVM

1. **Créer un snapshot du volume logique** :
   - Prenez un snapshot du volume logique `secure_data` pour sécuriser les données actuelles.

2. **Tester le snapshot** :
   - Montez le snapshot et vérifiez qu’il contient bien les données actuelles.

3. **Simuler une restauration** :
   - Supprimez un fichier dans `/mnt/secure_data`, puis restaurez-le à partir du snapshot.

---

## Étape 3 : Renforcement du pare-feu avec des règles dynamiques

1. **Bloquer les attaques par force brute** :
   - Configurez `firewalld` pour détecter et bloquer automatiquement les connexions répétées sur SSH.

2. **Restreindre l’accès SSH à une plage IP spécifique** :
   - Autorisez uniquement les connexions SSH depuis votre réseau local (192.168.x.x).

3. **Créer une zone sécurisée pour un service web** :
   - Configurez `firewalld` pour que seul HTTP/HTTPS soit autorisé dans une zone dédiée, et attribuez cette zone au serveur.

---

## Étape 4 : Création d'un script de surveillance avancé

1. **Écrivez un script `monitor.sh`** :
   - Surveille en temps réel :
     - Les connexions actives (`ss` ou `netstat`).
     - Les fichiers modifiés dans `/etc`.
   - Log les informations importantes dans `/var/log/monitor.log`.

2. **Ajoutez une alerte par e-mail** :
   - Configurez le script pour envoyer un e-mail lorsqu’un fichier dans `/etc` est modifié.

3. **Automatisez le script** :
   - Ajoutez une tâche cron pour exécuter le script toutes les 5 minutes.

---

## Étape 5 : Mise en place d’un IDS (Intrusion Detection System)

1. **Installer et configurer `AIDE`** :
   - Installez l’outil `AIDE` (Advanced Intrusion Detection Environment).
   - Initialisez la base de données.
   - Configurez `AIDE` pour surveiller les fichiers critiques du système.

2. **Tester `AIDE`** :
   - Faites une modification dans un fichier sous `/etc`.
   - Lancez une vérification avec `AIDE` et observez le rapport.

---