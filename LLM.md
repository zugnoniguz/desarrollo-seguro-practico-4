# Revisión de código con LLM

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

#### Debug Mode Enabled by Default (Remote Code Execution / Information Disclosure)

```python
debug = os.environ.get('DEBUG_MODE', 'true').lower() in ('true', '1', 'yes')
app.run(debug=debug, host=host)
```

The application defaults to Flask debug mode when the `DEBUG_MODE` environment
variable is not set. In production, an unhandled exception triggers the
Werkzeug interactive debugger, which renders full stack traces, source code,
configuration values, and file system paths to any user who can reach the error
page. The debugger also exposes an interactive Python console that, if the
automatically generated PIN can be obtained (e.g., via values leaked in the
traceback), allows arbitrary remote code execution on the server.

#### Flask Development Server Used in Production

```python
if __name__ == '__main__':
    import os
    host = os.environ.get('FLASK_HOST', '0.0.0.0')
    ...
    app.run(debug=debug, host=host)
```

The application is served with Flask's built-in development WSGI server
(`app.run`), listening on `0.0.0.0`. This server is single-threaded by default,
is not designed for production traffic, and lacks the hardening, timeout
handling, and resilience of a production WSGI server. Combined with the
Dockerfile's `CMD`, this is the only mechanism serving traffic, so the entire
application runs on a development-grade HTTP server.

#### Container Runs as Root

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["sh", "-c", "python init_db.py && python app.py"]
```

The Dockerfile contains no `USER` directive, so the application process runs as
root inside the container. Any code execution vulnerability (such as the
debugger console) therefore grants root privileges within the container, and
the SQLite database file `cine.db` lives in the same writable directory as the
application code and is owned and accessed by that root process.

#### Destructive Database Re-initialization on Every Start (Data Loss)

```dockerfile
CMD ["sh", "-c", "python init_db.py && python app.py"]
```

```python
cursor.executescript('''
    DROP TABLE IF EXISTS funciones;
    DROP TABLE IF EXISTS peliculas;
    ...
''')
```

The container's start command unconditionally runs `init_db.py`, which executes
`DROP TABLE` on both tables and recreates them with seed data. Every container
restart, crash-recovery restart (`restart: unless-stopped` in
`docker-compose.yml`), or redeploy destroys all production data in the SQLite
database and replaces it with the fixture data, silently discarding any state
written since the previous start.

#### Outdated and Unpinned Dependencies

```
# requirements.txt
flask==3.0.0
```

```
# requirements-dev.txt
-r ./requirements.txt
pytest==9.1.1
```

The pinned Flask version 3.0.0 is outdated, and its transitive dependencies
(notably Werkzeug, which provides the debugger and request parsing) are left
completely unpinned, so the image resolves whatever version pip selects at
build time. Older Werkzeug releases in the 3.0.x line are affected by known
vulnerabilities, including the interactive debugger executing arbitrary code
when active and denial-of-service flaws in multipart request parsing; because
the dependency is unpinned, a vulnerable version can be silently installed. The
same unpinned-dependency problem applies to the pip tooling and Python 3.12
base image, which are installed without version constraints.

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

#### SQL Injection (search parameter)

```python
sql = (
    f"SELECT peliculas.nombre as pelicula, funciones.fecha_hora, "
    ...
    f"WHERE peliculas.nombre LIKE '%{query}%' "
    ...
)
return db.execute(sql).fetchall()
```

The `query` value, taken directly from the user-controlled `buscar` request
parameter in the `index` route, is interpolated into the SQL statement with an
f-string instead of being passed as a bound parameter. An attacker can break
out of the `LIKE '%...%'` literal and inject arbitrary SQL (e.g. via `UNION
SELECT`) into the query executed against the SQLite database, allowing
extraction or manipulation of data from the `peliculas` and `funciones`
tables.

#### SQL Injection (ORDER BY direction)

```python
sort_dir = request.args.get('sentido', 'ASC')
...
f"ORDER BY {'peliculas.nombre' if sort_by == 'nombre' else 'funciones.fecha_hora'} "
f"{sort_dir}"
```

The `sort_dir` value comes from the user-controlled `sentido` request
parameter and is concatenated verbatim into the `ORDER BY` clause of the SQL
statement. While `sort_by` is restricted to a fixed set of column names,
`sort_dir` accepts any value, so an attacker can inject arbitrary SQL
expressions after the column name, injecting malicious SQL into the query.

#### Cross-Site Request Forgery (CSRF)

```python
@app.route('/edit/<int:pelicula_id>', methods=['POST'])
def edit_post(pelicula_id):
    nombre = request.form.get('nombre', '')
    ...
    db.execute(
        '''UPDATE peliculas
           SET nombre=?, genero=?, director=?, descripcion=?
           WHERE id=?''',
        (nombre, genero, director, descripcion, pelicula_id),
    )
    db.commit()
```

The state-changing `POST /edit/<id>` endpoint performs a database update with
no CSRF token or any other origin-validation mechanism, and the form in
`templates/edit.html` submits to it without any anti-CSRF protection. An
attacker can craft a page that, when visited by a victim, silently submits a
forged POST request and modifies or overwrites movie records in the database.

#### Missing Authentication / Broken Access Control

```python
@app.route('/edit/<int:pelicula_id>', methods=['GET'])
def edit_get(pelicula_id):
    db = get_db()
    pelicula = db.execute(
        'SELECT * FROM peliculas WHERE id = ?', (pelicula_id,)
    ).fetchone()
    return render_template('edit.html', pelicula=pelicula)
```

Neither the `GET` nor the `POST` handler for `/edit/<id>` performs any
authentication or authorization check; every "Editar" link on the public
index page exposes the editing functionality. Any anonymous user can access
the edit form and modify the name, genre, director, and description of any
movie in the production database.

#### Debug Mode Enabled in Production

```python
debug = os.environ.get('DEBUG_MODE', 'true').lower() in ('true', '1', 'yes')
app.run(debug=debug, host=host)
```

Debug mode is enabled by default: unless the `DEBUG_MODE` environment
variable is explicitly set to a falsy value, the application runs with
`debug=True`, and the Docker deployment (`docker-compose.yml`) sets no such
variable. The Werkzeug interactive debugger is therefore exposed on the
publicly bound interface (`0.0.0.0:5000`), allowing any attacker who triggers
an unhandled exception to execute arbitrary code on the server through the
debugger console.

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

#### SQL Injection (search parameter)

```python
sql = (
    f"SELECT peliculas.nombre as pelicula, funciones.fecha_hora, "
    ...
    f"WHERE peliculas.nombre LIKE '%{query}%' "
    ...
)
return db.execute(sql).fetchall()
```

The `query` value, taken directly from the user-controlled `buscar` request
parameter in the `index` route, is interpolated into the SQL statement with an
f-string instead of being passed as a bound parameter. An attacker can break
out of the `LIKE '%...%'` literal and inject arbitrary SQL (e.g. via `UNION
SELECT`) into the query executed against the SQLite database, allowing
extraction or manipulation of data from the `peliculas` and `funciones`
tables.

#### SQL Injection (ORDER BY direction)

```python
sort_dir = request.args.get('sentido', 'ASC')
...
f"ORDER BY {'peliculas.nombre' if sort_by == 'nombre' else 'funciones.fecha_hora'} "
f"{sort_dir}"
```

The `sort_dir` value comes from the user-controlled `sentido` request
parameter and is concatenated verbatim into the `ORDER BY` clause of the SQL
statement. While `sort_by` is restricted to a fixed set of column names,
`sort_dir` accepts any value, so an attacker can inject arbitrary SQL
expressions after the column name, injecting malicious SQL into the query.

#### Cross-Site Request Forgery (CSRF)

```python
@app.route('/edit/<int:pelicula_id>', methods=['POST'])
def edit_post(pelicula_id):
    nombre = request.form.get('nombre', '')
    ...
    db.execute(
        '''UPDATE peliculas
           SET nombre=?, genero=?, director=?, descripcion=?
           WHERE id=?''',
        (nombre, genero, director, descripcion, pelicula_id),
    )
    db.commit()
```

The state-changing `POST /edit/<id>` endpoint performs a database update with
no CSRF token or any other origin-validation mechanism, and the form in
`templates/edit.html` submits to it without any anti-CSRF protection. An
attacker can craft a page that, when visited by a victim, silently submits a
forged POST request and modifies or overwrites movie records in the database.

#### Missing Authentication / Broken Access Control

```python
@app.route('/edit/<int:pelicula_id>', methods=['GET'])
def edit_get(pelicula_id):
    db = get_db()
    pelicula = db.execute(
        'SELECT * FROM peliculas WHERE id = ?', (pelicula_id,)
    ).fetchone()
    return render_template('edit.html', pelicula=pelicula)
```

Neither the `GET` nor the `POST` handler for `/edit/<id>` performs any
authentication or authorization check; every "Editar" link on the public
index page exposes the editing functionality. Any anonymous user can access
the edit form and modify the name, genre, director, and description of any
movie in the production database.

#### Debug Mode Enabled in Production

```python
debug = os.environ.get('DEBUG_MODE', 'true').lower() in ('true', '1', 'yes')
app.run(debug=debug, host=host)
```

Debug mode is enabled by default: unless the `DEBUG_MODE` environment
variable is explicitly set to a falsy value, the application runs with
`debug=True`, and the Docker deployment (`docker-compose.yml`) sets no such
variable. The Werkzeug interactive debugger is therefore exposed on the
publicly bound interface (`0.0.0.0:5000`), allowing any attacker who triggers
an unhandled exception to execute arbitrary code on the server through the
debugger console.

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

#### 1. SpEL Injection (SSTI) leading to Remote Code Execution

```java
ExpressionParser parser = new SpelExpressionParser();

org.springframework.expression.EvaluationContext context =
    SimpleEvaluationContext.forReadOnlyDataBinding().build();

StandardEvaluationContext standardContext = new StandardEvaluationContext();

standardContext.setVariable("system", System.class);
standardContext.setVariable("runtime", Runtime.class);

var expr = parser.parseExpression(expression);
Object result = expr.getValue(standardContext);
```

`SpelEvaluator.evaluate()` parses an arbitrary attacker-controlled string as a
SpEL expression and evaluates it against a `StandardEvaluationContext`, which
grants full access to the Java type system (type references such as
`T(java.lang.Runtime)`, constructors, method invocation). Additionally, the
`System` and `Runtime` classes are explicitly registered as variables
(`#system`, `#runtime`), making command execution and server property
disclosure directly reachable. A `SimpleEvaluationContext` (which restricts
SpEL to simple data binding) is constructed but never used; the code path
instead evaluates with the unrestricted context. Any expression string that
reaches this method results in arbitrary code execution on the server,
e.g. `T(java.lang.Runtime).getRuntime().exec(...)`. The component is wired as a
Spring bean and injected into `FuncionController` (`private final SpelEvaluator
spelEval;`), leaving a live RCE sink present in the application's
request-serving codebase.

#### 2. Information Exposure via Debug Mode and Verbose SQL Logging

```dockerfile
ENTRYPOINT ["java", \
    "-jar", "app.jar", \
    "--spring.sql.init.mode=always", \
    "--spring.datasource.url=jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE", \
    "--spring.jpa.hibernate.ddl-auto=update", \
    "--app.upload-dir=/uploads", "--debug"]
```

```properties
spring.jpa.show-sql=true
```

The container entrypoint runs the application with `--debug`, enabling
DEBUG-level logging for core Spring components in production, and
`spring.jpa.show-sql=true` echoes every SQL statement to the logs. Both
channels expose internal application structure, database schema, queries, and
runtime details that aid an attacker in fingerprinting and further
exploitation, and the verbose logging is broadcast in a production deployment
that serves user traffic.

#### 3. Uncontrolled Resource Consumption (DoS)

```java
List<Funcion> resultados = funcionRepo.findAll().stream()
    .filter(f -> f.getNombreFuncion() != null &&
                 f.getNombreFuncion().toLowerCase().contains(buscar.toLowerCase()))
    .collect(Collectors.toList());
```

The unauthenticated `/` endpoint loads the entire `funciones` table into memory
on every request and performs a linear-scan substring filter over each entity,
instead of delegating the search to the database. The table size is unbounded
and grows over time, so repeated or concurrent requests let any unauthenticated
user force full-table reads and per-row string matching, consuming memory and
CPU without limit and degrading or denying service to other users.

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

#### 1. Hardcoded Encryption Key (CWE-798)

```java
// src/main/java/com/cinebuscador/config/EncryptionService.java
private static final String SECRET_KEY = "MySup3rS3cr3tK3y!2024CineBuscadorAES";
```

The AES-256 key is embedded as a string literal in the source code. Anyone with
access to the repository, the compiled JAR, or the Docker image can extract the
key and decrypt every piece of data encrypted with this service. Because the
key is shared by the entire application, a single leak compromises all
ciphertext at once and the key cannot be rotated without redeploying.

#### 2. Insecure Block Cipher Mode: AES/ECB (CWE-327)

```java
// src/main/java/com/cinebuscador/config/EncryptionService.java
private static final String CIPHER_ALGO = "AES/ECB/PKCS5Padding";
```

Encryption uses ECB mode, which is deterministic and requires no IV: identical
plaintext blocks always produce identical ciphertext blocks. For data such as
passwords or usernames this leaks structural patterns — an attacker with access
to the database can identify equal values across rows and build a codebook of
common values without knowing the key. ECB is explicitly discouraged for any
data beyond a single block.

#### 3. Key Material Exposed Through Helper Methods (CWE-200)

```java
// src/main/java/com/cinebuscador/config/EncryptionService.java
public static String getStaticKey() {
    return SECRET_KEY;
}

public static byte[] getKeyBytes() {
    return secretKey.getEncoded();
}

public static String getKeyHex() { ... }
```

The service exposes public static methods that return the raw secret key as a
string, as raw bytes, and as a hex string, along with a source-code comment
documenting how to decrypt with `openssl`. Any code running in the same JVM (or
any caller of these methods) can trivially obtain full key material, turning a
partial compromise into a complete one.

#### 4. Missing CSRF Protection (CWE-352)

```java
// src/main/java/com/cinebuscador/controller/AuthController.java
@PostMapping("/login")
public String login(@RequestParam String username, @RequestParam String password, Model model) {

@PostMapping("/register")
public String register(@RequestParam String username, @RequestParam String password,
                       @RequestParam String confirmPwd, Model model) {
```

The application does not include `spring-boot-starter-security` anywhere in
`pom.xml`, so no CSRF token mechanism is active. The state-changing POST
endpoints `/login` and `/register` accept plain request parameters with no
anti-CSRF token, and the Thymeleaf forms in `index.html` emit no CSRF field. An
attacker can host a page that silently submits a registration or login form on
behalf of a victim's browser session.

#### 5. User Account Enumeration (CWE-204)

```java
// src/main/java/com/cinebuscador/controller/AuthController.java
if (userRepository.findByUsername(username).isPresent()) {
    model.addAttribute("registerError", "El usuario ya existe");
```

The registration endpoint explicitly discloses whether a username already
exists in the system. An attacker can iterate over the registration form to
enumerate valid usernames, which facilitates credential-stuffing and targeted
password-guessing attacks against known accounts.

#### 6. Sensitive Data Exposure in HTML Response (CWE-200)

```java
// src/main/java/com/cinebuscador/controller/AuthController.java
model.addAttribute("encryptedPassword", user.getPassword());
```

```html
<!-- src/main/resources/templates/index.html -->
<div class="code" th:text="${encryptedPassword}"></div>
```

After a successful login or registration, the server sends the stored
credential material (the bcrypt hash of the user's password) back to the
browser and renders it in the page. Password hashes should never leave the
server; exposing them in the HTTP response leaks them to proxies, browser
caches, logs, and anyone with access to the rendered page or dev tools,
providing offline cracking material.

#### 7. No Brute-Force Protection on Authentication (CWE-307)

```java
// src/main/java/com/cinebuscador/controller/AuthController.java
@PostMapping("/login")
public String login(@RequestParam String username, @RequestParam String password, Model model) {
    User user = userRepository.findByUsername(username).orElse(null);
    if (user != null) {
        if (HashingService.matches(password, user.getPassword())) {
```

The login endpoint performs password verification with no rate limiting,
account lockout, delay, or captcha of any kind. An attacker can submit
unlimited password guesses per request in parallel, making online brute-force
and credential-stuffing attacks trivially feasible.

#### 8. No Password Policy Enforcement (CWE-521)

```java
// src/main/java/com/cinebuscador/controller/AuthController.java
if (!password.equals(confirmPwd)) {
    model.addAttribute("registerError", "Las contraseñas no coinciden");
    ...
}
User nuevoUsuario = new User();
nuevoUsuario.setPassword(HashingService.hash(password));
```

Registration only checks that the password matches its confirmation. There is
no minimum length, complexity, or deny-list check, so empty or single-character
passwords are accepted and stored, directly weakening authentication for all
accounts.

#### 9. Container Runs as Root (CWE-250)

```dockerfile
# Dockerfile
FROM eclipse-temurin:17-jre-noble
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar", ...]
```

The production image has no `USER` directive, so the application runs as root
inside the container. Any code-execution or deserialization vulnerability in
the app grants root privileges within the container, increasing the impact of a
compromise and easing container-escape attempts.

#### 10. Debug Logging and SQL Tracing Enabled in Production (CWE-532)

```dockerfile
# Dockerfile
ENTRYPOINT ["java", "-jar", "app.jar", ..., "--debug"]
```

```properties
# src/main/resources/application.properties
spring.jpa.show-sql=true
```

The application is launched with `--debug` (enabling verbose debug logging of
core Spring components) and Hibernate SQL tracing is enabled. This writes
detailed internal runtime and query information to production logs, which can
disclose internals, parameters, and data to anyone with log access.

#### 11. Database Without Authentication (CWE-306)

```dockerfile
# Dockerfile
"--spring.datasource.url=jdbc:h2:file:/data/cine3;AUTO_SERVER=TRUE"
```

The H2 database is opened with `AUTO_SERVER=TRUE` and no explicit credentials
are configured anywhere, so the default H2 user `sa` with an empty password
applies. AUTO_SERVER additionally starts a TCP listener that lets other
processes connect to the same database file, meaning any process reaching the
host or container can read and write the full user table (including password
hashes) without authentication.
