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

Luego, una vez aplicadas todas las reglas, el programa reune las reglas que
fallaron y los fragmentos de código relevantes y recopila un reporte para luego
ser procesado. El mismo puede ser visto de manera visual, en la terminal, o
exportado en algún formato estructurado como JSON para ser procesados por otra
herramienta.

