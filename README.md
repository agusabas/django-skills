# Django Skills

Agent Skills for Django development with Clean Architecture, SOLID principles, and modern best practices.

## Installation

```bash
# List available skills
npx skills add agustin/django-skills --list

# Install all skills
npx skills add agustin/django-skills --all

# Install specific skills
npx skills add agustin/django-skills --skill django-clean-drf
npx skills add agustin/django-skills --skill django-celery-expert
npx skills add agustin/django-skills --skill refactordjango
```

## Available Skills

### django-clean-drf

Comprehensive skill for building production-quality Django REST Framework APIs following Clean Architecture patterns.

**Features:**
- Use Cases with Input/Output DTOs
- Service layer (static queries, instance mutations)
- Validation returns `tuple[bool, str]` instead of exceptions
- Read/Write serializer separation
- Constructor injection for testability
- Complete testing patterns by layer
- Query optimization (N+1 prevention, select_related, prefetch_related)
- Pagination, filtering, search, and ordering
- JWT authentication and permissions
- Production deployment (Gunicorn, Docker, caching, logging, Sentry)
- Django Admin configuration for API backends
- Coding standards with Ruff

**When to use:** Creating new Django DRF applications, implementing CRUD operations, designing API endpoints with clean separation of concerns, optimizing queries, configuring authentication, or preparing for production deployment.

### django-celery-expert

Celery task processing with Django integration.

**Features:**
- Task design patterns
- Error handling and retries
- Periodic tasks (beat)
- Production monitoring
- Django integration patterns

**When to use:** Background task processing, async operations, scheduled jobs, or Celery configuration.

### refactordjango

Refactor existing Django code toward Clean Architecture.

**Features:**
- Transform fat views into Use Cases
- Extract services from views
- Convert exceptions to return values
- Separate Read/Write serializers
- Apply Python 3.12+ and Django 5+ patterns
- Fix N+1 queries and anti-patterns

**When to use:** Refactoring legacy Django code, modernizing projects, applying Clean Architecture to existing codebases.

## Architecture Overview

```
Views (HTTP) → Use Cases (Business Logic) → Services (Data Access) → Models (Domain)
```

### Key Principles

1. **Dependencies flow inward** - outer layers depend on inner layers
2. **No exceptions for business logic** - return `Output` DTOs or `tuple[bool, str]`
3. **Queries are static, mutations are instance methods**
4. **Views don't access models directly** - use services
5. **Serializers don't contain business logic** - use cases do
6. **DTOs use `@dataclass(frozen=True, slots=True)`**
7. **Use `@transaction.atomic` for state changes**
8. **Constructor injection for testability**

## Requirements

- Python 3.12+
- Django 5.0+
- Django REST Framework 3.14+

## License

MIT
