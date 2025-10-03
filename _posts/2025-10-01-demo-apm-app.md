---
title: "Building a SOAP Authentication Demo for APM Testing"
date: 2025-10-02
categories: [big-ip]
tags: [f5, big-ip, apm]
excerpt: "A lightweight Python application demonstrating SOAP-based authentication with features designed specifically for APM observability testing - no database required."
toc: true
---
<figure>
    <a href="/assets/apm-demo-app/screenshot-home-page.png"><img src="/assets/apm-demo-app/screenshot-home-page.png"></a>
</figure>

## Overview

This is a basic demo app that you can run to demo some F5 APM features. It's a single .py file but allows for login with multiple usernames and passwords, and a few API endpoints.

## Why SOAP in 2025?

My customer had a SOAP interface for his app, and so part of this application is a login via SOAP authentication.

## Architecture Overview

The application is built with Python using Flask for the web interface and Spyne for SOAP service implementation. It runs as a single file application that can be deployed with systemd, making it perfect for quick demos on a Ubuntu server.

Here's what the architecture looks like:

```
┌─────────────┐
│   Browser   │
│  (Client)   │
└──────┬──────┘
       │ HTTP
       ▼
┌─────────────────────────────────┐
│      Flask Web Application      │
├─────────────────────────────────┤
│  - Login Page                   │
│  - Dashboard                    │
│  - Logout Page                  │
│  - REST API Endpoints           │
└──────┬──────────────────────────┘
       │ Internal SOAP Calls
       ▼
┌─────────────────────────────────┐
│   SOAP Authentication Service   │
├─────────────────────────────────┤
│  - authenticate()               │
│  - validate_session()           │
│  - get_user()                   │
│  - validate_password()          │
│  - logout()                     │
└──────┬──────────────────────────┘
       │
       ▼
┌─────────────────────────────────┐
│     In-Memory Data Stores       │
├─────────────────────────────────┤
│  - User Database (dict)         │
│  - Active Sessions (dict)       │
└─────────────────────────────────┘
```

## Key Features for APM Testing

### 1. Separate Logout Page

When using the BIG-IP APM system, you can enter URIs that various websites may use to terminate sessions in the access profile configuration. The access profile monitors the URIs accessed  and it simultaneously terminates the BIG-IP APM session to the website as the session to the website ends. So, we have a dedicated logout page at `/logout`. 

The logout implementation looks like this:

```python
@app.route('/logout', methods=['POST', 'GET'])
def logout_page():
    """Separate logout page"""
    session_token = session.get('token')
    if session_token:
        # Logout via SOAP
        service = AuthenticationService()
        service.logout(None, session_token)
        session.pop('token', None)
    
    logout_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    return render_template_string(LOGOUT_TEMPLATE, logout_time=logout_time)
```

Users see a dedicated page confirming their logout with a timestamp, and F5 APM sees this as a distinct request it can use to terminate the APM session.

### 2. In-Memory User Database

Rather than requiring MySQL, PostgreSQL, or another database, the demo uses a simple Python dictionary for user storage:

```python
USERS_DB = {
    'admin': {
        'password_hash': hashlib.sha256('password123'.encode()).hexdigest(),
        'email': 'admin@example.com'
    },
    'user1': {
        'password_hash': hashlib.sha256('mypassword'.encode()).hexdigest(),
        'email': 'user1@example.com'
    },
    'demo': {
        'password_hash': hashlib.sha256('demo123'.encode()).hexdigest(),
        'email': 'demo@example.com'
    },
}
```

This design decision makes the demo:
- **Portable**: No database installation or configuration required
- **Fast**: Instant startup without database connections or migrations
- **Simple**: Easy to add test users by editing the dictionary
- **Stateless**: Can be restarted without losing test credentials

For F5 APM testing, this is perfect. You can spin up the application in seconds and start generating traffic immediately.

### 3. Session Management

The application implements realistic session management with expiration:

```python
ACTIVE_SESSIONS = {}
SESSION_TIMEOUT = 3600  # 1 hour

# Background cleanup thread
def cleanup_expired_sessions():
    while True:
        current_time = datetime.now()
        expired_tokens = []
        
        for token, session_info in ACTIVE_SESSIONS.items():
            if current_time - session_info['last_accessed'] > timedelta(seconds=SESSION_TIMEOUT):
                expired_tokens.append(token)
        
        for token in expired_tokens:
            del ACTIVE_SESSIONS[token]
        
        time.sleep(300)  # Check every 5 minutes
```

So this app doesn't *need* to be behind F5 APM, and F5 APM will need to handle SSO to this app. This way, you can use F5 APM to do things like:
- **Enforce MFA.** Example: username + password + token at F5 APM login, and then have APM send just username + password to the app if authentication succeeds. 
- **Perform OAuth Client/Resource Server.** Example: have F5 APM rely on Azure AD for auth, and then pass access token to app. (In this case, you would edit the app to receive OAuth token and claims, not username + password. Today the app is configured to expect username + password)
- **Perform SAML Auth.** Example: similar to OAuth but use a SAML IdP. (Again, you would edit the app to receive SAML claims, not username + password. Today the app is configured to expect username + password)

### 4. Multiple API Endpoints

The application exposes several REST API endpoints alongside the SOAP service, demonstrating different interaction patterns:

**Password Validation API**
```python
@app.route('/api/validate-password', methods=['POST'])
def api_validate_password():
    """API endpoint to validate password and return email"""
    data = request.get_json()
    username = data['username']
    password = data['password']
    
    service = AuthenticationService()
    email = service.validate_password(None, username, password)
    
    return jsonify({
        'success': True,
        'username': username,
        'email': email,
        'timestamp': datetime.now().isoformat()
    })
```

**Email Lookup API**
```python
@app.route('/api/get-email', methods=['GET', 'POST'])
def api_get_email():
    """API endpoint to get email address by username"""
    # Handles both GET and POST
    username = request.args.get('username') if request.method == 'GET' else request.get_json()['username']
    
    return jsonify({
        'success': True,
        'username': username,
        'email': USERS_DB[username]['email'],
        'timestamp': datetime.now().isoformat()
    })
```

These endpoints let you test:
- Different HTTP methods (GET vs POST). Example, after user enters username + password at F5 APM login screen, validate the creds are correct before continuing with further steps, like MFA or other checks.
- JSON request/response handling
- Error scenarios (404s for unknown users, 401s for bad credentials)
- MFA. Example: lookup a user's email address after they enter their username and password, so you can perform MFA with OTP via email.

### 5. Comprehensive Logging

Something I added later but find very handy is **comprehensive file-based logging**. The application implements rotating log files that capture every important event:

```python
def setup_logging():
    """Setup file logging with rotation"""
    log_dir = '/var/log/soap-auth-demo'
    
    # Main application log
    app_log_file = os.path.join(log_dir, 'soap_auth_demo.log')
    
    # Rotating file handler (10MB files, keep 5 backups)
    file_handler = logging.handlers.RotatingFileHandler(
        app_log_file, 
        maxBytes=10*1024*1024,
        backupCount=5
    )
    
    # Separate access log
    access_log_file = os.path.join(log_dir, 'access.log')
```

The logging system captures:
- **Authentication events**: Every login attempt (successful and failed)
- **Session operations**: Creation, validation, expiration
- **API calls**: All REST and SOAP endpoint usage
- **Client information**: IP addresses and user agents
- **Errors and warnings**: Failed authentication, invalid sessions

Example log output:
```
2025-10-02 10:30:15,123 - __main__ - INFO - Web login attempt for user 'admin' from IP 192.168.1.100
2025-10-02 10:30:15,125 - __main__ - INFO - User 'admin' authenticated successfully via SOAP
2025-10-02 10:30:15,126 - __main__ - INFO - Session created: AbC123XyZ456...
2025-10-02 10:32:45,789 - __main__ - INFO - Dashboard accessed by user 'admin' from IP 192.168.1.100
```

For APM testing, this is invaluable. You can:
- Correlate APM data with application logs
- Verify APM captured all transactions
- Debug any missing or incorrect traces
- Monitor from SSH with `tail -f /var/log/soap-auth-demo/soap_auth_demo.log`

### 6. SOAP Service Implementation

The core authentication logic uses Spyne to create a proper SOAP web service:

```python
class AuthenticationService(ServiceBase):
    """SOAP Web Service for Authentication"""
    
    @rpc(Unicode, Unicode, _returns=Unicode)
    def authenticate(ctx, username, password):
        """Authenticate user and return session token"""
        # Validation logic
        session_token = secrets.token_urlsafe(32)
        ACTIVE_SESSIONS[session_token] = {
            'username': username,
            'email': USERS_DB[username]['email'],
            'created_at': datetime.now()
        }
        return session_token
    
    @rpc(Unicode, _returns=Boolean)
    def validate_session(ctx, session_token):
        """Validate session token"""
        return session_token in ACTIVE_SESSIONS
```

The SOAP service exposes a WSDL at `http://localhost:5000/soap?wsdl`, making it easy to:
- Test SOAP client libraries
- Verify APM tools can trace SOAP calls
- Demonstrate distributed tracing across SOAP boundaries

## Deployment with systemd

The application is designed to run as a systemd service on Ubuntu. Here's a sample unit file:

```ini
[Unit]
Description=SOAP Authentication Demo Application
After=network.target

[Service]
Type=simple
User=azureuser
WorkingDirectory=/opt/soap-auth-demo
ExecStart=/usr/bin/python3 /opt/soap-auth-demo/soap-auth-demo.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

This makes the application:
- Start automatically on boot
- Restart on failure
- Easy to manage with `systemctl` commands

## Testing the Application

Once deployed, you can test all the features:

**Web Interface**
```bash
# Access the login page
curl http://localhost:5000/

# Test authentication
curl -X POST http://localhost:5000/ \
  -d "username=admin&password=password123"
```

**REST API**
```bash
# Validate password
curl -X POST http://localhost:5000/api/validate-password \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "password123"}'

# Get email address
curl http://localhost:5000/api/get-email?username=admin
```

**SOAP Service**
```bash
# View WSDL
curl http://localhost:5000/soap?wsdl

# Call SOAP service (using appropriate SOAP client)
```

**Monitor Logs**
```bash
# Tail the main application log
sudo tail -f /var/log/soap-auth-demo/soap_auth_demo.log

# Tail the access log
sudo tail -f /var/log/soap-auth-demo/access.log
```

## Generating Load for Testing

You can easily generate traffic for APM testing:

```python
import requests
import random
import time

users = ['admin', 'user1', 'demo', 'michaeloleary']
passwords = ['password123', 'mypassword', 'demo123', 'TooHotSummer2025']

while True:
    # Random login
    user_idx = random.randint(0, len(users) - 1)
    response = requests.post('http://localhost:5000/', 
                            data={'username': users[user_idx], 
                                  'password': passwords[user_idx]})
    
    if response.status_code == 200:
        # Random API calls
        for _ in range(random.randint(1, 5)):
            requests.get(f'http://localhost:5000/api/get-email?username={users[user_idx]}')
            time.sleep(random.uniform(0.5, 2.0))
        
        # Logout
        requests.post('http://localhost:5000/logout')
    
    time.sleep(random.uniform(1.0, 3.0))
```

## Conclusion

This SOAP authentication demo application demonstrates how to build a realistic, production-like application that's handy for F5 APM testing. The key design decisions - in-memory storage, comprehensive logging, separate logout page, and multiple API endpoints - make it both easy to deploy and rich in observable behavior.

The complete source code and deployment instructions are available in [this repository](https://github.com/mikeoleary/moleary-apm-demo-app), and you can have it running on Ubuntu in less than five minutes. Happy demo'ing!

## Further Reading

- [Flask Documentation](https://flask.palletsprojects.com/)
- [Spyne SOAP Library](http://spyne.io/)
- [Python Logging Best Practices](https://docs.python.org/3/howto/logging.html)
- [systemd Service Management](https://www.freedesktop.org/software/systemd/man/systemd.service.html)