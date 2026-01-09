# WebSocket Chat Application

Uma aplicação de chat em tempo real desenvolvida com Spring Boot, WebSocket e MongoDB. O projeto demonstra a implementação do padrão **Pub/Sub** para comunicação entre clientes através de um servidor WebSocket.

## 📋 Visão Geral

Esta aplicação implementa um sistema de chat um-para-um com suporte a múltiplos usuários conectados simultaneamente. Os mensagens são entregues em tempo real utilizando o protocolo WebSocket e persistidas em um banco de dados MongoDB.

### Principais Características

- ✅ Chat em tempo real (1:1)
- ✅ Múltiplos usuários simultâneos
- ✅ Notificações de mensagens não lidas
- ✅ Status de usuários online/offline
- ✅ Persistência de mensagens em MongoDB
- ✅ Comunicação bidirecional via WebSocket

---

## 🏗️ Arquitetura do Projeto

### Estrutura de Diretórios

```
websocket/
├── src/
│   ├── main/
│   │   ├── java/com/samuel/websocket/
│   │   │   ├── WebsocketApplication.java          # Classe principal Spring Boot
│   │   │   ├── chat/                              # Lógica de mensagens
│   │   │   │   ├── ChatController.java
│   │   │   │   ├── ChatMessage.java
│   │   │   │   ├── ChatMessageRepository.java
│   │   │   │   ├── ChatMessageService.java
│   │   │   │   └── ChatNotification.java
│   │   │   ├── chatroom/                          # Gerenciamento de salas de chat
│   │   │   │   ├── ChatRoom.java
│   │   │   │   ├── ChatRoomRepository.java
│   │   │   │   └── ChatRoomService.java
│   │   │   ├── config/                            # Configuração WebSocket
│   │   │   │   └── WebSocketConfig.java
│   │   │   └── user/                              # Gerenciamento de usuários
│   │   │       ├── User.java
│   │   │       ├── UserController.java
│   │   │       ├── UserRepository.java
│   │   │       ├── UserService.java
│   │   │       └── Status.java
│   │   └── resources/
│   │       ├── application.yml                    # Configurações da aplicação
│   │       └── static/                            # Frontend
│   │           ├── index.html
│   │           ├── css/main.css
│   │           ├── js/main.js
│   │           └── img/
│   └── test/                                      # Testes unitários
├── pom.xml                                        # Dependências Maven
├── docker-compose.yml                             # Container MongoDB
└── README.md
```

---

## 🔄 Padrão Pub/Sub (Publisher/Subscriber)

### O Que É?

O padrão Pub/Sub é um padrão de mensageria onde:
- **Publishers** enviam mensagens sem conhecer quem as receberá
- **Subscribers** se inscrevem em tópicos de interesse para receber mensagens
- Um **Message Broker** gerencia a distribuição de mensagens

### Implementação no Projeto

#### Backend (Spring WebSocket Broker)

O Spring gerencia os tópicos e canais de mensagem:

```
/user/{userId}/queue/messages    → Fila privada de cada usuário (mensagens diretas)
/topic/public                     → Tópico público (atualizações de usuários online)
```

#### Fluxo de Publicação de Mensagem

```
1. Cliente envia mensagem via WebSocket
   stompClient.send("/app/chat", {}, JSON.stringify(chatMessage))

2. ChatController recebe em @MessageMapping("/chat")
   └─> Salva mensagem no MongoDB
   └─> Cria ChatNotification

3. Spring publica para subscriber via messagingTemplate.convertAndSendToUser()
   └─> Entrega a `/user/{recipientId}/queue/messages`

4. Cliente subscriber recebe em stompClient.subscribe()
   └─> Função onMessageReceived() processa a notificação
   └─> Exibe a mensagem na interface
```

#### Fluxo de Atualização de Usuários

```
1. Usuário se conecta
   stompClient.send("/app/user.addUser", {}, JSON.stringify(user))

2. UserController recebe em @MessageMapping("/user.addUser")
   └─> Com @SendTo("/user/topic") publica para todos os subscribers

3. Todos os clientes recebem em stompClient.subscribe("/topic/public")
   └─> Função onUserListUpdate() atualiza lista de usuários
```

---

## 🌐 WebSocket & STOMP

### O Que É WebSocket?

WebSocket é um protocolo de comunicação bidirecional que permite:
- Conexão persistente entre cliente e servidor
- Comunicação full-duplex em tempo real
- Menor overhead em comparação com HTTP polling

### STOMP (Simple Text Oriented Messaging Protocol)

STOMP é um protocolo que roda sobre WebSocket fornecendo:
- Frames de mensagem estruturados
- Suporte a tópicos e filas
- Confirmações de entrega

### Configuração no Projeto

**WebSocketConfig.java**
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    // Habilita broker simples
    registry.enableSimpleBroker("/user");
    registry.setApplicationDestinationPrefixes("/app");
    
    // Registra endpoint WebSocket
    registry.addEndpoint("/ws").withSockJS();
}
```

### Cliente JavaScript (Frontend)

```javascript
// Estabelece conexão WebSocket
const socket = new SockJS('/ws');
stompClient = Stomp.over(socket);

// Se conecta ao servidor
stompClient.connect({}, onConnected, onError);

// Se inscreve em canais
stompClient.subscribe(`/user/${nickname}/queue/messages`, onMessageReceived);
stompClient.subscribe(`/topic/public`, onUserListUpdate);

// Publica mensagens
stompClient.send("/app/chat", {}, JSON.stringify(chatMessage));
```

---

## 💾 MongoDB para Persistência

### Por Que MongoDB?

- **Flexibilidade**: Documentos JSON sem schema rígido
- **Escalabilidade**: Distribuição horizontal de dados
- **Performance**: Consultas otimizadas com índices
- **Integração**: Spring Data MongoDB simplifica operações

### Modelos de Dados

#### User (Coleção: users)
```java
@Document
public class User {
    @Id
    private String nickName;           // Identificador único
    private String fullName;
    private Status status;             // ONLINE / OFFLINE
}
```

#### ChatMessage (Coleção: chatMessages)
```java
@Document
public class ChatMessage {
    @Id
    private String id;
    private String chatId;             // ID da sala de chat
    private String senderId;           // Quem enviou
    private String recipientId;        // Quem recebe
    private String content;            // Conteúdo da mensagem
    private Date timestamp;            // Quando foi enviada
}
```

#### ChatRoom (Coleção: chatRooms)
```java
@Document
public class ChatRoom {
    @Id
    private String id;
    private String senderName;
    private String recipientName;
}
```

### Repositórios (Data Access)

Spring Data MongoDB fornece métodos automáticos:

```java
public interface UserRepository extends MongoRepository<User, String> {
    List<User> findAllByStatus(Status status);  // Usuários online
}

public interface ChatMessageRepository extends MongoRepository<ChatMessage, String> {
    List<ChatMessage> findByChatId(String chatId);  // Mensagens de uma sala
}
```

### Fluxo de Persistência

```
1. Mensagem recebida no servidor
   ChatMessage recebida com senderId, recipientId, content

2. Serviço salva no MongoDB
   chatMessageService.save(chatMessage)
   ├─> Obtém/cria chatRoom
   ├─> Define timestamp
   └─> Persiste em MongoDB

3. Recuperação de histórico
   GET /messages/{senderId}/{recipientId}
   └─> Consulta MongoDB por chatId
   └─> Retorna array de mensagens ordenadas
```

---

## 🚀 Fluxo Completo da Aplicação

### 1️⃣ Conexão de Usuário

```
Cliente                          Servidor WebSocket             MongoDB
   │                                   │                           │
   ├─ Entra nickname/fullname ─────────>                           │
   │                                   │                           │
   │                           @MessageMapping("/user.addUser")    │
   │                                   │                           │
   │                                   ├─ saveUser ────────────────>│
   │                                   │                           │
   │                           @SendTo("/user/topic")              │
   │                                   │                           │
   │<─ Publica update de usuários ─────┤                           │
   │                                   │                           │
   ├─ Atualiza lista de usuários       │                           │
   
```

### 2️⃣ Envio de Mensagem

```
Cliente A                        Servidor WebSocket             MongoDB
   │                                   │                           │
   ├─ Envia mensagem ──────────────────>                           │
   │                                   │                           │
   │                           @MessageMapping("/chat")            │
   │                                   │                           │
   │                                   ├─ save message ────────────>│
   │                                   │                           │
   │                    convertAndSendToUser(recipientId)          │
   │                           /queue/messages                     │
   │                                   │                           │
   │                               Cliente B                       │
   │                                   │<─ Entrega mensagem        │
   │                                   │                           │
   │                                   ├─ Exibe na UI              │
   
```

### 3️⃣ Recuperação de Histórico

```
Cliente                          Servidor WebSocket             MongoDB
   │                                   │                           │
   ├─ GET /messages/{sid}/{rid}────────>                           │
   │                                   │                           │
   │                          findChatMessages                     │
   │                                   │                           │
   │                                   ├─ Consulta ────────────────>│
   │                                   │<─ Retorna mensagens ───────┤
   │<─ Retorna JSON array de mensagens ┤                           │
   │                                   │                           │
   ├─ Exibe no chat                    │                           │
   
```

---

## 📦 Dependências Principais

```xml
<!-- Spring Boot WebSocket -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>

<!-- Spring Data MongoDB -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>

<!-- STOMP WebSocket Client (Frontend) -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/stomp.js/2.3.3/stomp.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/sockjs-client/1.1.4/sockjs.min.js"></script>
```

---

## 🔧 Configuração da Aplicação

### application.yml

```yaml
spring:
  application:
    name: websocket
  
  # Configuração MongoDB
  data:
    mongodb:
      uri: mongodb://localhost:27017/websocket
      
  # Configuração WebSocket
  websocket:
    message-broker:
      broker-relay:
        # Usa broker simples (não STOMP relay)
        enabled: false

# Porta da aplicação
server:
  port: 8080
```

### Docker Compose (MongoDB)

```yaml
services:
  mongodb:
    image: mongo:latest
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_DATABASE: websocket
```

---

## 🎯 Endpoints da API

### WebSocket Endpoints

| Endpoint | Tipo | Descrição |
|----------|------|-----------|
| `/ws` | WebSocket | Conexão STOMP principal |
| `/app/user.addUser` | STOMP | Registra novo usuário online |
| `/app/user.disconnectUser` | STOMP | Registra usuário offline |
| `/app/chat` | STOMP | Envia mensagem privada |
| `/user/{userId}/queue/messages` | STOMP Subscribe | Recebe mensagens diretas |
| `/topic/public` | STOMP Subscribe | Recebe atualizações de usuários |

### REST Endpoints

| Endpoint | Método | Descrição |
|----------|--------|-----------|
| `/users` | GET | Lista usuários online |
| `/messages/{senderId}/{recipientId}` | GET | Recupera histórico de chat |

---

## 🚀 Como Executar

### Pré-requisitos
- Java 17+
- MongoDB 5.0+

### Passos

1. **Clone o repositório**
```bash
git clone <repository-url>
cd websocket
```

2. **Inicie MongoDB com Docker**
```bash
docker-compose up -d
```

3. **Execute a aplicação**
```bash
./mvnw.cmd spring-boot:run
```

4. **Acesse no navegador**
```
http://localhost:8080
```

5. **Abra em múltiplas abas/janelas** para testar com vários usuários

---

## 📊 Exemplo de Fluxo: Enviando uma Mensagem

### Frontend (JavaScript)

```javascript
// 1. Usuário clica em enviar
function sendMessage(event) {
    const messageContent = messageInput.value.trim();
    
    if (messageContent && stompClient) {
        // 2. Cria objeto de mensagem
        const chatMessage = {
            senderId: nickname,
            recipientId: selectedUserId,
            content: messageContent,
            timestamp: new Date()
        };
        
        // 3. Envia via WebSocket
        stompClient.send("/app/chat", {}, JSON.stringify(chatMessage));
        
        // 4. Exibe na UI do remetente
        displayMessage(nickname, messageContent);
        messageInput.value = '';
    }
}
```

### Backend (Spring)

```java
// 1. Controller recebe mensagem
@MessageMapping("/chat")
public void processMessage(@Payload ChatMessage chatMessage) {
    // 2. Salva no MongoDB
    ChatMessage savedMsg = chatMessageService.save(chatMessage);
    
    // 3. Cria notificação
    ChatNotification notification = ChatNotification.builder()
        .id(savedMsg.getId())
        .senderId(savedMsg.getSenderId())
        .recipientId(savedMsg.getRecipientId())
        .content(savedMsg.getContent())
        .build();
    
    // 4. Publica para destinatário
    messagingTemplate.convertAndSendToUser(
        chatMessage.getRecipientId(),
        "/queue/messages",
        notification
    );
}
```

### Frontend Recebe

```javascript
// 1. Subscriber recebe notificação
stompClient.subscribe(`/user/${nickname}/queue/messages`, onMessageReceived);

// 2. Processa mensagem
async function onMessageReceived(payload) {
    const message = JSON.parse(payload.body);
    
    // 3. Se for do usuário selecionado, exibe
    if (selectedUserId === message.senderId) {
        displayMessage(message.senderId, message.content);
    }
    
    // 4. Marca notificação como recebida
    const notifiedUser = document.querySelector(`#${message.senderId}`);
    if (notifiedUser && !notifiedUser.classList.contains('active')) {
        const badge = notifiedUser.querySelector('.nbr-msg');
        badge.classList.remove('hidden');
    }
}
```

---

## 🔐 Melhorias Futuras

- [ ] Autenticação e autorização (JWT)
- [ ] Criptografia de mensagens E2E
- [ ] Suporte a grupos de chat
- [ ] Tipagem de mensagens (texto, arquivo, imagem)
- [ ] Reações e emojis
- [ ] Busca de mensagens
- [ ] Sincronização offline-first com IndexedDB
- [ ] Escalabilidade com Redis Pub/Sub
- [ ] Testes unitários e integração

---

## 📝 Conceitos-Chave Aplicados

### 1. **Pub/Sub Pattern**
Desacoplamento entre produtores e consumidores de mensagens através de tópicos intermediários.

### 2. **WebSocket Protocol**
Comunicação bidirecional persistente para entrega em tempo real.

### 3. **STOMP Protocol**
Abstração sobre WebSocket para fácil manipulação de mensagens estruturadas.

### 4. **Document-Oriented Database**
MongoDB armazena documentos JSON flexíveis e relacionados naturalmente.

### 5. **Spring Data Abstraction**
Repositórios simplificam operações CRUD sem escrita de SQL/queries.

### 6. **Event-Driven Architecture**
Componentes reagem a eventos (conexão de usuário, recebimento de mensagem).

---

## 📚 Referências

- [Spring WebSocket Documentation](https://spring.io/guides/gs/messaging-stomp-websocket/)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [STOMP Protocol Specification](https://stomp.github.io/stomp-specification-1.2.html)
- [WebSocket Protocol (RFC 6455)](https://tools.ietf.org/html/rfc6455)

---

## 📄 Licença

Este projeto está disponível sob a licença MIT.

---

**Desenvolvido com ❤️ usando Spring Boot, WebSocket e MongoDB**
