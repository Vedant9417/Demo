# Weekend Planner

NestJS backend for the Weekend Planner app, acting as a gateway to n8n workflows. NestJS handles validation, rate limiting, and connecting to MongoDB, while the AI/scoring/places logic lives in n8n.

## Prerequisites

- Node.js 20+
- MongoDB running locally or via Atlas
- n8n instance with a properly configured webhook

## Setup

1. Install dependencies:
   ```bash
   npm install
   ```

2. Configure environment variables:
   Copy `.env.example` to `.env` and adjust values:
   ```bash
   cp .env.example .env
   ```
   **Environment Variables:**
   - `MONGODB_URI`: Connection string for MongoDB database (e.g. `mongodb://localhost:27017/weekend-planner`)
   - `N8N_WEBHOOK_URL`: The URL for the n8n webhook processing the plan (e.g. `https://your-n8n-instance.com/webhook/weekend-planner`)
   - `N8N_INTERNAL_SECRET`: Shared secret used to authenticate internal requests from n8n to fetch PlaceTypes
   - `PORT`: The port on which the NestJS API will run (default: 3000)
   - `NODE_ENV`: The environment the app is running in (e.g. `development` or `production`)

## Seed

Seed the database with default PlaceTypes for Mumbai:
```bash
npx ts-node src/place-type/place-type.seed.ts
```

## Run

Run the application in development mode:
```bash
npm run start:dev
```

## Test the API

You can test the endpoint using `curl`:

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

## Architecture Note

This NestJS backend serves purely as a validation and rate-limiting gateway. All AI processing (like interacting with Anthropic), scoring mechanics, fetching data from Google Places, and intricate orchestration occurs inside the n8n workflow. NestJS provides the frontend with a clean, validated API and protects the system via rate limiting (`@nestjs/throttler`). NestJS also acts as the central hub for accessing MongoDB, exposing an internal endpoint for n8n to retrieve `PlaceType` properties safely.
