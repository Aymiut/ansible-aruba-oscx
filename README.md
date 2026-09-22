# Automatisation d'un parc Aruba OS-CX avec Ansible

Configurer une vingtaine de commutateurs Aruba OS-CX depuis un inventaire : VLAN, agrégations de
liens LACP, stacking VSF et relevé des versions logicielles.

**Ce dépôt est une reconstruction générique.** Le projet d'origine a été mené chez SPIE ICS pour un
établissement de santé : ce code et ces données appartiennent au client et ne sont pas publiables.
Ici, tout est fictif — vingt commutateurs d'exemple, adresses en `10.50.10.0/24`, identifiants dans
un fichier chiffré. Ce qui est réel, c'est la structure du projet, les tâches et les pièges
rencontrés en production.

## Ce que ça fait

| Playbook             | Effet                                                                                |
| -------------------- | ------------------------------------------------------------------------------------ |
| `versions.yml`       | Lecture seule : relève la version logicielle de chaque commutateur et signale les écarts. |
| `vlans.yml`          | Crée sur tout le parc les VLAN déclarés dans `group_vars`.                            |
| `lags.yml`           | Crée les agrégations LACP des commutateurs qui en déclarent, et pose leurs VLAN.      |
| `vsf.yml`            | Déclare le membre de stack VSF et ses liens, par le CLI.                              |
| `site.yml`           | Relevé, puis VLAN, puis agrégations.                                                  |

## Structure

```
inventory/
├─ hosts.yml                   20 commutateurs : cœur, agrégation, accès
├─ group_vars/aoscx/
│  ├─ main.yml                 connexion, identifiants par défaut, VLAN du parc, version attendue
│  └─ vault.yml.example        modèle du fichier chiffré (le vrai n'est pas versionné)
└─ host_vars/
   ├─ sw-acc-02.yml            ports d'agrégation et membre de stack de ce commutateur
   ├─ sw-acc-03.yml            identifiants du lot ancien
   └─ …                        un fichier par commutateur
roles/aoscx/tasks/             vlans, lags, vsf, versions
playbooks/                     un playbook par opération, plus site.yml
tests/                         vérification du garde-fou, sans matériel
```

Le parc a deux jeux d'identifiants : celui de `group_vars/aoscx/main.yml` s'applique partout, et les
quelques commutateurs du lot le plus ancien redéfinissent le leur dans leur `host_vars`. Un
commutateur qui ne déclare ni agrégation ni stack n'est pas modifié par les tâches correspondantes.

## Utilisation

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

cp inventory/group_vars/aoscx/vault.yml.example inventory/group_vars/aoscx/vault.yml
ansible-vault encrypt inventory/group_vars/aoscx/vault.yml

ansible-playbook playbooks/versions.yml --ask-vault-pass          # lecture seule
ansible-playbook playbooks/site.yml --check --ask-vault-pass      # simulation
ansible-playbook playbooks/site.yml --ask-vault-pass              # application
```

## Ce que la production a appris à ce code

- **Le numéro de membre de stack.** Sur un stack, un port s'écrit `membre/module/port`. Viser le
  mauvais membre ne provoque aucune erreur : le playbook reconfigure simplement un commutateur déjà
  en service. Le rôle refuse donc d'appliquer si un port ne correspond pas au membre déclaré dans le
  `host_vars`. C'est le test de `tests/stack-member-guard.yml`.
- **Les ports Aruba sont administrativement `down` par défaut.** Une agrégation se crée sans erreur
  et ne monte jamais. Les tâches activent explicitement les ports membres, puis l'agrégation.
- **Changer l'adresse de management coupe la session en cours.** Il n'existe pas de module pour
  cette opération et le CLI se coupe sous les pieds. La renumérotation n'est pas dans ce dépôt : sur
  le terrain, elle se fait en gardant l'ancienne adresse sur une autre interface VLAN le temps de la
  bascule.
- **La compatibilité des versions se vérifie en premier.** Un `ansible-core` trop récent pour la
  collection fait échouer la connexion sans message clair. C'est ce qui a bloqué le projet d'origine
  plusieurs semaines, pendant lesquelles les vraies pistes semblaient être l'hyperviseur ou Python.
- **`--check` avant toute application.** Sur un parc, une erreur se propage aussi vite qu'une
  bonne configuration.

## Limites

- Le stacking VSF n'est pas testable en simulation : les images de commutateurs d'EVE-NG ne le
  supportent pas.
- Ce dépôt n'a pas été exécuté contre du matériel réel dans cette version générique. La CI vérifie
  l'inventaire, la syntaxe des playbooks et le garde-fou des agrégations, sans matériel.
