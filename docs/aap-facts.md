# Tableau de bord aap-facts — SOURCE DE VÉRITÉ VERSIONNÉE

<!--
Ce fichier est la source de vérité versionnée du tableau de bord. La mémoire de session
peut en refléter une copie, mais GIT FAIT FOI. DÉSORMAIS : toute mise à jour significative
doit finir committée ici (docs/aap-facts.md). La mémoire de session reste un cache pratique ;
fini le mirroring `cp` entre racines de workspaceStorage (cf. section « MÉMOIRE & RÉCONCILIATION »).
Migré depuis la mémoire de session (racine canonique …ba8) le 2026-07-16, via l'outil mémoire.
-->

# aap-automation — faits vérifiés (AAP 2.6 containerized)

<!-- MAJ 2026-07-16 (micro-livrable réconciliation) : tableau de bord RAPATRIÉ depuis l'ancien
     workspace claude_app (workspaceStorage e1ce58f…, orphelin après renommage claude_app -> deploy_aap)
     vers le workspace courant deploy_aap. 2 points périmés corrigés : (a) seuils stockage = RÉSOLU
     (15/10 GB sourcés, sortis de catégorie B) ; (b) dépôt = SOUS git (origin/main, commit 23095b5).
     Annexe aap-storage-thresholds.md FUSIONNÉE ici (section « STOCKAGE — SEUILS SOURCÉS »). Voir aussi
     la section finale « MÉMOIRE & RÉCONCILIATION » pour la nuance du suffixe d'instance -1. -->

## Collection installeur
- `ansible.containerized_installer` N'EST PAS sur Galaxy public.
- Livrée DANS le setup bundle AAP 2.6 containerized (tarball extrait).
- Dev hors ligne: extraire le bundle localement + collections_path -> bundle/collections.
- Sans bundle, --syntax-check du play install échoue (attendu).

## Confirmé
- FQCN install: `ansible.containerized_installer.install` (correct).
- Groupes inventaire ENTERPRISE (external PG), source inventory L9-57:
  automationgateway, automationcontroller, execution_nodes, automationhub,
  automationeda, redis. PAS de [database] dans l'exemple enterprise.
- [database] existe SEULEMENT dans inventory-growth (L48), topo DB colocalisée.
  => TRANCHÉ: external PG => PAS de groupe [database]. Retiré des 2 hosts.yml.
     Base externe déclarée uniquement via *_pg_host en group_vars.
- Collection version 2.6.0 (MANIFEST.json). Playbook install: playbooks/install.yml
  => FQCN ansible.containerized_installer.install CONFIRMÉ.
- Forme d'invocation CONFIRMÉE: le bundle playbooks/install.yml est un playbook
  MULTI-PLAY (1er play hosts: localhost preflight, puis groupes). => on l'appelle
  via `ansible.builtin.import_playbook` (PAS un rôle, PAS include). L'assert de
  verrouillage vit dans playbooks/assert_env_lock.yml, importé AVANT dans install.yml.
- Verrou (assert_env_lock.yml, hosts: all/run_once/gather_facts:false, action locale
  => hors ligne OK): (1) prod interdit test_harness_*; (2) prod exige
  <svc>_pg_sslmode=verify-full x4; (3) couplage universel verify-full => custom_ca_cert
  défini+non vide (vérifie la VAR, pas le fichier; dépôt PEM = preflight).
  custom_ca_cert posé dans prod+qual group_vars (path À VÉRIFIER).

## Bundle local (source de vérité)
- Chemin: bundles/ansible-automation-platform-containerized-setup-bundle-2.6-10.1-x86_64/
- collections_path -> <bundle>/collections (contient ansible_collections/).
- ansible-doc -l résout ansible.containerized_installer via ce chemin (OK).
- Fichiers: `inventory` (enterprise), `inventory-growth`, README.md (table de vars), bundle/, images/.
- Collections vendorées: ansible.containerized_installer, ansible.controller,
  ansible.platform, ansible.posix, community.crypto, community.postgresql,
  containers.podman, infra.ah_configuration, infra.controller_configuration.

## CONFIRMÉ contre bundle (source: fichier:ligne)
- receptor_type: hop node = `receptor_type='hop'` (inventory L22) dans [execution_nodes];
  execution = défaut `receptor_type=execution` (README L735).
- Registry (README L55-61): registry_url (def registry.redhat.io), registry_username,
  registry_password, registry_tls_verify (def true), registry_ns_aap/ns_rhel.
- Bundle (inventory ~L70): bundle_install=true, bundle_dir='{{ lookup env PWD }}/bundle'
  (le chemin DOIT inclure /bundle).
- PG commun (README L692-706): postgresql_admin_username(def postgres),
  postgresql_admin_password, postgresql_admin_database(def postgres),
  postgresql_disable_tls(def false), postgresql_tls_cert/key/remote.
- PG par service <svc> in {gateway,controller,hub,eda}: <svc>_pg_host, <svc>_pg_port(5432),
  <svc>_pg_database, <svc>_pg_username, <svc>_pg_password, <svc>_pg_sslmode(def prefer),
  <svc>_pg_tls_cert, <svc>_pg_tls_key, <svc>_pg_cert_auth(def false).
  DB défaut: gateway=gateway, controller=awx, hub=pulp, eda=eda.
  => verify-full = poser <svc>_pg_sslmode=verify-full (défaut prefer).
- TLS service: <svc>_tls_cert, <svc>_tls_key, <svc>_tls_remote(false) pour
  gateway/controller/hub/eda; aussi postgresql_tls_*, receptor_tls_*.
- CA/PKI (README L43-51): ca_tls_cert, ca_tls_key, ca_tls_key_passphrase,
  ca_tls_remote(false), custom_ca_cert.
- Redis (README L744-748): redis_mode(def cluster), redis_port(6379),
  redis_disable_tls(false), redis_cluster_ip, redis_prefer_ipv6.
- Groupe [redis] enterprise example = gateways+hubs+eda (PAS controllers).
  => TRANCHÉ: on suit l'exemple (gw+hub+eda). hosts.yml corrigés + CLAUDE.md §3
     corrigé (compo Redis) + règle ajoutée "bundle fait foi sur la mécanique".

## Règle gravée (CLAUDE.md §3)
- Source de vérité topologie/variables = `inventory` enterprise + README du bundle.
  §3 = intention; si divergence sur un FAIT mécanique, le bundle gagne, §3 corrigé.

## À surveiller (mordra plus tard)
- <svc>_pg_sslmode défaut = `prefer` (PAS verify-full). Exigence CLAUDE.md verify-full
  => poser EXPLICITEMENT sur les 4 services + VERROUILLER par assert au play install.
- community.general ABSENT du bundle. Si preflight utilise community.general,
  ne se résoudra pas via collections_path bundle. Options: rester sur vendorées
  (ansible.posix, community.crypto, community.postgresql, containers.podman) OU
  requirements.yml distinct pour preflight.

## Preflight brique 1 (dépôt PKI + connectivité PG TLS) — CONFIRMÉ contre collection
- roles/preflight/tasks/postgresql.yml (collection): test PG = build bundle CA temp
  (copy /etc/pki/tls/certs/ca-bundle.crt remote_src:true + copy custom_ca_cert
  remote_src=ca_tls_remote + slurp + blockinfile append) puis
  community.postgresql.postgresql_ping(ca_cert=<temp>/ca-bundle.crt, ssl_mode=<svc>_pg_sslmode,
  login_host/port/user/password/db), failed_when: conn_err_msg|length.
- ca_tls_remote (défaut false): false => PEM lu sur le SEED (controller/localhost);
  true => sur le NŒUD (remote_src). Confirmé (copy remote_src: ca_tls_remote).
- Checks PG gardés par appartenance au groupe du composant (nodes.yml:
  inventory_hostname in groups.get('automationhub',[]) etc.).
- Notre rôle preflight: tasks/main.yml inclut pki_ca.yml (1a, tag preflight_ca) puis
  postgres_connectivity.yml (1b, tag preflight_pg) -> boucle preflight_pg_services
  (vars/main.yml) -> postgres_ping_service.yml (gardé par groupe).
- 1a: garde stat+assert présence du fichier custom_ca_cert (sur seed si !ca_tls_remote,
  sinon nœud). Démontré hors ligne (-c local): absent=>fail, présent=>ok.
- Dépendance déclarée: roles/preflight/requirements.yml (community.postgresql,
  vendorée bundle 3.14.3 + Galaxy). community.general NON utilisé (absent bundle).
- Ping PG réel NON exécutable en dev (base externe non joignable): syntax-check + lint OK.

## Enregistrement Satellite — méthode retenue (résolu, sourcé)
- Bundle NE vendorise PAS redhat.satellite ni community.general (source: collections tree).
  Vendorées: ansible.posix, community.crypto, community.postgresql, containers.podman, infra.*.
- L'installeur containerized NE fait AUCUN enregistrement RHSM/Satellite (les "register"
  = awx-manage register_peers/queue, sans rapport). Il suppose l'hôte déjà enregistré +
  repos actifs (ansible.builtin.package). => enregistrement = notre preflight.
- Méthodes comparées:
  a. community.general.redhat_subscription (+ rhsm_repository): 100% module (via
     subscription-manager D-Bus), pas de command/shell. Supporte activationkey, org_id,
     server_hostname (Satellite), environment, rhsm_repo_ca_cert. (source: docs.ansible.com,
     community.general 13.2.0). rhsm_repository: enable repos par ID, exige hôte déjà enregistré.
  b. Global Registration 6.19 = curl|bash portant JWT+AK (source: Satellite 6.19 Managing
     hosts §6.2). Exécution = shell => viole "pas de command/shell".
  c. redhat.satellite.registration_command (=theforeman.foreman): GENERE la commande via API
     (delegate_to localhost, needs python requests), mais l'EXECUTION sur l'hôte = 
     ansible.builtin.shell "{{ registration_command }}" (source: §6.2.6 + foreman-ansible-modules).
     redhat.satellite ABSENT du bundle.
- RECO: méthode (a) community.general redhat_subscription + rhsm_repository (anti-command OK,
  idempotent). Dépendance EXTERNE: community.general (pas dans le bundle) -> à fournir au seed
  (Galaxy/Automation Hub), déclarée dans roles/preflight/requirements.yml.
- Prérequis: CA Satellite déployée dans le truststore de l'hôte AVANT enregistrement
  (rhsm_repo_ca_cert ou /etc/rhsm/ca/) — cf. §6.2.2.
- À VÉRIFIER (infra, non figeable): nom activation key (n'existe pas encore), organization/org_id +
  location, FQDN Satellite + sa CA, IDs de repos exacts (baseos/appstream/aap-2.6) si non portés
  par l'AK.

## SELinux — résolution périmètre (sourcé, AUCUNE task écrite)
GÉRÉ PAR L'INSTALLEUR (ne pas toucher):
- setype data_home_t sur $HOME/aap/containers/storage (common/tasks/executionplane.yml:6).
- Relabel des bind-mounts via :z/:Z (dizaines: controller/gateway/hub/eda/redis/receptor/
  pcp/postgresql/lightspeed tasks).
- security_opt ['label=disable'] sur UN conteneur controller (automationcontroller/tasks/containers.yml:43).
- Volume nommé podman :U (chown) hors NFS: hub_data:/var/lib/pulp:U (automationhub/tasks/facts.yml:34).
- HUB NFS: l'installeur MONTE lui-même le partage (automationhub/tasks/nfs.yml:22, ansible.posix.mount,
  src=hub_shared_data_path path=hub_data_dir opts=hub_shared_data_mount_opts). En mode NFS, bind
  hub_data_dir:/var/lib/pulp SANS :z/:Z/:U (volontaire, pas de relabel per-file sur NFS) + volume
  nommé hub_tmp:/var/tmp/pulp:U (facts.yml:44-50). hub_data_dir défaut=aap_volumes_dir/hub/data.
PRÉSUPPOSÉ / À NOTRE CHARGE (via variables, pas via task SELinux):
- SELinux state: AUCUN precheck dans l'installeur (grep preflight/ vide). Assert enforcing = choix
  projet, PAS exigence installeur => À VÉRIFIER doc AAP 2.6 containerized.
- NFS+SELinux: hub_shared_data_mount_opts défaut 'rw,sync,hard' SANS context=. Pour enforcing, poser
  context=system_u:object_r:container_file_t:s0 dans hub_shared_data_mount_opts (VARIABLE group_vars,
  pas une task). AUCUN booléen NFS posé par l'installeur (grep virt_use_nfs/container_use_nfs vide)
  => ne PAS inventer de booléen; la voie propre = context= au montage (fait par l'installeur).
- setype via ansible.builtin.file exige python3-libselinux/policycoreutils présents => À VÉRIFIER si
  défaut RHEL (sinon ajouter aux paquets preflight).
RECO brique SELinux future: se limiter à ASSERT de l'état SELinux (enforcing, si mandaté) — lecture
seule; NE PAS reposer contextes/:z/booléens (installeur). NFS = valeur de hub_shared_data_mount_opts.


## Firewalld / ports — résolution (sourcé, AUCUNE task écrite)
GÉRÉ PAR L'INSTALLEUR (chaque rôle a tasks/firewalld.yml, ansible.posix.firewalld,
permanent+immediate, zone=<comp>_firewall_zone, gaté « firewalld.service enabled »):
- gateway (_gateway_ports): envoy_http 80, envoy_https 443, gateway_nginx_http 8083,
  gateway_nginx_https 8446 (automationgateway/defaults:22-23,35-36 ; facts:6,14,19,24).
- controller (_controller_ports): nginx_http 8080, nginx_https 8443 (defaults:60-61 ; facts:6,11).
- hub (_hub_ports): nginx_http 8081, nginx_https 8444 (defaults:68-69 ; facts:6,12).
- eda (_eda_ports): nginx_http 8082, nginx_https 8445 (defaults:71-72 ; facts:6,17).
- redis: 6379/tcp + 16379/tcp (cluster bus, quand redis_cluster) (redis/defaults:15,19 ; firewalld.yml).
- receptor: receptor_port 27199 /(receptor_protocol=tcp) (receptor/defaults:17 ; firewalld.yml).
- optionnels (si groupes présents): metrics/pcp/lightspeed/mcp/postgresql(role).
NON firewallés (loopback, PAS dans _*_ports): uwsgi 8050/8052, daphne 8051/8001, gunicorn 8000,
  gateway_control_plane 50051, hub api 24817 / content 24816.
AUCUN precheck de ports dans preflight/ (grep vide). Zones = *_firewall_zone (défaut À VÉRIFIER).
À NOTRE CHARGE / INFRA (l'installeur n'y touche pas):
- 5432 vers PG EXTERNE = sortant -> firewall INBOUND côté DBA (postgresql role non utilisé en DB externe).
- SSH 22 exec -> nœuds gérés = sortant, infra. À VÉRIFIER.
- externe/LB -> VIP:443 = réseau/LB (host gateway:443 ouvert par l'installeur).
RECO brique firewalld: MINIMALE/quasi-nulle (installeur ouvre tout). Au plus assert d'état firewalld
(comme SELinux) + *_firewall_zone en group_vars si zone non défaut. NE PAS reposer les ports.
Sens mesh receptor exact = RÉSOLU (2026-07-16, voir section « MESH RECEPTOR — SENS DU PEERING »).
NGINX 8443/8444/8445/8446 = INTER-nœuds ;
backends (uwsgi/daphne/gunicorn/api/content/50051) = loopback.


## MESH RECEPTOR — SENS DU PEERING (RÉSOLU 2026-07-16, sourcé bundle fichier:ligne)
Source : rôle bundle ansible.containerized_installer/roles/receptor + inventory enterprise.
FAITS confirmés (qui écoute / qui initie / qui exécute) :
- Inventory enterprise (inventory L21-24) : [execution_nodes] = hop1 (receptor_type='hop'), exec1, exec2.
  AUCUN receptor_peers déclaré. Peering NON exprimé dans l'inventaire => DÉRIVÉ par l'installeur.
- Type de nœud (receptor/tasks/facts.yml L1-4) : automationcontroller => 'hybrid' (control+exec) ;
  hop1 => 'hop' (inventory) ; exec => 'execution' (receptor/defaults/main.yml:19).
- receptor_peers auto-posé UNIQUEMENT sur les controllers (facts.yml L22-31, gardé
  `when: inventory_hostname in groups.automationcontroller`). Défaut d'un controller =
  union( autres controllers [combinations(2), facts.yml L14-20], execution_nodes non "targeted" ).
  * peers = tous les execution_nodes MOINS `targeted` (facts.yml L27-30) ;
  * targeted = les execution_nodes nommés dans le receptor_peers EXPLICITE d'un autre execution_node.
    Sans receptor_peers explicite (inventaire par défaut) => targeted=[] => controller pose
    peers=[hop1, exec1, exec2].
- hop/exec : _receptor_peers = receptor_peers|default([]) (facts.yml L34-35) => VIDE par défaut =>
  ils ne DIALENT vers personne.
- Config générée (templates/receptor.conf.j2) :
  * tcp-listener sur 27199 sur CHAQUE nœud (L52-58), sauf cas dégénéré (1 controller & 0 exec =>
    local-only). => hop + exec + controllers ÉCOUTENT tous sur 27199.
  * tcp-peer {address:<peer>:27199, redial, tls_client} par nœud de _receptor_peers (L61-69).
    Un tcp-peer = CE nœud COMPOSE/INITIE la connexion TCP sortante vers le listener cible.
  * work-command seulement si receptor_type in [control, execution, hybrid] (L71-77) => le HOP n'a
    PAS de work-command : il N'EXÉCUTE AUCUN job, c'est un pur RELAIS/forwarder.
MODÈLE CONFIRMÉ (enterprise mono-site, défaut, aucun receptor_peers explicite) :
  * INITIATEUR TCP = les controllers UNIQUEMENT. Ils DIALENT vers hop1, exec1, exec2 et entre eux.
  * hop1 + exec1 + exec2 = LISTENERS passifs (n'initient rien). hop1 = relais sans exécution.
  * Sens TCP (controller -> exec/hop) peut différer du flux logique des jobs : une fois la connexion
    établie, le travail circule dans les deux sens quel que soit l'initiateur.
  * ATTENTION : par défaut le controller dial CHAQUE exec DIRECTEMENT (le hop n'est PAS sur le chemin
    de données tant qu'on ne redirige pas). Forcer le trafic exec À TRAVERS le hop => receptor_peers
    EXPLICITES (cf. multi-zone).
INVENTAIRE — CE QU'IL FAUT AJOUTER : RIEN au-delà du receptor_type='hop' déjà posé. En topologie plate
  mono-site, l'installeur dérive tout le peering de l'appartenance aux groupes (automationcontroller /
  execution_nodes). receptor_peers explicites = NON requis (juste un levier de personnalisation).
MULTI-ZONE / DMZ (anticipation, PAS d'implémentation) :
  * La politique réseau contraint le SENS d'initiation TCP à travers la frontière de zone.
  * On inverse/redirige le défaut « controller dial tout » via des receptor_peers EXPLICITES sur le
    côté AUTORISÉ à initier (mécanisme targeted/difference facts.yml L29-30).
  * # À VÉRIFIER : sens d'initiation TCP cross-zone (à trancher avec le réseau) => receptor_peers
    explicites + probablement routable_hostname sur le hop (facts.yml L8 : _receptor_hostname =
    routable_hostname | default(ansible_host)). Décision d'ARCHITECTURE réseau, à figer AVANT la DMZ.


## Partitionnement/stockage — résolution (sourcé, AUCUNE task ; VÉRIFIER pas CRÉER)
VERROU: preflight NE partitionne/formate/monte/crée RIEN (VM déjà provisionnée). Uniquement ASSERT.
CONFIRMÉ:
- Installeur ne fait AUCUN check espace/mount/partition (grep roles/preflight+common: disk/df/
  ansible_mounts/size_available/space = VIDE ; seul postgresql_effective_cache_size format). Rien à dupliquer.
- Podman graphroot (rootless) = {{ ansible_user_dir }}/aap/containers/storage
  (roles/common/templates/storage.conf.j2, rootless_storage_path). FS de ce chemin ne doit PAS
  être NFS (limite overlay/Podman) et doit avoir de l'espace.
- Données AAP sous $HOME/aap (ansible_user_dir/aap, aap_volumes_dir).
- Extraction bundle: images_tmp_dir défaut = System's TMPDIR (README L53) => /tmp (ou $TMPDIR)
  a besoin d'espace pour l'extraction des images.
- Hub shared storage requis si >1 hub backend 'file' (README L384) — déjà couvert brique 6.
ASSERTABLE (fait ansible_mounts, lecture seule, sans command): espace libre à un point de montage
  (size_available), fstype, graphroot pas sur NFS.
NON assertable => DOCUMENTER: IOPS (mesure = bench destructif), taille disque future/layout LVM.
SEUILS SOURCÉS (RÉSOLU 2026-07-16, table « Local disk » per-VM AAP 2.6 containerized — voir section
  « STOCKAGE — SEUILS SOURCÉS » plus bas + defaults/main.yml pour les 2 URL Red Hat) :
  Installation directory 15 GB ($HOME/aap) et TMPDIR /tmp 10 GB (offline/bundle) => POSÉS (défauts).
  /var/tmp (1 GB online / 3 GB bundle) et 3000 IOPS = provisioning (disque total 60 GB), documentés.
  FS type: la doc n'exige NI ne recommande xfs/ext4 => défaut vide. Table UNIQUE pour tous les noeuds
  => PAS de seuil disque par rôle (surplus hub = PARTAGE NFS de contenu, concern séparé, pas le disque local).
RECO brique: asserts sur ansible_mounts — (1) graphroot pas NFS (défendable, sans seuil) ;
  (2) espace libre >= seuil (défauts SOURCÉS 15/10, surchargeables) ; (3) fstype optionnel ; gating rôle possible.
  IOPS/layout = doc. Crée/formate/monte RIEN.

## STOCKAGE — SEUILS SOURCÉS (RÉSOLU 2026-07-16 ; annexe fusionnée depuis aap-storage-thresholds.md)
Sources (les 2 pages portent la MÊME table « Local disk », une seule pour TOUS les noeuds) :
  - Container enterprise topology :
    https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/plan-ref_cont_b_env_a
  - Containerized System requirements :
    https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/install-ref_cont_aap_system_requirements
Table « Local disk » per-VM : Total 60 GB · Installation directory 15 GB · /var/tmp online 1 GB ·
  /var/tmp offline/bundled 3 GB · TMPDIR (défaut /tmp) offline/bundled 10 GB
  (RAM 16 GB, CPU 4, IOPS 3000 = HORS périmètre stockage).
Décisions (posées dans roles/preflight/defaults/main.yml, source en commentaire) :
  - preflight_aap_min_free_gb = 15   (« Installation directory: 15 GB » ; $HOME/aap).
  - preflight_bundle_tmp_min_free_gb = 10  (« Temporary directory /tmp offline/bundled: 10 GB » ;
    contexte BUNDLE retenu, > online 1 GB).
  - preflight_storage_required_fstype = ""  (doc n'exige NI ne recommande xfs/ext4 : RECOMMANDATION
    absente aussi => rien à imposer ni documenter comme reco).
  - PAS de différenciation par rôle : table UNIQUE pour tous les VMs (aucun seuil disque hub supérieur
    publié). Le défaut global 15 GB ne masque donc AUCUN besoin hub local supérieur ; le surplus hub
    porte sur le partage NFS de contenu (brique 6), pas le disque local d'installation.
  - /var/tmp 3 GB (bundle) : exigence distincte NON modélisée (pas de variable dédiée, périmètre figé)
    => couverte par le provisioning (Total 60 GB), documentée seulement.
  - Garde dure « graphroot pas sur NFS » confirmée : « Podman does not support storing container
    images on an NFS share » (System requirements).
NB re-sourçage 2026-07-16 (micro-livrable réconciliation) : les 2 URL sont LIVE ; la ré-extraction de
  la table par fetch a été bloquée ce jour par le rendu JS (la page renvoie « New features »), MAIS les
  valeurs sont corroborées par (a) confirmation live antérieure le même jour, (b) concordance exacte avec
  les défauts réellement posés dans le code => différenciation par rôle : CONFIRMÉ chiffre UNIQUE.

## Reste ouvert
- ## ORDRE CANONIQUE preflight (revu & validé --list-tasks)
  Correctif de revue: gardes lecture seule DÉPLACÉES AVANT chrony (chrony mutait avant elles).
  Séquence réelle main.yml:
   1-2  SELinux (status enabled ; mode optionnel)          [garde lecture seule]
   3-4  firewalld (service_facts ; state optionnel)        [garde lecture seule]
   5-11 stockage (graphroot mount+NFS+espace+fstype ; tmp) [garde lecture seule]
   12-14 chrony présence + NTP défini                      [garde lecture seule]
   15-18 chrony config/enable/flush/verify                 [1re MUTATION = task 15]
   19-24 Satellite CA(guard)->register->repos              [mutation, TLS]
   25-28 paquets->group->user->linger                      [mutation]
   29-32 pki_ca custom_ca (guard défini+présence)          [garde, avant ping PG]
   33   postgres_connectivity ping TLS (par service)       [check réseau]
   34-36 hub NFS (cohérence + wait_for ; gaté hub)         [check réseau]
  INVARIANTS OK: gardes LS (1-11) AVANT toute mutation (15) ; chrony<register ;
  register+repos<paquets ; custom_ca guard(29-32)<ping PG(33) ; psycopg2(25)<ping PG(33).
  Aucune dépendance implicite inter-brique via set_fact (chaque _pf_* consommé dans sa brique).
  Facts: ansible_selinux/ansible_mounts (play start), service_facts (dans firewalld) — dispo au bon moment.
  Run --check -c local gw1: fail-fast au garde CA Satellite (task 22) AVANT register/paquets (non
  exécutés en dev). 1res mutations (chrony) en check = changed sans écriture. Gating hub OK (skip non-hub).


## POSTCHECK (rôle postcheck — lecture seule, exécuté APRÈS install sur le seed)
Points de contrôle SOURCÉS sur l'installateur (ne rien inventer) :
- gateway (DIRECT sur nœud, port nginx 8446) : uri /api/gateway/v1/ping/ -> status 200
  (bundle automationgateway/tasks/postinstall.yml).
- controller (DIRECT sur nœud, port nginx 8443) : uri /api/v2/ping/ -> status 200
  (automationcontroller/tasks/license.yml).
- eda (VIA proxy/VIP) : uri /api/eda/v1/status/ return_content -> content.status=='good'
  (+ rejet ''/'no healthy upstream') (automationeda/tasks/decision_environment.yml).
- hub (VIA proxy/VIP) : uri /pulp/api/v3/status/ -> status 200 (automationhub/tasks/upload_collections.yml).
Ports HTTPS defaults installer : gateway_nginx_https_port=8446, controller=8443, hub=8444,
  eda=8445, envoy_https_port=443 (proxy = point d'entrée LB/VIP).
Services systemd rootless (scope: user, SOURCÉ */tasks/systemd.yml) :
  gateway: automation-gateway.service, automation-gateway-proxy.service
  controller: automation-controller-{task,web,rsyslog}.service
  hub: automation-hub-{api,content,web}.service (+ workers -N À VÉRIFIER nombre)
  eda: automation-eda-{api,web}.service (+ daphne/worker-N/activation-worker-N/scheduler À VÉRIFIER)
  execution/hop: receptor.service
Statut systemd lu SANS `state` (module ansible.builtin.systemd, changed_when:false) =>
  LECTURE SEULE, aucun `command`. become_user=aap + env XDG_RUNTIME_DIR=/run/user/<uid>
  (uid résolu via getent passwd ; linger activé par preflight => /run/user existe).
TLS : validate_certs=true (jamais false par défaut) ; ca_path=custom_ca_cert (omit si vide -> trust OS).
Gating par groupe (include_tasks + when inventory_hostname in groups.get(...)) : main.yml ->
  gateway.yml / controller.yml / hub.yml / eda.yml / execution.yml. eda+hub HTTP en run_once via proxy,
  précédés d'un assert postcheck_gateway_proxy_url non vide (À VÉRIFIER : URL VIP réelle).
Variables : role-spécifiques postcheck_* dans defaults ; réutilise aap_user/custom_ca_cert de group_vars/all (§5.7).
FRONTIÈRE DEV : validé offline = syntax-check + ansible-lint (production, 0 failure, 10 fichiers) +
  gating (--check -c local gw1/eda1/hop1 : bonne branche incluse, autres skippées ; 1re sonde réelle
  échoue honnêtement = pas de plateforme, RIEN simulé). Aucun `command`/`shell` ajouté (liste inchangée).


## ORCHESTRATION site.yml (assemblage de bout en bout — validé)
Ordre global (--list-tasks, prod) : play#1 preflight [tag preflight] -> play#2 verrou
  [tag install] -> play#3..31 installeur Red Hat [tag install] -> play#32 postcheck [tag postcheck].
site.yml importe playbooks/{preflight,install,postcheck}.yml (JAMAIS l'installeur en direct).
  install.yml importe assert_env_lock.yml AVANT ansible.containerized_installer.install =>
  tout chemin via site.yml hérite du verrou (pas de porte dérobée). NON-CONTOURNEMENT PROUVÉ.
any_errors_fatal: true ajouté aux plays d'orchestration (preflight, assert_env_lock, postcheck) —
  mot-clé d'orchestration, aucune logique de rôle touchée ; l'installeur Red Hat n'est PAS modifié.
  But : un seul nœud qui échoue ses gardes preflight stoppe TOUTE la chaîne AVANT install.
Tags de phase sur les import_playbook (propagés à tout le contenu) : --tags preflight|install|postcheck.
  Isolation prouvée : --tags preflight=36 tâches preflight seules ; --tags postcheck=7 tâches postcheck
  seules ; --tags install=verrou+installeur (les "preflight" comptés sous install = rôle preflight
  INTERNE de l'installeur Red Hat, pas notre playbook). Rejouable par phase (postcheck sans re-preflight).
Démo verrou via site.yml (offline) : prod pollué (-e gateway_pg_sslmode=prefer) + --tags install
  --check -c local => verrou "exiger verify-full" FAILED (failed=1), installeur jamais démarré.
  Prod nominal (assert_env_lock direct) => ok=3 failed=0. Install RÉELLE non jouée en dev (déclaré).
Validation offline : syntax-check prod ET qual OK (warnings collection installeur = normaux) ;
  ansible-lint production 0 failure (29 fichiers). Fonctionne -i prod ET qual sans rien coder en dur.

## TABLEAU DE BORD "À VÉRIFIER" (revue de complétude — inventaire, aucune correction)
BLOQUANTS RÉELS avant run PROD = 9 (tous catégorie A infra/externe) :
  1. PG hosts x4 gateway/controller/hub/eda (DBA) ; ports/bases/users déjà figés.
  2. custom_ca_cert chemin+émission réels + CA serveur PG verify-full (PKI).
  3. Certs de service <svc>_tls_cert/key + SAN (VIP incluse) (PKI ; §3 interdit auto-signé).
  4. gateway_main_url = VIP/LB (Réseau) — MÊME valeur que postcheck_gateway_proxy_url (couplé).
  5. ntp_servers (Exploitation ; assert chrony le rend bloquant).
  6. hub_shared_data_path export NFS (Stockage ; 2 hubs).
  7. satellite_server_hostname/org_id/activation_key(à créer)/ca_src/repos (Satellite/AD).
  8. bundle_dir sur le seed (Provisioning ; bundle_install=true).
  9. postgresql_admin_username (DBA ; conditionnel si installeur crée bases/rôles).
  (+ qual-only : valeurs du banc de test qual). Catégorie B = 0 bloquant.
CATÉGORIE B (levable par lecture/doc, non bloquant) : enforcing exigé?, workers hub<N>/services EDA
  (lire bundle systemd.yml), context= NFS enforcing, stdout_callback yaml, périmètre hosts preflight.
  Chemin de résolution tracé par item (ne pas résoudre en masse).
  [RÉSOLU 2026-07-16, SORTIS de B] seuils stockage (15/10 GB sourcés RH, posés) ; fstype xfs
  strict-vs-reco (ni exigé ni recommandé par la doc => défaut vide) ; ÉPINGLAGE versions collections
  (voir section « ÉPINGLAGE COLLECTIONS » ci-dessous). B passe de 8 à 5 items.
INCOHÉRENCES — 3 résidus NETTOYÉS (micro-livrable) :
  - [FAIT] roles/preflight/defaults/main.yml : commentaire "STUB livrable 1" retiré (rôle complet).
    NB: le résidu était ICI (pas dans postcheck/tasks/main.yml, déjà propre — path corrigé).
  - [FAIT] roles/preflight/defaults/main.yml : "À VÉRIFIER noms de variables définitifs" orphelin retiré.
  - [FAIT] requirements.yml RACINE : question "collections Galaxy publiques" reformulée en CONSTAT
    (renvoie à roles/preflight/requirements.yml). NB: le résidu était dans requirements.yml racine,
    PAS roles/preflight/requirements.yml (dont les 3 À VÉRIFIER sont de vrais points ouverts, intacts).
  Décompte "À VÉRIFIER" hors bundles : 83 -> 81 (2 lignes contenant le texte retirées ; la ligne STUB
  n'en contenait pas). Bloquants catégorie A (9) INTACTS. Syntax prod+qual OK, lint 0 failure (29 fichiers).
  RESTE (non-défauts, à tracer, PAS corrigés) :
  - VIP dupliquée gateway_main_url <-> postcheck_gateway_proxy_url (couplage, 1 seule valeur).
  - Duplications prod/qual (NTP/NFS/PG/CA/VIP/bundle_dir) = VOULUES (§5.1), pas un défaut.
  Dépôt SOUS CONTRÔLE DE VERSION git (git init fait ; branche main, remote origin/main ; commit
  initial 23095b5 « First commit »). MAJ 2026-07-16 : l'ancienne note « pas un dépôt git » est PÉRIMÉE.
  Au moment de la réconciliation, 2 fichiers modifiés non committés (défauts stockage) + 1 non suivi
  (_demo_storage_assert.yml). DEPUIS COMMITTÉS + POUSSÉS + test déplacé sous tests/storage_assert.yml
  — voir section « FICHIERS STOCKAGE — COMMITTÉS ET POUSSÉS » en fin de tableau.
NON-ITEMS exclus du décompte : CLAUDE.md L73/76/81/117 + prod:9 (définition de la règle) ;
  meta/* des rôles (namespace/author/license/min_ansible/RHEL) = gouvernance, non bloquant.


## ÉPINGLAGE COLLECTIONS (RÉSOLU 2026-07-16, sourcé bundle MANIFEST.json)
Principe : la référence = version VENDORÉE DANS LE BUNDLE (plateforme testée RH), pas Galaxy latest
ni l'env dev. Épinglage STRICT (==) pour dev == seed. Seules les collections RÉELLEMENT utilisées
par nos rôles sont épinglées (grep FQCN task-key) :
  * community.postgresql.postgresql_ping (preflight/tasks/postgres_ping_service.yml:55)
  * community.general.redhat_subscription + rhsm_repository (preflight/tasks/satellite_register.yml:71,85)
  (ansible.posix / containers.podman / community.crypto : cités en COMMENTAIRE seulement, PAS utilisés
   comme modules par nos rôles => NON épinglés. ansible.builtin : pas de version.)
Table récap :
  | collection            | épinglée   | vendorée bundle | source                                                    |
  | community.postgresql  | ==3.14.3   | OUI             | bundle .../community/postgresql/MANIFEST.json => 3.14.3    |
  | community.general     | ==6.6.2    | NON (absente)   | non alignable bundle ; version validée dev (galaxy list)  |
Fichiers modifiés : roles/preflight/requirements.yml (les 2 pins + source en commentaire).
  requirements.yml RACINE : INCHANGÉ (collections: []) — l'installeur ansible.containerized_installer
  N'EST PAS sur Galaxy (vendoré, résolu via collections_path) => JAMAIS dans requirements.yml.
community.general = point sensible (register Satellite) : le backend de redhat_subscription dépend de
  la MAJEURE (D-Bus vs subscription-manager) => upgrade de majeure NON testé À PROSCRIRE. Absente du
  bundle => contrainte décidée séparément = 6.6.2 (dev-validée). # À VÉRIFIER (infra) : dispo de 6.6.2
  sur le seed (Galaxy public vs Automation Hub interne) ; repli borné majeure '>=6.0.0,<7.0.0' à
  trancher avec l'infra si le Hub n'offre pas 6.6.2.
Cohérence dev/bundle : en projet (collections_path = ./collections:./bundles/.../collections), la
  résolution prend community.postgresql 3.14.3 DU BUNDLE (le 2.4.2 système est shadowé) => l'env dev
  est aligné sur le pin, AUCUN downgrade silencieux effectué. community.general 6.6.2 = seule présente
  (système), = le pin. Validé offline : syntax-check prod+qual OK ; ansible-lint 0 failure (30 fichiers).


- Brique 8 (stockage asserts) POSÉE: roles/preflight/tasks/storage.yml, insérée APRÈS firewalld
  et AVANT satellite_register (garde d'état, tags preflight/preflight_storage). VERROU: asserts
  LECTURE SEULE sur ansible_mounts, ne partitionne/formate/monte/crée RIEN. Pas de command.
- Sourçage des seuils: RÉSOLU (2026-07-16, 2e tentative). Table « Local disk » per-VM (IDENTIQUE sur
  les 2 pages RH) extraite et vérifiée live. Défauts POSÉS et sourcés dans defaults/main.yml (voir
  section « STOCKAGE — SEUILS SOURCÉS »). Sources: plan-ref_cont_b_env_a (Container enterprise topology)
  + install-ref_cont_aap_system_requirements (Containerized System requirements), AAP 2.6.
- Asserts: (1) graphroot pas NFS = garde cœur sourcée (storage.conf.j2), toujours active, sans chiffre;
  (2) espace libre >= preflight_aap_min_free_gb / preflight_bundle_tmp_min_free_gb (0=skip);
  (3) fstype == preflight_storage_required_fstype (""=skip). Montage porteur = plus long préfixe
  (sort mount + overwrite via startswith ; VALIDÉ: graphroot /home/aap/... -> carrier /home).
- Vars: preflight_graphroot_path, preflight_bundle_tmp_path(/tmp), preflight_graphroot_forbid_nfs(true),
  preflight_aap_min_free_gb(15, sourcé « Installation directory 15 GB »),
  preflight_bundle_tmp_min_free_gb(10, sourcé « Temporary directory /tmp offline/bundled 10 GB »),
  preflight_storage_required_fstype("" ; doc n'impose NI ne recommande de FS)
  = role defaults préfixés (§5.7 ; surchargeables env/groupe). Non assertable => doc: IOPS,
  layout LVM, taille disque totale 60 GB, /var/tmp 3 GB bundle (provisioning).
- Démontré hors ligne (-c local, tag preflight_storage ; via tests/storage_assert.yml, var sim_home Gio):
  avec les défauts SOURCÉS 15/10, sim_home=5 Gio => FAIL (< 15 requis), sim_home >= 15 => pass ;
  NFS via ansible_mounts simulé (/ nfs4) => fail. L'assert d'espace MORD avec les nouveaux défauts.
  Stockage réel cible NON validé en dev.

- Brique 7 (firewalld assert d'état) POSÉE: roles/preflight/tasks/firewalld.yml, insérée APRÈS
  selinux et AVANT satellite_register (garde d'état, tags preflight/preflight_firewalld ; sous-tag
  preflight_firewalld_guard). Miroir strict de la brique 5. NE ROUVRE AUCUN PORT.
- État lu via ansible.builtin.service_facts (MODULE, pas de command) => ansible_facts.services
  ['firewalld.service'].status. Même source que la condition de l'installeur. AUCUN command ajouté.
- Décision état: installeur n'exige pas firewalld enabled (gère les 2 cas) => non imposé.
  preflight_firewalld_required_state = role default "" (miroir preflight_selinux_required_mode ;
  §5.7 consommateur unique = role => defaults préfixé ; surchargeable group_vars). Passer 'enabled'
  pour garantir que l'installeur ouvre les ports (sinon règles firewalld ignorées en silence).
- Démontré hors ligne (-c local, tag preflight_firewalld_guard): dev firewalld=disabled ;
  exigé 'enabled' => fail (avec conséquence), 'disabled' => pass. non imposé ("") => skip/pass.

- Brique 6 (prérequis NFS hub) POSÉE: roles/preflight/tasks/hub_nfs_prereq.yml, incluse dans
  main.yml APRÈS postgres_connectivity, GATÉE hub (when inventory_hostname in groups.automationhub),
  tags preflight/preflight_hub_nfs (+ sous-tag preflight_hub_nfs_guard). NE MONTE PAS.
- Installeur (automationhub/tasks/nfs.yml): monte le NFS (ansible.posix.mount, state mounted,
  fstype nfs) ET installe nfs-utils (package non-ostree / assert ostree). => on ne remonte pas,
  et on N'INSTALLE PAS nfs-utils (anti sur-couverture, comme podman). Test = ansible.builtin.wait_for
  (module TCP port 2049, PAS de mount, PAS de nfs-utils requis). NFSv3 rpcbind/111 = À VÉRIFIER.
- context= SELinux NFS: README ne documente que hub_shared_data_path + hub_shared_data_mount_opts
  (défaut rw,sync,hard, SANS context=); installeur ne pose pas context=; doc containerized non
  concluante => NON tranché. hub_shared_data_mount_opts laissé 'rw,sync,hard' SANS context=,
  À VÉRIFIER (context=system_u:object_r:<type>:s0 OU booléen). NE PAS inventer.
- Variables: hub_shared_data_path (À VÉRIFIER, non défini) + hub_shared_data_mount_opts = group_vars/all
  (prod+qual). preflight_nfs_port(2049)/preflight_nfs_probe_timeout(10) = role defaults.
- Garde cohérence hub HA (>1 hub => hub_shared_data_path requis) + format serveur:/export démontrée
  hors ligne (-c local, tag preflight_hub_nfs_guard, --limit hub1): absent=>fail, fourni=>pass,
  non-hub(gw1)=>skippé. Install nfs-utils + wait_for réseau NON exécutés en dev.

- Brique 5 (SELinux assert) POSÉE: roles/preflight/tasks/selinux.yml, insérée APRÈS chrony
  et AVANT satellite_register (garde d'état pure, tags preflight/preflight_selinux ;
  sous-tag preflight_selinux_guard). Lecture seule via fait ansible_selinux (PAS getenforce).
- Enforcing NON tranché par la doc consultée (containerized install/planning ne le fonde pas
  explicitement) + installeur sans precheck => assert obligatoire = MINIMAL: ansible_selinux.status
  == 'enabled' (non disabled). Mode strict = variable preflight_selinux_required_mode (defaults,
  défaut "" = non imposé; passer 'enforcing' en group_vars si mandaté). enforcing reste À VÉRIFIER.
- Bindings python3-libselinux/policycoreutils: STANDARD RHEL (SELinux core) => NON ajoutés à
  preflight_packages (et non installables pré-register). Si absents, ansible_selinux.status=
  'Missing selinux Python bindings' -> l'assert le signale avec remédiation. À VÉRIFIER profil minimal.
- Démontré hors ligne (-c local, tag preflight_selinux_guard): dev = enabled/enforcing/targeted;
  mode exigé permissive => fail, enforcing => pass. status assert passe (dev enabled).

## Reste ouvert (suite)
- Brique 4 (chrony) POSÉE: roles/preflight/tasks/chrony.yml + templates/chrony.conf.j2 +
  handler Restart chronyd. INSÉRÉE EN PREMIER dans main.yml (AVANT satellite_register).
- ORDRE décidé: chrony AVANT register/ping PG car register+verify-full = TLS, cassé par skew.
  Tension: chrony est un paquet mais hôte non enregistré => aucun repo => install impossible
  pré-register. Résolu = chrony DOIT être présent dans l'image RHEL + GARDE de présence
  (stat chronyd + assert). chrony présent par défaut sur RHEL standard, PAS garanti en minimal
  (# À VÉRIFIER image). Source prérequis horloge<register: Satellite 6.19 §6.2.2.
- chrony.conf: template REMPLACE /etc/chrony.conf (server iburst + driftfile/makestep/rtcsync).
  # À VÉRIFIER: aligner sur baseline org. notify Restart chronyd + meta flush_handlers avant verify.
- Vérif synchro: command `chronyc -n tracking`, changed_when: false, until rc==0 & pas
  'Not synchronised', retries 5/delay 6. Pas de module -> command lecture seule justifié.
- ntp_servers = group_vars/all (prod+qual, défaut [] + À VÉRIFIER; PAS de pool public en dur).
  preflight_chronyd_bin/chrony_conf/chrony_service = role defaults (préfixés).
- Config chrony + synchro NON exécutés en dev (root, horloge réelle): syntax+lint OK.
  Garde présence démontrée hors ligne (-c local, tag preflight_chrony_guard): absent=>fail, présent=>ok.

## Commands justifiés (pas de module)
- loginctl enable-linger (brique3) + creates marqueur linger.
- chronyc -n tracking (brique4) lecture seule, changed_when false.
- Brique 3 (paquets + user aap + linger) POSÉE: roles/preflight/tasks/packages_user.yml,
  incluse dans main.yml APRÈS satellite_register et AVANT le ping PG (tags preflight/preflight_packages).
- Linger: AUCUN module (vérifié: ni ansible.builtin.user ni community.general.systemd_service).
  Motif installeur (common/tasks/main.yml L110-113): command loginctl enable-linger + 
  creates: /var/lib/systemd/linger/<user> (idempotent). Reproduit tel quel.
- Paquets justifiés (non gonflés): ansible-core (installeur ASSERTE la version, preflight/core.yml,
  ne l'installe pas), podman (installeur l'installe aussi sur non-ostree), python3-psycopg2
  (requis par notre brique 1b ping). python3-cryptography NON inclus (installeur le gère, tls.yml).
- state: present (PAS latest) - version pilotée par repos Satellite.
- aap_user/group/home/shell = group_vars/all (prod+qual, valeur 'aap'), PAS defaults du rôle
  (var-naming[no-role-prefix] impose préfixe rôle ; et c'est un fait projet-wide). preflight_packages
  reste en role defaults (préfixé, OK).
- Paquets + linger NON exécutés en dev (repos injoignables, root): syntax-check + lint OK.
  Modules user/group idempotents par nature; non isolables proprement en --check ici (mêlés au
  package task qui touche les repos).
- Brique 2 (enregistrement Satellite) POSÉE: roles/preflight/tasks/satellite_register.yml
  (inclus EN PREMIER dans main.yml, tags preflight/preflight_satellite ; garde CA
  sous-taguée preflight_satellite_ca). Ordre: CA -> register -> repos.
- CA Satellite: redhat_subscription N'A PAS de param CA serveur. subscription-manager
  lit /etc/rhsm/ca/ (+ trust système). => déposer katello-server-ca.pem dans
  /etc/rhsm/ca/. rhsm_repo_ca_cert = CA du CDN de contenu (PAS la CA serveur).
- redhat_subscription idempotent: credentials consommés seulement si non enregistré
  (ou force_register). rhsm_repository: exige hôte déjà enregistré.
- community.general ajoutée à roles/preflight/requirements.yml (absente bundle, sur Galaxy;
  dispo réelle sur seed = À VÉRIFIER). Localement présente 6.6.2 (validation OK).
- Garde CA démontrée hors ligne (-c local, tag preflight_satellite_ca): absent=>fail, présent=>ok.
- Register + repos NON exécutés en dev (Satellite injoignable): syntax-check + lint OK seulement.
- group_vars prod+qual POSÉS (livrable fait). Restent en À VÉRIFIER (valeurs infra):
  bundle_dir (emplacement seed, doit finir /bundle), *_pg_host/port réels (DBA),
  gateway_main_url (VIP/LB), chemins+SAN certs PKI (non émis), registry Satellite.
- gateway_main_url = URL front-end gateway (README L283). Confirmé comme nom.
- PAS de variable pg sslrootcert/pg_ca dans README => RÉSOLU (source collection):
  verify-full valide via le TRUSTSTORE, pas via une var par service.
  * Runtime: roles/automationcontroller/templates/postgres.py.j2 L14-15 =>
    sslmode={{controller_pg_sslmode}}, sslrootcert={{ca_trust_bundle}}.
    ca_trust_bundle défaut=/etc/pki/tls/certs/ca-bundle.crt
    (automationcontroller/defaults/main.yml:79). Idem eda/hub/gateway.
  * Trust construit: roles/common/tasks/tls.yml L142-172 => custom_ca_cert copié en
    anchors/tls-custom.cert + ca_tls_cert en anchors/tls-aap.cert, puis update-ca-trust.
  * Preflight check DB externe: roles/preflight/tasks/nodes.yml L71 =>
    _postgresql_tls_ca_cert = custom_ca_cert ; postgresql.yml append custom au
    ca-bundle temp + postgresql_ping(ca_cert=..., ssl_mode=<svc>_pg_sslmode).
  => MÉCANISME: pour verify-full sur PG EXTERNE, fournir custom_ca_cert = chemin PEM
     de la CA ayant émis le cert serveur PG (déposé sur seed si ca_tls_remote=false,
     défaut; sur nodes si true). Aucune var *_pg_sslrootcert. ca_tls_cert = CA interne
     AAP (anchor tls-aap), distincte de custom_ca_cert (anchor tls-custom).
  * Nuance: custom_ca_cert = UN seul chemin => concaténer plusieurs CA en un PEM si besoin.
- controller_pg_host (et par analogie les *_pg_host) = REQUIS par l'installeur.
- Design vault: vault.yml MAPPE var_installeur -> "{{ vault_* }}" (aucune valeur);
  les vault_* réels seront dans un fichier chiffré séparé. main.yml = zéro secret.
- Banc de test: test_harness_enabled=true UNIQUEMENT en qual; absent en prod
  (vérifié via ansible-inventory --host). Assert de verrouillage = livrable play install.

## Structure dépôt
- Racine = workspace (pas de sous-dossier aap-automation/). Un dépôt Git = une racine.
- group_vars vit à CÔTÉ de chaque inventaire (inventories/{prod,qual}/group_vars/).
- Livraison itérative: un livrable à la fois, attendre validation (CLAUDE.md 6).


## FICHIERS STOCKAGE — COMMITTÉS ET POUSSÉS (état 2026-07-16 ; FAIT)
Dépôt git : branche main. 2 commits AJOUTÉS au-dessus de 23095b5 puis POUSSÉS (origin/main = b2f2c6f) :
  * a010184  preflight/storage: seuils disque sourcés (15 GB install, 10 GB /tmp bundle) + FS non imposé
             => roles/preflight/defaults/main.yml + roles/preflight/tasks/storage.yml
  * b2f2c6f  tests: harnais reproductible de l'assert d'espace stockage
             => tests/storage_assert.yml (ex-_demo_storage_assert.yml, déplacé + renommé)
- Harnais déplacé racine -> tests/storage_assert.yml (préfixe _ retiré) et rendu reproductible : include
  ajusté « {{ playbook_dir }}/../roles/preflight/tasks/storage.yml ». Rejoué depuis tests/ :
  sim_home=5 => FAIL (5 < 15 Gio), sim_home=20 => pass. Lancer :
  ansible-playbook -i localhost, tests/storage_assert.yml -e sim_home=5|20 -c local.
- ansible-lint 0 failure (profil production). Working tree PROPRE. push EFFECTUÉ par l'utilisateur
  (origin/main avancé à b2f2c6f).
NB (2026-07-16) : l'épinglage des collections (community.postgresql==3.14.3, community.general==6.6.2)
  a ensuite été committé par l'utilisateur (commit « requirements update »). Ce présent fichier
  docs/aap-facts.md est le commit de migration du tableau de bord sous git.


## MÉMOIRE & RÉCONCILIATION (2026-07-16 — pourquoi le tableau de bord semblait perdu)
- Renommage claude_app -> deploy_aap => nouveau hash de workspaceStorage.
  * Ancien : e1ce58f… (claude_app) — y vivait aap-facts.md (orphelin, 428 l.).
  * Courant : ce36137…ba8 (deploy_aap) — y vit aap-storage-thresholds.md (fusionné ici).
- PIÈGE outil mémoire (ÉTABLI 2026-07-16) : le workspace deploy_aap a DEUX racines d'instance :
  ce36137…ba8 (sans suffixe) et ce36137…ba8-1 (suffixe multi-fenêtre). resolve_memory_file_uri a
  renvoyé …ba8-1, MAIS l'outil mémoire ÉCRIT RÉELLEMENT dans …ba8 (sans suffixe) — vérifié par mtime.
  D'où l'impression initiale « aucune mémoire » (l'outil listait un root, le fichier était dans l'autre).
- HAZARD CONFIRMÉ : mirrorer par `cp` PENDANT l'édition a CLOBBERÉ des écritures fraîches de l'outil
  (des édits des livrables « commit » et « receptor » ont été PERDUS puis RE-POSÉS). RÈGLE :
  (1) éditer UNIQUEMENT via l'outil mémoire (il écrit dans …ba8) ; (2) NE JAMAIS cp …ba8-1 -> …ba8
  (stale -> canonique) ni cp en cours d'édition ; (3) synchro autorisée = SEULEMENT cp …ba8 -> …ba8-1
  (canonique -> stale) UNE fois, APRÈS tous les édits du tour. Racine canonique = …ba8 (freshest mtime).
- aap-storage-thresholds.md : contenu FUSIONNÉ ici ; l'original a été marqué .superseded (réversible).
- NOUVELLE RÈGLE DE TENUE (2026-07-16) : la SOURCE DE VÉRITÉ est DÉSORMAIS le fichier git
  docs/aap-facts.md (historisé + poussé sur origin). La mémoire de session reste un CACHE pratique.
  Toute mise à jour significative doit FINIR committée dans docs/aap-facts.md. FINI le mirroring `cp`
  entre racines de workspaceStorage — git fait foi et sert de backup distant récupérable.

## RHEL — version requise : RÉSOLU, sourcée dans le bundle (2026-07-16, suite)
- Suite du point ci-dessous : version trouvée HORS LIGNE dans le bundle, pas besoin de fetch web.
  Source EXACTE (collection ansible.containerized_installer, bundle 2.6-10.1-x86_64) :
  * roles/preflight/tasks/nodes.yml:12-13 — assert installeur RÉEL exécuté sur chaque nœud :
    `ansible_distribution == 'RedHat'` ET `ansible_distribution_version is version_compare('9.6', '>=')`,
    fail_msg: "Only Red Hat Enterprise Linux 9.6+ distributions are supported".
  * roles/preflight/tasks/core.yml:7 — assert ansible-core, fail_msg: "Supported combinations:
    RHEL 9 with Ansible 2.14 or RHEL 10 with Ansible 2.16" => confirme RHEL 10 comme 2e combinaison
    supportée (couplée à ansible-core 2.16 ; le garde-fou nodes.yml n'a PAS de borne haute, donc
    10.x satisfait aussi mathématiquement version_compare('9.6','>=')).
  => VALEUR RETENUE (la plus contraignante = celle réellement vérifiée par l'installeur) :
     RHEL 9.6+ (ou RHEL 10, avec ansible-core 2.16). C'est un ASSERT RÉEL de l'installeur (pas une
     doc externe) : plus fiable qu'une page web. Checklist .docx (section 4, les 2 variantes)
     corrigée en conséquence, avec note interne (commentaire Word) citant fichier:ligne.
- Reliquat noté en passant (PAS touché dans la checklist, hors périmètre de ce livrable) :
  nodes.yml:16-19 contient aussi un assert RAM réel : `ansible_memtotal_mb >= 15000`, fail_msg
  "Required RAM is 16GB..." => la valeur RAM 16 Go EST en fait sourçable dans le bundle (contrairement
  à vCPU/IOPS, pour lesquels aucun assert n'existe : `ansible_processor_vcpus | default(4)` n'est
  qu'un défaut de calcul de workers, PAS une exigence minimale ; IOPS n'apparaît nulle part dans les
  rôles). À reprendre dans un livrable dédié si on veut sourcer RAM avec le même luxe que RHEL/disque.

## RHEL — version requise : PAS DE VALEUR SOURCÉE (correctif checklist, 2026-07-16, PREMIÈRE tentative — voir suite ci-dessus)
- La checklist .docx (docs/AAP_2.6_prerequis_infra_par_equipe.docx, section 4) affirmait
  « RHEL 9.4+ ». Grep de ce fichier (aap-facts.md) : AUCUNE version RHEL n'y est mentionnée —
  la prémisse « c'est dans aap-facts.md, la vraie valeur est 9.6+ » était FAUSSE À CE STADE.
- Tentative de sourçage live (install-ref_cont_aap_system_requirements) : HTTP 403 (même mode
  d'échec que le blocage JS déjà noté § STOCKAGE — SEUILS SOURCÉS). Version NI 9.4 NI 9.6 confirmée
  À CE STADE (résolu ensuite hors ligne dans le bundle, voir section précédente).
- RAM 16 Go / vCPU 4 / IOPS ~3000 (mêmes docx) : déjà notés ci-dessus (§ STOCKAGE — SEUILS SOURCÉS,
  ligne « HORS périmètre stockage ») comme non sourcés avec la même rigueur que 60/15/10 Go — checklist
  corrigée pour les présenter comme ordre de grandeur à confirmer, pas comme exigence ferme (INCHANGÉ,
  cf. nuance RAM ci-dessus).
