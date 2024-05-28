# 01 WEBSOCKETS NODE

- Websockets sirve para conectar el cliente y el servidor de manera bilateral
- La información viaja del servidor al cliente sin que este lo solicite
- **Inicio de proyecto**

> npm init 
> npm i -D typescript @types/node ts-node-dev rimraf
> npx tsc --init --outDir dist/ --rootDir src

- Creo los scripts

~~~json
 {
    "dev": "tsnd --respawn --clear src/app.ts",
    "build": "rimraf ./dist && tsc",
    "start": "npm run build && node dist/app.js"
 }
~~~
-----

## WS - Websockets Library

- Hay varias librerías (Socket.io, SockJS...)
- WS es el más popular
- WebSocket es lo que vamos a usar desde el navegador web cliente porque viene nativamente
- Instalamos WS @types/ws

- Copio el código de ejemplo del server de la web de instalación de npm

~~~js
import { WebSocketServer } from 'ws';

//wss es web socket server y ws es el websocket cliente del index.html
const wss = new WebSocketServer({ port: 3000 });

wss.on('connection', function connection(ws) {

  console.log("client connected")
  
  ws.on('error', console.error);

  ws.on('message', function message(data) {
    console.log('received: %s', data);
  });

  ws.send('hola desde el server');
});
~~~

- Para probarlo usaremos POSTMAN
- El URL es el puerto indicado (3000) localhost:3000 (el protocolo no es http es ws, POSTMAN lo agrega automáticamente, no hace falta ponerlo)
- Recibo el mensaje hola desde el server sin que haya ninguna petición
- Frontend - Websockets
- Creo una nueva carpeta para el front con un archivo index.html

~~~html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>

<body>
  <h1>Websockets - <small>Status</small></h1>


  <form>
    <input type="text" placeholder="Enviar mensaje" />
    <button>Enviar</button>
  </form>

  <ul id="messages">

  </ul>

  <script>
    console.log("Hola mundo")
  </script>


</body>

</html>
~~~

- Si lo abro de manera habitual parecerá que funciona pero lo ejecutará con el protocolo file. No me sirve
- En la terminal entro en la carpeta del front y ejecuto

> npx http-server -o

- Lo instalo si no lo tengo
- Lo abre en http://127.0.0.1:8080 
- En consola tengo el Hola mundo
- En el script del index creo un webSocket usando la implementación nativa de los websockets
- Uso el protocolo ws

~~~html
<script>
    const socket = new webSocket('ws://localhost:3000')
</script>
~~~

- Cada vez que recargue la página aparecerá en consola client connected como indiqué en el server con un console.log
- Puedo añadir un console.log al cierre de la conexión en el server (dentro de ws.on('connection', )) para ver la desconexión

~~~js
ws.on('close', ()=> console.log("client disconnected"))
~~~

- Puedo hacer un console.log del ws para ver mucha info del cliente que se está conectando
- Diferentes pestañas del navegador van a ser diferentes conexiones
- Ya estamos list@s para enviar info del cliente al servidor
- En el index ahora he creado un socket, dispongo del onclose, onerror, onmessage, onopen, OPEN
- Con onopen tengo el event, puedo verlo con un console.log desde el script del html

~~~html
<script>
     const socket = new webSocket('ws://localhost:3000')
    socket.onopen((event)=>{
        console.log(event)
        console.log('Connected')
    })
    socket.onclose((event)=>{
        console.log(event)
        console.log('disconnected')
    })
</script>
~~~

- Si bajo el servidor dice disconnected
- Si lo vuelvo a levantar no dice Connected
- Tengo que programar ese mecanismo
- El webscoket que he creado en el script del html le faltan ciertas características que optras librerías hacen por nosotros
- Eso no significa que no lo podamos implementar
-------

## Enviar mensajes al servidor

- 