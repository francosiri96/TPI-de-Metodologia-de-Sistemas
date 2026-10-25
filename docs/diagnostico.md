# Diagnóstico del sistema base

Versión analizada: tag `estado-inicial`, archivo `solicitudes.py` (356 líneas, una sola clase `Sistema` más 3 funciones sueltas).

## 1. Calidad externa vs. calidad interna (Etapa 2)

PENDIENTE

## 2. Deuda técnica y code smells (Etapa 3)

Criterio de prioridad:
- **Alta**: genera errores o hace que cualquier cambio sea riesgoso. Se corrige primero.
- **Media**: complica entender o extender el código, pero no rompe nada hoy.
- **Baja**: es un problema de prolijidad, se corrige si queda tiempo.

### Resumen

| # | Smell | Ubicación | Prioridad |
|---|---|---|---|
| 1 | God Class | clase `Sistema` (líneas 64-351) | Alta |
| 2 | Long Method | `reg_sol` (100-157), `cambiar_estado` (177-216), `reporte` (297-332) | Alta |
| 3 | Long Parameter List | `agregar` (159), `filtrar` (275) | Media |
| 4 | Código duplicado | guardado del JSON, búsqueda por ID | Alta |
| 5 | Magic strings / magic numbers | estados, prioridades, cantidad de tipos | Media |
| 6 | Cadenas de if/elif (viola OCP) | prioridad por tipo, transiciones, nombres de estado | Media |
| 7 | PENDIENTE | | |
| 8 | PENDIENTE | | |
| 9 | PENDIENTE | | |
| 10 | PENDIENTE | | |
| 11 | PENDIENTE | | |
| 12 | PENDIENTE | | |

---

### Smell 1: God Class

**Ubicación:** clase `Sistema`, líneas 64 a 351 (casi todo el archivo).

**Evidencia:** la misma clase pide datos por teclado, valida, aplica las reglas de negocio, arma los textos que se muestran y escribe el archivo. Por ejemplo, en `asignar` (218-254) hay `input()`, la validación del estado, la búsqueda del responsable, el guardado con `json.dump` y el `print` del resultado, todo junto:

```python
    def asignar(self):
        print("--- Asignar responsable ---")
        try:
            id = int(input("ID de la solicitud: "))
        ...
        if s["estado"] == "C" or s["estado"] == "X":
            print("No se puede asignar responsable a una solicitud finalizada")
        ...
        f = open(ARCHIVO, "w")
        json.dump(self.d, f, indent=2)
        f.close()
        print("Responsable " + nombre + " asignado a la solicitud " + str(id))
```

**Impacto:** no hay forma de usar o probar una regla de negocio (por ejemplo "no se puede asignar responsable a una solicitud cerrada") sin pasar por la consola. Las pruebas que armamos tienen que simular a un usuario escribiendo en el teclado.

**Riesgo de mantenimiento:** alto. Si mañana se quiere una interfaz web o guardar en una base de datos, hay que tocar todos los métodos de la clase. Viola SRP: la clase tiene al menos 3 motivos para cambiar (interfaz, reglas, persistencia).

**Prioridad:** Alta.

---

### Smell 2: Long Method

**Ubicación:** `reg_sol` (líneas 100-157, 58 líneas), `cambiar_estado` (177-216, 40 líneas), `reporte` (297-332, 36 líneas).

**Evidencia:** `reg_sol` hace 7 cosas distintas en un solo método:
1. busca el estudiante por legajo (102-106)
2. si no existe, pide sus datos, los valida y lo da de alta (107-124)
3. muestra los tipos y pide uno (125-136)
4. pide y valida la descripción (137-140)
5. calcula la prioridad (141-153)
6. genera el ID (154)
7. arma la solicitud y la guarda (155-157)

**Impacto:** para entender cómo se calcula la prioridad hay que leer todo el método. Además, al estar todo mezclado aparece un error real: el estudiante se agrega a la lista en la línea 121, **antes** de validar el tipo y la descripción. Si después falla la validación, la solicitud no se crea pero el estudiante queda cargado igual. Lo comprobamos con la prueba `validaciones`: en el quinto intento ya no pregunta si se quiere registrar al estudiante 2002, porque quedó agregado en el intento anterior que falló.

**Riesgo de mantenimiento:** alto. Cualquier cambio en una de las 7 partes obliga a releer las otras 6 y es fácil romper algo sin darse cuenta.

**Prioridad:** Alta.

---

### Smell 3: Long Parameter List

**Ubicación:** `agregar` (línea 159) y `filtrar` (línea 275).

**Evidencia:**

```python
    def agregar(self, id, leg, nom, ape, tipo, desc, prio, est, resp, fecha):
```

```python
        self.agregar(id, leg, est["nombre"], est["apellido"], tipo, desc, p, "P", None, fecha)
```

```python
    def filtrar(self, e, t, p, r):
```

**Impacto:** con 10 parámetros es muy fácil pasarlos en otro orden (por ejemplo `nom` y `ape`, o `tipo` y `desc`, que son todos strings) y Python no avisa nada. En `filtrar` los nombres de una letra obligan a ir a `buscar` para saber qué es cada uno. Además, `estado` y `responsable` siempre valen `"P"` y `None` cuando se crea una solicitud, así que no deberían ser parámetros.

**Riesgo de mantenimiento:** medio. Agregar un filtro nuevo (por ejemplo por fecha) obliga a cambiar la firma de `filtrar` y todas las llamadas.

**Prioridad:** Media. Se corrige con Introduce Parameter Object (un objeto `Solicitud` y un objeto `FiltroSolicitud`).

---

### Smell 4: Código duplicado

**Ubicación y evidencia:**

a) Guardar el archivo se repite igual en 5 lugares (líneas 25, 173, 211, 251 y 346):

```python
        f = open(ARCHIVO, "w")
        json.dump(self.d, f, indent=2)
        f.close()
```

b) Buscar una solicitud por ID se repite en `cambiar_estado` (185), `asignar` (226) y `eliminar` (341):

```python
        s = None
        for x in self.d["solicitudes"]:
            if x["id"] == id:
                s = x
```

c) Buscar un responsable por ID se repite en `asignar` (243) y en `buscar` (269).

**Impacto:** si se cambia la forma de guardar (otro nombre de archivo, otra codificación, una base de datos) hay que cambiarlo en 5 lugares. Si se olvida uno, algunas operaciones guardan de una forma y otras de otra.

**Riesgo de mantenimiento:** alto. Además, `open` sin `with` deja el archivo abierto si `json.dump` falla.

**Prioridad:** Alta. Se corrige con Extract Method y moviendo la persistencia a una clase repositorio.

---

### Smell 5: Magic strings y magic numbers

**Ubicación:**
- Estados como letras sueltas: `"P"`, `"E"`, `"R"`, `"C"`, `"X"` en las líneas 40-49, 156, 195-207, 232, 307-316 y 326.
- Prioridades como números: `1`, `2`, `3` en 54-61, 142-153 y 326.
- La cantidad de tipos escrita a mano en la línea 133, aunque la lista está en la línea 68:

```python
        if t < 1 or t > 5:
```

**Impacto:** hay que saber de memoria que `"X"` es Rechazada y que `3` es Alta. Un error de tipeo como `"x"` en minúscula o `"Rechazado"` no da ningún error, simplemente la condición nunca se cumple. Si se agrega un sexto tipo a la lista, la línea 133 lo rechaza como inválido.

**Riesgo de mantenimiento:** medio.

**Prioridad:** Media. Se corrige con constantes o con `Enum` para estados y prioridades.

---

### Smell 6: Cadenas de if/elif que violan OCP

**Ubicación:** prioridad según tipo (líneas 142-153), transiciones de estado (195-207), nombre de cada estado (39-52) y conteo por estado en el reporte (306-316).

**Evidencia:**

```python
        if tipo == "Reclamo de nota":
            p = 3
        elif tipo == "Equivalencia":
            p = 2
        elif tipo == "Cambio de comision":
            p = 2
        elif tipo == "Certificado":
            p = 1
        else:
            p = 1
```

```python
        if s["estado"] == "P":
            if n == "E":
                ...
            elif n == "X":
                ok = True
        elif s["estado"] == "E":
            if n == "R" or n == "X":
                ok = True
        elif s["estado"] == "R":
            if n == "C":
                ok = True
```

**Impacto:** el conjunto de estados está repetido en 4 cadenas de if distintas. Agregar un estado nuevo (por ejemplo "En revisión") obliga a modificar las 4 y no hay nada que avise si se olvida una. Esto viola el principio Abierto/Cerrado: para extender el sistema hay que modificar código que ya funciona.

**Riesgo de mantenimiento:** medio.

**Prioridad:** Media. Se corrige reemplazando los if por tablas (diccionarios de prioridad por tipo y de transiciones permitidas) dentro de las clases del modelo.

---

### Smells 7 a 12

PENDIENTE
