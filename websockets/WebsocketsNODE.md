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
- El webscoket que he creado en el script del html le faltan ciertas características que otras librerías hacen por nosotros
- Eso no significa que no lo podamos implementar
-------

## Enviar mensajes al servidor

- Quiero usar el 'message' del ws.on (cliente websocket) que hay en el server
- Es decir, cuando el websocket emita un mensaje quiero ejecutar el console.log que tiene 
- Si quisera mandar binarios, strings, información usaría isBinary como segundo parámetro 

~~~js
 ws.on('message', function message(data, isBinary) {
    console.log('received: %s', data);
  });
~~~

- Voy al index.html a la etiqueta script

~~~html
<script>
  const socket = new WebSocket('ws://localhost:3000')
   
    const form = document.querySelector( 'form' ); //capturo el form
    const input = document.querySelector( 'input' ); //capturo el input


    function sendMessage( message ) {
      console.log({message})
    }

  //añado un listener al form para cuando le dé al botón pasarle a sendMessage el valor del input
    form.addEventListener('submit', event=>{
      event.preventDefault() //siempre para prevenir la recarga de la página
      const message = input.value
      sendMessage(message)
    })
  
</script>
~~~

- Si ahora escribo en el input del navegador Hola mundo me lo devuelve en consola
- Puedo verificar (sería necesario) que el message venga con algo con un if y message.length
- Si quiero mandar un m ensaje uso el socket.send(data)

~~~js
    function sendMessage( message ) {
      socket.send(message)
    }
~~~

- Este message cae en el ws.on('message') del server

~~~js
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 3000 });

wss.on('connection', function connection(ws) {

  console.log("client connected")
  
  ws.on('error', console.error);

  //cae aquí!! porque ws es el cliente
  ws.on('message', function message(data) {
    console.log('received: %s', data);
    //TODO:enviar la data de regreso al cliente
  });

  ws.send('hola desde el server');
});
~~~

- Con socket.io yo puedo escuchar mensajes personalizados
- Al usar ws que viene de forma nativa, solo puedo usar el 'message' desde el server
- Desde el index.html si pongo socket.on tengo onclose, onerror, onmessage, onopen
- Si quiero mandar diferente info o diferentes eventos habrá que ingeniárselas para recibirlo
- Para enviar la data de regreso al cliente uso ws.send en el server

~~~js
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 3000 });

wss.on('connection', function connection(ws) {

  console.log("client connected")
  
  ws.on('error', console.error);

  
  ws.on('message', function message(data) {
    console.log('received: %s', data);
  ws.send(data.toString().toUpperCase()) //Le devolveré lo que sea que me escribió pero en mayúsculas
});

  ws.send('hola desde el server');
});
~~~

- En el cliente (index.html)
- Si hago un console.log del event (en socket.onmessage) recibo un MessageEvent con mucha info
- Solo quiero la data que hay internamente, que se encuentra en el prototype/data

~~~html
<script>
  const socket = new WebSocket('ws://localhost:3000')
   
    const form = document.querySelector( 'form' ); //capturo el form
    const input = document.querySelector( 'input' ); //capturo el input


    function sendMessage( message ) {
      console.log({message})
    }

    form.addEventListener('submit', event=>{
      event.preventDefault() 
      const message = input.value
      sendMessage(message)
    })

    socket.onmessage=(event)=>{
      //console.log(event.data)

    }
  
</script>
~~~

- Lo que sea que escriba en el input del navegador lo recibo en consola en mayúsculas
- Lo que quiero es que este mensaje lo reciban también otras personas (chat), pero esto es lo que tengo hasta ahora
- Suponiendo que quisiera mandar un objeto como payload desde el server, para que no de error debo serializarlo con stringify

~~~js
 ws.on('message', function message(data) {
    console.log('received: %s', data);

    const payload = {
      type: "custom-message",
      payload: data.toString()
    }
  //ws.send(payload) esto devuelve error porque no es bufferLike
  ws.send( JSON.stringify(payload))
 })
~~~

- En el cliente (index.html) lo parseo

~~~html
<script>
  socket.onmessage=(event)=>{
       //console.log(event.data)

      const payload = JSON.parse(event.data)
      console.log(payload)
</script>
}
~~~

- Entonces, necesito usar stringify para mandarle la data al cliente que se conecto y usare parse para recibirlo desde el cliente
- Creemos un método para insertar en pantalla los mensajes que vamos recibiendo de vuelta del server

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
    let socket = null;

    const form = document.querySelector( 'form' );
    const input = document.querySelector( 'input' );
    const messagesElem = document.querySelector( '#messages' ); //tomo el ul por el id

    function sendMessage( message ) {
      socket?.send( message );
    }

  //para renderizar los mensajes
    function renderMessage( message ) {
      const li = document.createElement( 'li' ); //creo el list item
      li.innerHTML = message; //lo meto como contenido del li
      messagesElem.prepend( li ); //uso prepend porque quiero los primeros mensajes arriba (de lo contrario usaría append)
    }

    form.addEventListener( 'submit', ( event ) => {
      event.preventDefault();
      const message = input.value;
      sendMessage( message );
     
    } );

      socket.onmessage = ( event ) => {
        const { payload } = JSON.parse( event.data ); //desestructuro el payload

        renderMessage( payload ); //s el o paso al método para que lo renderice
      };

  </script>

</body>

</html>
~~~
---------

## Server Broadcast

- Queremos mandarle el mensaje a otras personas
- Creo otra instancia del navegador web
- Server Broadcast: Si queremos que el servidor emita a todo el mundo cuando un cliente WebSocket envíe data, incluso a si mismo
- La magia sucede aquí

~~~js
import {WebSocketServer, WebSocket} from 'ws'

(..codigo del server)

ws.on('message', function message(data, isBinary){ //podría tener solo la data y mandar el binary en false en el client.send
  
  const payload = JSON.stringify({
    type: 'custom-message',
    payload: data.toString()
  })
  
  wss.clients.forEach(function each(client){
    if (client.readyState === WebSocket.OPEN){ //si el readyState está abierto vamos a mandar la data 
      client.send(payload, {binary: isBinary}) //binary: false
    }
  })
})
~~~

- En cualquiera de las dos instancias del navegador que escriba se va a mostrar el mensaje en las dos pantallas
----

## Client Broadcast - A tod@s menos al emisor

- Para enviar a todos menos al emisor hago una evaluación de quién está emitiendo
- Si quien emite es el mismo que el cliente ws entonces no recibirá el mensaje

~~~js
import { WebSocketServer, WebSocket } from 'ws';



const wss = new WebSocketServer({ port: 3000 });

wss.on('connection', function connection(ws) {


  console.log('Client connected');

  ws.on('error', console.error);

  ws.on('message', function message(data) {

    const payload = JSON.stringify({
      type: 'custom-message',
      payload: data.toString(),
    })
    // ws.send( JSON.stringify(payload) );

    //* Todos - incluyente
    // wss.clients.forEach(function each(client) {
    //   if (client.readyState === WebSocket.OPEN) {
    //     client.send(payload, { binary: false });
    //   }
    // });

    // * Todos excluyente
    wss.clients.forEach(function each(client) {
      if (client !== ws && client.readyState === WebSocket.OPEN) { //si el cliente es diferente a ws (el emisor) y tiene el WebSocket abierto le envío la data
        client.send(payload, { binary: false });
      }
    });
    

  });

  // ws.send('Hola desde el servidor!');

  ws.on('close', () => {
    console.log('Client disconnected');
  })

});


console.log('http://localhost:3000');
~~~
----

## Reconexión

- Con ws (WebSockets de forma nativa) no viene nada para la reconexión por lo que será puro ingenio
- Por este motivo hay personas que optan por librerías en la parte del cliente
- En public/ index.html

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
    let socket = null;

    const form = document.querySelector( 'form' );
    const input = document.querySelector( 'input' );
    const messagesElem = document.querySelector( '#messages' );
    const statusElem = document.querySelector('small'); //mostraré en pantalla si estoy online u offline

    function sendMessage( message ) {
      socket?.send( message );
    }

    function renderMessage( message ) {
      const li = document.createElement( 'li' );
      li.innerHTML = message;
      messagesElem.prepend( li );
    }

    form.addEventListener( 'submit', ( event ) => {
      event.preventDefault();
      const message = input.value;
      sendMessage( message );
      input.value = null;
    } );

    //Creo la función para conectar al server
    function connectToServer() {
      socket = new WebSocket( 'ws://localhost:3000' );

      socket.onopen = ( event ) => {
        statusElem.innerText = 'Online';
      };

      //con onclose yo se cuando se pierde la conexión
      socket.onclose = ( event ) => {
        statusElem.innerText = 'Offline';
        setTimeout(() => {      //le doy un tiempo de gracia para reconectar
          connectToServer();
        }, 1500); //le doy un segundo y medio
      };

      socket.onmessage = ( event ) => {
        const { payload } = JSON.parse( event.data );

        renderMessage( payload );
      };

    }

    connectToServer(); //mando llamar a la función

  </script>

</body>

</html>
~~~

