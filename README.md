## Минимальное публичное API

API заявленное ниже является публичным для системы в целом. Данное API будет использовать сервис тестирования для эмулирования действий пользователя, а так же для проверки консистентности системы и ее отдельных компонентов (даже в условиях асинхронного взаимодействия система должна приходить в согласованное состояние в течение некоторого времени). См. "eventual consistency"

Отдельные компоненты ("сервисы") системы могут заявлять дополнительные методы API, как REST, так и messaging API для внутреннего общения сервисов системы.

Методы описанные ниже могут быть реализованы как прямое перенаправление (проксирование) к сервисам системы. Кроме того, они могут быть композитными, то есть включать в себя несколько запросов к сервисам и возвращение агрегированного результата пользователю. См. "API composition pattern"

### Структуры данных

Ниже расположено описание структур данных, которые используются в качестве ответа сервера

```jsx
UserDto {
  id: UUID,
  name: String
}

OrderStatus: COLLECTING | DISCARD | BOOKED | PAID | SHIPPING | REFUND | COMPLETED

OrderItemDto {
  id: UUID,
  title: String,
  price: Int
}

PaymentStatus: FAILED | SUCCESS

PaymentLogRecordDto {
  timestamp: Long,
  status: PaymentStatus,
  amount: Amount,
  transactionId: UUID,
}

OrderDto {
	id: UUID,
	timeCreated: Long,
	status: OrderStatus,
	itemsMap: Map<UUID, Amount>, 
	deliveryDuration: Int?,
	paymentHistory: List<PaymentLogRecord>
}

FinancialOperationType: WITHDRAW | REFUND

UserAccountFinancialLogRecordDto {
	type: FinancialOperationType,
	amount: Int,
	orderId: UUID,
	paymentTransactionId: UUID,
	timestamp: Long
}

CatalogItemDto {
	id: UUID,
	title: String,
	description: String,
	price: Int = 100,
	amount: Int
}

BookingDto {
	id: UUID,
	failedItems: Set<UUID>
}

PaymentSubmissionDto {
	timestamp: Long,
	transactionId: UUID
}

TokenResponseDto {
	accessToken: String,
	refreshToken: String
}

BookingStatus: FAILED | SUCCESS

BookingLogRecord {
    bookingId: UUID,
    itemId: UUID,
    status: BookingStatus,
    amount: Int,
    timestamp: Long
}

DeliverySubmissionOutcome: SUCCESS | FAILURE | EXPIRED

DeliveryInfoRecord {
    outcome: DeliverySubmissionOutcome,
    preparedTime: Long,
    attempts: Int,
    submittedTime: Long,
    transactionId: UUID,
    submissionStartedTime: Long
}

```


### Создание пользователя (user service)

```jsx
REQUEST:
	HTTP verb: POST
	URL: /users
	
	BODY FORMAT:
		{
			name: String
			password: String
		}

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: UserDto
```

### Получение пользователя (user service)

```jsx
REQUEST:	
	HTTP verb: GET
	URL: /users/{user_id}
	HEADERS: 
		Authorization: Bearer access_token
	PARAMETERS:
        user_id: UUID

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: UserDto
```

### Аутентификация (user service)

```jsx
REQUEST:	
	HTTP verb: POST
	URL: /authentication
	BODY FORMAT:
		{
			name: String
			password: String
		}

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: TokenResponseDto
```

### Обновление токена (user service)

```jsx
REQUEST:	
	HTTP verb: POST
	URL: /authentication/refresh
	HEADERS:
		Authorization: Bearer refresh_token 

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: TokenResponseDto
```

### Создание нового заказа (order service)

```jsx
REQUST:
	HTTP VERB: POST
	URL: /orders
	HEADERS: 
		Authorization: Bearer access_token

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: OrderDto
```

### Получение заказа (order service)

```jsx
REQUEST:
	HTTP verb: GET
	URL: /orders/{order_id}
	HEADERS: 
            Authorization: Bearer access_token
	PARAMETERS:
            order_id: UUID

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: OrderDto
```

### Получение информации о финансовых операциях с аккаунтом пользователя (payment service)

```jsx
REQUST:
	HTTP VERB: GET
	URL: /finlog?orderId={order_id}
	HEADERS:
            Authorization: Bearer access_token
	PARAMETERS:
            order_id: UUID (опциональный параметр. если не указан - вернуть все финансовые операции пользователя)

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: List<UserAccountFinancialLogRecordDto>
```

### Получение товаров каталога (warehouse service)

```jsx
REQUEST:
	HTTP verb: GET
	URL: /items?available={available}
	HEADERS:
            Authorization: Bearer access_token
	PARAMETERS:
            available: Boolean

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: List<CatalogItemDto>
```

### Помещение товара в корзину (order service)

```jsx
REQUST:
	HTTP VERB: PUT
	URL: /orders/{order_id}/items/{item_id}?amount={amount}
	HEADERS: 
		Authorization: Bearer access_token
	PARAMETERS:
		order_id: UUID
		item_id: UUID
		amount: Int

RESPONSE:
	HTTP CODES: 2** |400| any other 
	(200 - товар помещен в корзину, 400 - товар не может быть помещен в корзину)
```

### Оформление (финализация/бронирование) заказа (order service + sync request to warehouse service)

```jsx
REQUST:
	HTTP VERB: POST
	URL: /orders/{order_id}/bookings
	HEADERS: 
		Authorization: Bearer access_token
	PARAMETERS:
		order_id: UUID

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: BookingDto
```

### Получение возможных сейчас слотов доставки (delivery service)

```jsx
REQUEST:
	HTTP verb: GET
	URL: /delivery/slots?number={number}
	HEADERS: 
		Authorization: Bearer access_token
	PARAMETERS:
		number: Int [Positive] // number of slots we want to get

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: List<Int>
```

### Установление желаемого времени доставки (order service)

```jsx
REQUST:
	HTTP VERB: POST
	URL: /orders/{order_id}/delivery?slot={slot_in_sec}
	HEADERS: 
		Authorization: Bearer access_token
	PARAMETERS:
		order_id: UUID
		slot_in_sec: Int

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: BookingDto
```

### Оплата заказа (payment service)

```jsx
REQUST:
	HTTP VERB: POST
	URL: /orders/{order_id}/payment
	HEADERS: 
		Authorization: Bearer access_token
	PARAMETERS:
		order_id: UUID

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: PaymentSubmissionDto
```


## Непубличное API
Методы, необходимые для тестирования ваших сервисов

### Добавление товаров в каталог

```jsx
REQUEST:	
	HTTP verb: POST
	URL: /_internal/catalogItem
	BODY FORMAT:
		{
			title: String,
			description: String,
			price: Int,
			amount: Int
		}

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: CatalogItemDto
```

### Получить список забронированных товаров по bookingId
```bookingId = BookingDto.id``` получаемый при финализации заказа

```jsx
REQUEST:	
	HTTP verb: GET
	URL: /_internal/bookingHistory/{bookingId}
    HEADERS:
        Authorization: Bearer access_token
    PARAMETERS:
        bookingId: UUID

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: List<BookingLogRecord>
```

### Получить историю доставки заказа по orderId

```jsx
REQUEST:	
	HTTP verb: GET
	URL: /_internal/deliveryLog/{orderId}
    HEADERS:
        Authorization: Bearer access_token
    PARAMETERS:
        orderId: UUID

RESPONSE:
	HTTP CODES: 2** | any other
	BODY FORMAT: List<DeliveryInfoRecord>
```
