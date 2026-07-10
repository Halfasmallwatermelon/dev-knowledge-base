# 安全基础

## 认证与授权

### JWT 认证

```python
from jose import jwt
from datetime import datetime, timedelta

SECRET_KEY = "your-secret-key"

def create_token(user_id):
    payload = {
        "user_id": user_id,
        "exp": datetime.utcnow() + timedelta(hours=24)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")

def verify_token(token):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload["user_id"]
    except jwt.ExpiredSignatureError:
        return None
```

### OAuth 2.0

```
1. 用户点击"使用GitHub登录"
2. 重定向到GitHub授权页面
3. 用户同意授权
4. GitHub回调你的应用，带authorization code
5. 用code换取access token
6. 用token获取用户信息
```

## 常见攻击

### SQL 注入

```python
# ❌ 危险
query = f"SELECT * FROM users WHERE username = '{username}'"

# ✅ 安全
query = "SELECT * FROM users WHERE username = %s"
cursor.execute(query, (username,))
```

### XSS（跨站脚本）

```python
# ❌ 危险
html = f"<div>{user_input}</div>"

# ✅ 安全
from markupsafe import escape
html = f"<div>{escape(user_input)}</div>"
```

### CSRF（跨站请求伪造）

```python
# 使用 CSRF token
@app.route("/transfer", methods=["POST"])
@csrf_protect
def transfer():
    # 处理转账
    pass
```

## 密码安全

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# 哈希密码
hashed = pwd_context.hash("password123")

# 验证密码
is_valid = pwd_context.verify("password123", hashed)
```

## HTTPS

```nginx
# Nginx HTTPS 配置
server {
    listen 443 ssl
    server_name example.com
    
    ssl_certificate /path/to/cert.pem
    ssl_certificate_key /path/to/key.pem
    
    # 强制 HTTPS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always
}
```