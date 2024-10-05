---
title: Permisos en Linux
date: 2024-10-03
description: Información de permisos en ficheros y directorios de Linux.
categories:
    - Permissions
tags:
    - Permissions
media_subpath: /assets/img/commons/permissions/
# image:
---

## Niveles de acceso:

1. **Usuario (Owner)**: El propietario del archivo o directorio.
2. **Grupo**: Los usuarios que pertenecen al mismo grupo que el archivo.
3. **Otros (Others)**: Todos los demás usuarios del sistema.

## Permisos

- **r** (read): Permiso para leer el archivo o listar el contenido de un directorio.
- **w** (write): Permiso para modificar el contenido del archivo o modificar el contenido de un directorio (añadir, eliminar archivos dentro del directorio).
- **x** (execute): Permiso para ejecutar el archivo (si es un archivo ejecutable) o para entrar en un directorio.

## Representación de los permisos

Cuando ejecutas el comando `ls -l` en la terminal, verás algo como esto:

```bash
drwxr-xr-x 2 user group 4096 Oct 4 12:34 nombre_del_directorio
```

Este es el formato en el que se muestran los permisos de un archivo o directorio. Aquí desglosamos qué significa cada parte de la primera columna:

- El primer carácter: Indica el tipo de archivo o directorio:
    - `d`: Directorio.
    - `-`: Archivo regular.
    - `l`: Enlace simbólico.
- Los siguientes nueve caracteres están agrupados en tres conjuntos, y cada uno representa los permisos de **usuario**, **grupo** y **otros**, respectivamente:
    - `rwx`: El propietario tiene permisos de **lectura (r)**, **escritura (w)** y **ejecución (x)**.
    - `r-x`: El grupo tiene permisos de **lectura (r)** y **ejecución (x)**, pero no de escritura (sin `w`).
    - `r-x`: Otros usuarios (todos los demás) también tienen permisos de **lectura (r)** y **ejecución (x)**, pero no de escritura.

# Representación numérica

Los permisos también pueden representarse numéricamente usando un sistema de valores octales:

- **r (read)** = 4
- **w (write)** = 2
- **x (execute)** = 1

Para cada conjunto de permisos (usuario, grupo, otros), sumamos los valores correspondientes:

- **777**: Todos (usuario, grupo y otros) tienen permisos de **lectura (r)**, **escritura (w)** y **ejecución (x)**. (4+2+1 = 7)
- **755**: El propietario tiene permisos de **lectura (r)**, **escritura (w)** y **ejecución (x)** (4+2+1 = 7), pero el grupo y los otros usuarios solo tienen permisos de **lectura (r)** y **ejecución (x)** (4+0+1 = 5).
- **644**: El propietario tiene permisos de **lectura (r)** y **escritura (w)** (4+2=6), mientras que el grupo y los otros usuarios solo tienen permisos de **lectura (r)** (4+0+0 = 4).
