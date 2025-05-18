## Introducción 

Encontrarse con códigos 403 Forbidden es común dentro de aplicaciones web, normalmente detrás de ellos podemos encontrar paneles administrativos, APIs internas o endpoints sensibles. Aunque puedan parecer un callejón sin salida, debido a malas configuraciones en servidores, proxies o sistemas de acceso de control a veces pueden surgir grietas en las defensas por las que colarse. En estos apuntes voy a describir como funciona este código, por qué sucede y diferentes técnicas que pueden superarlo para acceder a recursos restringidos durante las auditorías web. 

### ¿Qué es un código 403 Forbidden? 

Un 403 Forbidden es un código de estado HTTP que nos indica que nuestra petición ha sido procesada por el servidor correctamente pero que no tenemos acceso al recurso al que estamos intentando acceder. Podemos verlo como si nos encontrásemos dentro de una biblioteca buscando un libro en concreto y al encontrarlo el bibliotecario nos dijera que nos tenemos que ir por qué no tenemos el carné.

### Causas comunes de códigos 403 (de más a menos comunes)

1. **Fallos de autentificación/autorización**: Errores debido a credenciales erróneas o tokens inválidos que impiden la correcta autenticación o autorización.

2. **Bloqueo de direcciones IP o whitelists**: Acceso denegado para rangos específicos de direcciones IP o ubicaciones, generalmente por políticas de seguridad que pueden permitir o restringir el tráfico proveniente de ciertas fuentes. 

3. **Problemas de permisos en ficheros o directorios**: Configuración incorrecta de permisos de archivos o directorios que impide el acceso a los recursos solicitados. 

4. **Limitación de peticiones**: Restricciones basadas en la frecuencia de solicitudes, lo que provoca códigos 403 si se excede el límite permitido en un periodo de tiempo determinado. 

5. **Restricciones mediante User-Agent, Referer o método**: Filtrado o bloqueo de solicitudes basadas en el encabezado User-Agent, Referer o método HTTP que se esté usando (GET,POST...), comúnmente usado para el bloqueo de bots. 

6. **Configuraciones erróneas en los reverse proxies**: Bloqueo de acceso a recursos específicos debido a políticas de seguridad mal configuradas (como en NGINX, Apache...).

7. **Configuraciones de permisos incorrectas (ACLs, IAM..)**:  Configuraciones incorrectas de Listas de Control de Acceso (ACL) o gestión de identidad y acceso (IAM), que impiden que usuarios autorizados accedan a los recursos.

8. **Firewall o softwares de seguridad**: Bloqueo de solicitudes por parte de un firewall o un software de seguridad (como un WAF), que detecta patrones sospechosos o intentos de ataque.

9. **Restricciones de acceso geográfico**: Restricciones de acceso basadas en la ubicación geográfica desde la que se intenta acceder, bloqueando las peticiones provenientes de ciertos países o regiones bloqueadas. 

***
## 403 Forbidden Bypass Cheat Sheet

A continuación categorizo diferentes técnicas reales y efectivas recopiladas a lo largo de los dos últimos años: 

## Manipulación de métodos HTTP

Hay muchos servidores en los que sólo se aplican controles de acceso a los métodos HTTP más comunes como pueden ser GET o POST. Al cambiar a otros métodos menos usados como PUT, DELETE, PATCH o TRACE puede ser que podamos saltarnos la configuración que no tenía los tenía en cuenta. 

Ejemplos: 

```bash
curl -X OPTIONS --path-as-is https://ejemplo.es/admin/
curl -X GET --path-as-is https://ejemplo.es/admin/
curl -X POST --path-as-is https://ejemplo.es/admin/
curl -X PUT --path-as-is https://ejemplo.es/admin/
curl -X DELETE --path-as-is https://ejemplo.es/admin/
curl -X PATCH --path-as-is https://ejemplo.es/admin/
curl -X HEAD --path-as-is https://ejemplo.es/admin/
curl -X TRACE --path-as-is https://ejemplo.es/admin/
curl -X CONNECT --path-as-is https://ejemplo.es/admin/
curl -X PROPFIND --path-as-is https://ejemplo.es/admin/
curl -X MKCOL --path-as-is https://ejemplo.es/admin/
curl -X COPY --path-as-is https://ejemplo.es/admin/
curl -X MOVE --path-as-is https://ejemplo.es/admin/
curl -X LOCK --path-as-is https://ejemplo.es/admin/
curl -X UNLOCK --path-as-is https://ejemplo.es/admin/
curl -X SEARCH --path-as-is https://ejemplo.es/admin/
```

- **-X**: Para cambiar el método HTTP usado. 
- **--path-as-is**: Previene la sanitización en la URL introducida (esencial para URL encoding).

#### ¿Qué es --path-as-is? 

Cuando mandas una petición usando curl, normalmente la propia herramienta va a "limpiarte" la URL sin que tu lo hagas, pero habrá veces en las que queramos mandar URLs aparentemente sin sentido o rotas para ver si podemos saltarnos las medidas de seguridad.

Al agregar el parámetro --path-as-is lo que le estamos diciendo al curl es: Manda la URL exactamente como la he escrito yo, no cambies nada. 

#### **PRO TIP**:

Usar siempre el método OPTIONS para descubrir los métodos HTTP permitidos dentro del servidor, luego podemos aprovechar alguna herramienta como el intruder de burpsuite para automatizar un bruteforce y testear los métodos no admitidos para potencialmente encontrar vulnerabilidades.

--- 
## Manipulación de cabeceras

Para saltarnos los 403 Forbidden u otros controles de acceso mal configurados lo que podemos se puede hacer es manipular las cabeceras HTTP para engañar al servidor y que esté nos deje pasar. A continuación dejo las cabeceras más usadas, con sus valores más comunes y una pequeña descripción de lo que intentan conseguir: 

```markdown
# Cabeceras usadas comúnmente para bypassear 403

| Cabecera                   | Valor asignado             | Descripción                                        |
|---------------------------|----------------------------|---------------------------------------------------------|
| X-Original-URL            | /admin                     | Access restricted paths via rewritten URLs              |
| X-Rewrite-URL             | /admin                     | Similar to X-Original-URL; processed by some proxies    |
| X-Custom-IP-Authorization | 127.0.0.1                  | Spoof internal IP (localhost)                           |
| X-Forwarded-For           | 127.0.0.1                  | Spoof client IP to appear as localhost                  |
| X-Client-IP               | 127.0.0.1                  | Another way to impersonate internal IP                  |
| X-Host                    | localhost                  | Manipulate host-based access controls                   |
| Referer                   | http://websegura.com/    | Trick server into trusting the source of the request    |
```

Usando las cabeceras X-Original-URL o X-Rewrite-URL podemos sobrescribir la ruta original solicitada, esto funciona especialmente bien en sistemas que usen NGINX como proxy inverso. Ejemplo:

```bash
curl -H “X-Original-URL: /admin” https://ejemplo.es/home
curl -H “X-Rewrite-URL: /admin” https://ejemplo.es/home
```

## Modificación del User-Agent

Algunos servidores bloquean las peticiones de herramientas como burp o curl inspeccionando la cabecera User-Agent, modificándola para que parezca de un navegador real podremos pasar a través de algunos filtros básicos.

Ejemplo:

````bash
curl -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64)" http://ejemplo.es/privado/
`````

Este truco hace que el servidor crea que la petición viene de un navegador normal y no de una herramienta automática. 

