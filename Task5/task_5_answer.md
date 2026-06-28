# Задание 5. Проектирование GraphQL API для `client-info`

## Анализ текущего REST API

Существующий REST-контракт `client-info` состоит из трёх операций чтения:

| REST endpoint | Назначение | GraphQL-представление |
| --- | --- | --- |
| `GET /clients/{id}` | Получить базовые данные клиента | `client(id)` с полями `id`, `name`, `age` |
| `GET /clients/{id}/documents` | Получить документы клиента | `client(id) { documents { ... } }` |
| `GET /clients/{id}/relatives` | Получить родственников клиента | `client(id) { relatives { ... } }` |

Проблема REST-подхода в этом кейсе в том, что разные сценарии требуют разные части большой карточки клиента. Если сделать много маленьких endpoint-ов, один экран или бизнес-процесс начинает выполнять несколько запросов к `client-info`. Если сделать один большой endpoint, потребители будут получать лишние данные.

## Предлагаемая GraphQL-модель

Основная сущность схемы - `Client`. Документы и родственники являются частями клиентской карточки, поэтому они представлены вложенными полями:

- `Client.documents`;
- `Client.relatives`.

Схема вынесена в отдельный файл: `Task5/client-info.schema.graphql`.

В схеме нет `Mutation`, потому что исходный Swagger-контракт содержит только операции чтения.

## Примеры запросов

Получить только базовые данные клиента:

```graphql
query GetClient($id: ID!) {
  client(id: $id) {
    id
    name
    age
  }
}
```

Получить документы клиента:

```graphql
query GetClientDocuments($id: ID!) {
  client(id: $id) {
    id
    documents {
      id
      type
      number
      issueDate
      expiryDate
    }
  }
}
```

Получить родственников клиента:

```graphql
query GetClientRelatives($id: ID!) {
  client(id: $id) {
    id
    relatives {
      id
      relationType
      name
      age
    }
  }
}
```

Комбинированный сценарий, который в REST потребовал бы несколько запросов:

```graphql
query GetClientForPolicySale($id: ID!) {
  client(id: $id) {
    id
    name
    documents {
      id
      type
      number
    }
    relatives {
      id
      relationType
      name
    }
  }
}
```
