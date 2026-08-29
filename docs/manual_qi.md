# Manual técnico: uso de la librería qi para NAO

## Tabla de contenido

1. Introducción
2. Requisitos
3. Instalación
4. Conceptos básicos
5. Conexión al robot
6. Servicios del robot
7. Llamadas síncronas y asíncronas
8. Eventos y señales
9. Parámetros de qi.Application
10. Manejo de errores y problemas comunes
11. Preguntas frecuentes

---

## 1. Introducción

Para manejar el robot NAO se puede hacer desde Python o desde Choregraphe. Este manual se centra en cómo controlarlo con Python usando el framework qi.

Existen dos frameworks principales con los que se puede controlar al NAO: NAOqi y qi. El framework NAOqi solo funciona con versiones de Python menores o iguales a Python 2.7, por lo que está bastante desactualizado y puede resultar más difícil de usar si se requieren funciones o librerías modernas. Por esto, este manual documenta el uso de qi, que sí está actualizado para versiones de Python mayores a la 3.7. La versión de Python usada como estándar es la 3.10, salvo que se requiera una versión distinta por alguna razón puntual.

## 2. Requisitos

* Python >= 3.7
* Sistema operativo Linux con distro Ubuntu (se puede usar el WSL de Windows con Ubuntu)
* CMake >= 3.23
* Un compilador para C++17

El paquete oficial `qi` de PyPI no cuenta con distribución para Windows. Si el entorno de trabajo es Windows, es necesario activar el WSL con una distro de Ubuntu antes de instalar.

## 3. Instalación

### 3.1 Activar WSL (solo Windows)

En PowerShell, como administrador:

```powershell
wsl --install -d Ubuntu
```

Reiniciar la computadora al terminar. Al volver a abrir Windows se abrirá una terminal de Ubuntu pidiendo crear un usuario y contraseña para el entorno Linux.

Para confirmar que la instalación quedó correcta:

```powershell
wsl --list --verbose
```

Debe aparecer la distro de Ubuntu con la versión 2.

### 3.2 Crear entorno virtual

```bash
python -m venv <nombre_del_entorno>
source <nombre_del_entorno>/bin/activate
```

### 3.3 Instalar qi

```bash
pip install qi
```

### 3.4 Verificación

```python
import qi
from sys import argv

NAO_IP = "TU_IP"
NAO_PORT = 9559

def main():
    app = qi.Application([], url=f"tcp://{NAO_IP}:{NAO_PORT}")
    app.start()

    tts = app.session.service("ALTextToSpeech")
    tts.say("Hola mundo")

if __name__ == "__main__":
    main()
```

Si el robot dice "Hola mundo", la instalación quedó correcta.

## 4. Conceptos básicos

qi funciona sobre un modelo cliente-servidor: el robot (o el simulador) corre un proceso NAOqi que expone servicios a través de un bus de mensajería. Un script en Python se conecta a ese bus, pide acceso a un servicio específico, y llama a sus métodos como si fueran funciones normales de Python.

El patrón general es siempre el mismo:

1. Abrir una conexión (sesión) al robot.
2. Pedir el servicio que se necesita con `session.service("NombreDelServicio")`.
3. Llamar a los métodos de ese servicio.

## 5. Conexión al robot

Existen dos formas de abrir la conexión.

### 5.1 qi.Session

Es la forma más simple y la recomendada para scripts sueltos:

```python
import qi

session = qi.Session()
session.connect("tcp://<IP_DEL_ROBOT>:9559")

tts = session.service("ALTextToSpeech")
tts.say("Hola mundo")
```

### 5.2 qi.Application

Útil cuando se necesita manejar argumentos de terminal, registrar servicios propios, o correr un loop de eventos:

```python
import qi

app = qi.Application([], url="tcp://<IP_DEL_ROBOT>:9559")
app.start()

tts = app.session.service("ALTextToSpeech")
tts.say("Hola mundo")
```

Para la mayoría de los casos de uso, `qi.Session` es suficiente.

## 6. Servicios del robot

`session.service("NombreDelServicio")` es el punto de entrada para acceder a cualquier módulo del robot. Los servicios más usados son:

| Servicio | Para qué sirve |
|---|---|
| `ALTextToSpeech` | Hacer que el robot hable |
| `ALMotion` | Controlar el movimiento del robot |
| `ALMemory` | Leer sensores y suscribirse a eventos |

## 7. Llamadas síncronas y asíncronas

Por defecto, las llamadas a un servicio son bloqueantes: el script espera a que termine una acción antes de continuar con la siguiente línea.

```python
tts.say("Esto es una prueba")
motion.moveTo(1, 0, 0)
# El robot primero termina de hablar y hasta entonces empieza a moverse
```

Para que dos acciones ocurran al mismo tiempo se usa el parámetro `_async=True`, que devuelve un objeto `qi.Future` en vez de esperar:

```python
say_op = tts.say("Esto es una prueba", _async=True)
move_op = motion.moveTo(1, 0, 0, _async=True)
# Ambas acciones se disparan sin esperar a que la anterior termine
```

## 8. Eventos y señales

El servicio `ALMemory` permite suscribirse a eventos del robot (por ejemplo, un toque en la cabeza o la detección de una cara) mediante señales:

```python
memory = session.service("ALMemory")

def mi_callback(valor):
    print("Evento recibido:", valor)

suscripcion = memory.subscriber("NombreDelEvento")
suscripcion.signal.connect(mi_callback)
```

Es importante mantener una referencia al objeto `suscripcion`; si se pierde (por ejemplo, se sobreescribe la variable o sale de alcance), el callback deja de ejecutarse.

## 9. Parámetros de qi.Application

`qi.Application` recibe una lista de argumentos (normalmente `sys.argv`, aunque puede ser una lista vacía `[]`) y un parámetro `url` con la dirección del robot. Además reconoce automáticamente los siguientes flags si se pasan por terminal:

| Flag | Para qué sirve |
|---|---|
| `--qi-url=tcp://IP:9559` | A qué robot o sesión conectarse |
| `--qi-listen-url=tcp://0.0.0.0:PUERTO` | En qué dirección escuchar, si el script también expone un servicio propio |
| `--qi-standalone` | Levanta una sesión propia en vez de conectarse a un robot |
| `--qi-log-filters` | Filtra qué categorías de logs se muestran |

Para el uso cotidiano, el único parámetro relevante es `url` (o `--qi-url`) con la IP y el puerto del robot. El puerto por defecto es siempre `9559`.

## 10. Manejo de errores y problemas comunes

### 10.1 Connection refused al usar el robot virtual desde WSL

Dentro de WSL, `127.0.0.1` apunta al propio WSL y no a Windows. Si el robot virtual (Choregraphe) corre en Windows, el script nunca lo va a encontrar en `127.0.0.1`.

Pasos para resolverlo:

1. Obtener la IP de Windows desde Powershell:

   ```bash
   ipconfig
   ```

   La línea `Dirección IPv4` indica la IP a usar en `NAO_IP`.

2. Verificar en Windows que el robot virtual escuche en todas las interfaces y no solo en localhost:

   ```powershell
   netstat -an | findstr 9559
   ```

   `0.0.0.0:9559 LISTENING` indica que está correctamente expuesto. `127.0.0.1:9559` indica que solo escucha localhost.

3. Si sigue fallando, revisar el Firewall de Windows, que puede estar bloqueando el tráfico entrante desde la interfaz virtual de WSL:

   ```powershell
   New-NetFirewallRule -DisplayName "Allow WSL to 9559" -Direction Inbound -LocalPort 9559 -Protocol TCP -Action Allow
   ```

### 10.2 pip install qi falla

Los wheels precompilados de qi en PyPI no siguen automáticamente cada versión nueva de Python; suelen estar limitados a un rango específico. Usar una versión muy reciente de Python (por ejemplo, 3.13 o 3.14) suele resultar en que no se encuentre distribución compatible. Se recomienda usar Python 3.10.

### 10.3 Necesidad de correr qi en Windows nativo

El paquete oficial de PyPI no tiene build para Windows. Existe un paquete no oficial de la comunidad, `libqi-windows`, que sí trae binarios para Windows, pero no está mantenido por Aldebaran, por lo que se recomienda usarlo solo como alternativa cuando no es posible trabajar desde WSL o Linux.

## 11. Preguntas frecuentes

**¿Qué IP debo usar si trabajo con el robot virtual?**
`127.0.0.1` si el simulador corre en la misma máquina Linux donde ejecutas el script. Si el simulador corre en Windows y el script en WSL, se debe usar la IP obtenida de `/etc/resolv.conf` (ver sección 10.1).

**¿Cuál es el puerto del robot?**
El puerto por defecto de NAOqi es `9559`, tanto para robots físicos como virtuales.
