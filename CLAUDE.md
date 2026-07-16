CLAUDE.md — Règles du dépôt `aap-automation`
Ce fichier fige les contraintes du projet. Il fait autorité sur toute session de
travail. Le lire et le respecter avant toute génération de code. En cas de
conflit entre une demande ponctuelle et ces règles, signaler le conflit plutôt que
de contourner silencieusement une règle.
---
1. Objectif du dépôt
Déployer Red Hat Ansible Automation Platform 2.6 en topologie
containerized enterprise, pour deux environnements :
production et qualification.
L'installation elle-même est déléguée à la collection officielle
`ansible.containerized_installer` (playbook `install`). Ce dépôt orchestre
l'installeur et prépare les cibles ; il ne réimplémente pas l'installeur.
---
2. Contexte d'exécution
Développement hors infrastructure cible. Aucun accès réseau aux futurs nœuds
AAP pendant le dev. Tout doit être validable hors ligne : `ansible-lint`,
`ansible-playbook --syntax-check`, `--check` quand c'est possible. Ne jamais
supposer un accès live aux cibles, ne pas inventer de connexion.
Exécution réelle depuis des nœuds seed ansible-core (control nodes), pas
depuis le poste de dev. Le playbook doit être autonome : ne rien présupposer du
poste de dev (chemins locaux, secrets déjà en place, collections déjà installées).
Serveur Red Hat Satellite fraîchement installé. Abonnement, repos, activation
keys et registry conteneurs passent par Satellite. Aucun content view ni activation
key AAP n'existe encore : l'enregistrement et l'abonnement des hôtes font partie du
périmètre (rôle preflight), ils ne sont pas un pré-requis déjà satisfait.
---
3. Architecture cible (ne pas ré-inventer)
Source de vérité sur la mécanique de topologie et de variables : le fichier
`inventory` enterprise du setup bundle AAP 2.6 + le README du bundle. Ce §3
exprime l'INTENTION ; là où l'intention et le bundle divergent sur un FAIT
mécanique (composition de groupe, nom de variable, valeur par défaut), le bundle
fait foi et ce §3 est corrigé — on ne dévie pas de la topologie testée par Red Hat.
AAP 2.6 containerized, Podman rootless sous l'utilisateur `aap`.
Topologie enterprise :
gateway ×2, controller ×2, hub ×2, EDA ×2,
1 hop node + 2 execution nodes.
Redis en mode cluster, 6 nœuds colocalisés sur gateways / hubs / EDA
(source : bundle `inventory` enterprise — gateway ×2 + hub ×2 + EDA ×2).
Interdit sur les controllers, sur les execution nodes et sur la base.
PostgreSQL externe (équipe DBA), TLS `verify-full`, 4 bases distinctes
(gateway, controller/awx, hub/pulp, eda) avec un rôle propriétaire par base.
Images de la plateforme servies par bundle OU registry interne / Satellite
(paramétrable). Le hub sert les Execution Environments au runtime.
Certificats de la PKI d'entreprise (jamais la CA auto-signée de l'installeur).
Les certificats de services répartis sur plusieurs nœuds doivent porter tous les
FQDN concernés en SAN, VIP incluse pour le gateway.
Point d'entrée unique : load balancer (VIP + FQDN) devant les gateways.
---
4. Périmètre du playbook
Preflight / préparation des VM cibles : enregistrement Satellite, repos,
paquets (ansible-core, podman, …), utilisateur `aap` + `linger`, partitionnement,
chrony, firewalld, SELinux, dépôt des certificats PKI, montage du partage du hub,
test de connectivité PostgreSQL en TLS depuis chaque nœud.
Gestion des inventaires enterprise (prod et qual) et des `group_vars`.
Invocation de `ansible.containerized_installer.install`.
Post-checks : idempotence et vérification de service basique.
---
5. Discipline de développement — NON NÉGOCIABLE
5.1 Séparation prod / qual explicite
Deux inventaires strictement distincts : `inventories/prod/`, `inventories/qual/`.
Aucune variable partagée par accident. Aucune valeur de prod atteignable depuis
qual, ni l'inverse.
Vérifier qu'aucun `group_vars`/`host_vars` commun ne fait fuiter une valeur d'un
environnement vers l'autre.
5.2 Verrouillage des variables de banc de test
Toute variable de type test-harness (endpoints factices, credentials de test,
mocks) doit être bloquée en production par une `assert` explicite, placée en
début de play, AVANT toute évaluation qui en dépend.
Vérifier l'ordre d'évaluation : l'assertion doit précéder l'usage de la variable
qu'elle protège (piège déjà rencontré — une variable évaluée avant l'assertion de
verrouillage rend l'assertion inutile).
5.3 Règle `# À VÉRIFIER :`
Si un nom de module, une option, une variable d'inventaire de la collection ou un
comportement n'est pas certain à 100 %, ne pas l'inventer.
Écrire la ligne avec un commentaire `# À VÉRIFIER : <ce dont on n'est pas sûr et où le confirmer>` plutôt qu'un nom plausible mais potentiellement faux.
5.4 Non-invention / affirmations validées
Chaque nom d'option et chaque comportement Ansible utilisé doit correspondre au
comportement réel : doc officielle AAP 2.6 ou `ansible-doc`.
Ne jamais présenter comme acquis un comportement non vérifié. En cas de doute →
`# À VÉRIFIER :`, pas d'invention.
Pièges déjà rencontrés, à ne pas reproduire :
`group_vars/` à la racine du dépôt n'est pas résolu par Ansible ; il doit
vivre à côté de l'inventaire.
`{{ playbook_dir }}` se résout depuis le CWD, pas depuis l'emplacement du
fichier de config.
valider les regex (ex. validation de nom de repo) sur des cas réels avant de les
considérer correctes.
5.5 Qualité Ansible
Idempotence exigée.
`changed_when` / `failed_when` explicites là où le module ne les gère pas.
Pas de `command` / `shell` là où un module existe.
Nommage des tasks clair, `ansible-lint` propre.
5.6 Secrets
Aucun secret en clair. `ansible-vault` (ou placeholders `{{ vault_* }}`) pour
tous les mots de passe : admin des composants, PostgreSQL, registry.
Ne jamais committer de valeur secrète, même en exemple.
5.7 Placement des variables
Constante projet transverse (consommée par plusieurs rôles/briques ou par le
contexte de l'installeur) : vit dans `inventories/<env>/group_vars/all/`, sans
préfixe. Exemples validés : `aap_user`, `ntp_servers`.
Variable spécifique à un rôle : vit dans les `defaults/` du rôle, préfixée par le
nom du rôle (`preflight_*`). C'est aussi ce qu'exige `ansible-lint`
(`var-naming[no-role-prefix]`) en profil production.
L'isolation prod/qual (5.1) prime sur le DRY : une constante projet transverse est
DÉCLARÉE SÉPARÉMENT dans le `group_vars` de CHAQUE environnement, même à valeur
identique. Dupliquer une valeur identique est voulu ; partager un fichier de
variables entre prod et qual est interdit.
---
6. Méthode de livraison
Livraison itérative, un livrable à la fois.
Attendre validation avant de passer au livrable suivant.
Ne pas générer l'ensemble du dépôt d'un coup.
Après chaque livrable, s'arrêter et demander confirmation.
---
7. Rappel de posture
Préférer un `# À VÉRIFIER :` honnête à une réponse confiante mais fausse.
Signaler explicitement toute hypothèse prise.
Distinguer ce qui est documenté (intention) de ce qui est réellement
appliqué / vérifié (comportement constaté).