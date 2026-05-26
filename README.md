Vulnérabilité : Script Injection dans GitHub Actions. Référence OWASP CI/CD Top 10 — CICD-SEC-4 (Poisoned Pipeline Execution), CWE-94.
Mécanisme : un workflow interpole directement ${{ github.event.* }} dans une commande run:. GitHub fait la substitution avant que bash parse la ligne → l'attaquant contrôle une partie du shell via le titre d'issue, nom de branche, message de commit, etc.
Impact : RCE dans le runner, vol du GITHUB_TOKEN et des secrets du repo, compromission de la supply chain.
Pourquoi c'est infra : c'est le pipeline CI/CD qui est compromis, pas l'application. La faille est dans la config de l'infra de build.
