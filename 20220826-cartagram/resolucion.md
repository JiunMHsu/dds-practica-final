# CartaGram

## Arquitectura

**1. Especifique los métodos que considere más importantes del componente "CartaGram API". Utilice el nivel de detalle que considere adecuado y haga explícitas todas sus suposiciones.**

**Respuesta:**

Dado el contexto arquitectónico de dicho componente (API REST que sirve directo a clientes pesados), se asume que se trata de un aplicativo estructurado con clases controladores y clases servicios. Por simplicidad, especificaré principalmente los métodos de la capa service.

Contexto general, el componente contará con dos servicios concretos encargados de comunicarse con `CartaGram Core` y `CartaGram Seguridad`. Y, se asume que el sistema de autenticación esta basada en tokens.

_Métodos de Autenticación:_

- `login()`: Autentica al usuario mediante contraseña y emite un token. Internamente llamará al servicio de Seguridad.
- `authenticate()` Autentica al usuario mediante el token recibido. Internamente llamará al servicio de Seguridad para resolver dicho token en usuario.

_Métodos de Mensajería:_

- `createMessage()`: Crea un mensaje para un usuario emisor. Internamente autenticará el token de la request y llamará al servicio Core.
- `findMessagesByChatId()`: Recupera mensajes de un chat. Internamente autenticará el token de la request y llamará al servicio Core. Si el usuario no pertenece al chat, la petición a Core fallará. Permite paginación, filtrado y ordenamiento.
- `updateMessageStatus()`: Actualiza el estado del mensaje (Enviado, Recibido, Leído). Internamente autenticará el token de la request y llamará al servicio Core. Es específicamente de `status` porque por requerimiento, no se habilita otra modificación sobre el recurso `Mensaje`.

_Métodos de Chats:_

- `createChatroom()`: Crea un chat grupal, quien realiza la petición será asignado como creador del grupo. Internamente autenticará el token de la request y llamará al servicio Core.
- `findChatsByUser()`: Recupera todos los chats de un usuario, sin importar si es un grupo o un chat privado. Internamente autenticará el token de la request y llamará al servicio Core. Permite paginación, filtrado y ordenamiento.
- `findChatById()`: Recupera un único chat por Id. Internamente autenticará el token de la request y llamará al servicio Core. Si el usuario no pertenece al chat, la petición a Core fallará.
- `updateChatroom()`: Actualiza los atributos de un grupo, sea nombre, descripción, participantes o administradores. Internamente autenticará el token de la request y llamará al servicio Core. Si el actor (usuario que realiza la petición) no es administrador, la petición a Core fallará.
- `leaveChatroom()`: Remueve el usuario actor del chat de grupo. Internamente autenticará el token de la request y llamará al servicio Core. Si el usuario no pertenece al chat, la petición a Core fallará.
- `deleteChatroom()`: Remueve el chat de grupo. Internamente autenticará el token de la request y llamará al servicio Core. Si el Si el actor no es administrador, la petición a Core fallará.

_Métodos de Bots:_

- `createBot()`: Crea un Bot. Internamente autenticará el token de la request y llamará al servicio Core.
- `updateBot()`: Actualiza atributos de un Bot. Internamente autenticará el token de la request y llamará al servicio Core.

Del conjunto de métodos mencionado, no se contempla la posibilidad de notificación, dado que dicha funcionalidad implicaría otro mecanismo de integración que no es puramente REST. No obstante, si se considerara dicha posibilidad, habría el siguiente método:

- `notify()`: Notifica a los clientes conectados (refiriéndose a web app, mobile o desktop) según usuario destinatario del mensaje nuevo. Una implementación para dicha funcionalidad es la del WebSocket.

**2. Explique las diferencias en la facilidad de mantenimiento de las alternativas A y B.**

**Respuesta:**

En primer lugar, las diferencias arquitectónicas destacables entre las alternativas A y B son las siguientes:

1. Cliente pesado en A y cliente liviano en B, además, el componente que sirve el cliente liviano no consume CartaGram API, sino que consume directo de Core y Seguridad.
2. La alternativa B mantiene una base de datos diferente y un componente extra para la funcionalidad de estadísticas.

Respecto a la facilidad de mantenimiento:

La la diferencia (1), es decir, los componentes de cliente y conectividad de componentes de la funcionalidad principal, hace que la alternativa A sea **más mantenible** que la B. Esto se debe a que la A presenta dependencias más lineales y simples, por ejemplo: Si la interfaz que expone `Core` o `Seguridad` sufre un cambio, en la alternativa B, hay **dos componentes** que se ven impactados de manera directa y ambos se verán obligados a modificarse. Mientras que en la A, se deberá trabajar sobre un único componente.

En cuanto a la diferencia (2), sobre la funcionalidad de análisis y estadísticos, la alternativa B supone una mayor facilidad de mantenimiento que la A, ya que ésta ultima agrega una complejidad y componentes extra. Haciendo que, por ejemplo, un cambio en los schemas de la DB principal impacte en múltiples componentes, a diferencia de la alternativa B, en donde sólo se debería adaptar `CartaGram Análisis`

**3. Dado el siguiente escenario, indicar el atributo de calidad que considere que está implicado en el problema detallado, definirlo y comparar cómo impactaría la elección de las alternativas Ao B.**

|           |                                                                                |
| --------- | ------------------------------------------------------------------------------ |
| Estímulo  | CartaGram Análisis realiza una consulta que procesa un gran volumen de datos   |
| Ambiente  | Producción                                                                     |
| Respuesta | El servicio de mensajería no sufre ninguna degradación en su nivel de servicio |

**Respuesta:**

El atributo de calidad en juego es **Disponibilidad**. Dicho atributo se define como la capacidad del sistema o componente de estar operativo y accesible para su uso cuando se requiere.

Para lograr el escenario descrito, la alternativa A es la óptima o, mejor dicho, es la viable. Dado que, en la alternativa B, el aplicativo de análisis está directamente conectada a la única base de datos, la realización de una consulta de gran volumen y su posterior análisis implicaría una lectura intensa y, eventualmente, también una intensa escritura. Ésto afectaría a las operaciones de lectoescritura del servicio de mensajería.

En la alternativa A, `CartaGram Store` copia de forma diaria y a ritmo constante, los datos de la DB principal a la `DB Stats` NoSQL. Luego, para cuando `CartaGram Análisis` realice la lectura pesada, impactará únicamente sobre `DB Stats`. Es más, al ser ésta una base de datos NoSQL, optimizaría aún más los tiempos de lectura. Por lo que la alternativa A es superior en cualquier aspecto para el problema detallado.
