---
title: Quoi de neuf sur la gestion de configuration ?
created: 2026-11-16
author:
  - mikamakusa
category:
  - Cloud
tags:
  - Ansible
  - Salt
  - Puppet
  - Chef
  - CFEngine
  - Rudder
  - Foreman
---
# Quoi de neuf sur la gestion de configuration ?

Un petit rappel de l'historique des outils de gestion des configuration s'impose avant même de démarrer cet article :  
- **CFEngine** a été rendu disponible en 1993,  
- **Puppet** et **Chef** sont sortis, respectivement, en 2005 et 2009,  
- **Rudder**, **Salf** et **Ansible**, sont sortis en 2011 (pour les deux permiers) et 2012,  
- **Uyuni**, le tout dernier, est disponible depuis 2018. Celui-ci est beaucoup plus récent en tant que projet indépendant, mais hérite d'une longue histoire technologique car issu de **Spacewalk** et constitue depuis 2018 l'amont communautaire de **SUSE Manager**.

Je pourrais également citer **OpenText Server Automation**, **Foreman**, ou **Powershell DSC**, mais j'ai préféré me limiter à ceux que j'ai déjà abordé durant mes précédentes expériences.  

### Comparatif synthétique
Voici déjà un comparatif assez synthétique de chaque outils (je rentrerai dans les détails plus tard) :  
| Outils | Licence / Modèle | Agent | Linux | Windows | Gestion déclarative |  
| ------ | ---------------- | ----- | ----- | ------- | ------------------- |  
| Rudder Core | Libre - Gratuit | Oui | Oui | En fonction de l'édition | Oui |  
| Ansible | Libre - Gratuit | Non | Oui | Oui, mais pas nativement | Oui |  
| Puppet | Open Source / Commercial | Oui | Oui | Oui | Oui |  
| CFEngine | Open Source / Commercial | Oui | Oui | Oui | Oui |  
| Salt | Libre - Grauit | Optionnel | Oui | Oui | Oui | Oui |  
| Chef Infra | Open source / commercial |	Oui | Oui | Oui | Oui |

## L'architecture
Concernant **Salt**, on peut distinguer deux modes de fonctionnement :  
- Le mode **Masterless**, pas besoin d'un serveur central pour exécuter les commandes, il suffit (par exemple) qu'un runner *gitlab-ci* puisse exécuter la commande à distance sur la cible, tout en ayant un accès aux *pillars* (contenant la configuration à déployer).  
- Le mode **Master/Minion**. C'est le mode de fonctionnement classique de **Salt**. Un serveur contient les *pillars* ainsi que la base des *minions* (worker sur lesquels seront exécutés les configurations). La communication entre eux est chiffrée rendant les transactions sécurisées.

Dans le cas d'**Ansible**, pas de serveurs *maître* ou *esclave* à proprement parler. Pas d'agent installé sur les serveurs de destination, juste un fichier d'inventaire ou un inventaire dynamique (selon le cas d'usage) et une connexion SSH directe...ou à travers un bastion (ce qui demande un effort supplémentaire en terme de configuration).

Du côté de **CFEngine**, l'approche **Client/Serveur** avec agent permanent obligatoire...De même que **Rudder**, **Chef Infra** et **Puppet**.

Cependant tous ne proposent pas nativement de gestion centralisée, d'interface graphique, de gestion des relais et de Correction automatique des dérives comme proposé par **Rudder**

**Avantage** : Je pencherais pour **Rudder**, il semble bien plus complet que les autres. J'ai pratiqué assez longtemps **Ansible** pour me rendre compte que sa simplicité n'est pas forcément une force.

## La syntaxe et la gestion des configurations
**Ansible**, **Salt** et **Rudder** proposent une syntaxe simple à base de YAML et de Jinja2 (en ce qui concerne la gestion des templates, uniquement pour les deux permiers), là où celles de **Puppet**, de **Chef Infra** et de **CFEngine** me semble moins naturelles. 

Exemples : 
#### Salt
```yaml
nginx:
  pkg.installed: []

nginx_service:
  service.running:
    - name: nginx
    - enable: true
    - require:
      - pkg: nginx
```

#### Ansible
```yaml
- name: Installer nginx
  hosts: webservers
  become: true

  tasks:
    - name: Installer le package
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Démarrer nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

#### Puppet
```puppet
package { 'nginx':
  ensure => installed,
}

service { 'nginx':
  ensure => running,
  enable => true,
}
```

#### CFEngine
```cfengine3
bundle agent nginx {
  packages:
    "nginx"
      policy => "present";

  services:
    "nginx"
      service_policy => "start";
}
```

#### Rudder
```yaml
---
id: "nginx_configuration"
name: "Nginx configuration"
version: "1.0"
description: "Install and configure Nginx"

items:
  - name: "Install Nginx"
    method: package_present
    params:
      name: "nginx"
      version: "latest"

  - name: "Configure Nginx"
    method: file_content
    params:
      path: "/etc/nginx/nginx.conf"
      lines: |
        user www-data;
        worker_processes auto;
        events {
          worker_connections 1024;
        }
        http {
          include /etc/nginx/mime.types;
          server {
            listen 80;
            server_name _;
            location / {
              return 200 "Nginx managed by Rudder";
            }
          }
        }
      enforce: "true"

  - name: "Enable Nginx service"
    method: service_enabled
    params:
      service_name: "nginx"

  - name: "Start Nginx service"
    method: service_started
    params:
      service_name: "nginx"
```

#### Chef Infra
```ruby
# recipes/nginx.rb

# Installation de Nginx
package 'nginx' do
  action :install
end

# Configuration du serveur
template '/etc/nginx/nginx.conf' do
  source 'nginx.conf.erb'
  owner 'root'
  group 'root'
  mode '0644'

  notifies :restart, 'service[nginx]', :delayed
end

# Gestion du service
service 'nginx' do
  action [:enable, :start]
end
```

**Avantage** : **Ansible**, **Salt** et **Rudder**. Le ticket d'entrée est moins compliqué pour maîtriser la syntaxe et réaliser des *Playbooks/Pillars/Techniques*. De plus, **Ansible** et **Rudder** ont une avance non négligeable sur **Salt** grâce à leur CLI permettant de réaliser un squelette de configuration déployable rapidement.

## Conformité et sécurité
Ce point est particulier car aucun des outils n'est un **SIEM**, un **EDR** ni un scanner de vulnérabilités réseau ou CVE.  
On peut distinguer plusieurs fonctions sur lesquelles chacun d'eux pourront être jugés :  
- Security Configuration Management (SCM) : maintenir les systèmes dans une configuration sécurisée.  
- Security Compliance Management : vérifier que les configurations respectent des référentiels et des politiques.  
- Security Remediation : corriger automatiquement les écarts de sécurité.  
- Security Auditing : conserver des preuves des contrôles réalisés et des résultats obtenus.

Contrairement aux autres, **Chef** dispose d'un écosystème permettant de gérer aussi bien la configuration que l'audit en lui même, grâce à **Chef Inspec**.
J'avais testé ce dernier il y a quelques années, pour du contrôle de conformité sur des ressources déployées sur Azure. Bien que la syntaxe soit basée sur **Ruby**, j'avais trouvé cela assez puissant pour le recommander.

Voici un exemple : 
```ruby
control 'ssh-001' do
  impact 1.0
  title 'Disable SSH root login'
  desc 'Root must not log in directly through SSH.'

  describe sshd_config do
    its('PermitRootLogin') {
      should cmp 'no'
    }
  end
end

control 'ssh-002' do
  impact 1.0
  title 'Disable SSH password authentication'

  describe sshd_config do
    its('PasswordAuthentication') {
      should cmp 'no'
    }
  end
end

control 'audit-001' do
  impact 0.8
  title 'Auditd must be installed'

  describe package('auditd') do
    it { should be_installed }
  end
end

control 'audit-002' do
  impact 0.8
  title 'Auditd service must be enabled'

  describe service('auditd') do
    it { should be_enabled }
    it { should be_running }
  end
end
```

**Ansible** et **Salt** ont plutôt une approche centrée sur l'application de politique de sécurité.

**CFEngine** a un langage moins accessible que les autres concernant la sécurité et la conformité en raison des *Promises*. Exemple :  
```cfengine3
bundle agent security {

  files:

    "/etc/ssh/sshd_config"
      edit_line => set_ssh_root_login;

  services:

    "sshd"
      service_policy => "start";
}

bundle edit_line set_ssh_root_login {

  replace_patterns:

    "PermitRootLogin.*"
      replace_with => "PermitRootLogin no";
}
```

**Rudder** est probablement celui qui propose l'approche la plus directement orientée vers la gouvernance des configurations de sécurité. Sa philosophie est de transformer une politique de sécurité en règles techniques applicables aux machines en intégrant directement les benchmarks CIS via sa solution *Policy and Benchmark Compliance*, ainsi qu'aux mode *Audit* (vérifie la configuration et signale les écarts, sans modifier la machine) et *Enforce* (vérifie la configuration et corrige les écarts).

**Avantage** : **Chef** (notamment pour l'apport de **Inspec** dans l'écosystème de **Chef** permettant la gestion de la compliance sur le Cloud) et **Rudder**. 


## Compatibilité systèmes
Tous les outils sont compatibles avec les distributions **Linux** ainsi qu'avec **Windows**. Par contre, la compatibilité de **Rudder** avec l'environnement **BSD** ainsi qu'avec les **macOS** est assez limité.
Mention spéciale pour **Ansible** : Bien que compatible avec **Windows**, ce dernier nécessite une configuration particulière.

**Avantage** : Difficile de départager l'un d'eux...sauf si on souhaite déployer sur **BSD** ou **macOS** ou sur **Windows** puisque certaines éditions peuvent bloquer des fonctionnalités de **Rudder**

## Licences et coûts
C'est un critère essentiel si l'objectif est de construire une solution entièrement libre et gratuite, y compris pour un usage professionnel.
| Solution | Édition communautaire | Édition commerciale | Principale limitation |
| -------- | --------------------- | ------------------- | --------------------- |
| Rudder | Rudder Core | Rudder Enterprise | Fonctions avancées et plateformes selon édition |
| Ansible | Ansible Community | Red Hat AAP | Gouvernance et fonctionnalités avancées |
| Puppet | Puppet Core | Puppet Enterprise | Fonctionnalités avancées et support |
| Chef Infra | Composants open source | Chef 360 | Plateforme commerciale et évolution du serveur historique |
| Salt | Salt Open | SaltStack Config | Gouvernance et interface avancées |
| CFEngine | Community | Enterprise | Reporting et fonctionnalités avancées |

**ATTENTION** : **Chef Infra** est en fin de vie et sera remplacé par **Chef 360**.
Il restera néanmoins utilisable et documenté.

**Avatnage** : Dommage pour **Chef Infra** et sa fin de vie imminente.

## Conclusion
**Chef** et **Puppet** ont une courbe d'apprentissage assez complexe ainsi qu'une architecture assez lourde, **Rudder** et **CFEngine** un ecosystème bien trop restreint, **Ansible** et **Salt** assez limités en ce qui concerne la gourvenance...cependant, chacun d'entre eux ont des nombreuses qualités et en recommander un plutôt qu'un autre serait occulter les différences notables.  
Donc à vous de vous faire votre propre idée sur le sujet.  
De mon côté, je vais peut-être jeter un oeil du côté de **Rudder**.