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

- La CE soporta los 30+ lenguajes estándar, y las versiones comerciales agregan
  soporte para lenguajes "enterprise" habilitados con `--pro-languages`
  (actualmente Apex, C# y Elixir).

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
por defecto. Los resultados se leyeron de la terminal directamente, sin
exportarlos a archivos externos.

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

```
dockerfile.security.missing-user.missing-user
```

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

```
dockerfile.security.missing-user.missing-user
```

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

```
dockerfile.security.missing-user-entrypoint.missing-user-entrypoint
```

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

```
python.django.security.django-no-csrf-token.django-no-csrf-token
```

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

Este hallazgo es un falso positivo, ya que esta vulnerabilidad, como indica
su nombre aplica a aplicaciones escritas en Python y Django, lo cual no es el
caso de esta aplicación en Spring Boot y Java.

Más allá de eso, tampoco aplica en este caso, ya que el formulario no incluye
autenticación, entonces no existe vulnerabilidad CSRF. Esto también podría
ser considerado vulnerabilidad, y sería un falso negativo, pero no será
contemplado.

#### Falsos negativos

No se detectan las vulnerabilidades de File Upload en el proyecto, incluyendo
el path traversal y la posible inclusión de archivos que no sean imágenes en el
servidor.

### Rama `practico-2`

Se mantienen las mismas detecciones de la rama `main`, y por ende se mantiene
su estado como verdadero positivo y falso positivo respectivamente.

En cambio, la vulnerabilidad que era un falso negativo sí fue remediada, y por
ende pasa a ser un verdadero negativo.

## Ejercicio 4

### Rama `main`

#### Hallazgo 1

```
dockerfile.security.missing-user-entrypoint.missing-user-entrypoint
```

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

Por la misma razón que anteriores, es un verdadero positivo.

#### Hallazgo 2

```
java.spring.security.audit.spel-injection.spel-injection
```

```java
public String evaluate(String expression) {
   if (expression == null || expression.isBlank()) {
       return "";
   }

   ExpressionParser parser = new SpelExpressionParser();

   org.springframework.expression.EvaluationContext context =
    SimpleEvaluationContext.forReadOnlyDataBinding().build();
```

```
A Spring expression is built with a dynamic value. The source of the
value(s) should be verified to avoid that unfiltered values fall into
this risky code evaluation.
```

Esto es un verdadero positivo. Es la vulnerabilidad a remediar de este
ejercicio, y el proceso de explotación fue detallado en el práctico 2.

#### Falsos negativos

No se identificaron falsos negativos en este análisis

### Rama `practico-2`

El hallazgo 1 se mantuvo, y sigue siendo válido. En cambio, el segundo hallazgo
sigue siendo reportado, pese a que se haya remediado la vulnerabilidad. Esto se
debe a que el código que genera la vulnerabilidad sigue estando en el
repositorio, pese a que nunca sea utilizado por la aplicación.

Esto implicaría que es un falso positivo, pero igualmente es importante que la
herramienta reporte código inseguro, pese a que este nunca sea utilizado en la
aplicación, ya que la clase podría ser utilizada en el futuro si el
desarrollador no es cuidadoso.

## Ejercicio 5

### Rama `main`

#### Hallazgo 1

```
dockerfile.security.missing-user-entrypoint.missing-user-entrypoint
```

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

Como los ejercicios anteriores, esto es un verdadero positivo.

#### Hallazgo 2

```
java.lang.security.audit.crypto.use-of-aes-ecb.use-of-aes-ecb
```

```java
private static final String CIPHER_ALGO = "AES/ECB/PKCS5Padding";
Cipher cipher = Cipher.getInstance(CIPHER_ALGO);
```

```
Use of AES with ECB mode detected. ECB doesn't provide message
confidentiality and  is not semantically secure so should not be used.
Instead, use a strong, secure cipher:
Cipher.getInstance("AES/CBC/PKCS7PADDING"). See https://owasp.org/www-
community/Using_the_Java_Cryptographic_Extensions for more information.
Details: https://sg.run/dB2Y
```

Esto es técnicamente un verdadero positivo, ya que es cierto que al utilizar
AES, no se debe utilizar `ECB` para encripción.

#### Hallazgo 3

```
python.django.security.django-no-csrf-token.django-no-csrf-token
```

```html
<form th:action="@{/login}" method="post">
   <label>Usuario</label>
   <input type="text" name="username" placeholder="nombre de usuario" required>
   <label>Contrase&#241;a</label>
   <input type="password" name="password" placeholder="tu contrase&#241;a" required>
   <button type="submit">Entrar</button>
</form>

<form th:action="@{/register}" method="post">
   <label>Usuario</label>
   <input type="text" name="username" placeholder="elige un usuario" required>
   <label>Contrase&#241;a</label>
   <input type="password" name="password" placeholder="elige una contrase&#241;a" required>
   <label>Confirmar Contrase&#241;a</label>
   <input type="password" name="confirmPwd" placeholder="repite la contrase&#241;a" required>
   <button type="submit">Registrarse</button>
</form>
```

```
Manually-created forms in django templates should specify a csrf_token to
prevent CSRF attacks.
Details: https://sg.run/N0Bp
```

Aplica la misma lógica que en el ejercicio anterior, ya que esta aplicación
también utiliza Spring Boot.

#### Falsos negativos

No se identifica la vulnerabilidad de utilizar encripción y no hashing para
procesar la contraseña.

### Rama `practico-2`

Se produce el mismo análisis, y las evaluaciones realizadas en la sección
anterior siguen aplicando para el análisis de semgrep en esta rama.

La única diferencia es con el falso negativo, que en esta rama es un verdadero
negativo ya que la vulnerabilidad es remediada.

# Análisis con LLMs

Para este análisis, se corrió el prompt con `GLM-5.3-Flash Free` en el nivel de
esfuerzo `high` en el harness `opencode`. Para cada ejercicio, se utilizó el
siguiente prompt y se inició el prompt desde el directorio del ejercicio, una
vez para cada rama.

```
Perform a security audit of the codebase within this directory. Analyse only
the code within this directory, and assume this is a production codebase that
serves traffic to users.

Report findings individually. Each finding should contain a short heading
describing what categoy this vulnerability falls within (SQL injection, XSS,
Directoy Travsel, etc), a short snippet of the vulnerable code and an
explanation as to what the vulnerability is below. There should be nothing
other than what is mentioned here, neither recommended fixes nor further
analysis.

Output the findings in a file called `ANALYSIS.md` in the root of the project,
including only the vulnerabilities found.
```

TODO: Justify why english and not spanish

El análisis producido por cada LLM será adjunto aquí, por completitud.

## Ejercicio 1

### Rama `main`

#### SQL Injection (search parameter)

```python
def buscar_funciones(query, sort_by='nombre', sort_dir='ASC'):
    db = get_db()
    sql = f"SELECT peliculas.nombre as pelicula, funciones.fecha_hora, " \
          f"(funciones.asientos_totales - funciones.asientos_ocupados) as disponibles " \
          f"FROM funciones " \
          f"JOIN peliculas ON funciones.pelicula_id = peliculas.id " \
          f"WHERE peliculas.nombre LIKE '%{query}%' " \
```

The `query` value comes directly from the `buscar` request parameter
(`request.args.get('buscar', '')` in the `/` route) and is interpolated
verbatim into the SQL string via an f-string before being executed with
`db.execute(sql)`. There is no parameterization, escaping, or validation. An
attacker can close the single quote and inject arbitrary SQL into the `WHERE`
clause, including `OR 1=1` tautologies, `UNION SELECT` payloads to extract data
from any table in the SQLite database, and blind/error-based injection
techniques. Because the query results are rendered back into the HTML table, a
`UNION`-based injection directly leaks database contents to the attacker.

#### SQL Injection (ORDER BY direction)

```python
f"ORDER BY {'peliculas.nombre' if sort_by == 'nombre' else 'funciones.fecha_hora'} " \
f"{sort_dir}"
return db.execute(sql).fetchall()
```

The `sort_dir` value comes directly from the `sentido` request parameter
(`request.args.get('sentido', 'ASC')`) and is appended unvalidated to the end
of the SQL statement. While `sort_by` is constrained by a ternary whitelist,
`sort_dir` is not, so any attacker-supplied string is concatenated into the
query. SQL can be injected after `ORDER BY <column>`, e.g. using expressions
such as `ASC; ...` or subexpressions like `ORDER BY funciones.fecha_hora
(SELECT ...)`, enabling error-based and blind (boolean/time-style) SQL
injection to exfiltrate or manipulate data.

#### Remote Code Execution via Enabled Debugger (Insecure Default Configuration)

```python
if __name__ == '__main__':
    import os
    host=os.environ.get('FLASK_HOST', '0.0.0.0')
    debug = os.environ.get('DEBUG_MODE', 'true').lower() in ('true', '1', 'yes')
    app.run(debug=debug,host=host)
```

Flask/Werkzeug debug mode is enabled by default: `DEBUG_MODE` defaults to
`'true'` unless explicitly overridden in the environment, and the app binds to
all interfaces (`0.0.0.0`). The Dockerfile's CMD (`python app.py`) is exactly
this entrypoint, so the production container runs with the interactive Werkzeug
debugger active and exposed on the mapped port 5000. When an unhandled
exception occurs, the debugger error page is served to any remote user and
permits arbitrary Python code execution protected only by a weakly random PIN
that can be brute-forced, giving unauthenticated remote code execution on the
server.

#### Excessive Privileges (Container Runs as Root)

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["sh", "-c", "python init_db.py && python app.py"]
```

The Dockerfile declares no `USER` directive, so the Flask application runs as
`root` inside the container. Combined with the debugger-enabled default above,
any code execution obtained through the application (e.g. via the Werkzeug
debugger) runs with full root privileges within the container, maximizing the
impact of a compromise of the host's namespace, mounted resources, and any
other root-level capabilities granted to the container.

### Rama `practico-2`

TODO:

## Ejercicio 2

### Rama `main`

#### SQL Injection — search query interpolated into LIKE clause

```python
sql = (
    f"SELECT peliculas.nombre as pelicula, funciones.fecha_hora, "
    ...
    f"WHERE peliculas.nombre LIKE '%{query}%' "
    ...
)
return db.execute(sql).fetchall()
```

In `app.py`, `buscar_funciones()` builds the SQL statement with an f-string and
interpolates `query`, which comes directly from `request.args.get('buscar',
'')` in the `/` route, into the `WHERE` clause. A request such as `/buscar=x%'
UNION SELECT ...--` allows an attacker to inject arbitrary SQL, since the value
is concatenated into the statement instead of being passed as a parameter.

#### SQL Injection — sort direction interpolated into ORDER BY

```python
def buscar_funciones(query, sort_by='nombre', sort_dir='ASC'):
    sql = (
        ...
        f"ORDER BY {'peliculas.nombre' if sort_by == 'nombre' else 'funciones.fecha_hora'} "
        f"{sort_dir}"
    )
```

In `app.py`, `sort_dir` is taken from `request.args.get('sentido', 'ASC')` and
interpolated verbatim into the `ORDER BY` clause. While `sort_by` is
constrained by a ternary check, `sort_dir` has no validation, allowing an
attacker to inject arbitrary SQL after `ORDER BY` (e.g. with subqueries or
multi-statement payloads depending on driver behavior).

#### Stored XSS — description rendered with the `| safe` filter

```html
{{ pelicula['descripcion'] | safe }}
```

In `templates/edit.html`, the movie description stored in the database is
rendered with the `| safe` filter, which disables Jinja's automatic HTML
escaping. Any HTML/JavaScript previously saved through the `/edit/<id>` POST
endpoint (e.g. `<script>alert(document.domain)</script>`) is executed in the
browser of every user who visits the edit page. This test file confirms the
vulnerable behavior exists (`test_xss.py` asserts the escaped rendering, which
the template's `| safe` filter defeats).

#### Missing Authentication and Authorization on edit endpoints

```python
@app.route('/edit/<int:pelicula_id>', methods=['GET'])
def edit_get(pelicula_id):
    ...

@app.route('/edit/<int:pelicula_id>', methods=['POST'])
def edit_post(pelicula_id):
    nombre = request.form.get('nombre', '')
    ...
```

Both edit endpoints in `app.py` perform no authentication or authorization
checks of any kind. Any visitor can view and modify the content of any movie
record, including injecting content that later gets rendered to all users (see
the stored XSS finding).

#### CSRF — no token on state-changing POST endpoint

```python
@app.route('/edit/<int:pelicula_id>', methods=['POST'])
def edit_post(pelicula_id):
    nombre = request.form.get('nombre', '')
    ...
    db.commit()
```

The `/edit/<id>` POST handler mutates database content but the form in
`templates/edit.html` contains no CSRF token and the application has no CSRF
protection (no `SECRET_KEY` is even configured). A third-party page can
silently submit forged POST requests to modify the application's data on behalf
of any visiting user.

#### Debug Mode Enabled by Default in Production

```python
debug = os.environ.get('DEBUG_MODE', 'true').lower() in ('true', '1', 'yes')
app.run(debug=debug, host=host)
```

In `app.py`, the Werkzeug development server runs with the interactive debugger
enabled unless `DEBUG_MODE` is explicitly set to a falsy value — the default is
`'true'`. When an unhandled exception occurs, the debugger exposes an
interactive Python console reachable by any remote user, which allows arbitrary
remote code execution. Combined with the `0.0.0.0` default bind address and the
`docker-compose.yml` port mapping, this is directly exposed to the network.

### Rama `practico-2`
TODO:

## Ejercicio 3

### Rama `main`

#### Unrestricted File Upload / Arbitrary File Write (Path Traversal)

```java
String filename = archivo.getOriginalFilename();
Path uploadPath = Paths.get(uploadDir);
...
Files.copy(archivo.getInputStream(), uploadPath.resolve(filename));
```

`PeliculaController.uploadFile` uses the client-supplied `getOriginalFilename()` directly as the destination filename with no sanitization or validation. A multipart filename containing traversal sequences (e.g. `../../etc/cron.d/pwn` or `..\..\` variants) causes `uploadPath.resolve(filename)` to escape the upload directory, letting any anonymous user write arbitrary files to any location writable by the application process.

#### Directory Traversal (Arbitrary File Read)

```java
@GetMapping("/uploads/{filename:.+}")
@ResponseBody
public ResponseEntity<Resource> serveFile(@PathVariable String filename) throws IOException {
    Path filePath = Paths.get(uploadDir).resolve(filename).normalize();
    Resource resource = new UrlResource(filePath.toUri());
```

`serveFile` resolves a user-controlled URL path segment against the upload directory and only calls `normalize()`, which collapses `../` sequences without checking that the resulting path stays inside `uploadDir`. A request such as `/uploads/..%2F..%2Fdata%2Fcine3.mv.db` (or `/uploads/../../etc/passwd`) lets any anonymous user read arbitrary files from the container filesystem, including the H2 database file itself, which contains the full contents of the database.

#### Stored XSS via Uploaded File Content

```java
Files.copy(archivo.getInputStream(), uploadPath.resolve(filename));
```

The upload endpoint performs no validation of file type, extension, or
content. Uploaded files are subsequently served by `serveFile` with
`Content-Disposition: inline` and a content type resolved from the file itself:

```java
MediaType mediaType = MediaTypeFactory.getMediaType(resource).orElse(MediaType.APPLICATION_OCTET_STREAM);
...
.header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"" + resource.getFilename() + "\"")
```

An attacker can upload a file named `x.html` or an SVG containing `<script>` on
the same origin (`/uploads/x.html`), and it will be rendered inline in the
browser by the application's own origin, executing JavaScript in the context of
any user who visits the link (stored XSS).

#### HTTP Response Header Injection

```java
.header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"" + resource.getFilename() + "\"")
```

The `Content-Disposition` header value is built by string concatenation with
the file's name, which is ultimately attacker-controlled via the upload
endpoint's `getOriginalFilename()`. A filename containing `"` or CRLF
characters breaks out of the quoted string and allows injection of arbitrary
response headers (HTTP response splitting), since no encoding or sanitization
is applied to the value.

#### Missing Access Control (Broken Authorization)

```java
@PostMapping("/upload/{id}")
public String uploadFile(@PathVariable Integer id, @RequestParam("afiche") MultipartFile archivo) ...
```

The entire application (controller and configuration) contains no
authentication or authorization mechanism; the file upload endpoint is fully
anonymous. Any unauthenticated visitor can upload or attach arbitrary files to
any movie record, and via the traversal in the filename, write files anywhere
on the host the application can reach.

#### Insecure Production Configuration

```java
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.sql.init.mode=always", ... "--debug"]
```

```properties
spring.jpa.show-sql=true
```

The container entrypoint enables Spring's `--debug` mode in production, which
turns on debug logging for core loggers and exposes the auto-configuration
report through actuator-style diagnostics, and `show-sql` prints all SQL
statements (including user-controlled query values) to logs. Both leak
sensitive internal information to anyone with access to container
logs. Additionally, the H2 database is opened with no credentials
(`jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE` with no user/password configured),
so any code path or adjacent process that can reach the database file or server
gets full unauthenticated access.

### Rama `practico-2`
TODO:

## Ejercicio 4

### Rama `main`

#### SpEL Injection (SSTI) → Remote Code Execution

```java
// src/main/java/com/cinebuscador/config/SpelEvaluator.java
public String evaluate(String expression) {
    ...
    ExpressionParser parser = new SpelExpressionParser();

    org.springframework.expression.EvaluationContext context =
        SimpleEvaluationContext.forReadOnlyDataBinding().build();

    StandardEvaluationContext standardContext = new StandardEvaluationContext();

    standardContext.setVariable("system", System.class);
    standardContext.setVariable("runtime", Runtime.class);

    var expr = parser.parseExpression(expression);
    Object result = expr.getValue(standardContext);
```

```java
// src/main/java/com/cinebuscador/controller/FuncionController.java
String spelResultado = spelEval.evaluate(buscar);
```

The raw, user-controlled `buscar` request parameter is passed directly into
`SpelExpressionParser.parseExpression()` and evaluated with a
`StandardEvaluationContext`. `StandardEvaluationContext` permits full SpEL
expression power, including type references (`T(...)`) and constructor
invocation, and this code additionally registers `System.class` and
`Runtime.class` as expression variables. An attacker can submit a search term
such as `T(java.lang.Runtime).getRuntime().exec('...')` or
`#runtime.getRuntime().exec('...')` and have arbitrary commands executed on the
server, resulting in full remote code execution. The restricted
`SimpleEvaluationContext` is created but never used; every evaluation goes
through the unrestricted `standardContext`.

#### Container Runs as Root

```dockerfile
# Dockerfile
FROM eclipse-temurin:17-jre-noble
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar", ...]
```

The Dockerfile contains no `USER` directive, so the application runs as root
inside the container. Combined with the SpEL remote code execution above, any
command executed through the application runs with root privileges within the
container, maximizing the impact of a compromise of the running service.

#### Debug Mode Enabled in Production

```dockerfile
# Dockerfile
ENTRYPOINT ["java", \
    "-jar", "app.jar", \
    "--spring.sql.init.mode=always", \
    "--spring.datasource.url=jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE", \
    "--spring.jpa.hibernate.ddl-auto=update", \
    "--app.upload-dir=/uploads", "--debug"]
```

The production container entrypoint launches the application with the `--debug`
flag, which enables debug-level logging for core Spring components. Verbose
debug output from a traffic-serving service can leak internal details
(configuration, framework internals, class names, SQL statements) into logs and
error handling paths, aiding attackers in reconnaissance. Relatedly,
`src/main/resources/application.properties` sets `spring.jpa.show-sql=true`,
causing every executed SQL statement to be logged.

#### Database Connection Without Credentials / Insecure JDBC URL

```yaml
# docker-compose.yml
environment:
  - SPRING_DATASOURCE_URL=jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE
```

```dockerfile
# Dockerfile
"--spring.datasource.url=jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE", \
```

The H2 database URL specifies no user or password, so the database runs with
default, unauthenticated access, and the `AUTO_SERVER=TRUE` flag starts a
mixed-mode automatic server that allows additional processes connecting to the
same database file to attach over TCP. If code execution is achieved inside the
container (or any other process reaches the file/socket), the application's
data store can be read or modified without any authentication.

### Rama `practico-2`
TODO:

## Ejercicio 5

### Rama `main`

#### 1. Hardcoded Cryptographic Key (Hardcoded Credentials / Insecure Cryptography)

```java
private static final String SECRET_KEY = "MySup3rS3cr3tK3y!2024CineBuscadorAES";
```

`src/main/java/com/cinebuscador/config/EncryptionService.java`

The AES-256 key used to encrypt all user passwords is embedded directly in the
source code. Anyone with access to the codebase, the compiled JAR, or the
container image can extract the key and decrypt every password stored in the
database. The key is never rotated, and the static initializer even silently
zero-pads short keys, guaranteeing the embedded value always resolves to the
same AES key.

#### 2. Use of AES/ECB Mode (Weak Cryptography)

```java
private static final String CIPHER_ALGO = "AES/ECB/PKCS5Padding";
```

`src/main/java/com/cinebuscador/config/EncryptionService.java`

Passwords are encrypted in ECB mode with no initialization vector. ECB is
deterministic: identical passwords always produce identical ciphertexts,
leaking whether two users share a password and enabling pattern analysis across
the stored credentials without needing to know the key.

#### 3. Passwords Stored Reversibly Instead of Hashed (Improper Password Storage)

```java
String decryptedPassword = EncryptionService.decrypt(user.getPassword());
if (password.equals(decryptedPassword)) {
```

`src/main/java/com/cinebuscador/controller/AuthController.java`

Passwords are encrypted rather than passed through a slow, one-way password
hash (e.g., bcrypt/argon2). Because decryption is required at login, every
stored password is fully recoverable by anyone who obtains the database
together with the key from the source code — an offline attacker recovers all
plaintext credentials immediately.

#### 4. Key-Material Exposure API (Sensitive Data Exposure)

```java
public static String getStaticKey() {
    return SECRET_KEY;
}

public static byte[] getKeyBytes() {
    return secretKey.getEncoded();
}

public static String getKeyHex() {
    ...
}
```

`src/main/java/com/cinebuscador/config/EncryptionService.java`

The service ships public accessor methods that return the raw key material (as
string, bytes, and hex) purely to facilitate decryption by external tools such
as `openssl`. Exposing the key bytes keeps them reachable from any code path
and widens the attack surface for key exfiltration.

#### 5. Unauthenticated H2 Database Console Exposure (Remote Code Execution / Exposed Debug Interface)

```java
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

`pom.xml`

```java
"--spring.datasource.url=jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE",
```

`Dockerfile`

The application depends on H2 and runs with no Spring Security dependency and
no `spring.h2.console.enabled=false` setting, so the H2 web console at
`/h2-console` is auto-enabled (Spring Boot's console auto-configuration
activates by default when H2 is on the classpath). With no authentication layer
in front of it, an anonymous attacker can reach the console and supply an
arbitrary JDBC URL, which H2 executes — enabling remote code execution (H2
console / `CREATE ALIAS`-style RCE).

#### 6. No CSRF Protection on State-Changing Endpoints (Cross-Site Request Forgery)

```html
<form th:action="@{/register}" method="post">
```

`src/main/resources/templates/index.html`

`/login` and `/register` are state-changing POST endpoints handled by plain
Spring MVC controllers. The project does not include Spring Security or any
CSRF-token mechanism, so any request to these endpoints is accepted without a
session-bound token. An attacker-controlled page can therefore trigger
registrations or login attempts (including blind login probing with
victim-supplied credentials) on behalf of an unconsenting visitor.

#### 7. User Enumeration via Registration (Information Disclosure)

```java
if (userRepository.findByUsername(username).isPresent()) {
    model.addAttribute("registerError", "El usuario ya existe");
```

`src/main/java/com/cinebuscador/controller/AuthController.java`

The registration endpoint explicitly reports when a username already
exists. This lets an attacker distinguish registered from unregistered accounts
by submitting candidate usernames, enabling account enumeration for use in
targeted credential-stuffing or phishing campaigns.

#### 8. Password Cipher Text Rendered in the UI (Sensitive Data Exposure)

```java
model.addAttribute("encryptedPassword", user.getPassword());
...
model.addAttribute("encryptedPassword", EncryptionService.encrypt(password));
```

`src/main/java/com/cinebuscador/controller/AuthController.java`

```html
<div class="code" th:text="${encryptedPassword}"></div>
```

`src/main/resources/templates/index.html`

After a successful login or registration, the password's ciphertext (and a
freshly generated one, redundantly re-encrypting the password a second time) is
written back into the rendered HTML. Since the app uses reversible encryption
with a source-embedded key, this ciphertext is equivalent to the plaintext
password; it is also re-encrypted and displayed on every page load, expanding
where credential material is persisted (browser history, proxies, logs) and
yielding additional ciphertext copies of the same secret.

#### 9. Production Debug Logging Enabled (Information Disclosure)

```java
"--spring.jpa.hibernate.ddl-auto=update", \
"--app.upload-dir=/uploads", "--debug"]
```

`Dockerfile`

The container entrypoint launches the production JVM with the Spring Boot
`--debug` flag, forcing core debug logging in production. This surfaces verbose
internal details (auto-configuration reports, condition evaluations, property
and framework internals) in the container logs, and combined with
`spring.jpa.show-sql=true` in `application.properties`, every SQL statement
executed against the user database is printed to stdout — increasing the volume
of sensitive operational data exposed through log aggregation or log access.

### Rama `practico-2`
TODO:

