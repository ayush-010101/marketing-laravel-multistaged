# Marketing Service — Laravel + Docker

A containerized marketing platform built with PHP Laravel, orchestrated with Docker Compose.

## Tech Stack
- PHP 8.3 + Laravel 11
- Nginx
- MySQL 8.0
- Redis 7
- Docker + Docker Compose

## Project Structure
```
marketing-laravel-app/
├── docker/
│   ├── nginx/
│   │   └── default.conf
│   └── php/
│       ├── Dockerfile
│       └── php.ini
├── src/
├── .dockerignore
├── .gitignore
├── .env.example
├── docker-compose.yml
├── docker-compose.override.yml
└── docker-compose.prod.yml
```

## Getting Started

### Prerequisites
- Docker Desktop
- Git

### Steps
1. Clone the repo
2. Run `cp .env.example .env` and fill in your values
3. Run `docker compose up --build`
4. Visit http://localhost:80

adding a line to check automated CI/CD