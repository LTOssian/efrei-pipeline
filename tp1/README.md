# TP1 — Déploiement d'une landing page sur Alwaysdata

## Compte Alwaysdata

- **Hôte SSH** : `ssh-louisan.alwaysdata.net`
- **Utilisateur** : `louisan`
- **Site déployé** : [https://louisan.alwaysdata.net](https://louisan.alwaysdata.net)

## Déploiement via SCP

```bash
scp index.html louisan@ssh-louisan.alwaysdata.net:www/
```

Le fichier `index.html` est placé dans le répertoire `www/` du serveur Alwaysdata, qui correspond à la racine du site web.

## Vérification

Ouvrir [https://louisan.alwaysdata.net](https://louisan.alwaysdata.net) dans un navigateur. La page doit afficher **SIX SEVEN**.

![Site déployé](screens/site-deployed.png)
