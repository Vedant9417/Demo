# Weekend Planner — NestJS Backend Prompt

You are a senior backend engineer. Build a production-ready **NestJS REST API**
called **Weekend Planner** that acts as a validated gateway to an n8n workflow.

---

## PRODUCT GOAL

Accept structured user input, validate it, forward it to an n8n webhook for
processing, and return up to 3 weekend plans to the frontend. The NestJS
backend does NOT call any AI, scoring engine, or Places API directly —
those all live in n8n.

---

## TECH STACK

- **Runtime:** Node.js 20+
- **Framework:** NestJS (latest stable)
- **Language:** TypeScript — strict mode (`"strict": true` in tsconfig)
- **Database:** MongoDB via Mongoose (`@nestjs/mongoose`)
- **HTTP Client:** `@nestjs/axios` + `axios` (to call n8n webhook)
- **Rate Limiting:** `@nestjs/throttler`
- **Validation:** `class-validator` + `class-transformer`
- **Config:** `@nestjs/config` with `.env` file
- **Package manager:** npm

---

## ARCHITECTURE ROLE

```
Frontend
   │
   ▼
NestJS  ──── validates input, rate-limits, calls n8n
   │
   ▼
n8n Webhook  ──── scoring → AI → Places API → MongoDB save
   │
   ▼
NestJS  ──── receives response, returns to frontend
```

NestJS also exposes ONE internal endpoint that n8n calls to fetch PlaceTypes
from MongoDB. This keeps all MongoDB access centralised in NestJS.

---

## PROJECT STRUCTURE — GENERATE EXACTLY THIS

```
src/
├── main.ts
├── app.module.ts
├── config/
│   └── configuration.ts
├── place-type/
│   ├── place-type.module.ts
│   ├── place-type.service.ts
│   ├── place-type.schema.ts
│   └── place-type.seed.ts
├── plan/
│   ├── plan.module.ts
│   ├── plan.controller.ts
│   ├── plan.service.ts
│   ├── dto/
│   │   └── user-input.dto.ts
│   └── interfaces/
│       └── plan.interface.ts
└── common/
    ├── filters/
    │   └── global-exception.filter.ts
    └── interceptors/
        └── logging.interceptor.ts
```

Do NOT generate: ai/, scoring/, places-api/, or cache/ directories.
Those services do not exist in this architecture.

---

## ENVIRONMENT VARIABLES

Create `.env.example` with:

```
MONGODB_URI=mongodb://localhost:27017/weekend-planner
N8N_WEBHOOK_URL=https://your-n8n-instance.com/webhook/weekend-planner
N8N_INTERNAL_SECRET=change-me-random-string-32chars
PORT=3000
NODE_ENV=development
```

Create `src/config/configuration.ts`:

```typescript
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  mongodbUri: process.env.MONGODB_URI,
  n8nWebhookUrl: process.env.N8N_WEBHOOK_URL,
  n8nInternalSecret: process.env.N8N_INTERNAL_SECRET,
  nodeEnv: process.env.NODE_ENV || 'development',
});
```

---

## DATA MODELS

### PlaceType — Mongoose Schema (`place-type.schema.ts`)

```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type PlaceTypeDocument = PlaceType & Document;

@Schema({ collection: 'placetypes', timestamps: true })
export class PlaceType {
  @Prop({ required: true })
  name: string;

  @Prop({ required: true, lowercase: true, index: true })
  city: string;

  @Prop({ type: [String], required: true,
    enum: ['scenic', 'calm', 'active', 'social', 'food', 'ride'] })
  tags: string[];

  @Prop({ required: true, enum: ['morning', 'evening', 'anytime'] })
  best_time: string;

  @Prop({ required: true, enum: ['low', 'medium', 'high'] })
  energy_required: string;

  @Prop({ required: true, enum: ['near', 'around_city', 'outskirts'] })
  distance_bucket: string;

  @Prop({ required: true, enum: ['low', 'medium', 'high'] })
  crowd_tendency: string;

  @Prop({ required: true, enum: ['yes', 'no', 'conditional'] })
  rain_safe: string;

  @Prop({ required: true, enum: ['low', 'medium', 'high'] })
  budget_range: string;

  @Prop({ required: true, enum: ['car', 'public', 'both'] })
  transport_fit: string;

  @Prop({ required: true })
  api_search_prompt: string;
}

export const PlaceTypeSchema = SchemaFactory.createForClass(PlaceType);
```

### UserInput DTO (`user-input.dto.ts`)

```typescript
import {
  IsString, IsNotEmpty, IsIn, IsArray, ArrayMinSize,
  IsOptional, MaxLength, MinLength,
} from 'class-validator';

export class UserInputDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  city: string;

  @IsIn(['low', 'medium', 'high'])
  energy_level: string;

  @IsIn(['alone', 'partner', 'friends', 'family'])
  company: string;

  @IsIn(['low', 'medium', 'high'])
  budget_range: string;

  @IsIn(['car', 'public', 'both'])
  travel_mode: string;

  @IsIn(['near', 'around_city', 'outskirts'])
  distance_comfort: string;

  @IsArray()
  @ArrayMinSize(1)
  @IsIn(['scenic', 'calm', 'active', 'social', 'food', 'ride'], { each: true })
  selected_visual_tags: string[];

  @IsOptional()
  @IsString()
  @MaxLength(200)
  optional_vibe_text?: string;
}
```

### Plan Interface (`plan.interface.ts`)

```typescript
export interface Plan {
  plan_id: string;
  recommended: boolean;
  place_type_name: string;
  location_name: string;
  location_photos: string[];
  tags: string[];
  timeline: {
    saturday: string;
    sunday: string;
  };
  budget_range: 'low' | 'medium' | 'high';
  distance_label: string;
  transport_assumption: string;
  why_this_works: string;
}

export interface PlanResponse {
  plans: Plan[];
}
```

---

## ENDPOINTS — BUILD EXACTLY THESE TWO

### 1. POST /plan-weekend  (public — frontend calls this)

**Controller:** `PlanController`
**Method:** `planWeekend(@Body() dto: UserInputDto)`
**Flow:**
1. NestJS validates `UserInputDto` (auto via ValidationPipe)
2. Calls `PlanService.generatePlans(dto)`
3. Returns `{ plans: Plan[] }`

### 2. GET /internal/place-types  (internal — n8n calls this)

**Controller:** `PlaceTypeController` (create a simple one inside `place-type.module.ts`)
**Query param:** `city` (string, required)
**Header required:** `x-internal-secret` must match `N8N_INTERNAL_SECRET` env var
**Guard:** `InternalSecretGuard` — custom guard, returns 401 if header missing or wrong
**Method:** `getByCity(@Query('city') city: string)`
**Returns:** Array of PlaceType documents for that city (lowercase match)
**Throws:** `404` with `{ message: "No place types configured for city: {city}" }` if empty

---

## main.ts — IMPLEMENT EXACTLY

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { AppModule } from './app.module';
import { GlobalExceptionFilter } from './common/filters/global-exception.filter';
import { LoggingInterceptor } from './common/interceptors/logging.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Global validation
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,              // strip unknown fields
    forbidNonWhitelisted: true,   // throw 400 on unknown fields
    transform: true,              // auto-transform types
  }));

  // Global exception filter
  app.useGlobalFilters(new GlobalExceptionFilter());

  // Global logging interceptor
  app.useGlobalInterceptors(new LoggingInterceptor());

  const configService = app.get(ConfigService);
  const port = configService.get<number>('port');

  await app.listen(port);
  console.log(`Weekend Planner API running on port ${port}`);
}

bootstrap();
```

---

## app.module.ts — IMPLEMENT EXACTLY

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { HttpModule } from '@nestjs/axios';
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';
import configuration from './config/configuration';
import { PlaceTypeModule } from './place-type/place-type.module';
import { PlanModule } from './plan/plan.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [configuration] }),

    MongooseModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        uri: config.get<string>('mongodbUri'),
      }),
    }),

    ThrottlerModule.forRoot([{
      ttl: 3600,    // 1 hour window
      limit: 10,    // max 10 requests per IP
    }]),

    HttpModule,
    PlaceTypeModule,
    PlanModule,
  ],
  providers: [
    { provide: APP_GUARD, useClass: ThrottlerGuard },  // applies globally
  ],
})
export class AppModule {}
```

---

## PLAN MODULE

### plan.service.ts — IMPLEMENT EXACTLY

```typescript
import { Injectable, HttpException, ServiceUnavailableException } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { ConfigService } from '@nestjs/config';
import { firstValueFrom } from 'rxjs';
import { UserInputDto } from './dto/user-input.dto';
import { PlanResponse } from './interfaces/plan.interface';

@Injectable()
export class PlanService {
  private readonly n8nWebhookUrl: string;

  constructor(
    private readonly httpService: HttpService,
    private readonly configService: ConfigService,
  ) {
    this.n8nWebhookUrl = this.configService.get<string>('n8nWebhookUrl');
  }

  async generatePlans(input: UserInputDto): Promise<PlanResponse> {
    try {
      const response = await firstValueFrom(
        this.httpService.post<PlanResponse>(
          this.n8nWebhookUrl,
          input,
          {
            timeout: 45000,  // 45s — AI + Places API takes time
            headers: { 'Content-Type': 'application/json' },
          }
        )
      );
      return response.data;

    } catch (err: any) {
      // n8n returned a structured error (4xx/5xx from n8n Error Response node)
      if (err.response?.data) {
        throw new HttpException(
          err.response.data,
          err.response.status,
        );
      }
      // Network/timeout failure — n8n unreachable
      throw new ServiceUnavailableException(
        'Plan generation service is currently unavailable. Please try again shortly.',
      );
    }
  }
}
```

### plan.controller.ts

```typescript
import { Controller, Post, Body } from '@nestjs/common';
import { PlanService } from './plan.service';
import { UserInputDto } from './dto/user-input.dto';

@Controller()
export class PlanController {
  constructor(private readonly planService: PlanService) {}

  @Post('plan-weekend')
  async planWeekend(@Body() dto: UserInputDto) {
    return this.planService.generatePlans(dto);
  }
}
```

### plan.module.ts

```typescript
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';
import { PlanController } from './plan.controller';
import { PlanService } from './plan.service';

@Module({
  imports: [HttpModule],
  controllers: [PlanController],
  providers: [PlanService],
})
export class PlanModule {}
```

---

## PLACE-TYPE MODULE

### place-type.service.ts

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { PlaceType, PlaceTypeDocument } from './place-type.schema';

@Injectable()
export class PlaceTypeService {
  constructor(
    @InjectModel(PlaceType.name)
    private readonly placeTypeModel: Model<PlaceTypeDocument>,
  ) {}

  async findByCity(city: string): Promise<PlaceType[]> {
    const results = await this.placeTypeModel
      .find({ city: city.toLowerCase().trim() })
      .lean()
      .exec();

    if (!results.length) {
      throw new NotFoundException(
        `No place types configured for city: ${city}`,
      );
    }

    return results;
  }
}
```

### InternalSecretGuard — create at `src/place-type/guards/internal-secret.guard.ts`

```typescript
import {
  CanActivate, ExecutionContext, Injectable, UnauthorizedException,
} from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class InternalSecretGuard implements CanActivate {
  constructor(private readonly configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const provided = request.headers['x-internal-secret'];
    const expected = this.configService.get<string>('n8nInternalSecret');

    if (!provided || provided !== expected) {
      throw new UnauthorizedException('Invalid or missing internal secret');
    }
    return true;
  }
}
```

### place-type.controller.ts (internal endpoint for n8n)

```typescript
import { Controller, Get, Query, UseGuards } from '@nestjs/common';
import { PlaceTypeService } from './place-type.service';
import { InternalSecretGuard } from './guards/internal-secret.guard';

@Controller('internal')
export class PlaceTypeController {
  constructor(private readonly placeTypeService: PlaceTypeService) {}

  @Get('place-types')
  @UseGuards(InternalSecretGuard)
  async getByCity(@Query('city') city: string) {
    return this.placeTypeService.findByCity(city);
  }
}
```

### place-type.module.ts

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { PlaceType, PlaceTypeSchema } from './place-type.schema';
import { PlaceTypeService } from './place-type.service';
import { PlaceTypeController } from './place-type.controller';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: PlaceType.name, schema: PlaceTypeSchema },
    ]),
  ],
  controllers: [PlaceTypeController],
  providers: [PlaceTypeService],
  exports: [PlaceTypeService],
})
export class PlaceTypeModule {}
```

---

## COMMON — IMPLEMENT BOTH

### GlobalExceptionFilter (`common/filters/global-exception.filter.ts`)

```typescript
import {
  ExceptionFilter, Catch, ArgumentsHost,
  HttpException, HttpStatus, Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';
    let error = 'Internal Server Error';
    let details: any = undefined;

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const body = exception.getResponse();

      if (typeof body === 'object' && body !== null) {
        const b = body as any;
        message = b.message || message;
        error   = b.error   || error;
        details = b.details || (Array.isArray(b.message) ? b.message : undefined);
        // If message was an array (class-validator), flatten it
        if (Array.isArray(b.message)) {
          message = 'Validation failed';
          details = b.message;
        }
      } else {
        message = body as string;
      }
    }

    this.logger.error(
      `${request.method} ${request.url} → ${status}: ${message}`,
      exception instanceof Error ? exception.stack : String(exception),
    );

    const responseBody: Record<string, any> = { statusCode: status, error, message };
    if (details) responseBody.details = details;

    response.status(status).json(responseBody);
  }
}
```

### LoggingInterceptor (`common/interceptors/logging.interceptor.ts`)

```typescript
import {
  Injectable, NestInterceptor, ExecutionContext,
  CallHandler, Logger,
} from '@nestjs/common';
import { Observable, tap } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const { method, url, ip } = req;
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const ms = Date.now() - start;
        const status = context.switchToHttp().getResponse().statusCode;
        this.logger.log(`${method} ${url} ${status} ${ms}ms [${ip}]`);
      })
    );
  }
}
```

---

## SEED DATA (`place-type.seed.ts`)

Create `src/place-type/place-type.seed.ts` as a standalone runnable script.
Run with: `npx ts-node src/place-type/place-type.seed.ts`

Insert exactly these 10 PlaceTypes for city `"mumbai"`:

```typescript
import mongoose from 'mongoose';
import * as dotenv from 'dotenv';
dotenv.config();

const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://localhost:27017/weekend-planner';

const placeTypes = [
  // 2 scenic low-energy spots
  {
    name: 'Lakeside Picnic Spot',
    city: 'mumbai',
    tags: ['scenic', 'calm'],
    best_time: 'morning',
    energy_required: 'low',
    distance_bucket: 'near',
    crowd_tendency: 'low',
    rain_safe: 'conditional',
    budget_range: 'low',
    transport_fit: 'both',
    api_search_prompt: 'lakeside picnic spot peaceful',
  },
  {
    name: 'Sunset Viewpoint',
    city: 'mumbai',
    tags: ['scenic', 'calm'],
    best_time: 'evening',
    energy_required: 'low',
    distance_bucket: 'around_city',
    crowd_tendency: 'medium',
    rain_safe: 'no',
    budget_range: 'low',
    transport_fit: 'both',
    api_search_prompt: 'sunset viewpoint hill',
  },
  // 2 active high-energy spots
  {
    name: 'Hill Fort Trek',
    city: 'mumbai',
    tags: ['active', 'scenic', 'ride'],
    best_time: 'morning',
    energy_required: 'high',
    distance_bucket: 'outskirts',
    crowd_tendency: 'low',
    rain_safe: 'no',
    budget_range: 'low',
    transport_fit: 'car',
    api_search_prompt: 'hill fort trek nature trail',
  },
  {
    name: 'Adventure Water Sports',
    city: 'mumbai',
    tags: ['active', 'social'],
    best_time: 'morning',
    energy_required: 'high',
    distance_bucket: 'outskirts',
    crowd_tendency: 'medium',
    rain_safe: 'no',
    budget_range: 'high',
    transport_fit: 'car',
    api_search_prompt: 'water sports adventure beach',
  },
  // 2 social / food spots
  {
    name: 'Heritage Food Walk',
    city: 'mumbai',
    tags: ['food', 'social', 'calm'],
    best_time: 'morning',
    energy_required: 'low',
    distance_bucket: 'near',
    crowd_tendency: 'high',
    rain_safe: 'yes',
    budget_range: 'medium',
    transport_fit: 'public',
    api_search_prompt: 'heritage food walk old city tour',
  },
  {
    name: 'Rooftop Café District',
    city: 'mumbai',
    tags: ['food', 'social', 'scenic'],
    best_time: 'evening',
    energy_required: 'low',
    distance_bucket: 'near',
    crowd_tendency: 'medium',
    rain_safe: 'conditional',
    budget_range: 'medium',
    transport_fit: 'both',
    api_search_prompt: 'rooftop cafe restaurant city view',
  },
  // 2 calm solo-friendly spots
  {
    name: 'Botanical Garden',
    city: 'mumbai',
    tags: ['calm', 'scenic'],
    best_time: 'morning',
    energy_required: 'low',
    distance_bucket: 'near',
    crowd_tendency: 'low',
    rain_safe: 'yes',
    budget_range: 'low',
    transport_fit: 'public',
    api_search_prompt: 'botanical garden nature walk quiet',
  },
  {
    name: 'Monastery or Temple Retreat',
    city: 'mumbai',
    tags: ['calm', 'scenic'],
    best_time: 'morning',
    energy_required: 'low',
    distance_bucket: 'around_city',
    crowd_tendency: 'low',
    rain_safe: 'yes',
    budget_range: 'low',
    transport_fit: 'both',
    api_search_prompt: 'monastery temple peaceful retreat',
  },
  // 2 ride / adventure spots
  {
    name: 'Coastal Drive + Beach',
    city: 'mumbai',
    tags: ['ride', 'scenic', 'calm'],
    best_time: 'morning',
    energy_required: 'medium',
    distance_bucket: 'outskirts',
    crowd_tendency: 'low',
    rain_safe: 'no',
    budget_range: 'medium',
    transport_fit: 'car',
    api_search_prompt: 'coastal beach drive scenic road',
  },
  {
    name: 'Countryside Cycling Route',
    city: 'mumbai',
    tags: ['ride', 'active', 'scenic'],
    best_time: 'morning',
    energy_required: 'medium',
    distance_bucket: 'outskirts',
    crowd_tendency: 'low',
    rain_safe: 'conditional',
    budget_range: 'low',
    transport_fit: 'car',
    api_search_prompt: 'countryside cycling trail village road',
  },
];

async function seed() {
  await mongoose.connect(MONGODB_URI);
  const db = mongoose.connection.db;
  const col = db.collection('placetypes');

  await col.deleteMany({ city: 'mumbai' });
  await col.insertMany(placeTypes);

  console.log(`Seeded ${placeTypes.length} PlaceTypes for Mumbai`);
  await mongoose.disconnect();
}

seed().catch((err) => {
  console.error('Seed failed:', err);
  process.exit(1);
});
```

---

## ERROR HANDLING — HTTP CODES AND MESSAGES

| Scenario | Code | Who handles |
|---|---|---|
| Unknown field in request body | 400 | NestJS `ValidationPipe` |
| Invalid enum / missing required field | 400 | NestJS `ValidationPipe` |
| Rate limit exceeded (10/IP/hour) | 429 | NestJS `ThrottlerGuard` |
| Missing/wrong `x-internal-secret` header | 401 | `InternalSecretGuard` |
| No PlaceTypes for city (from n8n) | 404 | n8n → NestJS re-throws |
| No valid PlaceTypes after scoring (from n8n) | 422 | n8n → NestJS re-throws |
| AI returned invalid JSON (from n8n) | 500 | n8n → NestJS re-throws |
| n8n webhook unreachable / timeout | 503 | `PlanService` catch block |
| Any unexpected error | 500 | `GlobalExceptionFilter` |

All responses use this shape:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "details": ["energy_level must be one of: low, medium, high"]
}
```

---

## PACKAGES TO INSTALL

Run exactly:

```bash
npm install \
  @nestjs/common @nestjs/core @nestjs/platform-express \
  @nestjs/config @nestjs/mongoose @nestjs/axios @nestjs/throttler \
  mongoose axios rxjs reflect-metadata \
  class-validator class-transformer

npm install --save-dev \
  @types/node typescript ts-node \
  @nestjs/cli
```

---

## tsconfig.json — REQUIRED SETTINGS

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "strict": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "noImplicitAny": true
  }
}
```

---

## EXAMPLE REQUEST / RESPONSE

### Request

```
POST /plan-weekend
Content-Type: application/json

{
  "city": "mumbai",
  "energy_level": "medium",
  "company": "partner",
  "budget_range": "medium",
  "travel_mode": "car",
  "distance_comfort": "outskirts",
  "selected_visual_tags": ["scenic", "calm"],
  "optional_vibe_text": "Something peaceful, away from noise"
}
```

### Success Response (200)

```json
{
  "plans": [
    {
      "plan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "recommended": true,
      "place_type_name": "Coastal Drive + Beach",
      "location_name": "Kashid Beach, Alibaug",
      "location_photos": [
        "https://places.googleapis.com/v1/places/.../photos/..."
      ],
      "tags": ["ride", "scenic", "calm"],
      "timeline": {
        "saturday": "Early morning: start the coastal drive. Late morning: arrive at the beach, walk along the shore. Afternoon: lunch at a local shack, relax.",
        "sunday": "Morning: explore the nearby fort or a quiet cove before heading back."
      },
      "budget_range": "medium",
      "distance_label": "~2 hrs from Mumbai",
      "transport_assumption": "Car strongly preferred; Mandwa ferry is an alternative to cut drive time.",
      "why_this_works": "A scenic coastal escape that perfectly matches your medium energy and love of open landscapes — ideal for a peaceful two-day retreat with your partner."
    }
  ]
}
```

### Validation Error (400)

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "details": [
    "energy_level must be one of the following values: low, medium, high",
    "selected_visual_tags must contain at least 1 elements"
  ]
}
```

### Rate Limited (429)

```json
{
  "statusCode": 429,
  "error": "Too Many Requests",
  "message": "ThrottlerException: Too Many Requests"
}
```

### n8n Unreachable (503)

```json
{
  "statusCode": 503,
  "error": "Service Unavailable",
  "message": "Plan generation service is currently unavailable. Please try again shortly."
}
```

---

## README.md — GENERATE THIS FILE

Include these sections:
1. **Prerequisites** — Node 20+, MongoDB running, n8n instance with webhook configured
2. **Setup** — `npm install`, copy `.env.example` to `.env`, fill in values
3. **Seed** — `npx ts-node src/place-type/place-type.seed.ts`
4. **Run** — `npm run start:dev`
5. **Test the endpoint** — paste the curl example below
6. **Environment variables table** — all 5 vars with descriptions
7. **Architecture note** — one paragraph explaining NestJS is a gateway, n8n does the AI work

Curl example to include in README:

```bash
curl -X POST http://localhost:3000/plan-weekend \
  -H "Content-Type: application/json" \
  -d '{
    "city": "mumbai",
    "energy_level": "medium",
    "company": "partner",
    "budget_range": "medium",
    "travel_mode": "car",
    "distance_comfort": "outskirts",
    "selected_visual_tags": ["scenic", "calm"],
    "optional_vibe_text": "Something peaceful, away from noise"
  }'
```

---

## WHAT NOT TO BUILD

Do NOT generate any of the following:

- `ai/` directory or any AI service
- `scoring/` directory or any scoring service
- `places-api/` directory or any Places API client
- `cache/` directory or cache service
- Any direct calls to Anthropic API
- Any direct calls to Google Places API
- Authentication (JWT, sessions, API keys for users)
- Bookings, reviews, ratings, or social features
- Any frontend code or HTML

All of the above live in n8n. NestJS only validates, rate-limits, and proxies.

---

## FINAL CHECKLIST

Before finishing, Antigravity must verify:

- [ ] All 5 files in `src/config/`, `src/common/` generated correctly
- [ ] `ValidationPipe` registered globally in `main.ts` with `whitelist: true, forbidNonWhitelisted: true, transform: true`
- [ ] `GlobalExceptionFilter` registered in `main.ts`
- [ ] `ThrottlerModule` in `app.module.ts` — `ttl: 3600, limit: 10`
- [ ] `ThrottlerGuard` provided globally via `APP_GUARD` in `app.module.ts`
- [ ] `PlaceTypeController` exposes `GET /internal/place-types?city=`
- [ ] `InternalSecretGuard` checks `x-internal-secret` header against env var
- [ ] `PlanService.generatePlans()` POSTs to `N8N_WEBHOOK_URL` with 45s timeout
- [ ] `PlanService` catch block handles both n8n error responses AND network failures
- [ ] `PlaceTypeService.findByCity()` throws 404 if empty result
- [ ] `PlaceTypeSchema` has `index: true` on `city` field
- [ ] `company` enum is `'alone' | 'partner' | 'friends' | 'family'` — not `'solo'`
- [ ] Seed script is runnable standalone with `npx ts-node`
- [ ] `.env.example` has all 5 variables
- [ ] `README.md` generated with setup + curl example
- [ ] No AI, scoring, Places API, or cache code anywhere in the project
