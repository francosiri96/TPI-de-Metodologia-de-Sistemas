# Sistema de Gestion de Solicitudes Academicas

TPI de Metodologia de Sistemas II - UTN TUP a distancia.

Sistema de consola para registrar solicitudes de estudiantes, clasificarlas por tipo, prioridad y estado, asignarles un responsable, buscarlas y sacar un reporte.

**Version actual: estado inicial.** Funciona, pero tiene problemas de diseño que vamos a diagnosticar y corregir en las siguientes etapas del trabajo.

## Tecnologias

- Python 3 (solo libreria estandar)
- Los datos se guardan en `datos.json`, que se crea solo la primera vez que se ejecuta

## Como usarlo

```
git clone <url-del-repo>
cd <carpeta>
python solicitudes.py
```

## Entidades

- Estudiante: legajo, nombre, apellido, email, carrera
- Responsable: id, nombre, area (vienen precargados)
- Solicitud: id, estudiante, tipo, descripcion, prioridad, estado, responsable, fecha
- Estado: Pendiente, En proceso, Resuelta, Cerrada, Rechazada

## Funcionalidades

1. Registrar solicitud (si el estudiante no existe se lo da de alta; la prioridad se calcula sola segun el tipo)
2. Cambiar estado
3. Asignar responsable
4. Buscar / filtrar solicitudes por estado, tipo, prioridad o responsable
5. Reporte por estado, con alerta de solicitudes urgentes sin responsable
6. Eliminar solicitud

## Reglas

- Prioridad: Reclamo de nota es Alta, Equivalencia y Cambio de comision son Media, Certificado y Otro son Baja. Si la descripcion dice "urgente" pasa a Alta.
- Cambios de estado permitidos: Pendiente a En proceso (tiene que tener responsable), Pendiente a Rechazada, En proceso a Resuelta, En proceso a Rechazada, Resuelta a Cerrada.
- No se puede asignar responsable a una solicitud Cerrada o Rechazada.
- La descripcion tiene que tener al menos 10 caracteres y el email tiene que tener "@".

Los datos de prueba son inventados.
