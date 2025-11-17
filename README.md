# Cloudflare DDNS Worker

A Cloudflare Worker that implements a Dynamic DNS (DDNS) update service, allowing automatic updates of DNS records in Cloudflare when your IP address changes.

## Overview

This worker provides a compatible endpoint for DDNS clients (such as routers or update scripts) to automatically update Cloudflare DNS records. It implements the standard DDNS update protocol used by many services.

## Features

- **HTTPS Enforcement** - All connections must use HTTPS for security
- **HTTP Basic Authentication** - Uses standard authentication headers
- **Cloudflare API Integration** - Directly updates DNS A records via Cloudflare API
- **Standard DDNS Protocol** - Compatible with most DDNS clients
- **Detailed Error Messages** - Clear feedback for troubleshooting
- **Type A Records Only** - Currently supports IPv4 A records

## How It Works

1. **Authentication**: The worker extracts credentials from HTTP Basic Authentication headers
   - Username: Your domain name (zone name)
   - Password: Your Cloudflare API token

2. **Update Process**:
   - Accepts query parameters for the hostname and new IP address
   - Validates all required parameters
   - Finds the Cloudflare zone (domain)
   - Locates the specific DNS record
   - Updates the record with the new IP address

3. **Response**: Returns `good` on success (standard DDNS protocol response)

## API Endpoints

### Update DNS Record

**Endpoints:** `/nic/update` or `/update`

**Method:** GET or POST

**Authentication:** HTTP Basic Auth
- Username: Zone name (your domain)
- Password: Cloudflare API token

**Query Parameters:**
- `hostname` (required): The full DNS record to update (e.g., `home.example.com`)
- `ip` or `myip` (required): The new IP address

**Example Request:**
```bash
curl -u "example.com:your-api-token" \
  "https://your-worker.workers.dev/update?hostname=home.example.com&myip=1.2.3.4"
```

**Using verbose mode to see full response:**
```bash
curl -v -u "example.com:your-api-token" \
  "https://your-worker.workers.dev/update?hostname=home.example.com&ip=1.2.3.4"
```

**Success Response:**
```
good
```

### Other Endpoints

- `/favicon.ico` - Returns 204 No Content
- `/robots.txt` - Returns 204 No Content
- All other paths - Returns 404 Not Found

## Prerequisites

Before you begin, ensure you have:
- Node.js 16+ and npm installed
- A Cloudflare account with a domain/zone configured
- A DNS A record already created in Cloudflare (the worker updates existing records, it doesn't create new ones)
- Basic understanding of HTTP and command-line tools

## Installation

1. **Clone or download this repository**

2. **Install dependencies**:
   ```bash
   npm install
   ```

## Development

### Run Tests
```bash
npm test
```

Run tests in watch mode during development - tests will automatically re-run when files change.

### Local Development Server
```bash
npm run dev
# or
npm start
```

This starts a local development server (typically at `http://localhost:8787/`) where you can test the worker locally before deploying.

### Run Tests Once (CI Mode)
```bash
npm test -- --run
```

## Deployment

### Deploy to Cloudflare Workers
```bash
npm run deploy
```

This will deploy your worker to Cloudflare. Make sure you're logged in with Wrangler first.

### Wrangler Authentication
First-time setup requires authentication:
```bash
npx wrangler login
```

### View Deployment Info
```bash
npx wrangler whoami
```

### View Logs
```bash
npx wrangler tail
```

Stream live logs from your deployed worker.

### Additional Wrangler Commands

View worker details:
```bash
npx wrangler deployments list
```

Delete a deployment:
```bash
npx wrangler delete
```

## Setup Guide

### Step 1: Create DNS Record (Required)

**Important**: The DNS record must exist before using this worker. Create an A record in Cloudflare:

1. Go to your Cloudflare Dashboard → Select your domain
2. Navigate to **DNS** → **Records**
3. Add an A record:
   - Type: `A`
   - Name: Your subdomain (e.g., `home` for `home.example.com`, or `@` for root domain)
   - IPv4 address: Any initial IP (will be updated by the worker)
   - Proxy status: Your preference (DNS only or Proxied)
   - TTL: Auto or your preference

### Step 2: Create Cloudflare API Token

1. Go to Cloudflare Dashboard → **My Profile** → **API Tokens**
2. Click **Create Token**
3. Use the **Edit zone DNS** template or create a custom token with:
   - Permissions: `Zone` → `DNS` → `Edit`
   - Zone Resources: `Include` → `Specific zone` → Select your domain
4. Click **Continue to summary** → **Create Token**
5. **Save the token securely** - you won't be able to see it again!

### Step 3: Deploy Worker

1. Authenticate with Wrangler (first time only):
   ```bash
   npx wrangler login
   ```

2. Deploy the worker:
   ```bash
   npm run deploy
   ```

3. Note your worker URL (e.g., `https://cloudflare-worker-ddns.your-subdomain.workers.dev`)

### Step 4: Test the Worker

Test with curl to verify it works:
```bash
curl -v -u "example.com:your-api-token" \
  "https://your-worker.workers.dev/update?hostname=home.example.com&ip=1.2.3.4"
```

Expected response: `good`

### Step 5: Configure DDNS Client
   - Set update URL to: `https://your-worker.workers.dev/update`
   - Username: Your domain name (e.g., `example.com`)
   - Password: Your Cloudflare API token
   - Hostname: The DNS record to update (e.g., `home.example.com`)

## Error Responses

The worker returns appropriate HTTP status codes and descriptive error messages:

### Client Errors (400 Bad Request)

- `"Please use a HTTPS connection."` - Request not using HTTPS
- `"Please provide valid credentials."` - Missing Authorization header
- `"Invalid authorization value."` - Malformed Basic Auth credentials
- `"You must include proper query parameters"` - Missing query parameters
- `"You must specify a hostname"` - Missing `hostname` parameter
- `"You must specify an ip address"` - Missing both `ip` and `myip` parameters
- `"Zone 'example.com' not found"` - Domain not found in your Cloudflare account
- `"DNS record 'home.example.com' not found in zone 'example.com'"` - DNS A record doesn't exist
- `"Cloudflare API error: [message]"` - API authentication failed or other API errors
- `"Failed to update DNS record: [message]"` - DNS record update failed

### Not Found (404)

- `"Not Found."` - Invalid endpoint path

### Server Errors (500)

- Unexpected errors with stack trace

## Security

- Enforces HTTPS connections only
- Uses HTTP Basic Authentication
- Validates all input parameters
- Cloudflare API token is passed securely via authentication headers

## Compatible DDNS Clients

This worker is compatible with most DDNS clients that support custom update URLs, including:
- DD-WRT routers
- pfSense
- OPNsense
- Linux ddclient
- Custom scripts

## Troubleshooting

### Common Issues

#### "Zone 'example.com' not found"
- Verify the domain exists in your Cloudflare account
- Use the exact zone name as username (not a subdomain)
- Check your API token has access to this zone

#### "DNS record 'home.example.com' not found"
- Create the A record in Cloudflare DNS first
- Verify the hostname exactly matches the DNS record name
- The worker only updates existing A records, it doesn't create new ones

#### "Cloudflare API error: Invalid request headers"
- Check your API token is correct and not expired
- Verify the token has `Zone:DNS:Edit` permissions
- Make sure you're using the correct token (not API key)

### Testing Commands

#### 1. Verify API Token
```bash
curl "https://api.cloudflare.com/client/v4/user/tokens/verify" \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

Expected response:
```json
{
  "result": {
    "id": "...",
    "status": "active"
  },
  "success": true
}
```

#### 2. List Your Zones
```bash
curl -s -X GET \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=example.com" | jq
```

Expected response:
```json
{
  "result": [
    {
      "id": "zone-id-here",
      "name": "example.com",
      "status": "active"
    }
  ],
  "success": true
}
```

#### 3. List DNS A Records for Your Zone
```bash
curl -s -X GET \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/ZONE_ID/dns_records?type=A" | jq
```

#### 4. Test Worker Update
```bash
curl -v -u "example.com:YOUR_API_TOKEN" \
  "https://your-worker.workers.dev/update?hostname=home.example.com&ip=1.2.3.4"
```

Expected response:
```
good
```

### Manual DNS Record Update (Direct API)

If you need to update a DNS record directly:
```bash
curl -s -X PUT \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  --data '{"type":"A","name":"home.example.com","content":"1.2.3.4","ttl":1,"proxied":false}' \
  "https://api.cloudflare.com/client/v4/zones/ZONE_ID/dns_records/RECORD_ID"
```

## License

This is a standard Cloudflare Worker implementation for DDNS updates.