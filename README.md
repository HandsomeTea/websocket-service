# websocket-service

简单易用的websocket服务器，基于[ws](https://www.npmjs.com/package/ws)，遵循[JSON-RPC 2.0](https://wiki.geekdream.com/Specification/json-rpc_2.0.html)协议。

# 快速开始

## 安装

```
npm install --save @coco-sheng/websocket-service
```

## 示例代码

typing.d.ts

```typescript
interface SocketAttr {
    userId: string;
    role: string;
    type: string
    token: string;
}
```

server.ts

```typescript
import { WebsocketServer } from '@coco-sheng/websocket-service';


const port = 3403;

export default new WebsocketServer<SocketAttr>({ port });
```

start.ts

```typescript
import server from './server';

server.start();
```

`SocketAttr`是socket连接上的属性，详见[socket属性](#socket属性)部分。

# method

定义method(类似于定义一个api)

method.ts

```typescript
import server from './server';


server.register('hello', () => {
    return 'hello world!';
});
```

你也可以同时定义多个method

```typescript
import { MethodFn } from '@coco-sheng/websocket-service';
import server from './server';


server.register({
    hello1() {
        return 'hello world1!';
    },
    hello2() {
        return {
            result: 'hello world2!'
        };
    },
    hello3(){
        console.log('hello world3!');
    }
});


const testMethod: MethodFn<SocketAttr> = () => {
    console.log('test');
};


server.register({ testMethod });
```

client.ts

```typescript
import Websocket from 'ws';

const port = 3403;
const client = new Websocket(`ws://localhost:${port}`);


client.on('open',async ()=>{
    const result1 = await new Promise(resolve => {
        client.send(JSON.stringify({ method: 'hello', id: 1, params: [], jsonrpc: '2.0' }));
        client.once('message', data => resolve(JSON.parse(data.toString())));
    });

    console.log(result1);
    // {
    //     jsonrpc: '2.0',
    //     id: 1,
    //     method: 'hello',
    //     result: 'hello world!'
    // }

    const result2 = await new Promise(resolve => {
        client.send(JSON.stringify({ method: 'hello1', id: 2, params: [], jsonrpc: '2.0' }));
        client.once('message', data => resolve(JSON.parse(data.toString())));
    });

    console.log(result2);
    // {
    //     jsonrpc: '2.0',
    //     id: 2,
    //     method: 'hello1',
    //     result: 'hello world1!'
    // }

    const result3 = await new Promise(resolve => {
        client.send(JSON.stringify({ method: 'hello2', id: 3, params: [], jsonrpc: '2.0' }));
        client.once('message', data => resolve(JSON.parse(data.toString())));
    });

    console.log(result3);
    // {
    //     jsonrpc: '2.0',
    //     id: 3,
    //     method: 'hello2',
    //     result: {
    //         result: 'hello world2!'
    //     }
    // }

    const result4 = await new Promise(resolve => {
        client.send(JSON.stringify({ method: 'hello3', id: 4, params: [], jsonrpc: '2.0' }));
        client.once('message', data => resolve(JSON.parse(data.toString())));
    });

    console.log(result4);
    // {
    //     jsonrpc: '2.0',
    //     id: 4,
    //     method: 'hello3',
    //     result: ''
    // }

    const result5 = await new Promise(resolve => {
        client.send(JSON.stringify({ method: 'testMethod', id: 5, params: [], jsonrpc: '2.0' }));
        client.once('message', data => resolve(JSON.parse(data.toString())));
    });

    console.log(result5);
    // {
    //     jsonrpc: '2.0',
    //     id: 5,
    //     method: 'testMethod',
    //     result: ''
    // }
});
```

主动向客户端发送消息

```typescript
server.register('hello',(_params, socket)=>{
    // _params 为method的请求参数
    socket.sendout({
        method: 'test'
        result: 'pending hello'
    });

    // 或者
    socket.send(JSON.stringify({
        jsonrpc: '2.0',
        method: 'test',
        result: 'pending hello'
    }));
});
```

`sendout`和`send`的区别在于`sendout`会把要发送的数据转换为符合`jsonrpc2.0`规范的格式，同时也会根据数据压缩配置将数据进行压缩处理；`send`则需要手动组装`jsonrpc2.0`规范的数据，且不会根据配置压缩数据。

# 中间件

中间件是一个在请求到达method之前对请求的数据和业务进行处理的函数。该函数如果返回一个Object，则会将该Object的属性挂载到当前socket连接的属性(后续业务可获取)上；返回其它数据结构则不做任何处理，后续业务逻辑也无法获取该返回值。可以定义针对全部method的一个会多个中间件，也可以为某个method定义一个或多个中间件，中间件的执行顺序为中间件定义的代码逻辑顺序。

middleware.ts，定义全局的中间件：

```typescript
import { MiddlewareFn } from '@coco-sheng/websocket-service';

server.use(()=>{
    console.log('this is a middleware for all methods');
});

const mdw1: MiddlewareFn<SocketAttr> = (_params, socket, method) => {

    if (method !== 'login' && !socket.getAttr('userId')) {
        throw Error('you must login to do this!');
    }

    console.log('middleware 1 for all methods');
};
const mdw2: MiddlewareFn<SocketAttr> = () =>{
    // ...
    return { type: '1' };
};

server.use(mdw1, mdw2, ...);
```

定义针对某个method的中间件。

```typescript
import { MiddlewareFn } from '@coco-sheng/websocket-service';

server.use('login', () => {
    console.log('to login method');

    // ...
    return { role: 'admin' };
    // 将会为当前socket设置一个role的属性，值为admin
});


const checkLoginToken: MiddlewareFn<SocketAttr> = (params, socket) => {
    // ...
};
const checkPermission: MiddlewareFn<SocketAttr> = () => {
    // ...
    throw new Error('you are no permission')
};


server.use('login', checkoutLoginToken, checkPermission);
```

使用中间件函数的socket参数，也可以在中间件里向客户端主动发送消息。

# socket属性

socket的属性即挂在到当前socket连接上的数据。

## 属性的设置

属性的设置有两种方式，一种是在中间件函数里返回一个Object(详见中间件部分)，另一种是调用socket对象本身的属性操作函数，如下：

```typescript
// 在method中设置
server.register('hello', (_params, socket) => {
    socket.setAttr('key','value');
    socket.setAttr({
        user: '....'
        role: 'admin'
    });
});


// 在中间件中设置
server.use((_params, socket, method)=>{
    console.log('this is a middleware for all methods');

    const user = ...;
    if(method === 'login' && user.role === 'admin'){
        socket.setAttr('role', 'admin');
        socket.setAttr({
            user: '....'
            type: 'password-login'
        });
    }
});
```

## 属性的获取

获取全部属性

```typescript
const attr = socket.getAttr();

// 全部属性
// {
//     ...   
// }
```

获取某个属性

```typescript
const value = socket.getAttr('key');

// 该属性值
// ...
```

获取某些属性

```typescript
const value = socket.getAttr('key1', 'key2', ...);


// key1,key2,...的属性
// {
//     key1: ...,
//     key2: ...,
//     ...   
// }
```

# 回调

## server.online

有客户端连接成功的回调函数，可传入多个，按顺序执行

```typescript
import { OnlineCallbackFn } from '@coco-sheng/websocket-service';

const online1: OnlineCallbackFn = () => { 
    // ...
};
const online2: OnlineCallbackFn = () => { 
    // ...
};

server.online(online1, online2);
```

## server.offline

有客户端断开连接时的回调函数，可传入多个，按顺序执行

```typescript
import { OfflineCallbackFn } from '@coco-sheng/websocket-service';

const offline1: OfflineCallbackFn<SocketAttr> = () => { };
const offline2: OfflineCallbackFn<SocketAttr> = () => { };

server.offline(offline1, offline2);
```

## server.error

捕获到全局错误时的回调，系统内置了一个全局错误处理逻辑，当捕获到错误时，会发送一条符合`jsonrpc2.0`规范的错误信息到客户端（客户端已下线除外），当配置了log(详情见扩展配置)，该错误信息也会打印到控制台。如果设置`server.error`回调函数，则不会执行系统内置的错误逻辑，但是依然会根据log配置打印错误信息，但是`server.error`设置的**错误回调函数内部的错误并不会再次捕获处理**。

```typescript
import { ErrorCallbackFn } from '@coco-sheng/websocket-service';

const error1: ErrorCallbackFn<SocketAttr> = () => { };
const error2: ErrorCallbackFn<SocketAttr> = () => { };

server.error(error1, error2);
```

# 内置method

系统内置了两个method：`connect`，`ping`

```typescript
client.on('open',async ()=>{
    const result1 = await new Promise(resolve => {
        client.send(JSON.stringify({ method: 'connect', id: 1, params: [], jsonrpc: '2.0' }));
        client.once('message', data => resolve(JSON.parse(data.toString())));
    });

    console.log(result1);
    // {
    //     jsonrpc: '2.0',
    //     id: 1,
    //     method: 'connect',
    //     result: {
    //         msg: 'connected',
    //         session: 'xxxxxxxxxx'
    //     }
    // }

    const result2 = await new Promise(resolve => {
        client.send(JSON.stringify({ method: 'ping', id: 2, params: [], jsonrpc: '2.0' }));
        client.once('message', data => resolve(JSON.parse(data.toString())));
    });

    console.log(result2);
    // {
    //     jsonrpc: '2.0',
    //     id: 2,
    //     method: 'ping',
    //     result: 'pong'
    // }
});
```

# server

## 获取连接

每一个客户端socket连接都有一个id(这个id通常可以作为socket连接的session标识来使用)，是一个随机生成的字符串，在method和中间件中都可以通过socket获取，以method为例：

```typescript
server.register('hello', (_params, socket) => {
    console.log(socket.id);
});
```

当你知道某个socket连接的id时，可以通过server直接拿到对应的socket对象：

```typescript
const sessionId = 'xxxxxxxxx';
const socket = server.getSocket(sessionId);


socket.sendout({
    method: 'notice',
    result: 'noticed!'
});
```

server中所有的socket连接，可通过`server.clients`来获取，也可以通过自定义条件的方式来获取某些socket连接：

```typescript
const socket = server.getSockets((attr:SocketAttr)=>{
    if(attr.role === 'admin'){
        return true;
    }
});
```

`server.getSockets`接受一个回调函数，该函数的入参为某个socket的所有属性，需要返回一个`Boolean`值来判断某个soeket连接是否为需要获取的socket对象。

## 获取连接属性



## 设置连接属性





# 关于ping

服务器端不建议主动去ping客户端，以此对连接进行保活，这样做会消耗服务器性能，所以系统内置了一个`ping`的method，当客户端发送`ping`的method时，系统会回复一条数据，详见[内置method](#内置method)。当然，你也可以通过其它方式实现ping。

# 扩展配置

`new WebsocketServer(config, options);`

- `[options.log]`：`Boolean | Function`，默认`false`关闭日志打印，当为`true`时，将采用内置的`log4js`日志配置打印日志；如果为一个函数，则需要返回一个`Logger`对象，系统的日志将采用该对象打印。

- `[options.compression]`：`zlib`，默认`undefined`。当为`zlib`时，将对服务器发送到客户端的数据先进行zlib压缩，再发送。

```typescript
import { WebsocketServer } from '@coco-sheng/websocket-service';


const port = 3403;

export default new WebsocketServer<SocketAttr>({ port }, {
    log: () => console,
    compression: 'zlib'
});
```

# 其它

暂无。
