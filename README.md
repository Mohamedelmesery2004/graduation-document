# 5.3 Backend Implementation

## 5.3.1 Architecture Overview & Flow

The backend system implements a **Layered Architecture** with Model-View-Controller (MVC) principles, designed to achieve scalability, maintainability, and modularity. This architectural approach separates concerns into distinct layers, enabling independent development, testing, and maintenance of each component. The system is built using Node.js with the Express framework, leveraging MongoDB as the primary database through the Mongoose Object Data Modeling (ODM) library.

The architecture follows a clear request lifecycle: **Client → Controller → Application Layer → Domain Layer → Infrastructure Layer → Database → Response**. This flow ensures that each layer has a specific responsibility, promoting code reusability and reducing coupling between components. The choice of this architecture is driven by the need for a multi-tenant customer support platform that requires real-time communication, complex business logic, and integration with external AI services.

The system employs a **Service Layer Pattern** to encapsulate business logic, keeping controllers thin and focused on HTTP request handling. The **Repository Pattern** is implemented to abstract data access operations, providing a clean interface between the application logic and the database. This design facilitates easier testing and potential database migration in the future.

Key technologies utilized in the backend include:
- **Express.js**: Web framework for building RESTful APIs
- **MongoDB with Mongoose**: NoSQL database and ODM for data persistence
- **Socket.IO**: Real-time bidirectional event-based communication
- **JWT (JSON Web Tokens)**: Stateless authentication mechanism
- **Joi**: Schema validation library for request validation
- **Google Gemini AI**: AI model for generating intelligent responses
- **HuggingFace**: Embedding generation for semantic search
- **ElevenLabs**: Text-to-speech conversion for voice capabilities
- **Multer**: File upload handling middleware

## 5.3.1.1 Controllers / Entry Points

The system comprises 21 controllers organized by domain functionality, each responsible for handling specific business operations.

### Authentication Domain

**AuthController** (`authController.js`)

**Responsibility**: Handles user registration, login, and profile retrieval

**Main Endpoints**: POST `/api/v1/auth/register`, POST `/api/v1/auth/login`, GET `/api/v1/auth/me`, GET `/api/v1/auth/companies`

**Code Snippet**:

```javascript
class AuthController extends BaseController {
  login = this.catchAsync(async (req, res) => {
    const data = await authService.login(req.body);
    this.sendSuccess(res, data, 'Login successful');
  });

  getMe = this.catchAsync(async (req, res) => {
    const user = await authService.getMe(req.userId);
    this.sendSuccess(res, { user });
  });
}
```

**Explanation**: The AuthController extends BaseController to inherit common response methods. The login method delegates to authService for credential verification and token generation. The getMe method retrieves the current user's profile using the userId extracted from the JWT token by authentication middleware. This controller demonstrates the thin controller pattern, where HTTP-specific concerns are handled while business logic is delegated to services.

**Interaction**: Delegates to `authService` for business logic, communicates with `userRepo` and `companyRepo` for data access.

### User Management Domain

**AdminUserController** (`adminUserController.js`)

**Responsibility**: Manages user CRUD operations for administrators

**Main Endpoints**: POST `/api/v1/admin/users`, GET `/api/v1/admin/users`, GET `/api/v1/admin/users/:id`, PUT `/api/v1/admin/users/:id`, DELETE `/api/v1/admin/users/:id`

**Code Snippet**:

```javascript
class AdminUserController extends BaseController {
  createUser = this.catchAsync(async (req, res) => {
    const { name, email, password, phone, role, profileImage, teamLeaderId } = req.body;

    if (req.userRole === ROLES.TEAM_LEADER && role === ROLES.COMPANY_MANAGER) {
      throw ApiError.forbidden('Team leaders cannot create company managers');
    }

    const existing = await userRepo.findOne({ companyId: req.companyId, email });
    if (existing) {
      throw ApiError.conflict('User with this email already exists in this company');
    }

    const user = await userRepo.create({
      companyId: req.companyId,
      name,
      email,
      passwordHash: password,
      phone: phone || null,
      role,
      profileImage: profileImage || null,
    });

    await recordAudit({
      companyId: req.companyId,
      actor: req.user,
      action: 'user.created',
      resourceType: 'user',
      targetId: user._id,
      details: { email: user.email, role: user.role },
    });

    this.sendSuccess(res, { user: user.toJSON() }, 'User created successfully', 201);
  });
}
```

**Explanation**: This method demonstrates role-based validation before user creation, ensuring team leaders cannot create company managers. It checks for duplicate emails within the company scope, creates the user through the repository, and records an audit log for compliance. The password is stored as-is (hashing occurs in the User model pre-save hook). This shows how controllers enforce business rules at the entry point while delegating persistence to repositories.

**AgentController** (`agentController.js`)

**Responsibility**: Handles agent-specific operations including login, profile management, and ticket interactions

**Main Endpoints**: POST `/api/v1/agent/login`, GET `/api/v1/agent/profile`, PUT `/api/v1/agent/profile`, POST `/api/v1/agent/tickets/:ticketId/claim`

**Code Snippet**:

```javascript
class AgentController extends BaseController {
  claimTicket = this.catchAsync(async (req, res) => {
    const ticket = await agentTicketService.claimTicket(req.companyId, req.params.ticketId, req.userId);

    if (ticket.context?.sessionId) {
      const session = await chatSessionRepo.findOne({
        companyId: req.companyId,
        sessionId: ticket.context.sessionId,
      });

      if (session) {
        const customer = await userRepo.model.findById(session.userId);
        const company = await companyRepo.model.findById(req.companyId);

        if (session.channel === CHANNELS.TELEGRAM && customer?.telegramChatId) {
          const botToken = company?.channelsConfig?.telegram?.botToken;
          if (botToken) {
            try {
              await axios.post(`https://api.telegram.org/bot${botToken}/sendMessage`, {
                chat_id: customer.telegramChatId,
                text: `An agent (*${req.user.name}*) has picked up your ticket *#${ticket.ticketNumber}* and will assist you shortly. Please stay connected!`,
                parse_mode: 'Markdown',
              });
            } catch (err) {
              console.error('Telegram claim notify error:', err.response?.data || err.message);
            }
          }
        }
      }
    }

    try {
      const io = getIO();
      io.of('/admin').to(`company:${req.companyId}`).emit('ticket:assigned', {
        ticketId: ticket._id,
        ticketNumber: ticket.ticketNumber,
        agentId: req.userId,
        agentName: req.user.name,
      });
    } catch (err) {
      console.error('Socket emit error:', err.message);
    }

    this.sendSuccess(res, { ticket }, 'Ticket claimed');
  });
}
```

**Explanation**: This method demonstrates cross-channel notification logic. After claiming a ticket through the service, it checks if the ticket has an associated chat session. If the session is on Telegram, it retrieves the customer's Telegram chat ID and the company's bot token, then sends a notification via the Telegram API. Simultaneously, it emits a Socket.IO event to the admin namespace to notify other agents in real-time. This shows how controllers coordinate between multiple external services (Telegram API, Socket.IO) while maintaining the core business logic in services.

**Interaction**: Delegates to `agentTicketService`, `agentProfileService`, and `agentDashboardService`.

### Chat & Communication Domain

**ChatController** (`chatController.js`)

**Responsibility**: Manages chat sessions, message processing, and real-time communication

**Main Endpoints**: POST `/api/v1/chat/sessions`, POST `/api/v1/chat/sessions/:sessionId/messages`, POST `/api/v1/chat/sessions/:sessionId/media`, GET `/api/v1/chat/sessions/my`

**Code Snippet**:

```javascript
class ChatController extends BaseController {
  sendMessage = this.catchAsync(async (req, res) => {
    const { sessionId } = req.params;
    const { content } = req.body;

    const result = await messageProcessor.processMessage(req.companyId, sessionId, content, 'web');

    try {
      const io = getIO();

      io.of('/webchat').to(`session:${sessionId}`).emit('chat:message', {
        sessionId,
        message: {
          role: 'assistant',
          content: result.aiResponse.answer,
          timestamp: new Date(),
        },
      });

      if (result.ticket) {
        io.of('/admin').to(`company:${req.companyId}`).emit('ticket:new', {
          ticket: result.ticket,
          sessionId,
        });
      }

      io.of('/admin').to(`company:${req.companyId}`).emit('chat:sessionUpdated', {
        sessionId,
        status: result.session.status,
        messageCount: result.session.messageCount,
        lastActivity: result.session.lastActivity,
      });
    } catch (socketErr) {
      console.error('Socket emit error:', socketErr.message);
    }

    this.sendSuccess(res, {
      message: {
        role: 'assistant',
        content: result.aiResponse.answer,
      },
      intent: result.aiResponse.detectedIntent,
      confidence: result.aiResponse.confidence,
      escalated: result.escalated,
      ticketNumber: result.ticket?.ticketNumber || null,
    });
  });
}
```

**Explanation**: This method demonstrates real-time event broadcasting through Socket.IO. After processing the message through the messageProcessor (which handles AI response generation and potential ticket creation), it emits events to multiple Socket.IO namespaces. The webchat namespace receives the AI response for the customer, the admin namespace receives ticket creation notifications if a ticket was created, and session updates are broadcast to keep admin dashboards synchronized. The try-catch around Socket.IO operations ensures that HTTP responses are not affected by WebSocket failures. This pattern demonstrates how the system maintains consistency between HTTP responses and real-time notifications.

**Interaction**: Delegates to `chatSessionManager` and `messageProcessor`, emits Socket.IO events for real-time updates.

### Ticket Management Domain

**AdminTicketController** (`adminTicketController.js`)

**Responsibility**: Provides administrative ticket management capabilities

**Main Endpoints**: GET `/api/v1/admin/tickets`, PUT `/api/v1/admin/tickets/:ticketId`, DELETE `/api/v1/admin/tickets/:ticketId`

**Code Snippet**:

```javascript
class AdminTicketController extends BaseController {
  updateTicket = this.catchAsync(async (req, res) => {
    const ticket = await ticketService.updateTicket(
      req.companyId,
      req.params.ticketId,
      req.body,
      req.userId
    );

    try {
      const io = getIO();
      io.of('/admin').to(`company:${req.companyId}`).emit('ticket:updated', {
        ticketId: ticket._id,
        ticketNumber: ticket.ticketNumber,
        status: ticket.status,
        assignedTo: ticket.assignedTo,
      });
    } catch (err) {
      console.error('Socket emit error:', err.message);
    }

    await recordAudit({
      companyId: req.companyId,
      actor: req.user,
      action: 'ticket.updated',
      resourceType: 'ticket',
      targetId: ticket._id,
      details: {
        ticketNumber: ticket.ticketNumber,
        patchKeys: Object.keys(req.body || {}),
      },
    });

    this.sendSuccess(res, { ticket }, 'Ticket updated');
  });
}
```

**Explanation**: This method demonstrates the pattern of service delegation followed by cross-cutting concerns. The ticket update is performed by the service layer, which handles business logic and state transitions. After the update, the controller emits a Socket.IO event to notify all connected admin clients of the change. It then records an audit log entry capturing the actor, action, and modified fields. This separation ensures that the service focuses on business rules while the controller handles HTTP-specific concerns like real-time notifications and audit logging.

**Interaction**: Uses `ticketService` for comprehensive ticket administration.

### Knowledge Base Domain

**KnowledgeController** (`knowledgeController.js`)

**Responsibility**: Manages knowledge base items for AI-powered responses

**Main Endpoints**: POST `/api/v1/admin/knowledge`, GET `/api/v1/admin/knowledge`, GET `/api/v1/admin/knowledge/:id`, PUT `/api/v1/admin/knowledge/:id`, DELETE `/api/v1/admin/knowledge/:id`

**Code Snippet**:

```javascript
class KnowledgeController extends BaseController {
  createKnowledgeItem = this.catchAsync(async (req, res) => {
    const { type, title, subtitle, content, features, slug, isActive } = req.body;

    const itemSlug = slug || slugify(title, { lower: true, strict: true });

    const existing = await KnowledgeItem.findOne({ companyId: req.companyId, slug: itemSlug });
    if (existing) {
      throw ApiError.conflict('Knowledge item with this slug already exists');
    }

    const item = await KnowledgeItem.create({
      companyId: req.companyId,
      type,
      title,
      subtitle: subtitle || null,
      content,
      features: features || [],
      slug: itemSlug,
      isActive: isActive !== undefined ? isActive : true,
    });

    await recordAudit({
      companyId: req.companyId,
      actor: req.user,
      action: 'knowledge.created',
      resourceType: 'knowledge_item',
      targetId: item._id,
      details: { title: item.title, slug: item.slug, type: item.type },
    });

    this.sendSuccess(res, { item }, 'Knowledge item created', 201);
  });
}
```

**Explanation**: This controller method demonstrates direct model manipulation for simple CRUD operations. It generates a slug from the title if not provided, checks for duplicates within the company scope, creates the knowledge item directly using the Mongoose model, and records an audit log. Unlike other controllers that delegate to services, this controller handles the creation logic directly because the operation is straightforward and does not require complex business rules. This shows pragmatic decision-making in architecture—using services when complexity warrants it, but keeping simple operations in controllers.

**Interaction**: Directly manipulates `KnowledgeItem` model, integrates with audit logging.

## 5.3.1.2 Layered Architecture Breakdown

### API Layer

The API Layer serves as the entry point for all HTTP requests and is responsible for request handling, response formatting, and middleware orchestration.

**Responsibilities**: HTTP request/response handling, middleware composition, routing, validation, authentication, authorization, file upload handling, response formatting.

**Real Components**: Controllers (21 files), Routes (18 files), Middleware (5 files), app.js (application configuration).

**Code Snippet - Route Definition**:

```javascript
import { Router } from 'express';
import agentController from '../controllers/agentController.js';
import { protect, tenantIsolation, allowRoles } from '../middlewares/authMiddleware.js';
import validate from '../middlewares/validateMiddleware.js';
import * as agentValidator from '../validators/agentValidator.js';
import { ROLES } from '../constants/index.js';

const router = Router();

router.post('/auth/login', validate(agentValidator.agentLogin), agentController.login);

router.use(protect, tenantIsolation, allowRoles(ROLES.AGENT));

router.get('/profile', agentController.getProfile);
router.post('/tickets/:ticketId/claim', validate(agentValidator.ticketIdParam), agentController.claimTicket);
router.post('/tickets/:ticketId/reply', validate(agentValidator.agentReply), agentController.replyToTicket);

export default router;
```

**Explanation**: This route definition demonstrates middleware composition and route organization. The login endpoint is public and only requires validation. All subsequent routes use three middleware functions: protect (JWT authentication), tenantIsolation (multi-tenancy enforcement), and allowRoles (role-based authorization). This composition ensures that all protected routes are authenticated, scoped to the correct company, and restricted to agents only. Validation middleware is applied per-endpoint for specific request validation. This pattern shows how the API layer handles cross-cutting concerns declaratively through middleware composition.

**Code Snippet - Middleware**:

```javascript
const protect = asyncHandler(async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    throw ApiError.unauthorized('Access denied. No token provided.');
  }

  const decoded = jwt.verify(token, config.jwt.secret);

  const user = await User.findById(decoded.id).select('-passwordHash');
  if (!user) {
    throw ApiError.unauthorized('User belonging to this token no longer exists.');
  }

  if (!user.isActive) {
    throw ApiError.unauthorized('User account is deactivated.');
  }

  req.user = user;
  req.userId = user._id;
  req.companyId = user.companyId;
  req.userRole = user.role;

  next();
});
```

**Explanation**: The protect middleware implements JWT-based authentication. It extracts the Bearer token from the Authorization header, verifies it using the JWT secret, retrieves the associated user from the database, checks if the user is active, and attaches user context to the request object. This middleware enables subsequent middleware and controllers to access req.user, req.userId, req.companyId, and req.userRole without additional database queries. The use of asyncHandler ensures that any errors are caught and passed to the error handling middleware. This pattern centralizes authentication logic and provides consistent user context throughout the request lifecycle.

### Application Layer

The Application Layer encapsulates business logic and use cases, providing a clean separation between HTTP handling and domain operations.

**Responsibilities**: Business logic implementation, use case orchestration, external service integration, transaction coordination, domain rule enforcement.

**Real Components**: Services (22 files organized in directories), Validators (13 files), DTOs (implicit through Joi schemas).

**Code Snippet - Service Layer**:

```javascript
class ChatSessionManager {
  generateSessionId() {
    const id = uuidv4().replace(/-/g, '').toUpperCase();
    return `CHAT-${id.substring(0, 4)}-${id.substring(4, 8)}-${id.substring(8, 12)}`;
  }

  async createSession(companyId, userId, channel = 'web') {
    const MAX_RETRIES = 3;
    let lastError;

    for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
      const sessionId = this.generateSessionId();
      try {
        const session = await ChatSession.create({
          companyId,
          sessionId,
          userId,
          channel,
          status: CHAT_STATUS.ACTIVE,
          messages: [],
          messageCount: 0,
          startedAt: new Date(),
          lastActivity: new Date(),
        });

        await logEvent({
          companyId,
          eventType: EVENT_TYPES.CHAT_SESSION_CREATED,
          entityType: 'chat_session',
          entityId: session._id,
          metadata: { channel },
        });

        return session;
      } catch (err) {
        if (err.code === 11000 && attempt < MAX_RETRIES) {
          console.warn(`[ChatSession] sessionId collision on attempt ${attempt}, retrying...`);
          lastError = err;
          continue;
        }
        throw err;
      }
    }

    throw lastError;
  }
}
```

**Explanation**: This service demonstrates business logic that goes beyond simple CRUD operations. The generateSessionId method creates human-readable session IDs using UUID generation with formatting. The createSession method implements retry logic for handling duplicate key errors (session ID collisions), which is a business concern rather than a data access concern. It also logs events for analytics and audit purposes. This service encapsulates the complete use case of session creation including ID generation, collision handling, persistence, and event logging. Controllers can call this method without needing to understand these details, demonstrating the separation of concerns achieved by the service layer.

**Code Snippet - Validation**:

```javascript
const register = {
  body: Joi.object({
    companySlug: Joi.string().required().trim().lowercase(),
    name: Joi.string().required().trim().min(2).max(100),
    email: Joi.string().required().email().trim().lowercase(),
    password: Joi.string().required().min(6).max(128),
    phone: Joi.string().trim().allow(null, ''),
  }).options({ stripUnknown: true, abortEarly: false }),
};
```

**Explanation**: This Joi schema defines validation rules for user registration. Each field has specific constraints: companySlug and email are required and converted to lowercase, name has length constraints, email must be a valid email format, password has minimum and maximum length requirements, and phone is optional. The stripUnknown option removes any fields not defined in the schema, preventing mass assignment attacks. The abortEarly: false option collects all validation errors rather than failing on the first error, providing comprehensive feedback to the client. This declarative validation approach ensures data integrity at the API boundary before business logic execution.

### Domain Layer

The Domain Layer represents the core business entities and their relationships, encapsulated through Mongoose models.

**Responsibilities**: Entity definition, data integrity constraints, business rule enforcement at the entity level, relationship modeling, domain-specific behaviors.

**Real Components**: Models (14 files), Constants (enumerations and RBAC matrix), Domain-specific methods on models.

**Code Snippet - Domain Model**:

```javascript
const knowledgeItemSchema = new mongoose.Schema(
  {
    companyId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Company',
      required: [true, 'Company ID is required'],
      index: true,
    },
    type: {
      type: String,
      enum: Object.values(KNOWLEDGE_TYPE),
      required: [true, 'Knowledge type is required'],
    },
    title: {
      type: String,
      required: [true, 'Title is required'],
      trim: true,
      maxlength: 200,
    },
    content: {
      type: String,
      required: [true, 'Content is required'],
    },
    slug: {
      type: String,
      required: true,
      lowercase: true,
      trim: true,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
    embeddingVector: {
      type: [Number],
      default: [],
    },
    embeddingModel: {
      type: String,
      default: null,
    },
  },
  {
    timestamps: true,
  }
);

knowledgeItemSchema.index({ companyId: 1, slug: 1 }, { unique: true });
knowledgeItemSchema.index({ companyId: 1, isActive: 1 });
knowledgeItemSchema.index({ companyId: 1, type: 1 });
```

**Explanation**: This schema demonstrates domain modeling with Mongoose. The companyId field establishes the multi-tenant relationship with required validation and indexing. The type field uses an enum to restrict values to predefined knowledge types. The title field has length constraints and automatic trimming. The embeddingVector field stores vector data as an array of numbers for semantic search capabilities. The indexes at the bottom optimize query performance: a unique compound index on companyId and slug prevents duplicate slugs within a company, and single-field indexes support common query patterns. This schema enforces data integrity at the database level while supporting the application's semantic search functionality.

**Code Snippet - Domain Behavior**:

```javascript
userSchema.pre('save', async function () {
  if (!this.isModified('passwordHash')) return;
  this.passwordHash = await bcrypt.hash(this.passwordHash, 12);
});

userSchema.methods.comparePassword = async function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.passwordHash);
};
```

**Explanation**: This code demonstrates domain behavior encapsulation in the User model. The pre-save hook automatically hashes passwords before saving, but only if the password field has been modified. This ensures that passwords are always hashed regardless of how the user is created or updated, preventing plain text password storage. The comparePassword method provides a secure way to verify passwords by using bcrypt's comparison function, which is resistant to timing attacks. These behaviors are encapsulated within the model, ensuring consistent password handling throughout the application and preventing developers from accidentally storing plain text passwords or using insecure comparison methods.

### Infrastructure Layer

The Infrastructure Layer handles external concerns including database access, external service integrations, and file system operations.

**Responsibilities**: Database connection management, data access abstraction, external API integration, file system operations, configuration management, logging infrastructure.

**Real Components**: Repositories (base and specialized), Database configuration, External service utilities (AI, embeddings, TTS), File upload middleware, Configuration management.

**Code Snippet - Repository Pattern**:

```javascript
class BaseRepository {
  constructor(model) {
    this.model = model;
  }

  async create(data) {
    return await this.model.create(data);
  }

  async findById(id, select = '') {
    let query = this.model.findById(id);
    if (select) query = query.select(select);
    return await query.exec();
  }

  async findOne(filter, select = '') {
    let query = this.model.findOne(filter);
    if (select) query = query.select(select);
    return await query.exec();
  }

  async findWithPagination(filter = {}, page = 1, limit = 10, options = {}) {
    const skip = (page - 1) * limit;
    const [data, total] = await Promise.all([
      this.find(filter, { ...options, skip, limit }),
      this.count(filter),
    ]);
    return {
      data,
      total,
      page: Number(page),
      limit: Number(limit),
      pages: Math.ceil(total / limit),
    };
  }
}
```

**Explanation**: The BaseRepository implements the Repository Pattern, providing a common interface for data access operations. It encapsulates Mongoose query building, allowing the application layer to work with domain objects rather than database-specific queries. The findWithPagination method demonstrates optimization by executing the data query and count query in parallel using Promise.all, reducing total query time. This abstraction layer enables easy testing (repositories can be mocked) and potential database migration (the repository interface remains consistent even if the underlying database changes). The constructor injection of the Mongoose model follows dependency injection principles, making the repositories flexible and testable.

**Code Snippet - External Service Integration**:

```javascript
export const generateEmbedding = async (text) => {
  if (!config.huggingface.apiToken) {
    throw new Error('HuggingFace API token is not configured');
  }

  const response = await axios.post(
    HF_API_URL,
    { inputs: text },
    {
      headers: {
        Authorization: `Bearer ${config.huggingface.apiToken}`,
        'Content-Type': 'application/json',
      },
      timeout: 30000,
    }
  );

  return response.data;
};
```

**Explanation**: This utility function demonstrates external service integration for generating text embeddings using the HuggingFace API. It checks for required configuration before making the request, uses axios for HTTP communication with proper headers and timeout configuration, and returns the embedding vector. This encapsulation keeps the application logic decoupled from the specific API implementation—if the embedding provider changes, only this utility function needs to be modified. The timeout configuration prevents the application from hanging on slow API responses. This pattern of encapsulating external service calls in utility functions or services is repeated throughout the application for AI services, Telegram API, and other external dependencies.

## 5.3.2 Architectural Patterns

### Repository Pattern

**Purpose**: The Repository Pattern abstracts data access logic, providing a collection-like interface for querying domain objects. It separates business logic from data access logic, making the system more testable and maintainable.

**Implementation in This Project**:

```javascript
class BaseRepository {
  constructor(model) {
    this.model = model;
  }

  async create(data) {
    return await this.model.create(data);
  }

  async findOne(filter, select = '') {
    let query = this.model.findOne(filter);
    if (select) query = query.select(select);
    return await query.exec();
  }

  async findWithPagination(filter = {}, page = 1, limit = 10, options = {}) {
    const skip = (page - 1) * limit;
    const [data, total] = await Promise.all([
      this.find(filter, { ...options, skip, limit }),
      this.count(filter),
    ]);
    return {
      data,
      total,
      page: Number(page),
      limit: Number(limit),
      pages: Math.ceil(total / limit),
    };
  }
}

class UserRepository extends BaseRepository {
  async findByEmail(email, companyId) {
    return await this.findOne({ email, companyId });
  }
}

export const userRepo = new UserRepository(User);
```

**Explanation**: The BaseRepository provides common CRUD operations that work with any Mongoose model. Domain-specific repositories like UserRepository extend the base to add domain-specific methods like findByEmail. Repositories are instantiated as singletons and exported from a central index file. Services use these repositories without knowing the underlying database implementation, enabling easy mocking in tests and potential database migration. The findWithPagination method demonstrates optimization by parallelizing data and count queries.

### Service Layer Pattern

**Purpose**: The Service Layer Pattern encapsulates business logic and coordinates operations between repositories and external services. It keeps controllers thin and focused on HTTP concerns while services handle complex business rules.

**Implementation in This Project**:

```javascript
class AuthService {
  async register({ companySlug, name, email, password, phone }) {
    const company = await companyRepo.findOne({ slug: companySlug, isActive: true });
    if (!company) {
      throw ApiError.notFound('Company not found or inactive');
    }

    const existingUser = await userRepo.findOne({ companyId: company._id, email });
    if (existingUser) {
      throw ApiError.conflict('User with this email already exists in this company');
    }

    const user = await userRepo.create({
      companyId: company._id,
      name,
      email,
      passwordHash: password,
      phone: phone || null,
      role: ROLES.CUSTOMER,
    });

    const token = generateToken(user);
    return { user: user.toJSON(), token };
  }

  async login({ email, password, companySlug }) {
    const company = await companyRepo.findOne({ slug: companySlug });
    if (!company) {
      throw ApiError.notFound('Company not found');
    }

    const user = await userRepo.findOne({ companyId: company._id, email });
    if (!user) {
      throw ApiError.unauthorized('Invalid email or password');
    }

    if (!user.isActive) {
      throw ApiError.unauthorized('Account is deactivated');
    }

    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      throw ApiError.unauthorized('Invalid email or password');
    }

    user.lastLogin = new Date();
    await user.save();

    const token = generateToken(user);
    return { user: user.toJSON(), token };
  }
}

export default new AuthService();
```

**Explanation**: The AuthService implements the complete use cases for user registration and login. It coordinates multiple repository calls, enforces business rules (company must be active, user must not already exist, account must be active), and integrates with authentication utilities for token generation. The service handles all error cases with appropriate API errors. Controllers simply call these methods and format the response. This separation allows the same business logic to be reused across different interfaces (HTTP, WebSocket, CLI) and makes unit testing easier by isolating business logic from HTTP concerns.

### Middleware Pipeline Pattern

**Purpose**: Express middleware pipeline processes requests sequentially, with each middleware having the opportunity to modify the request or response object or terminate the request chain. This pattern promotes code reuse and separation of concerns.

**Implementation in This Project**:

```javascript
app.use(helmet());
app.use(cors({ origin: config.cors.origin, credentials: true }));

const limiter = rateLimit({
  windowMs: config.rateLimit.windowMs,
  max: config.rateLimit.max,
  message: { success: false, message: 'Too many requests, please try again later.' },
  validate: { xForwardedForHeader: false },
  skip: () => config.env === 'development',
});
app.use('/api/', limiter);

app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

app.use(compression());

if (config.env === 'development') {
  app.use(morgan('dev'));
} else {
  app.use(morgan('combined'));
}

app.use('/api/v1/auth', authRoutes);
app.use('/api/v1/chat', chatRoutes);
app.use('/api/v1/agent', agentRoutes);

app.use(notFound);
app.use(errorHandler);
```

**Explanation**: The middleware pipeline in app.js demonstrates the composition of cross-cutting concerns. Security middleware (Helmet, CORS) is applied first.Rate limiting is applied to all API routes. Body parsing middleware handles JSON and URL-encoded data. Compression reduces response sizes. Logging middleware provides request/response logging. Route-specific middleware is applied per route module. Finally, error handling middleware catches all errors. This ordered execution ensures that each concern is addressed at the appropriate point in the request lifecycle, with security and validation occurring before business logic, and error handling occurring as a fallback.

### Singleton Pattern

**Purpose**: The Singleton Pattern ensures a single instance of a class exists throughout the application lifecycle, ensuring consistent state management and resource sharing.

**Implementation in This Project**:

```javascript
class AuthService {
  async register({ companySlug, name, email, password, phone }) {
    // Service logic
  }
}

export default new AuthService();
```

**Explanation**: Services are implemented as classes but exported as instances using `export default new AuthService()`. This ensures that all imports of the service receive the same instance, which is important for services that might cache data or maintain connections. For example, if a service cached frequently accessed data, having multiple instances would lead to cache inconsistency. The singleton pattern also reduces memory overhead by avoiding multiple object instantiations. This pattern is used consistently across all services and repositories in the application.

### Role-Based Access Control (RBAC)

**Purpose**: RBAC provides centralized, declarative authorization based on user roles, defining what each role can do on each resource.

**Implementation in This Project**:

```javascript
export const RBAC_MATRIX = {
  [ROLES.PLATFORM_SUPER_ADMIN]: {
    [RESOURCES.COMPANIES]: [ACTIONS.CREATE, ACTIONS.READ, ACTIONS.UPDATE, ACTIONS.DELETE, ACTIONS.MANAGE],
    [RESOURCES.USERS]: [ACTIONS.CREATE, ACTIONS.READ, ACTIONS.UPDATE, ACTIONS.DELETE, ACTIONS.MANAGE],
    [RESOURCES.KNOWLEDGE_BASE]: [ACTIONS.CREATE, ACTIONS.READ, ACTIONS.UPDATE, ACTIONS.DELETE, ACTIONS.MANAGE],
    [RESOURCES.CHAT]: [ACTIONS.READ, ACTIONS.DELETE, ACTIONS.MANAGE],
    [RESOURCES.TICKETS]: [ACTIONS.CREATE, ACTIONS.READ, ACTIONS.UPDATE, ACTIONS.DELETE, ACTIONS.MANAGE],
  },
  [ROLES.AGENT]: {
    [RESOURCES.KNOWLEDGE_BASE]: [ACTIONS.READ],
    [RESOURCES.CHAT]: [ACTIONS.READ, ACTIONS.UPDATE],
    [RESOURCES.TICKETS]: [ACTIONS.READ, ACTIONS.UPDATE],
    [RESOURCES.ANALYTICS]: [ACTIONS.READ],
  },
  [ROLES.CUSTOMER]: {
    [RESOURCES.CHAT]: [ACTIONS.CREATE, ACTIONS.READ],
    [RESOURCES.TICKETS]: [ACTIONS.READ],
  },
};

const requirePermission = (resource, action) => {
  return (req, res, next) => {
    const role = req.userRole;
    if (!role) {
      throw ApiError.unauthorized('No role found for user.');
    }

    const permissions = RBAC_MATRIX[role];
    if (!permissions) {
      throw ApiError.forbidden('No permissions defined for this role.');
    }

    const resourcePermissions = permissions[resource];
    if (!resourcePermissions || !resourcePermissions.includes(action)) {
      throw ApiError.forbidden(
        `Access denied. Role '${role}' does not have '${action}' permission on '${resource}'.`
      );
    }

    next();
  };
};
```

**Explanation**: The RBAC_MATRIX defines permissions declaratively, mapping roles to resources and actions. This matrix is easy to audit and modify without changing code. The requirePermission middleware factory creates authorization middleware that checks the matrix. This pattern provides centralized authorization logic that is consistent across all endpoints. When permissions change, only the matrix needs to be updated. The middleware provides clear error messages when authorization fails, aiding debugging. This implementation supports hierarchical permissions where higher roles have broader access, as seen in the matrix where PLATFORM_SUPER_ADMIN has MANAGE permission on all resources while AGENT has only READ and UPDATE on specific resources.

## 5.3.3 Feature-Based Implementation

### 5.3.3.1 Authentication and Authorization

**Functional Description**: The authentication and authorization feature manages user identity verification, session management, and access control based on roles and permissions. It supports multi-tenant architecture where users belong to companies and have specific roles determining their capabilities.

**Real Endpoints**:
- POST `/api/v1/auth/register` - User registration
- POST `/api/v1/auth/login` - User login
- GET `/api/v1/auth/me` - Get current user profile
- GET `/api/v1/auth/companies` - Get public companies list

**Data Flow Summary**: User credentials are validated against the database, passwords are verified using bcrypt hashing, JWT tokens are generated with user context, and subsequent requests are authenticated using the token with role-based permissions enforced.

#### 5.3.3.1.1 Flow Explanation

**1. Request Received**:

```javascript
// Route definition in authRoutes.js
router.post('/login', validate(authValidator.login), authController.login);
```

**2. Controller Logic**:

```javascript
// authController.js
login = this.catchAsync(async (req, res) => {
  const data = await authService.login(req.body);
  this.sendSuccess(res, data, 'Login successful');
});
```

**3. Service/Handler**:

```javascript
// authService.js
async login({ email, password, companySlug }) {
  const company = await companyRepo.findOne({ slug: companySlug });
  if (!company) {
    throw ApiError.notFound('Company not found');
  }

  const user = await userRepo.findOne({ companyId: company._id, email });
  if (!user) {
    throw ApiError.unauthorized('Invalid email or password');
  }

  if (!user.isActive) {
    throw ApiError.unauthorized('Account is deactivated');
  }

  const isMatch = await user.comparePassword(password);
  if (!isMatch) {
    throw ApiError.unauthorized('Invalid email or password');
  }

  user.lastLogin = new Date();
  await user.save();

  const token = generateToken(user);
  return { user: user.toJSON(), token };
}
```

**4. Business Logic**: The service performs company lookup, user lookup within the company, account status verification, password comparison using bcrypt, last login timestamp update, and JWT token generation.

**5. Database Interaction**:

```javascript
// Repository calls
const company = await companyRepo.findOne({ slug: companySlug });
const user = await userRepo.findOne({ companyId: company._id, email });
await user.save();
```

**6. Response Construction**:

```javascript
// Token generation in authMiddleware.js
const generateToken = (user) => {
  return jwt.sign(
    {
      id: user._id,
      companyId: user.companyId,
      role: user.role,
    },
    config.jwt.secret,
    { expiresIn: config.jwt.expiresIn }
  );
};

// Response from controller
this.sendSuccess(res, { user: user.toJSON(), token }, 'Login successful');
```

### 5.3.3.2 AI-Powered Chat System

**Functional Description**: The chat system provides intelligent customer support through AI-generated responses, with automatic escalation to human agents when necessary. It supports multiple channels including web chat, Telegram, and voice, with real-time messaging capabilities.

**Real Endpoints**:
- POST `/api/v1/chat/sessions` - Create new chat session
- POST `/api/v1/chat/sessions/:sessionId/messages` - Send message to session
- POST `/api/v1/chat/sessions/:sessionId/media` - Send media message
- GET `/api/v1/chat/sessions/my` - Get user's chat sessions
- POST `/api/v1/chat/sessions/:sessionId/close` - Close chat session

**Data Flow Summary**: Messages are processed through an AI pipeline that retrieves relevant knowledge base content using semantic search, generates contextual responses using Google Gemini AI, detects user intent and confidence, and automatically creates support tickets when escalation is needed.

#### 5.3.3.2.1 Flow Explanation

**1. Request Received**:

```javascript
// Route definition in chatRoutes.js
router.post(
  '/sessions/:sessionId/messages',
  requirePermission(RESOURCES.CHAT, ACTIONS.CREATE),
  validate(chatValidator.sendMessage),
  chatController.sendMessage
);
```

**2. Controller Logic**:

```javascript
// chatController.js
sendMessage = this.catchAsync(async (req, res) => {
  const { sessionId } = req.params;
  const { content } = req.body;

  const result = await messageProcessor.processMessage(req.companyId, sessionId, content, 'web');

  try {
    const io = getIO();
    io.of('/webchat').to(`session:${sessionId}`).emit('chat:message', {
      sessionId,
      message: {
        role: 'assistant',
        content: result.aiResponse.answer,
        timestamp: new Date(),
      },
    });

    if (result.ticket) {
      io.of('/admin').to(`company:${req.companyId}`).emit('ticket:new', {
        ticket: result.ticket,
        sessionId,
      });
    }
  } catch (socketErr) {
    console.error('Socket emit error:', socketErr.message);
  }

  this.sendSuccess(res, {
    message: {
      role: 'assistant',
      content: result.aiResponse.answer,
    },
    intent: result.aiResponse.detectedIntent,
    confidence: result.aiResponse.confidence,
    escalated: result.escalated,
    ticketNumber: result.ticket?.ticketNumber || null,
  });
});
```

**3. Service/Handler**:

```javascript
// messageProcessor.js
async processMessage(companyId, sessionId, userMessage, channel = 'web', media = null, skipUserMessageSave = false) {
  const session = await chatSessionRepo.findOne({ companyId, sessionId, status: CHAT_STATUS.ACTIVE });
  if (!session) {
    throw new Error('Active chat session not found');
  }

  const [company, user, userTickets] = await Promise.all([
    companyRepo.model.findById(companyId),
    userRepo.model.findById(session.userId).select('name email phone role telegramChatId createdAt'),
    ticketRepo.model.find({ companyId, userId: session.userId })
      .sort({ createdAt: -1 })
      .limit(5)
      .select('ticketNumber category priority status createdAt'),
  ]);

  const relevantKnowledge = await this.findRelevantKnowledge(companyId, userMessage);
  const knowledgeContext = relevantKnowledge.map(
    (k) => `[${k.item.type}] ${k.item.title}: ${k.item.content}`
  );

  const aiResult = await getAIResponse({
    systemPrompt,
    messages: previousMessages,
    userMessage,
    knowledgeContext,
  });

  session.messages.push({
    role: 'assistant',
    content: aiResult.answer,
    timestamp: new Date(),
    meta: {
      intent: aiResult.detectedIntent,
      confidence: aiResult.confidence,
      knowledgeUsed: relatedKnowledgeIds.length,
    },
  });

  const needsEscalation = userAsksForAgent || userAsksForTicket || aiResult.shouldEscalate;

  if (needsEscalation && canCreateTicket) {
    const ticketNumber = await this.generateTicketNumber(companyId);
    ticket = await ticketRepo.create({
      companyId,
      ticketNumber,
      userId: session.userId,
      channel,
      category: aiResult.category || 'other',
      priority: aiResult.priority || TICKET_PRIORITY.MEDIUM,
      status: TICKET_STATUS.PENDING,
      context: {
        sessionId: session.sessionId,
        lastUserMessage: userMessage,
        aiSummary: `Intent: ${aiResult.detectedIntent}. ${aiResult.answer.substring(0, 300)}`,
      },
    });
    session.summary.linkedTicketId = ticket._id;
  }

  await session.save();

  return {
    session,
    aiResponse: aiResult,
    ticket,
    knowledgeUsed: relevantKnowledge.length,
    escalated: !!ticket,
  };
}
```

**4. Business Logic**: The service retrieves the chat session, fetches related data in parallel (company, user, tickets), retrieves relevant knowledge using semantic search, constructs a comprehensive system prompt, calls the AI service for response generation, appends the AI response to the session, checks for escalation conditions, creates a ticket if needed, links the ticket to the session, and saves the session.

**5. Database Interaction**:

```javascript
// Session retrieval
const session = await chatSessionRepo.findOne({ companyId, sessionId, status: CHAT_STATUS.ACTIVE });

// Parallel data fetching
const [company, user, userTickets] = await Promise.all([
  companyRepo.model.findById(companyId),
  userRepo.model.findById(session.userId).select('name email phone role telegramChatId createdAt'),
  ticketRepo.model.find({ companyId, userId: session.userId })
    .sort({ createdAt: -1 })
    .limit(5)
    .select('ticketNumber category priority status createdAt'),
]);

// Knowledge retrieval
const items = await KnowledgeItem.find({
  companyId,
  isActive: true,
  embeddingVector: { $exists: true, $ne: [] },
});

// Ticket creation
ticket = await ticketRepo.create({
  companyId,
  ticketNumber,
  userId: session.userId,
  channel,
  category: aiResult.category || 'other',
  priority: aiResult.priority || TICKET_PRIORITY.MEDIUM,
  status: TICKET_STATUS.PENDING,
  context: {
    sessionId: session.sessionId,
    lastUserMessage: userMessage,
    aiSummary: `Intent: ${aiResult.detectedIntent}. ${aiResult.answer.substring(0, 300)}`,
  },
});

// Session update
await session.save();
```

**6. Response Construction**: The controller receives the processed result, emits Socket.IO events for real-time updates to both webchat and admin namespaces, and sends an HTTP response with the AI response, detected intent, confidence score, escalation status, and ticket number if created.

### 5.3.3.3 Ticket Management System

**Functional Description**: The ticket management system provides structured support request tracking with assignment workflows, status management, priority categorization, and integration with chat sessions for context preservation.

**Real Endpoints**:
- GET `/api/v1/tickets` - Get customer's tickets (paginated)
- POST `/api/v1/tickets/:ticketId/reply` - Customer reply to ticket
- POST `/api/v1/tickets/:ticketId/feedback` - Submit feedback on resolved ticket
- GET `/api/v1/admin/tickets` - Get all tickets with filters
- PUT `/api/v1/admin/tickets/:ticketId` - Update ticket details

**Data Flow Summary**: Tickets are created automatically by the AI system or manually by agents. They flow through a lifecycle from pending to opened to closed, with assignment to agents, status updates, and feedback collection.

#### 5.3.3.3.1 Flow Explanation

**1. Request Received**:

```javascript
// Route definition in agentRoutes.js
router.post('/tickets/:ticketId/claim', validate(agentValidator.ticketIdParam), agentController.claimTicket);
```

**2. Controller Logic**:

```javascript
// agentController.js
claimTicket = this.catchAsync(async (req, res) => {
  const ticket = await agentTicketService.claimTicket(req.companyId, req.params.ticketId, req.userId);

  try {
    const io = getIO();
    io.of('/admin').to(`company:${req.companyId}`).emit('ticket:assigned', {
      ticketId: ticket._id,
      ticketNumber: ticket.ticketNumber,
      agentId: req.userId,
      agentName: req.user.name,
    });
  } catch (err) {
    console.error('Socket emit error:', err.message);
  }

  this.sendSuccess(res, { ticket }, 'Ticket claimed');
});
```

**3. Service/Handler**:

```javascript
// agentTicketService.js
async claimTicket(companyId, ticketId, agentId) {
  const ticket = await Ticket.findOneAndUpdate(
    {
      _id: ticketId,
      companyId,
      assignedTo: null,
      status: TICKET_STATUS.PENDING,
    },
    {
      $set: {
        assignedTo: agentId,
        status: TICKET_STATUS.OPENED,
      },
    },
    { new: true }
  )
    .populate('userId', 'name email phone')
    .populate('assignedTo', 'name email');

  if (!ticket) {
    const existing = await ticketRepo.findOne({ _id: ticketId, companyId });
    if (!existing) throw ApiError.notFound('Ticket not found');
    if (existing.assignedTo) throw ApiError.conflict('Ticket is already assigned to another agent');
    if (existing.status !== TICKET_STATUS.PENDING) throw ApiError.conflict(`Ticket status is '${existing.status}', cannot claim`);
    throw ApiError.conflict('Ticket cannot be claimed');
  }

  await logEvent({
    companyId,
    eventType: EVENT_TYPES.TICKET_CLAIMED,
    entityType: 'ticket',
    entityId: ticket._id,
    metadata: { agentId, ticketNumber: ticket.ticketNumber },
  });

  return ticket;
}
```

**4. Business Logic**: The service performs an atomic update operation to claim the ticket, ensuring race conditions are prevented through MongoDB's atomic findOneAndUpdate. It verifies the ticket exists, is unassigned, and has pending status. It logs the claim event for analytics. The controller then emits Socket.IO events to notify other agents.

**5. Database Interaction**:

```javascript
// Atomic ticket update
const ticket = await Ticket.findOneAndUpdate(
  {
    _id: ticketId,
    companyId,
    assignedTo: null,
    status: TICKET_STATUS.PENDING,
  },
  {
    $set: {
      assignedTo: agentId,
      status: TICKET_STATUS.OPENED,
    },
  },
  { new: true }
)
  .populate('userId', 'name email phone')
  .populate('assignedTo', 'name email');

// Event logging
await logEvent({
  companyId,
  eventType: EVENT_TYPES.TICKET_CLAIMED,
  entityType: 'ticket',
  entityId: ticket._id,
  metadata: { agentId, ticketNumber: ticket.ticketNumber },
});
```

**6. Response Construction**: The controller emits a Socket.IO event to the admin namespace to notify all connected agents of the assignment, then sends an HTTP response with the claimed ticket details including populated user and agent information.

## 5.3.4 Cross-Cutting Concerns

### Authentication and Authorization

The system implements JWT-based stateless authentication with role-based authorization.

**Code Snippet - Authentication Middleware**:

```javascript
const protect = asyncHandler(async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    throw ApiError.unauthorized('Access denied. No token provided.');
  }

  const decoded = jwt.verify(token, config.jwt.secret);

  const user = await User.findById(decoded.id).select('-passwordHash');
  if (!user) {
    throw ApiError.unauthorized('User belonging to this token no longer exists.');
  }

  if (!user.isActive) {
    throw ApiError.unauthorized('User account is deactivated.');
  }

  req.user = user;
  req.userId = user._id;
  req.companyId = user.companyId;
  req.userRole = user.role;

  next();
});
```

**Explanation**: The protect middleware extracts and verifies JWT tokens, retrieves the associated user, checks account status, and attaches user context to the request. This middleware is applied to protected routes to ensure only authenticated requests proceed.

### Validation Strategy

Request validation is implemented using Joi schemas defined in the validators directory.

**Code Snippet - Validation Middleware**:

```javascript
const validate = (schema) => {
  return (req, res, next) => {
    const errors = [];

    ['body', 'query', 'params'].forEach((key) => {
      if (schema[key]) {
        const { error, value } = schema[key].validate(req[key], {
          abortEarly: false,
          stripUnknown: true,
        });
        if (error) {
          error.details.forEach((detail) => {
            errors.push({
              field: detail.path.join('.'),
              message: detail.message,
            });
          });
        } else if (key === 'body') {
          req.body = value;
        } else {
          Object.keys(req[key]).forEach((k) => {
            if (!(k in value)) delete req[key][k];
          });
          Object.assign(req[key], value);
        }
      }
    });

    if (errors.length > 0) {
      throw ApiError.badRequest('Validation failed', errors);
    }

    next();
  };
};
```

**Explanation**: The validate middleware applies Joi schemas to request bodies, query parameters, and route parameters. It collects all validation errors rather than failing on the first error, strips unknown fields to prevent mass assignment attacks, and provides detailed error messages. This ensures data integrity at the API boundary.

### Exception Handling

Centralized error handling is implemented through the error middleware.

**Code Snippet - Error Middleware**:

```javascript
const errorHandler = (err, req, res, _next) => {
  let statusCode = err.statusCode || 500;
  let message = err.message || 'Internal server error';
  let errors = err.errors || [];

  if (err.name === 'ValidationError') {
    statusCode = 400;
    message = 'Validation failed';
    errors = Object.values(err.errors).map((e) => ({
      field: e.path,
      message: e.message,
    }));
  }

  if (err.code === 11000) {
    statusCode = 409;
    const field = Object.keys(err.keyValue)[0];
    message = `Duplicate value for field: ${field}`;
  }

  if (err.name === 'JsonWebTokenError') {
    statusCode = 401;
    message = 'Invalid token';
  }
  if (err.name === 'TokenExpiredError') {
    statusCode = 401;
    message = 'Token expired';
  }

  if (config.env === 'development') {
    console.error('Error:', {
      message: err.message,
      stack: err.stack,
      statusCode,
    });
  }

  res.status(statusCode).json({
    success: false,
    message,
    ...(errors.length > 0 && { errors }),
    ...(config.env === 'development' && { stack: err.stack }),
  });
};
```

**Explanation**: The errorHandler middleware processes all errors, converting them to consistent JSON responses. It handles specific error types including Mongoose validation errors, duplicate key errors, and JWT errors. In development mode, it includes stack traces for debugging. This centralized error handling ensures consistent error responses across the application.

## 5.3.5 Performance and Scalability Considerations

### Async Operations

The system extensively uses asynchronous operations throughout the stack to ensure non-blocking I/O.

**Code Snippet - Parallel Operations**:

```javascript
const [company, user, userTickets] = await Promise.all([
  companyRepo.model.findById(companyId),
  userRepo.model.findById(session.userId).select('name email phone role telegramChatId createdAt'),
  ticketRepo.model.find({ companyId, userId: session.userId })
    .sort({ createdAt: -1 })
    .limit(5)
    .select('ticketNumber category priority status createdAt'),
]);
```

**Explanation**: Using Promise.all for parallel database queries reduces total response time by executing independent queries concurrently rather than sequentially. This pattern is used throughout the application when multiple independent data fetches are needed.

### Database Optimizations

The system employs comprehensive indexing for query optimization.

**Code Snippet - Database Indexes**:

```javascript
ticketSchema.index({ companyId: 1, ticketNumber: 1 }, { unique: true });
ticketSchema.index({ companyId: 1, status: 1, createdAt: -1 });
ticketSchema.index({ companyId: 1, userId: 1 });
ticketSchema.index({ companyId: 1, assignedTo: 1 });
ticketSchema.index({ companyId: 1, assignedTo: 1, status: 1, createdAt: -1 });
```

**Explanation**: Compound indexes optimize common query patterns. The unique index on companyId and ticketNumber prevents duplicate ticket numbers within a company. The compound index on companyId, status, and createdAt optimizes queries for pending tickets sorted by creation time. These indexes significantly improve query performance for the most common access patterns.

### Pagination

Pagination is implemented consistently across list endpoints.

**Code Snippet - Pagination Implementation**:

```javascript
async findWithPagination(filter = {}, page = 1, limit = 10, options = {}) {
  const skip = (page - 1) * limit;
  const [data, total] = await Promise.all([
    this.find(filter, { ...options, skip, limit }),
    this.count(filter),
  ]);
  return {
    data,
    total,
    pages: Math.ceil(total / limit),
  };
}
```

**Explanation**: The findWithPagination method executes data and count queries in parallel using Promise.all, reducing total query time. It returns both the paginated data and metadata including total count and total pages, enabling clients to implement pagination controls. This pattern is reused across all list endpoints.

## 5.3.6 Testing and Quality Assurance

The current implementation does not include automated tests. The package.json includes a placeholder test script that returns an error indicating no tests are specified. This represents an area for improvement in the codebase.

Quality assurance is primarily achieved through:
- Type checking via Joi validation schemas
- Manual testing during development
- Code review processes
- Runtime error handling and logging

The system would benefit from implementing:
- Unit tests for services and repositories
- Integration tests for API endpoints
- End-to-end tests for critical user flows
- Load testing for performance validation
- Contract testing for API stability

## 5.3.7 Deployment and Environment

### Environment Variables

The system uses environment variables for configuration, managed through the dotenv package.

**Code Snippet - Configuration Management**:

```javascript
dotenv.config({ path: join(process.cwd(), '.env') });

const config = {
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT, 10) || 3000,
  mongo: {
    uri: process.env.MONGODB_URI,
  },
  jwt: {
    secret: process.env.JWT_SECRET || 'default-secret-change-me',
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  },
  gemini: {
    apiKey: process.env.GEMINI_API_KEY,
  },
  cors: {
    origin: process.env.CORS_ORIGIN || '*',
  },
  rateLimit: {
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS, 10) || 15 * 60 * 1000,
    max: parseInt(process.env.RATE_LIMIT_MAX, 10) || 100,
  },
};
```

**Explanation**: Configuration is centralized in config/index.js, loading variables from the .env file and providing default values where appropriate. This approach enables environment-specific configurations without code changes, supporting development, staging, and production environments.

### Deployment Configuration

The server implementation includes graceful shutdown handling.

**Code Snippet - Graceful Shutdown**:

```javascript
const startServer = async () => {
  await connectDB();

  const server = http.createServer(app);
  initializeSocket(server);

  server.listen(config.port, () => {
    console.log(`Server running on http://localhost:${config.port}`);
  });

  const gracefulShutdown = (signal) => {
    console.log(`\n${signal} received. Shutting down gracefully...`);
    server.close(() => {
      console.log('HTTP server closed.');
      process.exit(0);
    });
  };

  process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
  process.on('SIGINT', () => gracefulShutdown('SIGINT'));
};
```

**Explanation**: The server listens for SIGTERM and SIGINT signals to perform graceful shutdown. This ensures in-flight requests complete and connections close properly during deployment or restarts, preventing data loss and client errors.

## 5.3.8 Conclusion

The backend implementation demonstrates a well-architected system designed for scalability, maintainability, and robustness. The layered architecture with clear separation of concerns enables independent development and testing of components. The Repository Pattern and Service Layer Pattern provide clean abstractions that facilitate future modifications and testing.

The system successfully integrates multiple complex features including AI-powered chat, multi-channel communication, ticket management, knowledge base with semantic search, and real-time communication through Socket.IO. The comprehensive RBAC system ensures secure access control across different user roles and company contexts.

Strengths of the backend implementation include:
- **Modularity**: Clear separation between API, application, domain, and infrastructure layers
- **Scalability**: Async operations, database indexing, and pagination support horizontal scaling
- **Maintainability**: Consistent patterns across controllers, services, and repositories
- **Security**: JWT authentication, multi-tenant isolation, RBAC authorization, and input validation
- **Real-time Capabilities**: Socket.IO integration for live updates across multiple namespaces
- **AI Integration**: Sophisticated AI pipeline with knowledge retrieval and context-aware responses
- **Multi-channel Support**: Unified architecture supporting web, Telegram, and voice channels

Areas for future enhancement include:
- Implementation of comprehensive automated testing
- Addition of caching layer (Redis) for performance optimization
- Expansion of monitoring and observability features
- Implementation of API versioning strategy
- Addition of request tracing and distributed logging
- Enhanced error recovery and retry mechanisms for external service calls

The backend provides a solid foundation for a multi-tenant AI-powered customer support platform, with architecture decisions that support current requirements while allowing for future growth and feature expansion.
