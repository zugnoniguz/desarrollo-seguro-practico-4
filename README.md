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

