# Tuto : finir l'installation (12 minutes, 3 collages)

> Objectif : que chaque modification que je pousse finisse **toute seule** en `.jar` sur ton serveur,
> avec les deux fichiers (plugin + économie), une sauvegarde préalable, et un contrôle de taille.
> Tu n'as plus qu'à **redémarrer le serveur** — ça, aucun robot ne le fait à ta place.

Prérequis d'état (déjà vrai, vérifié) : build ✅ `aca3909` (15 étapes), PR #7 **mergeable**, les deux
`.jar` déjà testés sur ton serveur de test.

---

## Étape 1 — Merger la PR #7 (1 clic)

1. Ouvre **https://github.com/Souxch06/ValoriaTycoon/pull/7**
2. Onglet **Conversation** (le premier)
3. Descends tout en bas → bouton vert **Merge pull request** → **Confirm merge**

Sans le faire, rien ne bouge : tout le travail est sur la branche `arena/01a043a8-valoriatycoon`.
C'est sans danger : `main` n'a pour l'instant qu'un ancien `deploy.yml` qui se déclenche sur `push: main`,
et il ne sera neutralisé qu'à l'étape 2 — **enchaîne les deux tout de suite**, dans cet ordre.

---

## Étape 2 — Couper l'ancien déploiement (1 collage)

Pourquoi : l'ancien workflow n'envoie **qu'un seul** `.jar`. S'il tourne encore à chaque merge, il
laisse le serveur avec le plugin **sans son économie**.

1. Ouvre **https://github.com/Souxch06/ValoriaTycoon/edit/main/.github/workflows/deploy.yml**
2. Dans la zone de texte : **appui long → Select all → Delete** (ou Ctrl+A puis Suppr)
3. Colle le **bloc A** ci‑dessous (copie‑colle depuis le fichier `docs/paste/deploy-neutralise.yml` du
   dépôt, ou depuis ce tuto si tu le lis en version brute)
4. Déroule tout en bas → bouton vert **Commit changes** (laisse « Commit directly to `main` »)

## Étape 3 — Installer les deux nouveaux workflows (2 collages)

### 3.1 `build.yml` (celui qui publie la release qui déclenche le dépôt)
1. **https://github.com/Souxch06/ValoriaTycoon/edit/main/.github/workflows/build.yml**
   *(si « 404 / file not found » : « Add file → Create new file » et tape le nom
   `.github/workflows/build.yml`)*
2. Select all → Delete → colle le **bloc B**
3. **Commit changes**

### 3.2 `deploy-serveur.yml` (celui qui dépose les trois jar)
1. **https://github.com/Souxch06/ValoriaTycoon/new/main?filename=.github%2Fworkflows%2Fdeploy-serveur.yml**
   *(le nom du fichier est déjà rempli par ce lien)*
2. Colle le **bloc C**
3. **Commit changes** → GitHub affiche le fichier créé, c'est bon

⏱ Après ce dernier commit : `main` a changé → le build se relance → s'il est vert, la release
`build-latest` est publiée → le dépôt se déclenche **en mode simulation** (`dry_run` par défaut, il
n'envoie rien). C'est voulu : on regarde d'abord.

---

## Étape 4 — Tester sans risque, puis armer (2 clics)

1. **https://github.com/Souxch06/ValoriaTycoon/actions/workflows/deploy-serveur.yml**
2. **Run workflow** ▾ → branche `main` → laisse **`dry_run` coché** → **Run workflow**
3. Ouvre le run → étape **« Déploiement (simulation) »** : tu dois voir la liste exacte de ce qui serait
   fait, du genre
   ```
   -mkdir plugins/_sauvegarde-20260828-131502
   rename plugins/ValoriaTycoon-v1.6.3.jar plugins/_sauvegarde-…/ValoriaTycoon-v1.6.3.jar
   rename plugins/ValoriaEconomy-v1.6.3.jar plugins/_sauvegarde-…/ValoriaEconomy-v1.6.3.jar
   put target/ValoriaTycoon-v1.6.3.jar target/ValoriaEconomy-v1.6.3.jar plugins/
   ```
4. Si ça te va : refais **Run workflow**, **décoche `dry_run`** → les trois jar partent vraiment sur le
   serveur (les anciens sont d'abord sauvegardés), puis **redémarre le serveur**.

**Filet recommandé** (au choix, 30 s) : **Settings → Security → Environments → production →
Edit → Required reviewers → ajoute‑toi → Save**. Dès lors, chaque dépôt attendra ton
**Review deployments** avant d'écrire sur le serveur.

---

## Étape 5 — Ensuite, ton seul réflexe
| tu veux | tu fais |
| --- | --- |
| une correction / une nouvelle fonctionnalité | tu me le demandes → je pousse → build → je te dis « merge » → tu cliques → les jar arrivent sur le serveur |
| savoir si c'est bon | https://github.com/Souxch06/ValoriaTycoon/actions (vert = livré) |
| récupérer un jar à la main | https://github.com/Souxch06/ValoriaTycoon/releases/tag/build-latest |
| revenir en arrière | dans `plugins/_sauvegarde-<date>/`, un `rename` inverse + redémarrage |
| déployer sans rien changer | Actions → deploy-serveur → Run workflow (dry_run décoché) |

---

## Les 3 blocs (à copier tels quels, en entier)

> Ouvre ce tuto en `raw` (bouton **Copy raw content** en haut à droite du fichier) et copie les blocs
> depuis là : dans le rendu GitHub, les tabulations et les retraits restent intacts, mais en `raw` tu es
> certain de ne rien perdre.

### Bloc A → `.github/workflows/deploy.yml` (neutralisé)

```yaml
name: Build and Deploy ValoriaTycoon

on:
  # Neutralise le declencheur automatique : le depot des DEUX jars est assure par
  # deploy-serveur.yml (declenche par la release `build-latest`). Ce fichier ne sert plus
  # qu'a un declenchement manuel, et a l'historique.
  workflow_dispatch:

jobs:
  build-and-deploy:
    name: Build et Déploiement SFTP
    runs-on: ubuntu-latest

    steps:
      - name: Récupérer le code
        uses: actions/checkout@v4

      - name: Installer Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '25'
          cache: maven

      - name: Compiler le plugin
        run: mvn clean package -DskipTests

      - name: Identifier le JAR final du plugin
        id: select-jar
        run: |
          mkdir -p target/deploy
          # Recherche du JAR final en excluant les JARs annexes (original-*, sources, javadoc)
          FINAL_JAR=$(find target -maxdepth 1 -type f -name "*.jar" \
            ! -name "original-*" \
            ! -name "*-sources.jar" \
            ! -name "*-javadoc.jar" \
            | head -n 1)

          if [ -z "$FINAL_JAR" ]; then
            echo "::error::Aucun JAR final du plugin trouvé dans target/ !"
            exit 1
          fi

          echo "JAR final identifié : $(basename "$FINAL_JAR")"
          cp "$FINAL_JAR" target/deploy/
          echo "jar_path=target/deploy/$(basename "$FINAL_JAR")" >> "$GITHUB_OUTPUT"

      - name: Envoyer le plugin sur MCServerHost via SFTP
        env:
          SFTP_HOST: ${{ secrets.SFTP_HOST }}
          SFTP_PORT: ${{ secrets.SFTP_PORT }}
          SFTP_USERNAME: ${{ secrets.SFTP_USERNAME }}
          SSHPASS: ${{ secrets.SFTP_PASSWORD }}
          JAR_PATH: ${{ steps.select-jar.outputs.jar_path }}
        run: |
          sudo apt-get update -y && sudo apt-get install -y sshpass
          sshpass -e sftp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P "$SFTP_PORT" "$SFTP_USERNAME@$SFTP_HOST" <<EOF
          put "$JAR_PATH" plugins/
          bye
          EOF
```

### Blocs B et C → les deux workflows, sans recopie

Ils ne sont **pas collés dans ce document** : un contenu recopié diverge, et c'est arrivé ici
(la copie du tuto décrivait un build « deux jar », sans l'étape de release). Les deux fichiers à
coller sont donc lus **à la source**, par URL brute :

| à coller dans | contenu à copier depuis |
| --- | --- |
| `.github/workflows/build.yml` | https://github.com/Souxch06/ValoriaTycoon/raw/arena/01a043a8-valoriatycoon/docs/CI-A-COLLER.yml |
| `.github/workflows/deploy-serveur.yml` | https://github.com/Souxch06/ValoriaTycoon/raw/arena/01a043a8-valoriatycoon/docs/CI-DEPLOY-A-COLLER.yml |

Ouvre le lien, `Ctrl+A` puis `Ctrl+C`, va dans l'éditeur du fichier cible sur GitHub, `Ctrl+A`,
`Suppr`, `Ctrl+V`, puis **Commit**. Si le fichier cible n'existe pas encore (`deploy-serveur.yml`),
crée-le d'abord avec un contenu vide depuis `Add file → Create new file`.

`scripts/verify-ci-copies.py` (une étape du build) refuse désormais toute divergence entre ces
fichiers : la copie de travail `scripts/ci/build-workflow.yml` doit rester identique octet pour
octet à `docs/CI-A-COLLER.yml`.




Ils existent aussi en fichiers, avec bouton « Copy raw content », si c'est plus pratique :
https://github.com/Souxch06/ValoriaTycoon/tree/arena/01a043a8-valoriatycoon/docs/paste

Les trois blocs complets sont juste en dessous, dans ce fichier — copie depuis la version
brute (`raw`) pour ne pas casser l'indentation YAML.

Les blocs B et C sont **exactement** les fichiers `docs/CI-A-COLLER.yml` et `docs/CI-DEPLOY-A-COLLER.yml`
validés par le build vert `aca3909`. Une indentation YAML perdue = workflow invalide, et le refus
s'affiche sans détail : ne réécris rien à la main, copie. Le bloc A n'est que le `deploy.yml` actuel de
`main` avec son déclencheur `push:` retiré (rien d'autre n'a bougé, donc rien d'autre à re‑vérifier).

---
