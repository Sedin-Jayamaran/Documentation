# `🔄 Environment Variable Flow in Rails + Docker`
🤖 This is how environment variables flow from your `.env` file to your Rails app:

```.env
DATABASE_USERNAME=root
DATABASE_HOST=db
```

✅ This file holds your environment variable values. It is loaded by Docker Compose automatically (if in the same folder) or via the dotenv-rails gem inside Rails.

📦 docker-compose.yml
```dockerfile
environment:
  DATABASE_USERNAME: ${DATABASE_USERNAME}
  DATABASE_HOST: ${DATABASE_HOST}
```
✅ Docker Compose substitutes ${DATABASE_USERNAME} with the value from .env and injects it into the container.

🛠️ config/database.yml
```database.yml
username: <%= ENV['DATABASE_USERNAME'] %>
host:     <%= ENV['DATABASE_HOST'] %>
```
✅ Inside Rails, the values are accessed using ENV[]. This uses Ruby's ERB interpolation to read environment variables at runtime .

