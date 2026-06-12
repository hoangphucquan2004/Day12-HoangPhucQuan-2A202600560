# Deployment Information

## Public URL

https://hoangphucquan-2a202600560-production-2228.up.railway.app

## Platform

Railway — deployed via `railway up` CLI from `06-lab-complete/`

## Environment Variables Set

| Variable | Description |
|----------|-------------|
| `PORT` | Auto-injected by Railway |
| `AGENT_API_KEY` | API key for authentication |
| `ENVIRONMENT` | `production` |

## Test Commands

### Health Check
```bash
curl https://hoangphucquan-2a202600560-production-2228.up.railway.app/health
# Expected: {"status":"ok","version":"1.0.0",...}
```

### Readiness Probe
```bash
curl https://hoangphucquan-2a202600560-production-2228.up.railway.app/ready
# Expected: {"ready":true}
```

### Authentication Required (expect 401)
```bash
curl -X POST https://hoangphucquan-2a202600560-production-2228.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: {"detail":"Invalid or missing API key..."}
```

### API Test with Key (expect 200)
```bash
curl -X POST https://hoangphucquan-2a202600560-production-2228.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
# Expected: {"question":"...","answer":"...","model":"gpt-4o-mini","timestamp":"..."}
```

### Rate Limit Test (expect 429 after limit)
```bash
for i in {1..25}; do
  curl -s -X POST https://hoangphucquan-2a202600560-production-2228.up.railway.app/ask \
    -H "X-API-Key: dev-key-change-me" \
    -H "Content-Type: application/json" \
    -d '{"question": "test"}' | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Request {'"'"'$i'"'"'}: {d.get(\"answer\",d.get(\"detail\",\"\"))[:50]}')"
done
# After 20 requests: {"detail":"Rate limit exceeded: 20 req/min"}
```
