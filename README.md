# aap-automation

Déploiement de **Red Hat Ansible Automation Platform 2.6** en topologie
**containerized enterprise**, pour deux environnements : **production** et
**qualification**.

L'installation est déléguée à la collection officielle
`ansible.containerized_installer`. Ce dépôt **orchestre** l'installeur et
**prépare** les cibles (preflight) ; il ne réimplémente pas l'installeur.

> Les contraintes de projet font autorité et sont figées dans [CLAUDE.md](CLAUDE.md).
> Les lire avant toute contribution.

## Structure du dépôt

```
aap-automation/
├── CLAUDE.md                 # Règles du dépôt (font autorité)
├── README.md
├── ansible.cfg
├── requirements.yml          # Collections requises (galaxy)
├── .ansible-lint
├── .gitignore
├── site.yml                  # Orchestrateur : preflight -> install -> postcheck
├── playbooks/
│   ├── preflight.yml         # Préparation des VM cibles
│   ├── install.yml           # Invoque ansible.containerized_installer.install
│   └── postcheck.yml         # Vérifications post-installation
├── inventories/
│   ├── prod/                 # Environnement PRODUCTION (strictement isolé)
│   │   ├── hosts.yml
│   │   ├── group_vars/       # DOIT vivre à côté de l'inventaire (cf. CLAUDE.md 5.4)
│   │   └── host_vars/
│   └── qual/                 # Environnement QUALIFICATION (strictement isolé)
│       ├── hosts.yml
│       ├── group_vars/
│       └── host_vars/
└── roles/
    ├── preflight/            # Enregistrement Satellite, repos, paquets, PKI, ...
    └── postcheck/            # Idempotence / vérification de service basique
```

## Séparation prod / qual

Deux inventaires strictement distincts. Aucune variable partagée par accident,
aucune valeur d'un environnement atteignable depuis l'autre. Toujours cibler
l'inventaire explicitement :

```bash
# Qualification
ansible-playbook -i inventories/qual/hosts.yml site.yml

# Production
ansible-playbook -i inventories/prod/hosts.yml site.yml
```

## Validation hors ligne (dev hors infrastructure cible)

```bash
ansible-lint
ansible-playbook -i inventories/qual/hosts.yml site.yml --syntax-check
ansible-playbook -i inventories/prod/hosts.yml site.yml --syntax-check
```
