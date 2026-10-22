# GymPass API - SOLID Architecture

> API estilo GymPass para gerenciamento de academias, check-ins e usuários, construída seguindo princípios SOLID com injeção de dependências.

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Tecnologias](#tecnologias)
- [Arquitetura](#arquitetura)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Autenticação JWT & Refresh Token](#autenticação-jwt--refresh-token)
- [Sistema de Roles (user.role)](#sistema-de-roles-userrole)
- [Injeção de Dependências](#injeção-de-dependências)
- [Casos de Uso (Use Cases)](#casos-de-uso-use-cases)
- [Repositories & Data Access](#repositories--data-access)
- [Controllers & Rotas](#controllers--rotas)
- [Tratamento de Erros](#tratamento-de-erros)
- [Testes](#testes)
- [Configuração e Execução](#configuração-e-execução)

---

## Visão Geral

Esta é uma API RESTful desenvolvida em **Node.js + TypeScript + Fastify** que implementa um sistema estilo GymPass, permitindo:

- 👤 **Cadastro e autenticação de usuários** (MEMBER/ADMIN)
- 🏋️ **Gerenciamento de academias** (criação, busca, listagem por proximidade)
- ✅ **Check-ins em academias** (com validação de distância e limite diário)
- 📊 **Métricas e histórico** de check-ins do usuário
- 🔐 **Autenticação stateless com JWT + Refresh Token** (cookies HttpOnly)

O projeto segue **Clean Architecture** com separação clara de responsabilidades:
- **Domain/Use Cases**: Regras de negócio puras
- **Data/Repositories**: Abstração de acesso a dados
- **Presentation/Controllers**: Camada HTTP
- **Factories**: Injeção de dependências manual

---

## Tecnologias

| Categoria | Tecnologia | Versão |
|-----------|------------|--------|
| **Runtime** | Node.js | 18+ |
| **Language** | TypeScript | 4.9.5 |
| **Framework HTTP** | Fastify | 4.13.0 |
| **Database** | PostgreSQL + Prisma ORM | 4.10.1 |
| **Validação** | Zod | 3.20.6 |
| **Auth** | @fastify/jwt + @fastify/cookie | 6.7.0 / 8.3.0 |
| **Password Hash** | bcryptjs | 2.4.3 |
| **Date Handling** | dayjs | 1.11.7 |
| **Testes** | Vitest | 0.28.5 |
| **Lint/Format** | ESLint + Rocketseat Config | 8.34.0 |
| **Build** | tsup | 6.6.3 |
| **Dev Runtime** | tsx | 3.12.3 |

### Dependências Principais

```json
{
  "@fastify/cookie": "8.3.0",
  "@fastify/jwt": "6.7.0",
  "@prisma/client": "4.10.1",
  "bcryptjs": "2.4.3",
  "dayjs": "1.11.7",
  "dotenv": "16.0.3",
  "fastify": "4.13.0",
  "zod": "3.20.6"
}
```

---

## Arquitetura

```
src/
├── @types/                 # Extensões de tipos (ex: fastify-jwt)
├── env/                    # Configuração de variáveis de ambiente (Zod)
├── http/
│   ├── controllers/        # Controllers organizados por recurso
│   │   ├── users/          # register, authenticate, refresh, profile
│   │   ├── gyms/           # create, search, nearby
│   │   └── check-ins/      # create, validate, history, metrics
│   └── middlewares/        # verify-jwt, verify-user-role
├── lib/                    # Instâncias compartilhadas (Prisma Client)
├── repositories/           # Interfaces + Implementações
│   ├── prisma/             # Implementações Prisma
│   └── in-memory/          # Implementações em memória (testes)
├── use-cases/              # Regras de negócio (casos de uso)
│   ├── errors/             # Custom errors de domínio
│   └── factories/          # Factories para injeção de dependências
└── utils/                  # Utilitários (ex: distância entre coordenadas)
```

### Princípios SOLID Aplicados

| Princípio | Implementação |
|-----------|---------------|
| **S** - Single Responsibility | Cada use-case tem uma única responsabilidade |
| **O** - Open/Closed | Repositories via interfaces; novos repositórios não quebram use-cases |
| **L** - Liskov Substitution | `InMemoryUsersRepository` e `PrismaUsersRepository` implementam `UsersRepository` |
| **I** - Interface Segregation | Interfaces específicas por repositório (`UsersRepository`, `GymsRepository`, etc.) |
| **D** - Dependency Inversion | Use-cases dependem de interfaces, não de implementações concretas |

---

## Estrutura do Projeto

### Models (Prisma Schema)

```prisma
enum Role {
  ADMIN
  MEMBER
}

model User {
  id            String   @id @default(uuid())
  name          String
  email         String   @unique
  password_hash String
  role          Role     @default(MEMBER)
  created_at    DateTime @default(now())
  checkIns      CheckIn[]
}

model Gym {
  id          String   @id @default(uuid())
  title       String
  description String?
  phone       String?
  latitude    Decimal
  longitude   Decimal
  checkIns    CheckIn[]
}

model CheckIn {
  id           String    @id @default(uuid())
  created_at   DateTime  @default(now())
  validated_at DateTime?
  user         User      @relation(fields: [user_id], references: [id])
  user_id      String
  gym          Gym       @relation(fields: [gym_id], references: [id])
  gym_id       String
}
```

---

## Autenticação JWT & Refresh Token

### Configuração (app.ts)

```typescript
app.register(fastifyJwt, {
  secret: env.JWT_SECRET,
  cookie: {
    cookieName: 'refreshToken',
    signed: false,
  },
  sign: {
    expiresIn: '10m',        // Access token: 10 minutos
  },
})
```

### Fluxo de Autenticação

```
┌─────────────┐     POST /sessions      ┌──────────────────┐
│   Client    │ ──────────────────────► │  Authenticate    │
│             │ ◄────────────────────── │  Use Case        │
└─────────────┘   { token, refreshToken } └──────────────────┘
       │                                    │
       │  Access Token (JWT)                │  Refresh Token (HttpOnly Cookie)
       │  - Expires: 10min                  │  - Expires: 7 dias
       │  - Payload: { role, sub }          │  - Payload: { role, sub }
       │  - Enviado no header Authorization │  - Armazenado em cookie seguro
       ▼                                    ▼
```

### Login (authenticate.ts)

```typescript
// Gera Access Token (curta duração)
const token = await reply.jwtSign(
  { role: user.role },
  { sign: { sub: user.id } }
)

// Gera Refresh Token (longa duração, HttpOnly cookie)
const refreshToken = await reply.jwtSign(
  { role: user.role },
  { sign: { sub: user.id, expiresIn: '7d' } }
)

return reply
  .setCookie('refreshToken', refreshToken, {
    path: '/',
    secure: true,       // Apenas HTTPS
    sameSite: true,     // CSRF protection
    httpOnly: true,     // Inacessível via JavaScript
  })
  .status(200)
  .send({ token })
```

### Refresh Token (refresh.ts)

```typescript
// Verifica apenas o cookie (refreshToken)
await request.jwtVerify({ onlyCookie: true })

// Gera novos tokens
const token = await reply.jwtSign({ role }, { sign: { sub: request.user.sub } })
const refreshToken = await reply.jwtSign(
  { role },
  { sign: { sub: request.user.sub, expiresIn: '7d' } }
)

return reply
  .setCookie('refreshToken', refreshToken, { /* same options */ })
  .status(200)
  .send({ token })
```

### Por que Access Token curto + Refresh Token longo?

| Token | Duração | Armazenamento | Uso |
|-------|---------|---------------|-----|
| **Access Token** | 10 min | Memory (header) | Autorização em cada request |
| **Refresh Token** | 7 dias | HttpOnly Cookie | Renovar access token sem re-login |

**Vantagens:**
- ✅ Access token roubado expira rápido (janela de ataque pequena)
- ✅ Refresh token em HttpOnly cookie protege contra XSS
- ✅ Usuário não precisa fazer login frequente
- ✅ Possível revogar refresh token no servidor (logout real)

---

## Sistema de Roles (user.role)

### Definição no JWT (fastify-jwt.d.ts)

```typescript
declare module '@fastify/jwt' {
  export interface FastifyJWT {
    user: {
      role: 'ADMIN' | 'MEMBER'
      sub: string
    }
  }
}
```

### Middleware de Verificação de Role (verify-user-role.ts)

```typescript
export function verifyUserRole(roleToVerify: 'ADMIN' | 'MEMBER') {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const { role } = request.user  // Vem do JWT validado

    if (role !== roleToVerify) {
      return reply.status(401).send({ message: 'Unauthorized.' })
    }
  }
}
```

### Uso nas Rotas (gyms/routes.ts)

```typescript
export async function gymsRoutes(app: FastifyInstance) {
  // Todas as rotas de gyms exigem autenticação
  app.addHook('onRequest', verifyJwt)

  app.get('/gyms/search', search)
  app.get('/gyms/nearby', nearby)

  // Apenas ADMIN pode criar academias
  app.post('/gyms', { onRequest: [verifyUserRole('ADMIN')] }, create)
}
```

### Uso nas Rotas (check-ins/routes.ts)

```typescript
export async function checkInsRoutes(app: FastifyInstance) {
  app.addHook('onRequest', verifyJwt)

  app.get('/check-ins/history', history)
  app.get('/check-ins/metrics', metrics)
  app.post('/gyms/:gymId/check-ins', create)

  // Apenas ADMIN pode validar check-ins
  app.patch(
    '/check-ins/:checkInId/validate',
    { onRequest: [verifyUserRole('ADMIN')] },
    validate
  )
}
```

### Matriz de Permissões

| Endpoint | METHOD | Role Necessária |
|----------|--------|-----------------|
| `/users` | POST | Pública |
| `/sessions` | POST | Pública |
| `/token/refresh` | PATCH | Refresh Token válido |
| `/me` | GET | Autenticado (qualquer role) |
| `/gyms/search` | GET | Autenticado |
| `/gyms/nearby` | GET | Autenticado |
| `/gyms` | POST | **ADMIN** |
| `/gyms/:gymId/check-ins` | POST | Autenticado |
| `/check-ins/history` | GET | Autenticado |
| `/check-ins/metrics` | GET | Autenticado |
| `/check-ins/:checkInId/validate` | PATCH | **ADMIN** |

---

## Injeção de Dependências

Este projeto usa **injeção de dependências manual via Factories** (sem container IoC), seguindo o princípio da **Inversão de Dependência (DIP)**.

### Camadas da Injeção

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTROLLER (HTTP)                        │
│  - Recebe request/response                                  │
│  - Valida input (Zod)                                       │
│  - Chama Factory → UseCase                                  │
│  - Formata response HTTP                                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      FACTORY                                │
│  - Instancia Repository concreto (Prisma)                   │
│  - Injeta no UseCase                                        │
│  - Retorna UseCase pronto para uso                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      USE CASE                               │
│  - Regras de negócio puras                                  │
│  - Depende de INTERFACE (abstração)                         │
│  - Não conhece Prisma, HTTP, etc.                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   REPOSITORY (INTERFACE)                    │
│  - Define contrato: findById, create, etc.                  │
│  - Implementado por Prisma ou InMemory                      │
└─────────────────────────────────────────────────────────────┘
```

### Exemplo Completo: Autenticação

#### 1. Interface do Repository (`repositories/users-repository.ts`)

```typescript
import { Prisma, User } from '@prisma/client'

export interface UsersRepository {
  findById(id: string): Promise<User | null>
  findByEmail(email: string): Promise<User | null>
  create(data: Prisma.UserCreateInput): Promise<User>
}
```

#### 2. Implementação Prisma (`repositories/prisma/prisma-users-repository.ts`)

```typescript
import { prisma } from '@/lib/prisma'
import { Prisma } from '@prisma/client'
import { UsersRepository } from '../users-repository'

export class PrismaUsersRepository implements UsersRepository {
  async findById(id: string) {
    return prisma.user.findUnique({ where: { id } })
  }

  async findByEmail(email: string) {
    return prisma.user.findUnique({ where: { email } })
  }

  async create(data: Prisma.UserCreateInput) {
    return prisma.user.create({ data })
  }
}
```

#### 3. Implementação In-Memory (`repositories/in-memory/in-memory-users-repository.ts`)

```typescript
import { UsersRepository } from '@/repositories/users-repository'
import { User, Prisma } from '@prisma/client'
import { randomUUID } from 'node:crypto'

export class InMemoryUsersRepository implements UsersRepository {
  public items: User[] = []

  async findById(id: string) {
    return this.items.find(item => item.id === id) ?? null
  }

  async findByEmail(email: string) {
    return this.items.find(item => item.email === email) ?? null
  }

  async create(data: Prisma.UserCreateInput) {
    const user = {
      id: randomUUID(),
      name: data.name,
      email: data.email,
      password_hash: data.password_hash,
      created_at: new Date(),
    }
    this.items.push(user)
    return user
  }
}
```

#### 4. Use Case (`use-cases/authenticate.ts`)

```typescript
import { UsersRepository } from '@/repositories/users-repository'
import { InvalidCredentialsError } from '@/use-cases/errors/invalid-credentials-error'
import { User } from '@prisma/client'
import { compare } from 'bcryptjs'

interface AuthenticateUseCaseRequest {
  email: string
  password: string
}

interface AuthenticateUseCaseResponse {
  user: User
}

export class AuthenticateUseCase {
  // Depende da INTERFACE, não da implementação
  constructor(private usersRepository: UsersRepository) {}

  async execute({
    email,
    password,
  }: AuthenticateUseCaseRequest): Promise<AuthenticateUseCaseResponse> {
    const user = await this.usersRepository.findByEmail(email)

    if (!user) throw new InvalidCredentialsError()

    const passwordMatches = await compare(password, user.password_hash)
    if (!passwordMatches) throw new InvalidCredentialsError()

    return { user }
  }
}
```

#### 5. Factory (`use-cases/factories/make-authenticate-use-case.ts`)

```typescript
import { PrismaUsersRepository } from '@/repositories/prisma/prisma-users-repository'
import { AuthenticateUseCase } from '../authenticate'

export function makeAuthenticateUseCase() {
  const usersRepository = new PrismaUsersRepository()  // Implementação concreta
  const authenticateUseCase = new AuthenticateUseCase(usersRepository)
  return authenticateUseCase
}
```

#### 6. Controller (`http/controllers/users/authenticate.ts`)

```typescript
import { FastifyReply, FastifyRequest } from 'fastify'
import { z } from 'zod'
import { InvalidCredentialsError } from '@/use-cases/errors/invalid-credentials-error'
import { makeAuthenticateUseCase } from '@/use-cases/factories/make-authenticate-use-case'

export async function authenticate(request: FastifyRequest, reply: FastifyReply) {
  const authenticateBodySchema = z.object({
    email: z.string().email(),
    password: z.string().min(6),
  })

  const { email, password } = authenticateBodySchema.parse(request.body)

  try {
    // Usa Factory para obter UseCase com dependências resolvidas
    const authenticateUseCase = makeAuthenticateUseCase()

    const { user } = await authenticateUseCase.execute({ email, password })

    // Gera tokens JWT
    const token = await reply.jwtSign({ role: user.role }, { sign: { sub: user.id } })
    const refreshToken = await reply.jwtSign(
      { role: user.role },
      { sign: { sub: user.id, expiresIn: '7d' } }
    )

    return reply
      .setCookie('refreshToken', refreshToken, { /* options */ })
      .status(200)
      .send({ token })
  } catch (err) {
    if (err instanceof InvalidCredentialsError) {
      return reply.status(400).send({ message: err.message })
    }
    throw err
  }
}
```

### Exemplo com Múltiplas Dependências: Check-in

#### Use Case (`use-cases/check-in.ts`)

```typescript
import { CheckInsRepository } from '@/repositories/check-ins-repository'
import { GymsRepository } from '@/repositories/gyms-repository'
import { MaxDistanceError } from '@/use-cases/errors/max-distance-error'
import { MaxNumberOfCheckInsError } from '@/use-cases/errors/max-number-of-check-ins-error'
import { ResourceNotFoundError } from '@/use-cases/errors/resource-not-found-error'
import { getDistanceBetweenCoordinates } from '@/utils/get-distance-between-coordinates'
import { CheckIn } from '@prisma/client'

interface CheckInUseCaseRequest {
  userId: string
  gymId: string
  userLatitude: number
  userLongitude: number
}

interface CheckInUseCaseResponse {
  checkIn: CheckIn
}

export class CheckInUseCase {
  constructor(
    private checkInsRepository: CheckInsRepository,  // Interface
    private gymsRepository: GymsRepository,          // Interface
  ) {}

  async execute({
    userId,
    gymId,
    userLatitude,
    userLongitude,
  }: CheckInUseCaseRequest): Promise<CheckInUseCaseResponse> {
    const gym = await this.gymsRepository.findById(gymId)
    if (!gym) throw new ResourceNotFoundError()

    const distance = getDistanceBetweenCoordinates(
      { latitude: userLatitude, longitude: userLongitude },
      { latitude: gym.latitude.toNumber(), longitude: gym.longitude.toNumber() }
    )

    if (distance > 0.1) throw new MaxDistanceError()  // 100m

    const checkInOnSameDay = await this.checkInsRepository.findByUserIdOnDate(userId, new Date())
    if (checkInOnSameDay) throw new MaxNumberOfCheckInsError()

    const checkIn = await this.checkInsRepository.create({
      gym_id: gymId,
      user_id: userId,
    })

    return { checkIn }
  }
}
```

#### Factory (`use-cases/factories/make-check-in-use-case.ts`)

```typescript
import { PrismaCheckInsRepository } from '@/repositories/prisma/prisma-check-ins-repository'
import { PrismaGymsRepository } from '@/repositories/prisma/prisma-gyms-repository'
import { CheckInUseCase } from '../check-in'

export function makeCheckInUseCase() {
  const checkInsRepository = new PrismaCheckInsRepository()
  const gymsRepository = new PrismaGymsRepository()
  const useCase = new CheckInUseCase(checkInsRepository, gymsRepository)
  return useCase
}
```

#### Controller (`http/controllers/check-ins/create.ts`)

```typescript
import { FastifyReply, FastifyRequest } from 'fastify'
import { z } from 'zod'
import { MaxDistanceError } from '@/use-cases/errors/max-distance-error'
import { MaxNumberOfCheckInsError } from '@/use-cases/errors/max-number-of-check-ins-error'
import { ResourceNotFoundError } from '@/use-cases/errors/resource-not-found-error'
import { makeCheckInUseCase } from '@/use-cases/factories/make-check-in-use-case'

export async function create(request: FastifyRequest, reply: FastifyReply) {
  const createCheckInParamsSchema = z.object({
    gymId: z.string().uuid(),
  })

  const createCheckInBodySchema = z.object({
    latitude: z.number().refine(v => Math.abs(v) <= 90),
    longitude: z.number().refine(v => Math.abs(v) <= 180),
  })

  const { gymId } = createCheckInParamsSchema.parse(request.params)
  const { latitude, longitude } = createCheckInBodySchema.parse(request.body)

  try {
    const checkInUseCase = makeCheckInUseCase()

    const { checkIn } = await checkInUseCase.execute({
      gymId,
      userId: request.user.sub,  // Vem do JWT (verifyJwt middleware)
      userLatitude: latitude,
      userLongitude: longitude,
    })

    return reply.status(201).send({ checkIn })
  } catch (err) {
    if (err instanceof MaxDistanceError) {
      return reply.status(400).send({ message: err.message })
    }
    if (err instanceof MaxNumberOfCheckInsError) {
      return reply.status(400).send({ message: err.message })
    }
    if (err instanceof ResourceNotFoundError) {
      return reply.status(404).send({ message: err.message })
    }
    throw err
  }
}
```

### Benefícios desta Abordagem

| Benefício | Descrição |
|-----------|-----------|
| **Testabilidade** | Use-cases testados com `InMemoryRepository` (rápido, sem DB) |
| **Desacoplamento** | Use-case não sabe que existe Prisma ou Fastify |
| **Flexibilidade** | Trocar Prisma por outro ORM = nova implementação da interface |
| **Single Responsibility** | Cada classe tem um propósito claro |
| **Substituição Liskov** | `InMemoryUsersRepository` substitui `PrismaUsersRepository` transparentemente |

---

## Casos de Uso (Use Cases)

| Use Case | Arquivo | Descrição |
|----------|---------|-----------|
| **Register** | `register.ts` | Cadastro de usuário com hash de senha |
| **Authenticate** | `authenticate.ts` | Login com validação de credenciais |
| **GetUserProfile** | `get-user-profile.ts` | Busca perfil do usuário logado |
| **GetUserMetrics** | `get-user-metrics.ts` | Contagem total de check-ins do usuário |
| **FetchUserCheckInsHistory** | `fetch-user-check-ins-history.ts` | Histórico paginado de check-ins |
| **CheckIn** | `check-in.ts` | Realiza check-in (valida distância + limite diário) |
| **ValidateCheckIn** | `validate-check-in.ts` | Admin valida check-in (marca `validated_at`) |
| **FetchNearbyGyms** | `fetch-nearby-gyms.ts` | Academias próximas (coordenadas + Haversine) |
| **SearchGyms** | `search-gyms.ts` | Busca por título com paginação |
| **CreateGym** | `create-gym.ts` | Cria academia (apenas ADMIN) |

### Padrão de Use Case

```typescript
// 1. Interface de Request/Response (tipagem forte)
interface XxxUseCaseRequest { ... }
interface XxxUseCaseResponse { ... }

// 2. Classe com injeção de dependência via construtor
export class XxxUseCase {
  constructor(private repository: RepositoryInterface) {}

  async execute(request: XxxUseCaseRequest): Promise<XxxUseCaseResponse> {
    // Regras de negócio puras
    // Lança custom errors de domínio
  }
}

// 3. Custom Errors (domain errors)
export class BusinessRuleError extends Error {
  constructor() { super('Mensagem amigável') }
}
```

---

## Repositories & Data Access

### Interfaces (Contratos)

```typescript
// users-repository.ts
export interface UsersRepository {
  findById(id: string): Promise<User | null>
  findByEmail(email: string): Promise<User | null>
  create(data: Prisma.UserCreateInput): Promise<User>
}

// gyms-repository.ts
export interface GymsRepository {
  findById(id: string): Promise<Gym | null>
  findManyNearby(params: { latitude: number; longitude: number }): Promise<Gym[]>
  searchMany(query: string, page: number): Promise<Gym[]>
  create(data: Prisma.GymCreateInput): Promise<Gym>
}

// check-ins-repository.ts
export interface CheckInsRepository {
  findById(id: string): Promise<CheckIn | null>
  create(data: Prisma.CheckInUncheckedCreateInput): Promise<CheckIn>
  findManyByUserId(userId: string, page: number): Promise<CheckIn[]>
  countByUserId(userId: string): Promise<number>
  findByUserIdOnDate(userId: string, date: Date): Promise<CheckIn | null>
  save(checkIn: CheckIn): Promise<CheckIn>
}
```

### Implementações

| Repository | Prisma | In-Memory |
|------------|--------|-----------|
| Users | `PrismaUsersRepository` | `InMemoryUsersRepository` |
| Gyms | `PrismaGymsRepository` | `InMemoryGymsRepository` |
| CheckIns | `PrismaCheckInsRepository` | `InMemoryCheckInsRepository` |

---

## Controllers & Rotas

### Padrão de Controller

```typescript
export async function xxx(request: FastifyRequest, reply: FastifyReply) {
  // 1. Validação de input (Zod)
  const schema = z.object({ ... })
  const data = schema.parse(request.body)

  try {
    // 2. Obtém UseCase via Factory
    const useCase = makeXxxUseCase()

    // 3. Executa UseCase
    const result = await useCase.execute({ ... })

    // 4. Retorna response HTTP
    return reply.status(201).send(result)
  } catch (err) {
    // 5. Tratamento de erros de domínio
    if (err instanceof DomainError) {
      return reply.status(400).send({ message: err.message })
    }
    throw err  // Erros inesperados → global error handler
  }
}
```

### Registro de Rotas

```typescript
// users/routes.ts
export async function usersRoutes(app: FastifyInstance) {
  app.post('/users', register)           // Público
  app.post('/sessions', authenticate)    // Público
  app.patch('/token/refresh', refresh)   // Refresh token
  app.get('/me', { onRequest: [verifyJwt] }, profile)  // Protegido
}

// gyms/routes.ts
export async function gymsRoutes(app: FastifyInstance) {
  app.addHook('onRequest', verifyJwt)    // Todas protegidas

  app.get('/gyms/search', search)
  app.get('/gyms/nearby', nearby)
  app.post('/gyms', { onRequest: [verifyUserRole('ADMIN')] }, create)
}
```

---

## Tratamento de Erros

### Custom Errors de Domínio (`use-cases/errors/`)

| Error | HTTP Status | Uso |
|-------|-------------|-----|
| `InvalidCredentialsError` | 400 | Email/senha inválidos |
| `UserAlreadyExistsError` | 409 | Email já cadastrado |
| `ResourceNotFoundError` | 404 | Academia/check-in não encontrado |
| `MaxDistanceError` | 400 | Usuário > 100m da academia |
| `MaxNumberOfCheckInsError` | 400 | Já fez check-in hoje |
| `LateCheckInValidationError` | 400 | Check-in > 20 min para validar |

### Global Error Handler (`app.ts`)

```typescript
app.setErrorHandler((error, _, reply) => {
  // Erros de validação Zod
  if (error instanceof ZodError) {
    return reply.status(400).send({
      message: 'Validation error.',
      issues: error.format()
    })
  }

  // Log em dev, enviar para Sentry/DataDog em prod
  if (env.NODE_ENV !== 'production') {
    console.error(error)
  }

  return reply.status(500).send({ message: 'Internal server error.' })
})
```

---

## Testes

### Estrutura de Testes

```
src/
├── use-cases/
│   ├── *.spec.ts           # Unit tests (use-cases + in-memory repos)
└── http/
    └── controllers/
        └── **/*.spec.ts    # E2E tests (HTTP + Prisma test DB)
```

### Configuração Vitest

```json
{
  "test": "vitest run --dir src/use-cases",
  "test:e2e": "vitest run --dir src/http",
  "test:coverage": "vitest run --coverage"
}
```

### Exemplo: Unit Test (Use Case + InMemory)

```typescript
// authenticate.spec.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { InMemoryUsersRepository } from '@/repositories/in-memory/in-memory-users-repository'
import { AuthenticateUseCase } from './authenticate'
import { hash } from 'bcryptjs'
import { InvalidCredentialsError } from './errors/invalid-credentials-error'

let usersRepository: InMemoryUsersRepository
let sut: AuthenticateUseCase

describe('Authenticate Use Case', () => {
  beforeEach(() => {
    usersRepository = new InMemoryUsersRepository()
    sut = new AuthenticateUseCase(usersRepository)
  })

  it('should be able to authenticate', async () => {
    await usersRepository.create({
      name: 'John Doe',
      email: 'john@example.com',
      password_hash: await hash('123456', 6),
    })

    const { user } = await sut.execute({
      email: 'john@example.com',
      password: '123456',
    })

    expect(user.id).toEqual(expect.any(String))
  })

  it('should not be able to authenticate with wrong password', async () => {
    await usersRepository.create({
      name: 'John Doe',
      email: 'john@example.com',
      password_hash: await hash('123456', 6),
    })

    await expect(() =>
      sut.execute({ email: 'john@example.com', password: 'wrong' })
    ).rejects.toBeInstanceOf(InvalidCredentialsError)
  })
})
```

### Exemplo: E2E Test (Controller + Prisma)

```typescript
// authenticate.spec.ts (em src/http/controllers/users/)
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import request from 'supertest'
import { app } from '@/app'
import { createAndAuthenticateUser } from '@/utils/test/create-and-authenticate-user'

describe('Authenticate (e2e)', () => {
  beforeAll(async () => { await app.ready() })
  afterAll(async () => { await app.close() })

  it('should be able to authenticate', async () => {
    await request(app.server).post('/users').send({
      name: 'John Doe',
      email: 'john@example.com',
      password: '123456',
    })

    const response = await request(app.server).post('/sessions').send({
      email: 'john@example.com',
      password: '123456',
    })

    expect(response.statusCode).toEqual(200)
    expect(response.body).toEqual({ token: expect.any(String) })
    expect(response.get('Set-Cookie')).toEqual(
      expect.arrayContaining([expect.stringContaining('refreshToken=')])
    )
  })
})
```

### Ambiente de Teste Prisma

- `prisma/vitest-environment-prisma/` - Custom environment para isolar DB por test file
- Cria banco temporário, roda migrations, limpa após testes

---

## Configuração e Execução

### Variáveis de Ambiente (`.env`)

```env
# Database
DATABASE_URL="postgresql://user:pass@localhost:5432/gympass?schema=public"

# JWT
JWT_SECRET="your-super-secret-key-here"

# Server
PORT=3333
NODE_ENV=dev
```

### Instalação

```bash
# Instalar dependências
npm install

# Configurar banco (criar migrações)
npx prisma migrate dev

# Gerar Prisma Client
npx prisma generate
```

### Scripts Disponíveis

```bash
# Desenvolvimento (hot reload)
npm run start:dev

# Build para produção
npm run build

# Executar build
npm start

# Testes unitários (use-cases)
npm run test

# Testes unitários em watch mode
npm run test:watch

# Testes E2E (controllers + Prisma)
npm run test:e2e

# Cobertura de testes
npm run test:coverage

# UI do Vitest
npm run test:ui
```

### Docker (Opcional)

```bash
# Subir PostgreSQL
docker-compose up -d

# Ver logs
docker-compose logs -f
```

---

## Padrões de Código

### Naming Conventions

| Tipo | Convenção | Exemplo |
|------|-----------|---------|
| Interfaces | PascalCase + sufixo `Repository` | `UsersRepository` |
| Classes (Impl) | PascalCase + prefixo `Prisma`/`InMemory` | `PrismaUsersRepository` |
| Use Cases | PascalCase + sufixo `UseCase` | `AuthenticateUseCase` |
| Factories | `make` + PascalCase + `UseCase` | `makeAuthenticateUseCase` |
| Controllers | verbo + recurso | `authenticate`, `create`, `search` |
| Errors | PascalCase + sufixo `Error` | `InvalidCredentialsError` |
| Request/Response interfaces | `XxxUseCaseRequest/Response` | `AuthenticateUseCaseRequest` |

### Imports (Path Aliases)

```typescript
// tsconfig.json -> paths: { "@/*": ["./src/*"] }
import { UsersRepository } from '@/repositories/users-repository'
import { makeAuthenticateUseCase } from '@/use-cases/factories/make-authenticate-use-case'
import { env } from '@/env'
```

---

## Segurança

- ✅ **Senhas hasheadas** com bcryptjs (cost: 6)
- ✅ **JWT assinado** com segredo forte
- ✅ **Refresh Token** em cookie HttpOnly + Secure + SameSite
- ✅ **Validação de input** com Zod em todos endpoints
- ✅ **Rate limiting** recomendado para produção (não implementado)
- ✅ **HTTPS obrigatório** em produção (`secure: true` nos cookies)

---

## Próximos Passos / Melhorias Sugeridas

- [ ] Rate limiting (ex: `@fastify/rate-limit`)
- [ ] Logs estruturados (Pino) + integração Sentry/DataDog
- [ ] Documentação OpenAPI/Swagger (`@fastify/swagger`)
- [ ] Testes de integração com TestContainers
- [ ] CI/CD pipeline (GitHub Actions já configurado)
- [ ] Paginação padronizada (cursor-based)
- [ ] Soft delete para usuários/academias
- [ ] Refresh token rotation + detecção de reuse

---

## Licença

ISC License - Projeto educacional Rocketseat Node.js

---

> **Desenvolvido seguindo Clean Architecture + SOLID + Dependency Injection manual**  
> Para fins de estudo e demonstração de boas práticas em APIs Node.js