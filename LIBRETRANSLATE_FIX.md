# LibreTranslate Fix Summary

## The Problem

LibreTranslate appeared to be "stopped" or hanging with the error:
```
onnxruntime cpuid_info warning: Unknown CPU vendor. cpuinfo_vendor value: 0
```

## Root Causes

### 1. Port Conflict (Primary Issue)
**Port 5000 was already in use by macOS AirPlay Receiver service** (AirTunes). When you tried to access `localhost:5000`, you were connecting to Apple's AirPlay service instead of LibreTranslate.

**Evidence:**
```bash
$ curl -v http://localhost:5000/
< Server: AirTunes/925.5.1
< X-Apple-ProcessingTime: 0
```

### 2. Slow Initialization on Apple Silicon
LibreTranslate with onnxruntime has compatibility issues with ARM64/Apple Silicon, causing:
- The `cpuid_info warning` (non-fatal, just a warning)
- Slow model downloads and initialization (1GB+ of data)
- No progress logs during model loading
- Can take 2-5 minutes to fully start

## The Solution

### Changes Made to `docker-compose.yaml`:

```yaml
libretranslate:
  image: libretranslate/libretranslate:latest
  container_name: libretranslate
  platform: linux/arm64
  environment:
    - LT_LOAD_ONLY=en,es,fr,de,ja,zh          # Reduced from 11 to 6 languages
    - LT_DEBUG=true                           # Enable verbose logging
    - LT_THREADS=2                            # Limit CPU threads
  ports:
    - 5050:5000                               # Changed from 5000:5000 to 5050:5000
  restart: unless-stopped
  volumes:
    - ./libretranslate_data:/home/libretranslate/.local  # Persist models
```

### Key Changes:

1. **Port Mapping**: Changed from `5000:5000` to `5050:5000`
   - LibreTranslate now accessible at `http://localhost:5050` (not 5000)

2. **Reduced Languages**: From 11 to 6 languages (en, es, fr, de, ja, zh)
   - Faster initialization
   - Less memory usage
   - Fewer models to download

3. **Added Volume Mount**: Persists downloaded models
   - Models stored in `./libretranslate_data/`
   - Prevents re-downloading on container restart
   - 1GB+ saved locally

4. **Debug Mode**: Added `LT_DEBUG=true`
   - More verbose logging
   - Better visibility into initialization process

5. **Thread Limit**: Added `LT_THREADS=2`
   - Reduces CPU usage
   - Better performance on Apple Silicon

## Current Status

✅ **Both services are now running:**

| Service | Port | Status | Model |
|---------|------|--------|-------|
| faster-whisper | 10300 | ✅ Running | small |
| libretranslate | 5050 | ✅ Running | 6 languages, 8 models |

## How to Use LibreTranslate

### API Endpoint
```
http://localhost:5050
```

### Get Available Languages
```bash
curl http://localhost:5050/languages
```

**Response:**
```json
[
  {"code":"en","name":"English","targets":["de","en","es","fr","ja","zh"]},
  {"code":"es","name":"Spanish","targets":["de","en","es","fr","ja","zh"]},
  {"code":"fr","name":"French","targets":["de","en","es","fr","ja","zh"]},
  {"code":"de","name":"German","targets":["de","en","es","fr","ja","zh"]},
  {"code":"ja","name":"Japanese","targets":["de","en","es","fr","ja","zh"]},
  {"code":"zh","name":"Chinese","targets":["de","en","es","fr","ja","zh"]}
]
```

### Translate Text

**Request:**
```bash
curl -X POST http://localhost:5050/translate \
  -H "Content-Type: application/json" \
  -d '{
    "q": "Hello, how are you?",
    "source": "en",
    "target": "es"
  }'
```

**Response:**
```json
{
  "translatedText": "Hola, ¿cómo estás?"
}
```

### Examples

**English → Japanese:**
```bash
curl -s -X POST http://localhost:5050/translate \
  -H "Content-Type: application/json" \
  -d '{"q":"Good morning","source":"en","target":"ja"}' | jq .
```
Result: `おはようございます`

**English → French:**
```bash
curl -s -X POST http://localhost:5050/translate \
  -H "Content-Type: application/json" \
  -d '{"q":"Thank you","source":"en","target":"fr"}' | jq .
```
Result: `Je vous remercie`

**Japanese → English:**
```bash
curl -s -X POST http://localhost:5050/translate \
  -H "Content-Type: application/json" \
  -d '{"q":"ありがとうございます","source":"ja","target":"en"}' | jq .
```
Result: `Thank you very much.`

## API Documentation

### POST /translate

**Parameters:**
- `q` (string, required): Text to translate
- `source` (string, required): Source language code (e.g., "en")
- `target` (string, required): Target language code (e.g., "es")
- `format` (string, optional): "text" (default) or "html"
- `api_key` (string, optional): API key if authentication is enabled

**Response:**
```json
{
  "translatedText": "The translated text"
}
```

### GET /languages

Returns array of available languages with their supported target languages.

### GET /detect

Detect the language of provided text.

**Example:**
```bash
curl -X POST http://localhost:5050/detect \
  -H "Content-Type: application/json" \
  -d '{"q":"Bonjour"}'
```

## Node.js Integration Example

```javascript
const axios = require('axios');

async function translate(text, source, target) {
  try {
    const response = await axios.post('http://localhost:5050/translate', {
      q: text,
      source: source,
      target: target
    });
    return response.data.translatedText;
  } catch (error) {
    console.error('Translation error:', error.message);
    throw error;
  }
}

// Usage
translate("Hello, world!", "en", "es")
  .then(result => console.log(result)); // "Hola, mundo!"
```

## Python Integration Example

```python
import requests

def translate(text, source, target):
    response = requests.post('http://localhost:5050/translate', json={
        'q': text,
        'source': source,
        'target': target
    })
    return response.json()['translatedText']

# Usage
result = translate("Hello, world!", "en", "ja")
print(result)  # こんにちは、世界！
```

## Troubleshooting

### Issue: Service Not Responding

**Check if running:**
```bash
docker-compose ps
```

**Check logs:**
```bash
docker-compose logs libretranslate
```

**Restart service:**
```bash
docker-compose restart libretranslate
```

### Issue: Slow Startup

LibreTranslate needs to download translation models on first start:
- Expected: 1-2 GB download
- Time: 2-5 minutes on first start
- Subsequent starts: Fast (models are cached)

**Monitor download progress:**
```bash
docker-compose logs -f libretranslate
```

Wait for:
```
Loaded support for 6 languages (X models total)!
[INFO] Starting gunicorn
[INFO] Booting worker with pid: X
```

### Issue: Port Already in Use

If you see connection errors, check if port 5050 is available:
```bash
lsof -i :5050
```

Change port in `docker-compose.yaml` if needed:
```yaml
ports:
  - 5051:5000  # Use different host port
```

### Issue: Out of Memory

If LibreTranslate crashes due to memory:
1. Reduce number of languages in `LT_LOAD_ONLY`
2. Increase Docker memory limit
3. Reduce `LT_THREADS` value

## Performance Notes

- **First Translation**: May be slow (model initialization)
- **Subsequent Translations**: Fast (models loaded in memory)
- **Quality**: Good for general text, not as good as commercial APIs for specialized content
- **Offline**: Fully offline once models are downloaded
- **Languages Supported**: 6 languages with bidirectional translation

## Supported Language Pairs

All language pairs are supported bidirectionally:
- English ↔ Spanish
- English ↔ French
- English ↔ German
- English ↔ Japanese
- English ↔ Chinese
- Spanish ↔ French
- (and all other combinations)

## Adding More Languages

To add more languages, edit `docker-compose.yaml`:

```yaml
environment:
  - LT_LOAD_ONLY=en,es,fr,de,ja,zh,ko,pt,ru,ar,it
```

Available language codes: en, es, fr, de, ja, zh, ko, pt, ru, ar, it, nl, pl, tr, etc.

**Note:** More languages = slower startup and more memory usage.

## Common macOS Port Conflicts

If you encounter port conflicts on macOS:
- **Port 5000**: AirPlay Receiver (this issue)
- **Port 7000**: AirPlay
- **Port 3000**: Various development servers
- **Port 8080**: Common web proxy

Always check `netstat -an | grep <port>` before using a port.

---

**Updated:** 2026-02-13
**Services:** faster-whisper + libretranslate
**Platform:** Apple Silicon (ARM64)
