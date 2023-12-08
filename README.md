# Paso 1: Instalación de Django y Channels

## 1.1 Creación de entorno virtual:

```bash
python -m venv myenv
```

## 1.2 Activación del entorno virtual:

`En Windows: myenv\Scripts\activate`

`En Linux/Mac: source myenv/bin/activate`
## 1.3 Instalación de Django y Channels:


` pip install django channels daphne`

Explicación:

Django es un marco web de alto nivel para el desarrollo rápido de aplicaciones web.
Channels extiende Django para manejar conexiones WebSocket y otras funcionalidades en tiempo real.
# Paso 2: Creación del proyecto Django

## 2.1 Creación del directorio y proyecto:

```bash
mkdir mi_proyecto_chat
cd mi_proyecto_chat
django-admin startproject ChatApp
cd ChatApp
python manage.py startapp chat
```
Explicación:

startproject crea un nuevo proyecto Django llamado ChatApp.
# Paso 3: Estructura de carpetas y archivos

## 3.1 Organización de archivos:

Asegúrate de que la estructura de carpetas y archivos esté organizada correctamente.

Revisa el archivo `tree.txt`
# Paso 4: Configuración del archivo settings.py

## 4.1 Clave secreta y configuración de aplicaciones:


```python
# ChatApp/settings.py

INSTALLED_APPS = [
    'daphne',
    # ... otras aplicaciones ...
    'channels',
    'chat.apps.ChatConfig',  # Asegúrate de incluir la aplicación 'chat'
]
```
## 4.2 Configuración de ASGI_APPLICATION:

```python
# ChatApp/settings.py

ASGI_APPLICATION = 'ChatApp.asgi.application'

```
Explicación:

ASGI_APPLICATION apunta al archivo asgi.py para la configuración de Channels.

# Paso 5: Creación de consumidores en chat/consumers.py

## 5.1 Consumidor de WebSocket:

```python
# chat/consumers.py

import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.roomGroupName = "group_chat_gfg"
        await self.channel_layer.group_add(
            self.roomGroupName,
            self.channel_name
        )
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.roomGroupName,
            self.channel_layer
        )

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]
        username = text_data_json["username"]
        await self.channel_layer.group_send(
            self.roomGroupName, {
                "type": "sendMessage",
                "message": message,
                "username": username,
            }
        )

    async def sendMessage(self, event):
        message = event["message"]
        username = event["username"]
        await self.send(text_data=json.dumps({"message": message, "username": username}))
```
Explicación:

ChatConsumer es un consumidor de WebSocket que maneja la conexión, desconexión y recepción/envío de mensajes en tiempo real.


# Paso 6: Configuración de las rutas WebSocket en chat/routing.py

## 6.1 Definición de rutas de WebSocket:

```python

# chat/routing.py

from django.urls import path
from chat.consumers import ChatConsumer

websocket_urlpatterns = [
    path("", ChatConsumer.as_asgi()),
]

```
Explicación:

websocket_urlpatterns define la ruta de WebSocket que apunta al consumidor ChatConsumer.
# Paso 7: Configuración del archivo asgi.py

## 7.1 Configuración de ASGI_APPLICATION:

```python
# ChatApp/asgi.py

import os
from django.core.asgi import get_asgi_application
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
from chat import routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'ChatApp.settings')

application = ProtocolTypeRouter(
    {
        "http": get_asgi_application(),
        "websocket": AuthMiddlewareStack(
            URLRouter(
                routing.websocket_urlpatterns
            )
        ),
    }
)
```
Explicación:

asgi.py configura el manejo de conexiones WebSocket a través de Channels.
ProtocolTypeRouter permite que diferentes protocolos (HTTP, WebSocket) se manejen de manera diferente.
AuthMiddlewareStack agrega soporte para autenticación de Django a las conexiones WebSocket.
# Paso 8: Configuración del archivo urls.py

## 8.1 Configuración de rutas:

```python
# ChatApp/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path("", include("chat.urls")),
]

```
Explicación:

Configura las rutas para incluir las definidas en chat.urls y las proporcionadas por el administrador de Django.

# Paso 9: Creación de plantillas HTML

## 9.1 Estructura de chatPage.html:

```html
<!-- templates/chat/chatPage.html -->

<!DOCTYPE html>
<html>
<body>
    <center><h1>Hola, Bievenido a mi chat! {{ request.user }}</h1></center>
    <br>
    {% if request.user.is_authenticated %}
    <center> Sal del sitio <a href="{% url 'logout-user' %}">Cerrar sesion</a></center>
    {% endif %}
    <div class="chat__item__container" id="id_chat_item_container" style="font-size: 20px">
        <br />
        <input type="text" id="id_message_send_input" />
        <button type="submit" id="id_message_send_button">Enviar</button>
        <br />
        <br />
    </div>
        <script>
        // Script para la funcionalidad de chat en tiempo real
        const chatSocket = new WebSocket("ws://" + window.location.host + "/");
        
        chatSocket.onopen = function (e) {
            console.log("La conexión se estableció exitosamente.");
        };

        chatSocket.onclose = function (e) {
            console.log("¡Algo inesperado sucedió!");
        };

        document.querySelector("#id_message_send_input").focus();

        document.querySelector("#id_message_send_input").onkeyup = function (e) {
            if (e.keyCode == 13) {
                document.querySelector("#id_message_send_button").click();
            }
        };

        document.querySelector("#id_message_send_button").onclick = function (e) {
            var messageInput = document.querySelector("#id_message_send_input").value;
            chatSocket.send(JSON.stringify({ message: messageInput, username: "{{ request.user.username }}" }));
        };

        chatSocket.onmessage = function (e) {
            const data = JSON.parse(e.data);
            var div = document.createElement("div");
            div.innerHTML = data.username + " : " + data.message;
            document.querySelector("#id_message_send_input").value = "";
            document.querySelector("#id_chat_item_container").appendChild(div);
        };
    </script>
</body>
</html>
```


## 9.2. Explicación del script:

Se crea un nuevo WebSocket (chatSocket) que se conecta a la misma dirección que el servidor de desarrollo.
onopen y onclose son manejadores de eventos que se activan cuando la conexión WebSocket se establece y se cierra, respectivamente.
El script maneja el envío de mensajes cuando se presiona Enter o se hace clic en el botón de enviar.
onmessage maneja los mensajes recibidos y los muestra en la interfaz de chat.
Con esto, la configuración y funcionalidad del chat en tiempo real están completas
## 9.3 Estructura de LoginPage.html:

```html
<!-- templates/chat/LoginPage.html -->

<!DOCTYPE html>
<html>
<body>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <br>
        <button type="submit">Login</button>
    </form>
</body>
</html>
```
Explicación:

chatPage.html contiene la interfaz del chat y el script para la funcionalidad de WebSocket.
LoginPage.html tiene un formulario simple de inicio de sesión.
# Paso 10: Actualización de configuración en settings.py

## 10.1 Configuración de plantillas y algunos ajustes extra en settings:

```python
# ChatApp/settings.py

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```
Explicación:

Asegúrate de que la configuración de las plantillas de Django (TEMPLATES) esté correctamente configurada.



Tambien agregamos este bloque de codigo al final de `settings.py`

```python
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels.layers.InMemoryChannelLayer"
    }
}

LOGIN_REDIRECT_URL = "chat-page"
LOGOUT_REDIRECT_URL = "login-user"
```
# Paso 11: Aplicación de migraciones y ejecución del servidor de desarrollo

## 11.1 Aplicación de migraciones:

```bash
python manage.py makemigrations
python manage.py migrate
```
## 11.2 Inicio del servidor de desarrollo:

```bash
python manage.py runserver
```
Explicación:

migrate aplica las migraciones para crear las tablas de la base de datos.
runserver inicia el servidor de desarrollo.
# Paso 12: Verificación

## 12.1 Acceso y verificación:

Abre tu navegador y ve a http://127.0.0.1:8000/.
Inicia sesión con un usuario registrado o crea uno si aún no lo has hecho.
Accede a la página de chat haciendo clic en el enlace proporcionado.
Deberías ver la interfaz del chat y poder enviar mensajes en tiempo real.

