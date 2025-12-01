# Tabla de alias de Git (ordenados por frecuencia de uso)

| Alias     | Uso                     | Descripción                                                                                          |
| --------- | ----------------------- | ---------------------------------------------------------------------------------------------------- |
| `s`       | `git s`                 | Muestra el estado del repositorio en formato corto. Es lo que más se usa para ver qué cambió.        |
| `st`      | `git st`                | Igual que `git status`, pero sin el formato corto.                                                   |
| `ad`      | `git ad <archivo>`      | Agrega archivos al staging.                                                                          |
| `adp`     | `git adp`               | Agrega cambios por partes (modo interactivo).                                                        |
| `cm`      | `git cm "mensaje"`      | Crea un commit con mensaje.                                                                          |
| `ca`      | `git ca`                | Modifica el último commit, conservando los cambios.                                                  |
| `ps`      | `git ps`                | Envía commits al repositorio remoto.                                                                 |
| `pl`      | `git pl`                | Trae cambios del remoto.                                                                             |
| `prb`     | `git prb`               | Trae cambios con rebase para evitar merges innecesarios.                                             |
| `co`      | `git co <rama>`         | Cambia de rama. También sirve para restaurar archivos.                                               |
| `br`      | `git br`                | Lista ramas o crea nuevas si se usa con `-b`.                                                        |
| `lg`      | `git lg`                | Muestra el historial de commits en una sola línea, con grafo y decoración. Muy útil para orientarte. |
| `l`       | `git l`                 | Historial simplificado en una sola línea.                                                            |
| `df`      | `git df`                | Muestra diferencias sin agregar.                                                                     |
| `di`      | `git di`                | Muestra diferencias de lo que ya está en staging.                                                    |
| `rb`      | `git rb <rama>`         | Hace rebase sobre otra rama.                                                                         |
| `rbi`     | `git rbi`               | Rebase interactivo para ordenar, squash, etc.                                                        |
| `mg`      | `git mg <rama>`         | Hace un merge de otra rama.                                                                          |
| `brd`     | `git brd <rama>`        | Borra una rama local si ya está mergeada.                                                            |
| `brf`     | `git brf <rama>`        | Borra una rama local aunque no esté mergeada.                                                        |
| `undo`    | `git undo`              | Revierte el último commit pero mantiene los cambios en staging.                                      |
| `unstage` | `git unstage <archivo>` | Saca un archivo del staging sin borrar su contenido.                                                 |
| `last`    | `git last`              | Muestra el último commit.                                                                            |
| `ss`      | `git ss "mensaje"`      | Crea un stash.                                                                                       |
| `ssp`     | `git ssp`               | Aplica el último stash y lo borra.                                                                   |
| `ssl`     | `git ssl`               | Lista todos los stash.                                                                               |
| `ft`      | `git ft`                | Descarga referencias del remoto sin mezclar.                                                         |
| `fta`     | `git fta`               | Descarga referencias de todos los remotos.                                                           |
| `r`       | `git r`                 | Muestra la configuración de remotos.                                                                 |
| `cl`      | `git cl <url>`          | Clona un repositorio.                                                                                |



# Cargar todos los alias globales de una sola vez

## Copiá y pegá esto tal cual en tu terminal:

```
git config --global alias.st "status"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.cm "commit -m"
git config --global alias.ca "commit --amend"
git config --global alias.df "diff"
git config --global alias.di "diff --cached"
git config --global alias.ad "add"
git config --global alias.adp "add -p"

git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.l "log --oneline"

git config --global alias.pl "pull"
git config --global alias.prb "pull --rebase"
git config --global alias.ps "push"
git config --global alias.psf "push --force-with-lease"

git config --global alias.rb "rebase"
git config --global alias.rbi "rebase -i"
git config --global alias.mg "merge"

git config --global alias.brd "branch -d"
git config --global alias.brf "branch -D"

git config --global alias.s "status -sb"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
git config --global alias.undo "reset --soft HEAD~1"

git config --global alias.ss "stash save"
git config --global alias.ssp "stash pop"
git config --global alias.ssl "stash list"

git config --global alias.cl "clone"
git config --global alias.ft "fetch"
git config --global alias.fta "fetch --all"
git config --global alias.r "remote -v"
```

## Para verificar que todo quedó cargado

```
git config --global --get-regexp alias
```