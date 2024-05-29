# REST WebSockets

- Una API REST puede desencadenar notificaciones a sus clientes
- Crearemos una aplicación de tickets sin frameworks (no react, solo js)
- En una pantalla tengo el ticket y el escritorio en grande, con los otros escritorios y sus tickets en el lateral
- En otra pantalla tengo la de un escritorio
- Tengo una pantalla donde indico a qué escritorio quiero ingresar/crear
- En el escritorio tengo una cola de tickets
- Desde otra pantalla tengo "crear nuevo ticket" que si clico aumento la cola
- Si le doy a "atender el siguiente ticket", el ticket aparece en la otra pantalla grande
- Le doy a terminar y pone atendiendo a nadie, le vuelvo a dar a "atender el siguiente..." y así
- También puedo crear nuevos tickets desde una petición POST a un endpoint y lo añade a la cola
----

## Socket Server

- En src creo
  - config
  - domain/interfaces
  - presentation/services
  - presentation/tickets 

- Quiero manejar el server como un singtleton. Solo quiero una instancia de mi websocket server
- 
- src/services/wss.service.ts

~~~js
import { Server } from 'http';
import { WebSocket, WebSocketServer} from 'ws';

interface Options {
  server: Server;
  path?: string; // ws donde quiero que mis websockets se conecten
}


export class WssService {

  private static _instance: WssService;
  private wss: WebSocketServer;

  private constructor( options: Options ) {
    const { server, path = '/ws' } = options; /// ws://localhost:3000/ws aqui lanzaremos nuestra conexión

  //conecto el servidor de WebSockets con el servidor de Express
    this.wss = new WebSocketServer({ server, path }); //este server que le paso como argumento va a ser nuestro servidor de Express

    this.start(); //llamo aquí el método de conexión que hay más abajo
  }

  //al ser una instancia privada debo crear un getter 
  static get instance(): WssService {
    if ( !WssService._instance ) {
      throw 'WssService is not initialized'; //si no está creada la instancia devuelvo un string "not initializated"
    }

    return WssService._instance; //si hay una instancia la devuelvo
  }

  //lo llamo inicializar una sola vez, en todos los lugares voy a apuntar a la misma instancia
  static initWss( options: Options ) {
    WssService._instance = new WssService(options);
  }


  //creo un método para la conexión
  public start() {
                                //debo importarlo de ws, si no usará la versión nativa de WebSockets que tiene otros métodos
    this.wss.on('connection', (ws: WebSocket ) => {

      console.log('Client connected');

      ws.on('close', () => console.log('Client disconnected') ) //cuando se desconecte mandaremos el string de disconnected

    });

  }

}
~~~

- La idea es poder usar este servicio para mandar y lanzar las peticiones que queramos desde cualquier lado de mi aplicación
- Al ser un singleton apuntará siempre a la misma instancia del servicio
-----

## Conectar servidores Express y WS

- 