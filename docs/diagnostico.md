# Diagnóstico del sistema base

Versión analizada: tag `estado-inicial`, archivo `solicitudes.py` (356 líneas, una sola clase `Sistema` más 3 funciones sueltas).

## 1. Calidad externa vs. calidad interna (Etapa 2)

1. Calidad Externa (La mirada del usuario)
La calidad externa se refiere a cómo se comporta el software ante los ojos de los usuarios finales (en este caso, el personal administrativo que gestiona las solicitudes académicas). Desde esta perspectiva, el sistema funciona correctamente y cumple con los requerimientos planteados:

- **Flujo completo de negocio**: El menú interactivo permite realizar todas las operaciones esperadas: registrar estudiantes (con validaciones básicas de email y campos vacíos), crear solicitudes, asignar prioridades automáticas (según el tipo o la palabra clave "urgente"), cambiar estados mediante una máquina de estados controlada (Pendiente -> En Proceso -> Resuelta -> Cerrada), asignar responsables, filtrar, emitir reportes con estadísticas y eliminar registros.

- **Persistencia de datos**: Los datos se guardan de forma persistente en un archivo JSON (datos.json), lo que evita la pérdida de información al cerrar la aplicación.

- **Experiencia interactiva inmediata**: La interfaz de consola es directa y proporciona realimentación textual clara de lo que va ocurriendo en cada paso.


2. Calidad Interna(La mirada del desarrollador)

La calidad interna abarca la arquitectura del código, su diseño, legibilidad y mantenibilidad. Desde esta perspectiva, el código presenta importantes deficiencias y deudas técnicas:

 - **Acoplamiento extremo (UI y Lógica mezcladas)**: Toda la lógica de negocio, validaciones y reglas de transición de estados están fuertemente acopladas con las operaciones de E/S de consola (input() y print()). No hay separación de responsabilidades.

- **Nombres crípticos y legibilidad reducida**: Se utilizan variables de una sola letra (d, e, f, s, al, rid, t, leg), lo que dificulta la lectura y comprensión del código por otros desarrolladores (o por el mismo autor tiempo después).

- **Código muerto**: Se declara una variable global cont = 0  # contador de solicitudes al inicio del script que jamás se utiliza en todo el programa.


3. Relación: Funcionalidad Visible vs. Mantenibilidad, Legibilidad y Evolución Futura


- **Legibilidad y Mantenibilidad**: Al carecer de una estructura modular (clases especializadas, separación entre modelos de datos, persistencia y vistas), cualquier modificación simple —como cambiar el formato de la fecha, agregar un nuevo campo al estudiante o modificar la regla de una prioridad— requiere modificar funciones extensas donde todo está mezclado. Esto aumenta drásticamente el riesgo de introducir bugs colaterales.

- **Evolución Futura bloqueada**: Debido a que la lógica está atrapada en funciones de consola (input/print), es imposible evolucionar este sistema hacia una interfaz web (ej. Flask o FastAPI), una API REST o una interfaz gráfica de escritorio. Para hacerlo, prácticamente habría que reescribir todo el código desde cero, ya que la lógica de negocio no puede reutilizarse de forma independiente.

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
| 7 | Manejo de Exepciones silencioso |cargar(), reg_sol(),cambiar_estado(),asignar(),eliminar() | Alta |
| 8 | Nombres pobres de variables |Presente en todo el script (ej. cargar(),menu(),filtrar()) | Media |
| 9 | Primitive Obssesion | En todo el sistema | Alta |
| 10 | Acoplamiento fuerte al almacenamiento fisico | cargar(),agregar(),cambiar_estado(),etc | Media |
| 11 | Código muerto (Dead Code) | Línea 7 del código original. | Baja |
| 12 | Desnormalización y duplicación de datos | Método agregar() | Media |

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

### Smells 7 Manejo de Exepciones silencioso

**Ubicación:** Funciones cargar(), reg_sol(), cambiar_estado(), asignar() y filtrar().

**Evidencia:** Uso de bloques except: vacíos o sin especificar el tipo de excepción (ej. except: en cargar()), lo cual captura incluso interrupciones del sistema (KeyboardInterrupt), además de ocultar errores en filtrar() mediante pass.

**Impacto:** Enmascara fallos reales de programación o corrupción de archivos, dificultando el debugging y provocando comportamientos erráticos silenciosos.

**Prioridad:** Alta. Se corrige capturando excepciones especificas, y si ocurren errores debe informarse al usuario o registrarse(log)

### Smells 8 Nombres pobres de variables

**Ubicacion:** Presente en todo el script (ej. cargar(), menu(), filtrar()).

**Evidencia:** Uso masivo de identificadores crípticos o abreviados como d (para los datos globales), x / x2 (para iteraciones), al (para alertas), enc (bandera de encontrado) y id.


**Impacto:** Dificulta severamente la lectura, comprensión y legibilidad del código. Obliga al desarrollador a retener en mente el contexto de abreviaturas ambiguas


**Prioridad:** Media. A medida que crezca el código, las variables genéricas causarán confusión y errores por colisión de nombres. Pero se corrige cambiando el nombre de las variables y asignandoles nombres mas especificos para su funcion o lo que guardan.



### Smells 9 Primitive Obsession

**Ubicación:** En todo el sistema (manipulación directa de claves como s["estado"], s["resp"], etc.)

**Evidencia:** Las entidades del dominio (estudiantes, responsables, solicitudes) se manejan como diccionarios puros basados en claves de texto plano en lugar de clases u objetos tipados con métodos propios.

**Impacto:** Alto acoplamiento a estructuras de datos primitivas, propenso a errores tipográficos, ausencia de autocompletado y validación de tipos tardía.

**Prioridad de corrección:** Alta.


### Smells 10 Acoplamiento fuerte al almacenamiento fisico

**Ubicación:** Funciones cargar(), agregar(), cambiar_estado(), asignar(), eliminar().

**Evidencia:** La lógica de negocio y las funciones de interfaz interactúan de manera directa con operaciones de archivos de texto plano mediante json.load() y json.dump(), sin ninguna abstracción o capa de repositorio de datos.

**Impacto:** Imposibilita la migración a otro sistema de persistencia (como una base de datos relacional) sin reescribir prácticamente toda la aplicación.

**Prioridad de corrección:** Meida.

### Smells 11 Codigo muerto (Dead Code)

**Ubicación:** Línea 7 del código original.

**Evidencia:** Declaración de la variable global cont = 0  # contador de solicitudes, la cual nunca se lee ni se modifica en ninguna parte del script.

**Impacto:** Genera ruido visual y confusión en quien lee el código, haciéndole creer que existe un mecanismo contador activo.

**Prioridad de corrección:** Baja. Se corrige dandole una funcionalidad a la variable o eliminandola si su uso no es necesario.


### Smells 12 Desnormalización y duplicación de datos

**Ubicación:** Método agregar().

**Evidencia:** El nombre completo del estudiante ("nombre_est": nom + " " + ape) se almacena duplicado dentro de cada solicitud en lugar de resolverse dinámicamente mediante la relación con el legajo del estudiante.

**Impacto:** Si un estudiante actualiza sus datos personales (nombre o apellido), las solicitudes históricas mantendrán el nombre viejo, generando inconsistencia de datos.

**Prioridad de corrección:** Media.



## Etapa-Diagnostico
PENDIENTE
