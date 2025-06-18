# Oracle Personal Appliance

## Materiel

- i3-2130 2 core de 3,40Ghz
- 14G de ram
- 4 DD de 3To

## Clone du projet github

```
cd projets
ssh-keygen -f id_rsa.opa
git clone git@github.com:sebcb1/OPA.git
cd OPA
git config --global core.sshCommand "ssh -i /Users/sebastien/projets/id_rsa.opa"
git config --global user.email "sebastienbrillard@icloud.com"
git config --global user.name  "Sébastien Brillard"
git config --global push.default simple
```

## Installation de l'OS

