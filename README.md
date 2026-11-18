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
