---
theme: apple-basic
background: https://source.unsplash.com/collection/94734566/1920x1080
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
drawings:
  persist: false
transition: slide-up
mdc: true
layout: intro-image-right
image: 'https://shub.sfo2.digitaloceanspaces.com/docs/web-tips/server-sent-events/sse-demo.gif'
---

<div class="d-flex">
<div>
    <h1>Уведомления на фронтенде</h1>

    Реализация через Node.js и SSE
</div>

[//]: # (<img src="https://shub.sfo2.digitaloceanspaces.com/docs/web-tips/server-sent-events/sse-demo.gif" />)

</div>


<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">

[//]: # (    Press Space for next page <carbon:arrow-right class="inline"/>)
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
  <a href="https://github.com/TanyaLagodich/notifications" target="_blank" alt="GitHub" title="Open in GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<style>
div {
    background-size: contain;
}
</style>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

<style>
div {
  background-size: cover;
}
</style>

---
transition: slide-up
---

# Задача

Уведомлять пользователя о различных событиях на бекенде.
![Example of notification](/assets/notification.png "Example")

# Проблемы
- Как уведомлять пользователя в режиме real time?
- Как минимизировать трудозатраты бекенд-разработчиков?


<!--
You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/guide/syntax#embedded-styles
-->

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

<!--
Here is another comment.
-->

---
layout: default
transition: slide-up
level: 2
---

# Решение

Использовать Node.js для создания сервера, который будет принимать POST-запросы от основного сервера и отправлять их подключенным пользователям с помощью Server-Sent events

## Преимущества
- Минимальные трудозатраты на стороне бекенда
- Простота и гибкость, тк обработка уведомлений вынесена в отдельный сервис -- работает независимо
- Возможность интеграции с другими сервисами (на будущее)
- (Практика в Node.js для меня 🤓)

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

---
layout: default
transition: slide-up
level: 2
---
# Stack

- Node.js
- Fastify
- Server-Sent events


---
layout: default
transition: slide-up
level: 2
---
# Code

Class SSE, который отвечает за хранение подключенных пользователей при их подключении или отключении

Так же отправляет уведомление подключенномму клиенту

```js {all|2|1-6|9|all}
class SSE {
    constructor() {
        this.clients = new Map();

        EventEmitter.on('send-to-client', ({userId, text}) => {
            this.sendToClient({userId, text});
        });
    }
}
```

---
layout: default
transition: slide-up
level: 2
---
# Подключение пользователей
```javascript
  addClient(req, reply) {
        const headers = {
            'Content-Type': 'text/event-stream',
            'Connection': 'keep-alive',
            'Cache-Control': 'no-cache',
            'Access-Control-Allow-Origin': '*',
        };
        reply.raw.writeHead(200, headers);

        const userId = req.params.userId;

        if (!userId) {
            // TODO add handling error
        }

        if (!this.clients.has(userId)) {
            this.clients.set(userId, []);
        }

        const clientsArray = this.clients.get(userId);
        clientsArray.push(reply);
    }
```
---
layout: default
transition: slide-up
level: 2
---
# Обработка отключенных пользователей
```javascript
const removeListener = () => {
            const index = clientsArray.indexOf(reply);
            if (index !== -1) {
                clientsArray.splice(index, 1);
                if (clientsArray.length === 0) {
                    this.clients.delete(userId);
                }
            }
        };

        reply.raw.once('close', removeListener);
```
---
layout: default
transition: slide-up
level: 2
---

# Отправка уведомлений
При каждом POST- запросе на `/notification` с телом (формат обсуждается)
```
{
  "text": "notification",
  "userId": "1234"
}
```

```javascript
   sendToClient({ userId, text }) {
        const clientsArray = this.clients.get(userId);

        if (clientsArray) {
            clientsArray.forEach((client) => {
                try {
                    const formattedData = JSON.stringify({ text });
                    client.raw.write(`data: ${formattedData}\n\n`);
                } catch (err) {
                    console.error(err);
                }
            });
        }
    }
```

---
layout: default
transition: slide-up
level: 2
---
# Как подключить на фронтенде
(на примере front-office)
```javascript
    const evtSource = new EventSource(`http://localhost:3000/events/${userId}`);
    evtSource.onopen = () => {
      console.log('Событие: open');
    };

    evtSource.onmessage = (event) => {
      const parsedData = JSON.parse(event.data);
      const { text } = parsedData;
      ctx.uiNotifier.success(text); // пример как можно показать уведомление
    };
```

---
layout: default
transition: slide-up
level: 2
---
# Демонстрация

---
layout: default
transition: slide-up
level: 2
---
# Что нужно еще сделать?

- авторизация
- оптимизация

---
layout: default
transition: slide-up
level: 2
---
# Обсуждение

- Вместо SSE можно использовать WebSocket, но нужно ли?
- 
