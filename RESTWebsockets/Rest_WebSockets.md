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

- tengo el WssService, todavía no he inicializado el WebSocketServer (wss) en ningún lugar
- En el server.ts tengo a express(). Express es una implementación del Server from 'http'
- Entonces, tengo que crear mi Server http porque eso es lo que estoy esperando desde la interface de Options de wssService
- En src/server.ts

~~~js
import express, { Router } from 'express';
import path from 'path';

interface Options {
  port: number;
  // routes: Router;
  public_path?: string;
}


export class Server {

  public readonly app = express(); //express es una implementación del Server from 'http'
  private serverListener?: any;
  private readonly port: number;
  private readonly publicPath: string;
  // private readonly routes: Router;

  constructor(options: Options) {
    const { port, public_path = 'public' } = options;
    this.port = port;
    this.publicPath = public_path;

    this.configure();
  }

  private configure() {


    //* Middlewares
    this.app.use( express.json() ); // raw
    this.app.use( express.urlencoded({ extended: true }) ); // x-www-form-urlencoded

    //* Public Folder
    this.app.use( express.static( this.publicPath ) ); //doy acceso a express a la carpeta public

    //* Routes
    // this.app.use( this.routes );

    //* SPA
    this.app.get(/^\/(?!api).*/, (req, res) => { //esta regExp hace que responda siempre el index.html siempre que la ruta no empiece por api
      const indexPath = path.join( __dirname + `../../../${ this.publicPath }/index.html` );
      res.sendFile(indexPath); //en la respuesta envío con sendFile el html
    });

  }

  public setRoutes(  router: Router ) {
    this.app.use(router);
  }
  
  
  async start() { 

    this.serverListener = this.app.listen(this.port, () => {
      console.log(`Server running on port ${ this.port }`);
    });

  }

  public close() {
    this.serverListener?.close();
  }

}
~~~

- En app.ts

~~~js
import { createServer } from 'http';
import { envs } from './config/envs';
import { AppRoutes } from './presentation/routes';
import { Server } from './presentation/server';
import { WssService } from './presentation/services/wss.service';


(async()=> {
  main();
})();


function main() {

  const server = new Server({
    port: envs.PORT,
  });

  const httpServer = createServer( server.app );
  WssService.initWss({ server: httpServer }); // si no le indico el path tiene /ws por defecto
                                              //este es el server en el que tengo el Wss


  server.setRoutes( AppRoutes.routes ); //En el momento que inicializo mis rutas , estas rutas van a  necesitar acceso al Wss
                                        //Pero mi Wss no está inicializado hasta después, por eso aquí le paso las rutas



  httpServer.listen( envs.PORT, () => { 
    console.log(`Server running on port: ${ envs.PORT }`);
  })
}
~~~
- En src/routes

~~~js
import { Router } from 'express';
import { TicketRoutes } from './tickets/routes';


export class AppRoutes {


  static get routes(): Router {

    const router = Router();
    
    // Definir las rutas
    // router.use('/api/todos', /*TodoRoutes.routes */ );

    router.use('/api/ticket',  TicketRoutes.routes  );



    return router;
  }
}
~~~

- En presentation creo tickets con el archivo controller.ts y routes.ts
- ticket/routes.ts

~~~js
import { Router } from 'express';
import { TicketController } from './controller';




export class TicketRoutes {


  static get routes() {

    const router = Router();
    const ticketController = new TicketController();


    router.get('/', ticketController.getTickets );
    router.get('/last', ticketController.getLastTicketNumber );
    router.get('/pending', ticketController.pendingTickets );


    router.post('/', ticketController.createTicket);

    router.get('/draw/:desk', ticketController.drawTicket);
    router.put('/done/:ticketId', ticketController.ticketFinished);

    router.get('/working-on', ticketController.workingOn );


    return router;
  }


}
~~~

- El tickets/controller.ts

~~~js
import { Request, Response } from 'express';
import { TicketService } from '../services/ticket.service';



export class TicketController {

  // DI - WssService
  constructor(
    private readonly ticketService = new TicketService(),
  ) {}


  public getTickets = async (req: Request, res: Response) => {
    res.json( this.ticketService.tickets );
  }

  public getLastTicketNumber = async (req: Request, res: Response) => {
    res.json( this.ticketService.lastTicketNumber );
  }

  public pendingTickets = async (req: Request, res: Response) => {
    res.json( this.ticketService.pendingTickets );
  }

  public createTicket = async (req: Request, res: Response) => {
    res.status(201).json( this.ticketService.createTicket() );
  }

  public drawTicket = async (req: Request, res: Response) => {
    const { desk } = req.params;
    res.json( this.ticketService.drawTicket(desk) );
  }

  public ticketFinished = async (req: Request, res: Response) => {
    const { ticketId } = req.params;
    
    res.json( this.ticketService.onFinishedTicket(ticketId) );
  }

  public workingOn = async (req: Request, res: Response) => {
    res.json( this.ticketService.lastWorkingOnTickets );
  }
} 
~~~

- El ticketService lo tengo aquí!! en src/presentation/services/ticket.service.ts

~~~js
import { UuidAdapter } from '../../config/uuid.adapter';
import { Ticket } from '../../domain/interfaces/ticket';
import { WssService } from './wss.service';



export class TicketService {

  constructor(
    private readonly wssService = WssService.instance,
  ) {}


  public tickets: Ticket [] = [
    { id: UuidAdapter.v4(), number: 1, createdAt: new Date(), done: false },
    { id: UuidAdapter.v4(), number: 2, createdAt: new Date(), done: false },
    { id: UuidAdapter.v4(), number: 3, createdAt: new Date(), done: false },
    { id: UuidAdapter.v4(), number: 4, createdAt: new Date(), done: false },
    { id: UuidAdapter.v4(), number: 5, createdAt: new Date(), done: false },
    { id: UuidAdapter.v4(), number: 6, createdAt: new Date(), done: false },
  ];

  private readonly workingOnTickets: Ticket[] = [];

  public get pendingTickets():Ticket[] {
    return this.tickets.filter( ticket => !ticket.handleAtDesk );
  }

  public get lastWorkingOnTickets():Ticket[] {
    return this.workingOnTickets.slice(0,4);
  }

  public get lastTicketNumber(): number {
    return this.tickets.length > 0 ? this.tickets.at(-1)!.number : 0;
  }

  public createTicket() {

    const ticket: Ticket = {
      id: UuidAdapter.v4(),
      number: this.lastTicketNumber + 1,
      createdAt: new Date(),
      done: false,
      handleAt: undefined,
      handleAtDesk: undefined,
    }

    this.tickets.push(ticket);
    this.onTicketNumberChanged();

    return ticket;
  }

  public drawTicket(desk: string) {

    const ticket = this.tickets.find( t => !t.handleAtDesk );
    if ( !ticket ) return { status: 'error', message: 'No hay tickets pendientes' };

    ticket.handleAtDesk = desk;
    ticket.handleAt = new Date();


    this.workingOnTickets.unshift({...ticket});
    this.onTicketNumberChanged();
    this.onWorkingOnChanged();


    return { status: 'ok', ticket }

  }

  public onFinishedTicket( id: string ) {
    const ticket = this.tickets.find( t => t.id === id );
    if ( !ticket ) return { status: 'error', message: 'Ticket no encontrado' };

    this.tickets = this.tickets.map( ticket => {

      if ( ticket.id === id ) {
        ticket.done = true;
      }

      return ticket;
    });

    return { status: 'ok' }
  }

  private onTicketNumberChanged() {
    this.wssService.sendMessage('on-ticket-count-changed', this.pendingTickets.length );
  }

  private onWorkingOnChanged() {
    this.wssService.sendMessage('on-working-changed', this.lastWorkingOnTickets );
  }
}
~~~

- La interfaz de ticket está en domain/interfaces/ticket.interface.ts

~~~js
export interface Ticket {
  id: string;
  number: number;
  createdAt: Date;
  handleAtDesk?: string; // Escritorio 1,
  handleAt?: Date;
  done: boolean;
}
~~~

- Para las variables de entorno  creo src/config e instalo dotenv y env-var para validar
- src/config/envs.ts

~~~js
import 'dotenv/config';
import { get } from 'env-var';


export const envs = {

  PORT: get('PORT').required().asPortNumber(),

}
~~~

- Creo también el uuidAdapter en la carpeta config, usado en el arreglo de tickets creado anteriormente

~~~js
import { v4 as uuidv4 } from 'uuid';

export class UuidAdapter {

  public static v4() {
    return uuidv4();
  }


}
~~~

- 


