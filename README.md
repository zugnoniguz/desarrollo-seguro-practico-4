# Herramienta

Semgrep es la herramienta que utilizaré en este informe.

## Funcionamiento

Semgrep es un motor determinista de reglas que analiza la sintaxis del
código.

Primero se determina el lenguaje utilizado para el proyecto recorriendo los
archivos del mismo a través del FS o git, para así determinar qué reglas
aplicar. Una vez se determina qué lenguajes se utilizan, se aplican una serie
de reglas predefinidas que corresponden con código mal-escrito o inseguro en
los lenguajes encontrados.

Estas reglas son definidas como patrones de cadenas, es decir, se escribe el
código no deseado en el lenguaje que se va a analizar. Luego Semgrep convierte
la estructura del programa y las reglas en sus ASTs correspondientes y verifica
la correspondencia utilizando estos, y no simplemente el código fuente
original. Esto permite una fácil escritura de las reglas, ya que implica
escribir código y no reglas sobre ASTs, pero a su vez permite definir reglas
que verifiquen aspectos más complejos del programa y a su vez limita la
ocurrencia de falsos positivos y negativos.

Estas reglas pueden venir de varias fuentes. En principio, Semgrep ofrece un
repositorio de reglas que el usuario puede descargar, pero el usuario también
puede definir las reglas que desee o utilizar otras, según las necesidades del
análisis a realizar.

Finalmente, una vez aplicadas todas las reglas, el programa reune las reglas que
fallaron y los fragmentos de código relevantes y recopila un reporte para luego
ser procesado. El mismo puede ser visto de manera visual, en la terminal, o
exportado en algún formato estructurado como JSON para ser procesados por otra
herramienta.

## Limitaciones de la Community Edition

Semgrep CE es la versión gratuita del producto, que respecto a las versiones
comerciales presenta varias limitaciones.

- El análisis es limitado a un solo archivo. No contempla el flujo de datos
  para reglas que puedan involucrar varios archivos.

- En la CE las reglas de taint (manchado de variables) solo siguen el flujo
  dentro de la misma función. Las versiones comerciales ofrecen análisis de
  flujo de datos entre funciones (taint analysis, constant propagation,
  metavariables tipadas) para detectar si un input llega a un sink a través de
  una cadena de llamadas.

- La CE solo incluye reglas de la comunidad. Las versiones comerciales añaden
  reglas avanzadas mantenidas por Semgrep, con menor tasa de falsos positivos.
  Además se ofrece un acceso más amplio al repositorio de Semgrep.

- La CE soporta los 30+ lenguajes estándar; las versiones comerciales agregan
  soporte para lenguajes como Apex (Salesforce). TODO: Add something else?

- La CE es solo CLI, con resultados en la terminal o exportados como JSON. Las
  versiones comerciales incluyen dashboard web, reportes, gestión de políticas y
  trazabilidad de hallazgos a lo largo del tiempo.

- Las versiones comerciales suman Semgrep Secrets (secretos con análisis
  semántico), Semgrep Supply Chain (dependencias/SCA), Semgrep Guardian (código
  generado por IA) y Agentic Workflows (LLMs para análisis más profundo y triage
  automatizado).

- Las versiones comerciales ofrecen escaneos gestionados, guardrails y
  priorización centralizada para CI/CD.

## Fuentes

- Semgrep Community Edition: <https://semgrep.dev/products/community-edition/>
- Semgrep Pro Engine: <https://semgrep.dev/products/pro-engine/>
- Documentación general: <https://semgrep.dev/docs/>
- Semgrep Registry (repositorio de reglas): <https://semgrep.dev/explore>

# Análisis con la herramienta

Para ejecutar las reglas, se ejecutó el comando `semgrep scan`, sin ninguna
configuración, en cada proyecto. Esto ejecuta las reglas de la CE que incluye
por defecto.

Nótese que en el análisis la única vulnerabilidad que se asume que existe es la
que está definida que existe en la letra del ejercicio. Para el análisis con
semgrep no se contempló la existencia de otras vulnerabilidades fuera de las
que la herramienta destaque.

También se ignoraron vulnerabilidades resaltadas en los archivos de test
introducidos en el práctico 4, ya que estos no son relevantes para la
aplicación.

## Ejercicio 1

### Rama `main`

#### Hallazgo 1

```dockerfile
# ...
CMD ["sh", "-c", "python init_db.py && python app.py"]
```

```
By not specifying a USER, a program in the container may run as 'root'.
This is a security hazard. If an attacker can control a process running
as root, they may have control over the container. Ensure that the last
USER in a Dockerfile is a USER other than 'root'.
Details: https://sg.run/Gbvn
```

Este resultado es correcto. En un contenedor de docker, el proceso de la
aplicación nunca debería correr como el usuario `root`. Este es un verdadero
positivo.

#### Falsos negativos

La inyección SQL en la aplicación, que es evidente y en lo que consitía el
ejercicio, no fue detectada.

### Rama `practico-2`

Se mantienten las mismas detecciones que en la rama `main`, y por ende estas
siguen siendo verdaderos positivos.

La única diferencia es que ahora la vulnerabilidad de inyección SQL fue
remediada, y por ende que no ocurra en el análisis es un verdadero negativo.

## Ejercicio 2

### Rama `main`

#### Hallazgo 1

```dockerfile
CMD ["sh", "-c", "python init_db.py && python app.py"]
```

```
By not specifying a USER, a program in the container may run as 'root'.
This is a security hazard. If an attacker can control a process running
as root, they may have control over the container. Ensure that the last
USER in a Dockerfile is a USER other than 'root'.
Details: https://sg.run/Gbvn
```

Al igual que en el ejercicio 1, esto es un verdadero positivo.

#### Falsos negativos

La vulnerabilidad del ejercicio, el xss en el template de la aplicación, no es
detectado.

### Rama `practico-2`

Se mantienten las mismas detecciones que en la rama `main`, y por ende estas
siguen siendo verdaderos positivos.

La única diferencia es que ahora la vulnerabilidad de xss fue
remediada, y por ende que no ocurra en el análisis es un verdadero negativo.

## Ejercicio 3

### Rama `main`

#### Hallazgo 1

```dockerfile
ENTRYPOINT ["java", \
    "-jar", "app.jar", \
    "--spring.sql.init.mode=always", \
    "--spring.datasource.url=jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE", \
    "--spring.jpa.hibernate.ddl-auto=update", \
    "--app.upload-dir=/uploads", "--debug"]
```

```
By not specifying a USER, a program in the container may run as 'root'.
This is a security hazard. If an attacker can control a process running
as root, they may have control over the container. Ensure that the last
USER in a Dockerfile is a USER other than 'root'.
Details: https://sg.run/k281
```

Como en el ejercicio 1 y 2, esto es un verdadero positivo.

#### Hallazgo 2

```html
<form method="post"
      th:action="@{/upload/{id}(id=${pelicula.id})}"
      enctype="multipart/form-data">
    <label for="afiche">Seleccionar archivo:</label>
    <input type="file" name="afiche" id="afiche" accept="*/*" required>
    <br><br>
    <button type="submit">Subir afiche</button>
</form>
```

```
Manually-created forms in django templates should specify a csrf_token to
prevent CSRF attacks.
Details: https://sg.run/N0Bp
```

TODO: Fill in this is a false positive

#### Falsos negativos

No se detectan las vulnerabilidades de File Upload en el proyecto, incluyendo
el path traversal y la posible inclusión de archivos que no sean imágenes en el
servidor.

### Rama `practico-2`
