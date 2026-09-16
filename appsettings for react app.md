In a **.NET application**, configuration is usually stored in `appsettings.json`:

```json
{
  "APIBaseUrl": "https://api.example.com",
  "ConnectionStrings": {
    "DefaultConnection": "..."
  }
}
```

and accessed through `IConfiguration`.

For a **React application**, there is no built-in `appsettings.json` equivalent. Common approaches are:

### 1. Environment Files (Most Common)

Create files like:

**.env**

```env
REACT_APP_API_URL=https://dev-api.example.com
```

**.env.production**

```env
REACT_APP_API_URL=https://api.example.com
```

Access in React:

```javascript
const apiUrl = process.env.REACT_APP_API_URL;
```

For Vite projects:

```env
VITE_API_URL=https://api.example.com
```

```javascript
const apiUrl = import.meta.env.VITE_API_URL;
```

---

### 2. Runtime Configuration File

Create a file such as:

**public/config.json**

```json
{
  "apiUrl": "https://api.example.com"
}
```

Load it when the app starts:

```javascript
const config = await fetch('/config.json').then(r => r.json());
```

This is closest to `.NET appsettings.json` because you can change the file without rebuilding the React app.

---

### 3. Config JavaScript File

**src/config.js**

```javascript
export default {
  apiUrl: "https://api.example.com"
};
```

Use:

```javascript
import config from './config';

console.log(config.apiUrl);
```

---

### In Docker Deployments

A common pattern is:

```yaml
volumes:
  - ./config.json:/usr/share/nginx/html/config.json
```

Then React reads `config.json` at runtime. This allows changing API URLs or environment-specific settings without rebuilding the React image.

If you're working on your DevSecOps deployment, check whether your React UI container contains:

```bash
docker exec -it <container> sh
find / -name "*.env" 2>/dev/null
find / -name "config.json" 2>/dev/null
```

or look for:

```bash
cat /usr/share/nginx/html/config.json
```

Many production React applications deployed through Nginx use a runtime `config.json` instead of `.env` files.
