# Fork local de trilium_ynh (Elektro121)

Ce dépôt est un fork de <https://github.com/YunoHost-Apps/trilium_ynh>, hébergé
pour un usage personnel. Il n'est pas destiné à être installé par quelqu'un d'autre.

## Le seul changement

Le paquet d'origine ne déclare qu'une permission YunoHost : `main`, sur `/`.
Tout le chemin de l'application est donc derrière le SSO de YunoHost.

Conséquence : l'API REST **ETAPI** de Trilium (`/etapi`) est elle aussi capturée
par le SSO. Un client externe (serveur MCP, script, app desktop) qui envoie
`Authorization: <token>` reçoit une redirection `302` vers `/yunohost/sso`
au lieu d'une réponse d'API, et le token n'est jamais lu.

Ce fork ajoute une seconde permission :

```toml
etapi.url = "/etapi"
etapi.show_tile = false
etapi.allowed = "visitors"
```

Résultat :

- `https://<domaine>/<chemin>/etapi/...` n'est **plus** intercepté par le SSO.
- Le reste de l'application (`main`) reste derrière le SSO, inchangé.
- La protection de l'API est le **token ETAPI**, pas le SSO. Un token révoqué
  coupe l'accès ; il n'y a pas d'autre barrière sur ce chemin.

La version est passée à `0.106.1~ynh2` pour que `yunohost app upgrade` accepte
de rejouer l'installation des permissions. Aucun autre fichier du paquet n'est
modifié : ni le service systemd, ni nginx, ni la configuration Trilium.

## Mettre à jour une instance existante

```bash
sudo yunohost app upgrade trilium -u https://github.com/Elektro121/trilium_ynh
```

Les données (`/home/yunohost.app/trilium`) ne sont pas touchées : le script
`upgrade` du paquet conserve la base et réécrit la configuration.

Vérifier ensuite que la permission existe :

```bash
sudo yunohost user permission list | grep -A3 trilium
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: <token>" \
  https://<domaine>/<chemin>/etapi/app-info     # attendu : 200
```

Un `302` vers `/yunohost/sso` signifie que la permission n'a pas été créée :
relancer l'upgrade et lire la sortie.

## Revenir au paquet officiel

```bash
sudo yunohost app upgrade trilium -u https://github.com/YunoHost-Apps/trilium_ynh
```

La permission `etapi` n'est plus déclarée : le paquet la supprime, et le chemin
repasse derrière le SSO.

## Note d'usage

Exposer `/etapi` sans SSO rend aussi joignable publiquement l'endpoint
`/etapi/auth/login` (authentification par mot de passe, protégée par le rate
limit de Trilium). Si l'accès doit rester strictement local, préférer le port
direct `http://<ip-lan>:8095/etapi` qui ne passe ni par nginx ni par le SSO.
