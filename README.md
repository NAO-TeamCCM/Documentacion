# NAO + qi

Proyecto de práctica para manejar al robot NAO desde Python usando el framework qi.

## Introducción

Para manejar el robot NAO se puede hacer desde Python o desde Choregraphe. Este proyecto se centra en cómo controlarlo con Python.

Existen dos frameworks principales con los que se puede controlar al NAO: NAOqi y qi. El framework NAOqi solo funciona con versiones de Python menores o iguales a Python 2.7, por lo que está bastante desactualizado. Por eso aquí usamos qi, que sí está actualizado para versiones de Python mayores a la 3.7. La versión de Python usada en este proyecto es la 3.10.

## Requisitos mínimos

* Python >= 3.7
* Sistema operativo Linux con distro Ubuntu (se puede usar el WSL de Windows con Ubuntu)
* CMake >= 3.23
* Un compilador para C++17

**NOTA:** el paquete oficial `qi` de PyPI no tiene distribución para Windows. Si trabajas en Windows, activa el WSL con Ubuntu antes de instalar.

## Instalación

Activar WSL (solo Windows, PowerShell como administrador):

```powershell
wsl --install -d Ubuntu
```

Crear y activar un entorno virtual:

```bash
python -m venv <nombre_del_entorno>
source <nombre_del_entorno>/bin/activate
```

Instalar qi:

```bash
pip install qi
```

## Verificación rápida

```python
import qi
from sys import argv

NAO_IP = "TU_IP"  # 127.0.0.1 si usas el robot virtual en Linux,
                  # o la IP de Windows si corres el script desde WSL
NAO_PORT = 9559

def main():
    app = qi.Application([], url=f"tcp://{NAO_IP}:{NAO_PORT}")
    app.start()

    tts = app.session.service("ALTextToSpeech")
    tts.say("Hola mundo")

if __name__ == "__main__":
    main()
```

Si el robot dice "Hola mundo", todo quedó instalado correctamente.

## Cómo usar la librería

Todo script con qi sigue el mismo patrón: conectarse al robot, pedir el servicio que se necesita, y llamar a sus métodos.

Hay dos formas de conectarse:

```python
# qi.Session: la más simple, recomendada para scripts sueltos
session = qi.Session()
session.connect("tcp://<IP_DEL_ROBOT>:9559")

tts = session.service("ALTextToSpeech")
tts.say("Hola mundo")
```

```python
# qi.Application: útil cuando necesitas manejar argumentos de terminal,
# registrar tus propios servicios, o correr un loop de eventos
app = qi.Application([], url="tcp://<IP_DEL_ROBOT>:9559")
app.start()

tts = app.session.service("ALTextToSpeech")
tts.say("Hola mundo")
```

Servicios más usados: `ALTextToSpeech` (habla), `ALMotion` (movimiento), `ALMemory` (sensores y eventos).

Las llamadas son síncronas por defecto; para que dos acciones ocurran al mismo tiempo se usa `_async=True`, que devuelve un `qi.Future` en vez de esperar a que termine.

## Problemas comunes

* **`Connection refused` desde WSL:** dentro de WSL, `127.0.0.1` no apunta a Windows. Obtén la IP real con `cat /etc/resolv.conf` y úsala en `NAO_IP`. Verifica también en Windows con `netstat -an | findstr 9559` que el simulador escuche en `0.0.0.0` y no solo en localhost, y revisa el Firewall de Windows si el puerto sigue bloqueado.
* **Falla `pip install qi`:** probablemente tu versión de Python es demasiado nueva y no hay wheel disponible. Usa Python 3.10.
* **Necesitas correr qi en Windows nativo:** el paquete oficial no lo soporta; existe una alternativa no oficial de la comunidad, `libqi-windows`, pero no está mantenida por Aldebaran.

## Documentación adicional

Para la referencia técnica completa de la librería (conceptos, parámetros de `qi.Application`, eventos y señales, manejo de errores a detalle), ver `docs/manual_qi.md`.
